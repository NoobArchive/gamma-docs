# 共同模块: Android 手游

## 编译偏好：Thumb

通常会使用许多 Thumb 代码，需要小心

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