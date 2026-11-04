# 雷霆射击 v99.0.47.6

Gamma 引擎 AS3 分支的第二个样本，和《全民猎妖》共用
`DeltaxCpp.swf` + `Main.swf` 结构。包内媒体代号是 `ftxnew`，
资源域名 `dl.kos.ftxgame.com`。

> AS3 源导出在 `_agent-sandbox-snake261103/as3_export/lt_*`。


## 包体

- 资源目录：`别的游戏/雷霆射击--百度`
- filelist：`filelist_99.0.47.8.r`，`fl-verify` **2395/2395**
- 原生库：`lib/armeabi/libShellClient.so`，5,496,188 B，MD5
  `56ce42e0694971e080d879c694c9d151`
- SWF：`assets/bin/release/`
  - `DeltaxCpp_62bc8c50b9814295824fbae7ec5b6fc6_99.0.46.26.4.swf`，113,473 B
  - `Main_6412eb105e1112db8527dc8cdddb4cc0_99.0.47.6.swf`，1,506,214 B

## 脚本后端

`etc/gac_config.xml` 同样是：

```xml
<Script>as3</Script>
<SWF Path = "bin/release/" />
```

`version.inf`：

```ini
[DeltaXCode]
Version = 99.0.46.26.4
Size = 113473
Md5 = 62bc8c50b9814295824fbae7ec5b6fc6

[ShellCode]
Version = 99.0.47.6
Size = 1506214
Md5 = 6412eb105e1112db8527dc8cdddb4cc0
```

`.so` 里 `CScriptAS3` 字符串出现 242 次，`tamarin`/`avmplus`/`MMgc` 齐全；
没有 `CScriptLua`、`lua_newstate`、`LuaJIT`。和全民猎妖一样，是
Tamarin AVM2 的 AS3 构建。

## 和全民猎妖的关系

用 FFDec 导出两边脚本后对比：

| | 全民猎妖 | 雷霆射击 |
|---|---:|---:|
| `DeltaxCpp` 类数 | 247 | 247 |
| 同名 `DeltaxCpp` 类 | — | 247/247 |
| `DeltaxCpp` 内容不同的文件 | — | 6 |
| `Main` 类数 | 1,928 | 1,957 |
| 两边同名的 `Main` 类 | — | 1,921 |
| `webguyu.*` 类数 | 1,757 | 1,786 |

`DeltaxCpp` 只差 6 个文件，主要是：

- `CApplication` / `Application` / `BaseApplication` / `IApplication`：
  新增 `CheckShareData`
- `CameraController`：新增 `setCameraOfs`
- `TickManager` / `Application`：帧时间上限、`CURRENT_TICK_COUNT` 改成实例字段

`Main` 多出的类集中在射击玩法：`ShootSkillInfo`、`ShootSkillConfig`、
`PickWeapon`、`ChipScript`、`ShareConfig` 等；少掉的类是猎妖的
`pkshow`/`QuenchingConfig`/`SwitchLineFrame` 等。

这组对比说明：

- `DeltaxCpp.swf` 是**跨项目复用的 AS3 运行时/绑定层**，每个项目只做少量补丁；
- `Main.swf` 是**各游戏自己的 AS3 游戏层**，共享 `webguyu.*` 框架但玩法类不同。

## `misc/ShellCommon` 的 Lua

`assets/misc/ShellCommon/c0203e19a8474c4cbc914d6c1534ff18.r` 里同样是 9 条
Lua 5.1 字节码，源路径是 `D:/13a_kos/src/game/cpp/game/lua/ShellCommon/*.lua`。
本 build 没有 Lua VM，客户端不执行；是 13a/13a_kos 代码库的打包残留。

## 结论

雷霆射击与全民猎妖是同一套 AS3 Gamma 技术栈的两个游戏：
`DeltaxCpp.swf`（`[DeltaXCode]`）提供 DeltaX AS3 运行时和 C++ 绑定，
`Main.swf`（`[ShellCode]`）提供 `webguyu.*` 游戏逻辑/UI，
`libShellClient.so` 提供引擎、Tamarin AVM2 和原生类。
