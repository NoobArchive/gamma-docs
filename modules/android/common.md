# 共同模块: Android 手游

## 编译偏好：ARM Thumb

通常会使用许多 Thumb 代码，需要小心

## 脚本层

### 后端选择：AS3 与 Lua 是两套编译产物

`Core::CAppClient::CAppClient` 在运行时按 `gac_config.xml` 选 `0`/`2`，
但二进制只编进其中一个后端：

```c
if (GetChild("Script") && strcmp(GetText("Script"), "as3") == 0)
    script = new Gamma::CScript(2);   // AS3
else
    script = new Gamma::CScript(0);   // Lua
```

- **Lua 版**：`CScript(0)` → `CScriptLua` + LuaJIT 2.0.4；
  `CScript(2)` → `throw "as3 is disable!!!!"`。
- **AS3 版**：`CScript(2)` → `CScriptAS3` + Tamarin AVM2；
  `CScript(0)` 返回空，二进制里没有 `CScriptLua`/`lua_newstate`/LuaJIT。

所以“AS3 硬禁用”只对 Lua 版成立；AS3 版是另一套编译产物。
判断一个包用哪种后端，要看 `gac_config.xml` 的 `<Script>` **加上** `.so` 里的
`CScriptAS3`/`CScriptLua` 符号，不能只看 `/misc`。

### AS3

**样本：**《全民猎妖》0.0.45.1、《雷霆射击》99.0.47.6。
版本细节见 [qmly-v0-0-45-1.md](qmly-v0-0-45-1.md)、
[leiting-v99-0-47-6.md](leiting-v99-0-47-6.md)。

| 项 | 全民猎妖 0.0.45.1 | 雷霆射击 99.0.47.6 |
|---|---|---|
| `gac_config.xml` | `<Script>as3</Script>`，`<SWF Path="bin/release/"/>` | 同 |
| `[DeltaXCode]` | `DeltaxCpp_ca04..._0.0.45.swf`，113,378 B | `DeltaxCpp_62bc..._99.0.46.26.4.swf`，113,473 B |
| `[ShellCode]` | `Main_506e..._0.0.45.1.swf`，1,478,217 B | `Main_6412..._99.0.47.6.swf`，1,506,214 B |
| AS3 类数 | `DeltaxCpp` 247 + `Main` 1,928 | `DeltaxCpp` 247 + `Main` 1,957 |
| AS3 VM | Tamarin AVM2（avmplus/MMgc/avmshell） | 同 |
| Lua VM | 无 | 无 |

`libShellClient.so` 里的 AS3 侧符号：

- `Gamma::CScriptAS3`（字符串出现 242 次，含类/方法/vtable 名）
- `tamarin::CreateTamarin`、`tamarin::Avm2CoreImpl`、`avmplus::*`、`MMgc::*`
- `avmshell::handleDoABC2` / `SwfParser`
- **没有** `Gamma::CScriptLua`、`lua_newstate`、`LuaJIT`
- **没有** `webguyu`、`CPlayer`、`CGameScene`、`CManor` 这些游戏类

#### `DeltaXCpp.swf` 是什么

它是 AS3 侧的 **DeltaX 运行时 + C++ 原生绑定层**，不是游戏本体；
`version.inf` 里对应 `[DeltaXCode]`。文档类是
`deltaxcpp.DeltaxCpp extends deltaxcpp.appframe.Application`。

内容大致三类：

- 149 个 `deltax.*`：纯 AS3 接口/框架（`IApplication`、`BaseApplication`、
  `DeltaXWindow`、`DeltaXRenderer`、`IGUIManager`、`IResourceManager`、……）
- 13 个 `deltaxcpp.*`：具体实现（`Application`、`DeltaXRenderer`、
  `ResourceManager`、`LoaderManager`、`Sound2D`、`ConnectionToGameServer`）
- 85 个根级 `C*`/`I*`/枚举：`CApplication`、`CBaseAppClient`、`CRenderer`、
  `CGWnd`、`CResourceManager`、`CLogicObject`、`CLogicScene`、…… 都是
  `native` 声明，绑定 `.so` 里同名 C++ 类

关键接线：

- `Application` 构造时 `CApplication.GetInstance()`，再 `SetHandler(this)`；
  原生 `Core::CApplication` 把 AS3 对象挂成 `IAppHandler`，每帧回调
  `OnStart`/`OnClose`/`OnCheckFrame`/`OnRenderBegin`/`OnRenderEnd`/
  `OnUserCmd`/`OnAllLoadingDone` 等。
