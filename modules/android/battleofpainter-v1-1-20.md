# 粉刷匠大作战 v1.1.20

2017 年 11 月官方包版本。

## 包体验证(2026-11)

- 资源目录:`旧版本/[171122]粉刷匠-官方包-1.1.20/assets`
- filelist:`filelist_1.1.20.r`,**391/391** 逻辑包校验通过
- 解包:1661 个文件
- 输出:`_agent-sandbox/old_1_1_20*`

| 格式 | 结果 |
|---|---|
| `.tex → PNG` | 489/489 |
| `.gui` 结构 | 159/159 精确闭合 |
| `.gui → PNG` | 159/159 |
| role GLB | 48/48 |
| fx GLB | 24/24 |
| `.ans` | 61/61,354 clip |
| `.anf` | 359/359 |
| `.eft` | 232/232 |
| `.mtr` | 12/12 |
| `.eqp` | 1/1 |

## 版本差异

- `.gui` 全部为 `0x271d`,与 1.3.3 同读取器。
- `equip.txt` 使用 `ID` / `szName` / `nSign`(不是 `nID` / `dName`);
  `skins` 导出 31 个已登记皮肤 + 8 个 model-only。
- UI 必须用本版本的 `gui/tex` atlas。
- `tcs_jm001.tex` 缺失(1 个 GUI 引用),属于包内资源缺失。

## 脚本层

- 这是第一个真正带完整 Lua UI 的公开包：`misc/` 下 32 个 `.r`，
  对应 `assets/lua/` 里 103 个 `.lua` 源文件。
- `CGameClient.lua` 用 `ClassCast` 把原生对象换成 Lua 类：
  `CGameStartLua` / `CGameMainLua` / `CGameSceneClientLua` / `CPlayerClientLua`；
  Lua 类先调用原生 `CGameStart.Ctor` / `CGameMain.OnCreated`，再追加 UI 逻辑。
- 因此核心战斗/场景/玩家仍是 `libShellClient.so` 里的 C++；Lua 是 UI/网络/活动胶水。
- 时间戳：`CGameStartLua.lua` / `CGameMainLua.lua` / `CGameSceneClientLua.lua`
  头部都是 `--Date 2017-08-05`；`CGameClient.lua` 保留 `2015-12-07` 的老框架。

> 反过来的边界：上一个 1.1.6.6 (2017-07-27) 完全没有 `misc/`，连 Lua 加载入口都没有；
> 详见 [battleofpainter-v1-1-6-6.md](battleofpainter-v1-1-6-6.md) 与
> [common.md](common.md) 的脚本层时间线。
