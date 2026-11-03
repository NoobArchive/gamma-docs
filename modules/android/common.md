# 共同模块: Android 手游

## 编译偏好：Thumb

通常会使用许多 Thumb 代码，需要小心

## 脚本层：Lua 可用、AS3 硬禁用

Gamma 的 `Gamma::CScript` 只认 `EScriptVM` 的两个值：

| 值 | 后端 | 实际行为 |
|---:|---|---|
| 0 | Lua | `new Gamma::CScriptLua` → `lua_newstate` + `luaL_openlibs`（静态 LuaJIT 2.0.4） |
| 2 | AS3 | 直接 `throw "as3 is disable!!!!"`；没有 AVM/ABC 实现 |

- `Core::CApp::CApp` 里固定传 `0`（《贪吃蛇乐园》`0x150fa4: MOV R1, #0`）。
- 这几套 APK 里没有 `.swf` / `.abc`；`libShellClient.so` 里没有 AVM/ABC 解释器符号。
- 所以 Android 这几套 build 实际只有 Lua 后端；AS3 只是引擎 ABI 里的遗留枚举。
- 压缩媒体里偶发的 `FWS`/`CWS`/`ZWS`（music/ani 分片）和 `1b 4c 4a` 不是
  SWF/LuaJIT，必须看文件头或容器结构，不能只靠字节扫描下结论。

**“有 Lua VM” ≠ “有 Lua 脚本”。** LuaJIT 是静态链进 `libShellClient.so` 的，
脚本要从 `misc/` 加载；没有脚本时 VM 只是常驻不工作。
`CScriptLua` 构造函数里确实内嵌了一小段 Lua（`class()`/继承辅助），那属于引擎运行时，
不是游戏逻辑脚本。

Lua 启动层的字符串指纹（有则说明这个 build 会挂 Lua）：

```
AsyncRequire( "GameClientLoader" )
Load Script :
OnGameClientLoaded
```

**Lua 与 C++ 的关系：** Lua 不是重写整个游戏，而是用 `ClassCast` 把原生对象的
虚表换成 Lua 类，再覆写 `OnCreated` / `OnTick` 等：

```lua
-- 1.1.20 CGameClient.lua
self.m_GameStart = ClassCast( CGameStartLua, CGameAppClient.GetGameStart( self ) );
self.m_GameMain  = ClassCast( CGameMainLua,  CGameAppClient.GetGameMain( self ) );
```

```lua
-- 1.1.20 CGameMainLua.lua
function CGameMainLua:OnCreated()
    CGameMain.OnCreated( self );  -- 先跑原生绑定/窗口创建
    ...
end
```

所以即使 1.1.20+ 有 Lua，核心场景、玩家、庄园、碰撞、渲染仍是 C++；
Lua 负责 UI、网络、活动、配置胶水。

### 版本时间线

| 版本 | `misc/` 脚本 | `AsyncRequire` 指纹 | 结论 |
|---|---|---|---|
| 贪吃蛇乐园 200.0.0 | 无 | 无 | 本地纯 Native；`gac_config.xml` 虽写 `<Script>lua</Script>`，但没有脚本/加载器 |
| 粉刷匠 1.1.6.6 (2017-07-27) | 无 | 无 | 纯 Native；`client_config.xml` 没有 `SearchPathProgrammer` |
| 粉刷匠 1.1.6.10 (2017-08-04) | `misc/` 5 个 `.lua` + 2 个 `.luaproj` | 有 | 开始挂 Lua；但 `CGameClient:OnStart` 只 `print`，UI 仍全 Native |
| 粉刷匠 1.1.20 (2017-11-22) | 32 个 `.r`（103 个 `.lua` 源文件） | 有 | Lua UI 真正上线（`CGameStartLua`/`CGameMainLua`，2017-08-05） |
| 粉刷匠 1.3.3 | 176 个 `.lua` | 有 | Lua UI 继续扩展；C++ 核心仍在 |

> 1.1.6.10 的占位脚本头部日期是 2015-12-07（来自更早的 15a_Snake 模板）；
> 真正的 Lua UI 类从 2017-08-05 开始写。这解释了 1.1.6.6 为什么完全没有脚本。

**1.1.6.10 的 filelist 新增项（可作为边界证据）：**

