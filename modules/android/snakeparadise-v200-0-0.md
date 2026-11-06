# 贪吃蛇乐园 v200.0.0
九艺网络/悦岩居 对 《贪吃蛇大作战》的粗劣模仿

后来代码库被重新利用作为《节奏粉刷匠》（粉刷匠大作战）的基础。

~~该版本也没有 `tcs_jm002.tex`;1.1.6.6 的部分 GUI 仍引用它,说明相关 UI在资源改名为 `fs_jm002` 之前就已废弃,属于历史残留。~~ tcs_jm002.tex 居然放在 /etc/gui/ 里了，草

## 脚本层：没有 Lua/AS3，客户端逻辑在 `libShellClient.so`

`assets/` 下没有 `misc/`，`filelist_200.0.0.r` 里也没有任何 `misc`/`lua` 条目。
`etc/gac_config.xml` 虽然写着 `<Script>lua</Script>` 和
`ShellClientLua/?.lua` / `ShellCommonLua/?.lua` 搜索路径，但没有任何脚本文件；
这是引擎的默认配置残留，不是本包实际加载的脚本。

`libShellClient.so`（v7a，5,325,668 B）在 IDA 里 15,812 个函数、10,759 个导出，
其中直接包含游戏侧 C++ 类：

- `CGameAppClient`、`CSnakeClient`、`CGameSceneClient`
- `CGameScene`、`CSnake`、`CBall`
- `CGameMain`、`CGameStart`、`CSettingWnd`、`CShareWnd`

UI 也是 C++ 直接建窗口：

- `CGameAppClient::OnAllLoaded` @ `0xf69c4`：
  `CreateFromRes("gui/cfg/tcs_denglu.gui")`、`tcs_zjm.gui`、`messagebox.gui`；
- `CGameStart::OnCreated` @ `0xfc7e0`：
  取 `B_wujin`/`B_xianshi`/`B_pifu`/`B_paihang`/`B_huodong`/`B_guize`/`B_fenxiang`
  等控件，并 `CreateFromRes` 载入 `tcs_guize.gui`、`tcs_pifu.gui`、`tcs_fenxiang.gui`、
  `tcs_huodong.gui`、`tcs_shezhi.gui`、`tcs_paihang.gui`、`tcs_zhanghaodenglu.gui`、
  `tcs_zhanghao.gui`、`tcs_quming.gui`。
- `.so` 里能直接扫到 19 个 `gui/cfg/*.gui` 路径；包内共 22 个 GUI，
  其余 3 个是更新/解压/下载页（`messagebox_gengxin.gui`、`jieya1.gui`、`tcs_xiazai.gui`）。

Lua/AS3 证据：

- `AsyncRequire`、`GameClientLoader`、`Load Script`、`OnGameClientLoaded` 在 `.so` 里
  都是 **0 次**；没有 Lua 加载入口。
- 没有 `.lua` 文件，没有 Lua 字节码魔数 `1b 4c 75 61`；`1b 4c 4a` 只在 music 分片里
  偶然出现，不是 LuaJIT bytecode。
- LuaJIT 2.0.4 确实静态链在 `.so` 里（`lua_newstate` @ `0x30fe14`,
  `luaL_openlibs` @ `0x32f570`），但只是引擎常驻能力。
- `Gamma::CScript::CScript(EScriptVM)` 只有两个分支：`0 → CScriptLua`，
  `2 → throw "as3 is disable!!!!"`；没有 AS3 后端。

`version.inf` 里 `[Data] URL = http://snakeres.joyegame.com/snake/data/`
（当前 502）说明它本来可以从数据包补资源/脚本；但就这个官网包里的二进制而言，
没有脚本加载器，也没有脚本文件，客户端逻辑是纯 Native。

**结论:** 本地包不是“Lua 被藏起来了”，而是这一版客户端根本没有启用 Lua 层；
`libShellClient.so` 同时是引擎和游戏模块，但不是把所有逻辑写成硬编码 if-else——
`.gui` 布局和 `config/*.txt` 仍是数据驱动，解释/执行这些数据的代码是 C++。

## 资源解包/转换(2026-11)

- 输入:`别的游戏/贪吃蛇乐园-200.0.0-官网包/assets`
- `filelist_200.0.0.r`:9/9 逻辑包校验通过;
- 解包:71 个文件 / ~6 MB;
- `.tex`:6/6,`etc-dual=4`、`etc2=1`、`raw=1`;
- `.gui`:22/22 渲染成功,428 state、232 可见、122 段文字;
- 产物:`_agent-sandbox-tcs261106/tcs_tex`、`tcs_ui`、`tcs_probe.md`;
- 登录页:`_agent-sandbox-tcs261106/tcs_ui/cfg/tcs_denglu.png`;
- 本包没有模型资源;`.map/.rgn` 地形仍待做。
