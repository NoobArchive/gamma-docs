# 粉刷匠大作战 v1.1.6.6

2017 年 7 月底一测阶段发布，最早公开版本。

明显基于《贪吃蛇乐园》代码库，且具有前作残留。

## 包体验证(2026-11)

- 资源目录:`旧版本/粉刷匠-1.1.6.6-一测/assets`
- filelist:`filelist_1.1.6.6.r`,**117/117** 逻辑包校验通过
- 解包:551 个文件
- 输出:`_agent-sandbox/old_1_1_6_6*`

| 格式 | 结果 |
|---|---|
| `.tex → PNG` | 140/140 |
| `.gui` 结构 | 49/49 精确闭合 |
| `.gui → PNG` | 49/49 |
| role GLB | 18/18 |
| fx GLB | 14/14 |
| `.ans` | 29/29,143 clip |
| `.anf` | 147/147 |
| `.eft` | 74/74 |
| `.mtr` | 12/12 |
| `.eqp` | 1/1 |

## 版本差异

- `.gui` 同时含 `0x271b`(1 个)、`0x271c`(2 个)、`0x271d`(46 个);
  旧版没有 `s17/s26` 两个字符串,`more[]` 数量也不同。
- `equip.txt` 使用 `ID` / `szName`(不是 `nID` / `dName`);
  `skins` 导出 13 个已登记皮肤 + 4 个 model-only。
- UI 必须用本版本的 `gui/tex` atlas;不能拿 1.1.20 的图集补。
- `tcs_jm002.tex` 在包内缺失(2 个 GUI 引用);前作《贪吃蛇乐园》也没有
  这个文件,推测该 UI 在改名为 `fs_jm002` 前就废弃了,不是解析错误。

## 脚本层：没有 Lua/AS3，客户端逻辑在 `libShellClient.so`

- `assets/` 下没有 `misc/`、`lua/`；`filelist_1.1.6.6.r` 的 117 条里没有
  `misc`/`lua` 条目。
- `etc/client_config.xml` 没有 `SearchPathProgrammer`，也没有 `<Script>`；
  `version.inf` 的 `[Data] URL = pkgroot:/`，数据全部从包内读，不存在
  “远程拉 Lua”的路径。
- `libShellClient.so`（v7a，4,512,396 B）在 IDA 里 24,981 个函数、12,426 个导出，
  游戏侧类直接在 `.so` 里：`CGameAppClient`、`CGameScene`、`CPlayer`、
  `CManor`、`CItem`、`CGameMain`、`CGameStart`、`CSettingWnd`、
  `CShareWnd`、`CWndSettlemet` 等。
- `CGameAppClient::OnStarted` @ `0x2c4f80` 只有 878 B：加载 `user.ini`、
  服务器列表、`StandarUISize`、创建 GUI/更新器；没有
  `SearchPathProgrammer` 循环，也没有 Lua 调用。
- `CGameAppClient::OnAllLoaded` @ `0x2c572c` 直接 `CreateFromRes`：
  `fsj_denglu.gui`、`fsj_zjm.gui`、`fsj_dengdai.gui`、`fsj_tongyongitem.gui`、
  `messagebox.gui`、`debug_input.gui`、`fsj_jiazaizhongduan.gui`。
- `CGameStart::OnCreated` @ `0x2d1c90` 取 `B_kaishiyouxi.anniu`、`B_Enter`、
  `B_shezhi`、`B_pifuchengjiu.*`、`B_fenxiang` 等控件，并加载
  `fsj_pifu.gui`、`fsj_fenxiang.gui`、`fsj_shezhi.gui`、`fsj_yinyuexuanze.gui`、
  `fsj_shujutongbu.gui`、`fsj_xinshouzhiyin.gui`、`fsj_zhanji.gui`、
  `fsj_xianshijiesuo.gui`。
- `CGameMain::OnCreated` @ `0x2cb850` 绑定 40+ HUD 控件（`B_anniu`、`jindutiao`、
  `B_jiasu`、`B_fanhui`、`gerenchengji.*`、`W_xinhao.*`、`renwu.*` 等）并加载
  `tcs_tuichutishi.gui`、`fsj_zhandixinxi.gui`、`fsj_jishaxinxi.gui`、
  `fsj_jiezoutishi.gui`、`fsj_zjm_renwuitem.gui`。
- `.so` 里直接出现 38 个 `gui/cfg/*.gui` 路径；包内 49 个 GUI，其余 11 个没有以
  完整路径出现（可能是动态拼接的列表项/更新解压页）。
- Lua/AS3 指纹：`AsyncRequire`、`GameClientLoader`、`Load Script`、
  `OnGameClientLoaded` 全部为 0；没有 `.lua` 文件、没有 `1b 4c 75 61` 字节码；
  LuaJIT 2.0.4 静态链在 `.so` 里（`lua_newstate` @ `0x2e35c8`、
  `luaL_openlibs` @ `0x2f6058`）但没有任何脚本。
  `Gamma::CScript::CScript(EScriptVM)` 的 `2` 分支直接抛
  `"as3 is disable!!!!"`。

**边界:** 下一个包 `1.1.6.10`（2017-08-04）才出现 `misc/ShellClientLua` 和
`AsyncRequire( "GameClientLoader" )`；再下一个 `1.1.20` 才有真正的 Lua UI
（`CGameStartLua` / `CGameMainLua`，2017-08-05）。所以 1.1.6.6 不是“Lua 藏起来了”，
而是 Lua 层尚未接入。

**结论:** 这个版本的客户端逻辑（战斗、场景、玩家、庄园、UI 事件）全是 C++；
`.gui`、`config/*.txt` 是数据，不是脚本。

## `.eqp`

本版仍有 `role/dress/male.eqp`(2,982B,18 条),但 `libShellClient.so`
只引用 `equip.txt` / `role/mod/` / `role/ani/`,没有 `.eqp` / `role/dress/`
字符串。它和 1.1.20/1.3.3 一样是遗留数据。详见
[`../../formats/eqp.md`](../../formats/eqp.md)。