```
misc/ShellClientLua/0c827fd2720ce6b3e5d29ea01b1452dd.z   4133   1562
misc/ShellCommonLua/f8fc1d6a209cb7c74cd87c1e0d8c36b9.z   1323    623
```

## 启动引导: DeltaxAdapter / AndroidBoot

- 早期 Gamma 手游用 `libDeltaxAdapter.so`(入口 `DeltaxAdapterEntry`,缓存指针
  `g_ShellClientStartUpFunction`);
- 新版用 `libAndroidBoot.so`(`AndroidBoot` / `g_StartUpFunction`);
- 二者都是 trampoline:Java `copyShellLib()` 给出真身路径 → `dlopen` →
  `dlsym("ANativeActivity_onCreate")` → 转发;真身是 `libShellClient.so`。



**两者对照:**

| | libDeltaxAdapter.so(早期) | libAndroidBoot.so(后期) |
|---|---|---|
| 大小/函数数 | 106 KB / 254 | 108 KB / 525 |
| 入口函数 | `DeltaxAdapterEntry` | `AndroidBoot` |
| 缓存指针 | `g_ShellClientStartUpFunction` | `g_StartUpFunction` |
| 取路径 | Java `copyShellLib()` | Java `copyShellLib()` |
| dlopen 模式 | `RTLD_LAZY` (0) | `RTLD_NOW` (2) |
| dlsym 目标 | 刚打开的 `handle` → 之后 `dlclose` | `RTLD_DEFAULT`(-1,全局查找)→ 不关闭 |
| 通知 Java | ❌ 不调用 | ✅ 额外调用 `loadShellLib(path)` |
| 失败处理 | 两处分别抛 `"can not load native lib"` | 合并为一个判断 |
| 载荷 | `libShellClient.so`(已确认同目录存在) | `libShellClient.so` |

**结论:**
1. **同一套机制**:两者都是 ~100 KB 的空壳 trampoline —— 通过 Java `copyShellLib()` 拿到运行时释放的真身路径 → `dlopen` → `dlsym("ANativeActivity_onCreate")` → 缓存到全局指针 → 每次调用直接转发。真正的 ShellClient 壳库在运行时才被加载。
2. **命名演进就是最直接的证据**:早期库的缓存变量直接叫 **`g_ShellClientStartUpFunction`**,明确指向载荷是 `libShellClient.so`(《贪吃蛇乐园》目录里确实有);后来改名为更通用的 `g_StartUpFunction` + 入口改名 `AndroidBoot`。
3. **细微差别只是实现细节的迭代**:后期版改用 `RTLD_NOW` 提前暴露未解析符号、不 `dlclose`、并新增 `loadShellLib` 回调让 Java 侧同步感知加载完成(推测是为了配合 Java 侧 `System.loadLibrary` 去重或初始化)。
4. **逆向时的启发**:分析这两批游戏,这两个 `.so` 都可以直接跳过,重点放在 `libShellClient.so`;它们唯一有意义的"指纹"就是 `copyShellLib`/`loadShellLib` 这对 Java 方法名——可以从 Java 层(如 `ShellClient` 类)入手追 `copyShellLib` 到底从 APK 哪个位置、以什么方式释放真身。

**入口 `DeltaxAdapterEntry` @ `0x4c30` 反编译结果:**

```c
INT DeltaxAdapterEntry(ANativeActivity *act, ...) {
  if (g_ShellClientStartUpFunction)                  // 已加载 → 直接转发
    return g_ShellClientStartUpFunction(act, ...);

  env = act->env; clazz = act->clazz;                // 手动解 ANativeActivity 结构
  methodID = env->GetMethodID(clazz, "copyShellLib", "()Ljava/lang/String;");
  path = env->CallObjectMethod(clazz, methodID);     // 问 Java 要真实 .so 路径
  s = env->GetStringUTFChars(path, 0);
  handle = dlopen(s, RTLD_LAZY);                     // 动态加载真身
  env->ReleaseStringUTFChars(path, s);
  if (!handle) throw "can not load native lib";
  g_ShellClientStartUpFunction = dlsym(handle, "ANativeActivity_onCreate");
  if (!fn) throw "can not load native lib";
  dlclose(handle);                                   // 用完即关
  return g_ShellClientStartUpFunction(act, ...);     // 转发
}
```