- `External.ClassCast` → `CScriptAS3::ClassCast`：把原生对象重绑到 AS3 派生类
  （和 Lua 版的 `ClassCast` 同一思路）。
- `DeltaxCpp` 构造函数把 `TableFile.s_TableFileImpCreate` 指向原生
  `flash.utils.TabFile`，AS3 侧读表走 C++ 的 `CTabFile`。

#### 加载顺序

1. `CGameAppClient::CGameAppClient` 调 `CApplication::CApplication(this, "Main")`，
   shell 基础名是 `Main`。
2. `CApplication::OnNewCodeVersionRetrieved` 从 `version.inf` 读版本，拼
   `DeltaxCpp_<version>.swf`（`[DeltaXCode]`）和
   `Main_<version>.swf`（`[ShellCode]`）。
3. `CApplication::OnSwfLoaded` 先识别并运行 `DeltaxCpp`，再触发 `Main` 的加载；
   `CScript::RunBuffer` → `CScriptAS3` → Tamarin `avmshell` 的 `DoABC2`/`SwfParser`。

#### 关于 `misc/ShellCommon` 的 Lua

AS3 包里确实有 `misc/ShellCommon/*.r`，里面是 9 条 Lua 5.1 字节码
（源路径 `D:/13a/src/game/cpp/game/lua/ShellCommon/*.lua`），
但这两个 build 都没有 Lua VM（`CScriptLua`/`lua_newstate`/`LuaJIT` 均为 0），
所以客户端不会执行它们；属于 13a 代码库的打包残留，或供其他工具使用。
**不要把“有 `/misc`”当成“在用 Lua”。**

### Lua

《爆彩童话》《贪吃蛇乐园》《粉刷匠大作战》等。

**Lua 构建里 AS3 被硬禁用。** `Gamma::CScript` 只认 `EScriptVM` 的两个值：

| 值 | 后端 | 实际行为 |
|---:|---|---|
| 0 | Lua | `new Gamma::CScriptLua` → `lua_newstate` + `luaL_openlibs`（静态 LuaJIT 2.0.4） |
| 2 | AS3 | 直接 `throw "as3 is disable!!!!"`；没有 AVM/ABC 实现 |

- 具体选择在 `Core::CAppClient::CAppClient`，见上；《贪吃蛇乐园》的
  `Core::CApp::CApp` 里也能看到 `MOV R1, #0`。
- 这几套 Lua APK 里没有 `.swf` / `.abc`；`libShellClient.so` 里没有 AVM/ABC 解释器符号。
- 所以**这几套 Lua build** 实际只有 Lua 后端；AS3 只是引擎 ABI 里的遗留枚举。
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

所以即使 1.1.7.1+ 有 Lua，核心场景、玩家、庄园、碰撞、渲染仍是 C++；
Lua 负责 UI、网络、活动、配置胶水。

### 版本时间线

| 版本 | `misc/` 脚本 | `AsyncRequire` 指纹 | 结论 |
|---|---|---|---|
| 贪吃蛇乐园 200.0.0 | 无 | 无 | 本地纯 Native；`gac_config.xml` 虽写 `<Script>lua</Script>`，但没有脚本/加载器 |
| 粉刷匠 1.1.6.6 (2017-07-27) | 无 | 无 | 纯 Native；`client_config.xml` 没有 `SearchPathProgrammer` |
| 粉刷匠 1.1.6.8 | 无 | 无 | 同样无 `misc/`（用户校正；本机无包，待补） |
| 粉刷匠 1.1.6.10 (2017-08-04) | `misc/` 5 个 `.lua` + 2 个 `.luaproj` | 有 | 开始挂 Lua；但 `CGameClient:OnStart` 只 `print`，UI 仍全 Native |
| 粉刷匠 1.1.7.1 (2017-09) | 二测起大量使用（本机无包，待补） | 有 | 与 1.1.6.10 相比 UI/逻辑脚本化大幅扩张 |
| 粉刷匠 1.1.20 (2017-11-22) | 32 个 `.r`（103 个 `.lua` 源文件） | 有 | （`CGameStartLua`/`CGameMainLua`，2017-08-05） |
| 粉刷匠 1.3.3 (2018-05-18) | 176 个 `.lua` | 有 | Lua UI 继续扩展；C++ 核心仍在 |

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