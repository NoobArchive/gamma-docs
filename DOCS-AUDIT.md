# 文档校对报告

> 日期:2026-10-31
> 范围:`Gamma引擎相关/README.md`、`FORMAT-NOTES.md`、`docs/**/*.md`

## 1. 已修正

| 项 | 原状 | 现状 |
|---|---|---|
| `FORMAT-NOTES.md` | 根目录旧笔记,数字/命令与 `docs/` 脱节 | 改成**迁移指针**;内容拆入 `docs/formats/` |
| 公共头/文本编码 | 只在旧笔记 §0/§3 | 新增 `docs/formats/common.md` |
| `.tex` | 没有正式格式文档 | 新增 `docs/formats/tex.md` |
| `.r`/filelist | 只在旧笔记 §2,且写 `635` | 新增 `docs/formats/package.md`,改为 **636/636** |
| `.tex` 数量 | 写“792 个全部解出 PNG” | 实际 **788 PNG + 4 JXR**(`loading_1..4`) |
| 多贴图 piece | “多材质分层未解” | 已确认是**配色变体**,`ams.md`/`STATUS.md` 已改 |
| `.eft` unit | “只切段/375/376” | 已升级为 **754/754 prop、5036 unit** |
| `.mtr` | “26B 未标定” | 已逐字段标定(0x2716=24B / 0x2717=26B);高光/自发光开关与 3 个模板遗留字节已命名 |
| `.gui` | 数字/字段旧 | 更新 `f[0]` style、`f` 旋转、翻转、wrap、状态代表 |
| `.ans` 共享骨架 | `box02/03`、`p_*_a` 丢蒙皮 | 增加 `bind.ans` / `box01.ans` 回退,6 个模型恢复 skin |
| `.anf` 帧率 | 30fps 假设 | 已由实机动作速度确认 |
| 配置编码 | `read_text_any` 先试 UTF-16LE, UTF-8 BOM 的 `fusion.txt` 会乱码 | 先看 BOM/UTF-8,再按 NUL 比例判断 UTF-16 |
| 文档入口 | 无 | 新增 `README.md`、`docs/GETTING-STARTED.md`、`docs/ARCHITECTURE.md` |

## 2. 新增文档

- `README.md` — 项目入口与最小工作流;
- `docs/GETTING-STARTED.md` — 解包 → 转换 → 打包;
- `docs/ARCHITECTURE.md` — 工具布局、`gammaray-py` 设计、迁移计划;
- `docs/formats/common.md` — `CCommonFileHead`/依赖/文本编码;
- `docs/formats/tex.md` — `.tex`/ATF/ETC/JXR;
- `docs/formats/package.md` — `.r`/filelist/分片与打包边界;
- `docs/formats/skins.md` — equip/fusion/model-only 皮肤与三套 ID;
- `docs/formats/legacy-notes.md` — 旧加载链、`.ani` 光标、包/哈希。

## 3. 仍然缺的内容(按优先级)

| 缺口 | 现状 | 建议 |
|---|---|---|
| `.lua` / `.txt` / `.xml` | 未研究 | 动态 UI、数值表、玩法参数;做表驱动填充 |
| `.ani` 光标 | 只有旧笔记入口地址 | 单独写 `formats/ani.md`,校对 RIFF 块 |
| `.264` | 未处理 | 直接封装 mp4,校验音视频轨 |
| `loading_1..4` JXR | 只抽 `.jxr` | jxrlib/texconv 转 PNG |
| `.mtr` +126/+127/+157/+158/+160 | +126/+127 是 SMaterial 内的高光/自发光开关;后 3 个是模板遗留字段 | 不影响 alphaMode/doubleSided |
| `.gui` wrap | 已解: `u2/u3` = texel 周期, `u1 bit4/5` 反向; login 100px 周期已对齐 | 保留 `--no-wrap-tile` 仅作对照 |
| `.gui` 文字色 | 已解: `SChildImageList.b`(ARGB); `c` 已用 1px 八向描边近似 | 有真机截图时校准宽度/混合 |
| `.gui` 运行时状态 | 选“内容最多”代表 | 用 Lua `ShowWnd`/段位/wifi 值精确选 |
| `.gui` 动态内容 | 未填 | 皮肤格子/排行榜/玩家名 |
| `.eft` 逐帧模拟 | 只有结构卡+粗略粒子云 | 把 `props` 映射成速度/生命周期/颜色曲线 |
| `p_046` | 未注册 | 仅资料整理,不影响工具 |

## 4. 链接/一致性检查

- `docs/formats/OVERVIEW.md` 顶部链接已指向 `../STATUS.md` / `../GETTING-STARTED.md`;
- `ams.md` 的“主笔记 §0”已改为 `common.md`;
- 所有 `docs/**/*.md` 相对链接检查通过(仅正文里的括号文本被误判)。
