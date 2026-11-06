# 解析器现状 · 缺陷 · 后续思路

> 更新: 2026-11-02
> 新用户/工作流见 [GETTING-STARTED.md](GETTING-STARTED.md);
> 工具布局/迁移见 [ARCHITECTURE.md](ARCHITECTURE.md)。

## 0. 数据基准与合规

- **唯一权威来源**: `V3-1.3.3` 的同版本全量解包
  (`_agent-sandbox/gx_dump`, 由 `filelist_1.3.3.4.7.r` 对齐), 所有结论都基于它。
  旧版本 `1.1.20` / `1.1.6.6` 只作为**兼容性样本**, 不反向改写 1.3.3 的结论。
- `Rhythm Painters.apk` 只是**原版的单机魔改版**(配置和资源都可能被替换),
  因此不作为格式依据, 也不作为名字/数值的依据。
- 聊天/名称过滤表(`nChatFilter` / `nNameFilter` 一类)属于敏感内容:
  **不读取、不导出、不引用**。本轮调研时误打开过一次, 相关中间产物已删除
  (`temp-another/rp*`, `rp_046.txt` 等), 后续工具也不会触碰。

## 1. 能力矩阵

### 1.1 `.tex` 贴图

| | 内容 |
|---|---|
| 已实现 | 头部/deps/索引、raw(RGBA)、DXT1/5、ETC2、双 ETC1(色彩+alpha)、mip 跳过、ATF 内嵌 JXR 抽取 |
| 覆盖 | 792 个全部可解:788 个 PNG + `loading_1..4` 的 4 个 JPEG-XR(仅 `.jxr`); `tex-batch` 批量 |
| 缺陷 | `fmt=1` 的 JXR 负载只能抽出来(需要 jxrlib 才能转 PNG); 少数 `Compressed` 变体只写 `.bin` |
| 思路 | JXR 用 `jxrlib`/`texconv` 转码; 或者直接取更低的 mip |

### 1.2 `.r` 容器 / filelist

| | 内容 |
|---|---|
| 已实现 | `parse_package` / `build_package`、按 filelist 分组、多分片拼接 |
| 覆盖 | `fl-verify` 636 条记录 0 mismatch |
| 缺陷 | 跨分片的记录需要按声明长度拼接, 个别文件仍会报 problems |
| 思路 | 用 filelist 的 path->包映射做二次校验 |

### 1.3 `.ams` 模型

| | 内容 |
|---|---|
| 已实现 | piece 解析(`u16 ? / u16 vcount / packed / skin / ic / indices`)、顶点位宽 8/9/10/11/12、`TTinyNormal`、蒙皮块、piece 包围盒 `min` 角、材质变体(配色槽) |
| 覆盖 | `role/mod` **78/78**; `fx/mod` **59/59**(Khronos 校验 0 error / 0 warning / 0 info) |
| 缺陷 | ① piece 贴图只取配色槽第 0 套(`--skin N` 可换); 实测 `mats` 是配色变体, 不存在多层材质 |
| 思路 | ① 用 `CPiece::SPieceMatrl` 的 tag 还原真正的材质槽语义; ② 对缺 extra 的文件做 `CStaticPiece::ReadMainData` 的逐字段对齐 |

### 1.4 `.ans` 骨架 / 绑定

| | 内容 |
|---|---|
| 已实现 | 骨骼名+父子、每骨 52B 绑定记录(size / 局部平移 B / 世界旋转 C / 世界逆平移 D)、`invBind=[C|D]`、动画表; **共享骨架回退**(`p_*_a`→`bind.ans`, `box02/03`→`box01.ans`) |
| 覆盖 | 93 个文件全部可解; 3 个模型 97 根骨交叉验证误差 < 0.0001; `box02/03`、`p_004_a/p_015_a/p_023_a/p_024_a` 已恢复蒙皮 |
| 缺陷 | `SFigureUnit`(附加 figure 数组)非 0 的情况没验证(实测都为 0) |
| 思路 | 找到带 figure 的模型(如换装/挂件)再补 |

### 1.5 `.anf` 动画

| | 内容 |
|---|---|
| 已实现 | `transScale/scaleScale`、每骨 (nS,nR,nT) 表、时间戳 u8/u16、旋转(axis u32 + angle u16)、平移、缩放; 缺通道时按引擎规则回退; **时间轴 30fps** |
| 覆盖 | **542/542**; 骨表短于骨架(如 `p_001_attack_1` 28 项)与 >255 帧(u16 时间戳)都能自动反推 |
| 缺陷 | `.ans` 动画表的 `kc` 事件本作全为 0, 结构已定位但无样本 |
| 思路 | 等出现非 0 kc 的版本再补事件语义 |

### 1.6 GLB 导出

| | 内容 |
|---|---|
| 已实现 | 蒙皮(skins/JOINTS_0/WEIGHTS_0)、绑定 TRS、`invBind` 直取文件值、单 clip `--anim`、**全 clip `--clips`**、贴图嵌入、配色 `--skin`、**`.mtr` 的 alphaMode/doubleSided** |
| 覆盖 | 78 个 role 模型 × **540 个动画**(共享骨架回退后;未登记 `attack_1` 只在 manifest 中标记); 早期 Khronos 校验 0 issue |
| 缺陷 | ① UV 滚动/自发光等 shader 参数未映射到 glTF; ② `.ams` 的 `mats` 实测是**配色变体**(tag=0、每个 sub 只有 1 个贴图索引),不是多层材质 |
| 思路 | 拆 `CModelMaterial::PrepareMaterial` 的额外参数; 若未来出现多 sub 材质再按 primitive 拆分 |

### 1.7 `.eqp` / 皮肤表

| | 内容 |
|---|---|
| 已实现 | `male.eqp` 的 29 个部件 + 29 个备注名; **`equip.txt` 权威皮肤表**; **`fusion.txt` 18 个融合皮肤**; **未实装 `p_*` 模型** |
| 覆盖 | `skins` 导出 66 条:equip 34 + fusion 18 + model 14 |
| 缺陷 | 未实装 model 条目没有正式名字/图标(只有模型 stem);`nShow=0` 的商店隐藏规则仍以配置为准 |
| 思路 | 把 `nIconID` 与 `iconconfig.txt` 联动;用 `nShow/nSort` 驱动商店模拟 |

### 1.8 `.gui` UI

| | 内容 |
|---|---|
| 已实现 | 完整窗口树、`src/dst`、UV 环绕、**负宽/负高翻转**、**`k` ARGB 染色**、**`f[0]` style 对齐**、**`i[]` 字号/行间距**、**CGRichWnd 自动换行**、**同矩形状态备选选择**、本地化字典取词、**`u2/u3` 周期平铺**、**`u1 bit3` 纯色块**、**`SChildImageList.b` 文本颜色**、**`c` 1px 描边近似** |
| 覆盖 | 结构 **213/213** 精确闭合; 10658 个 state(可见绘制 8434 + 纯色 35, 1445 环绕, 2979 翻转), 1054 段文字; 213 张预览图 |
| 缺陷 | ① `u[0]`/`u[1]` 的 `0x40`/`u5` 高 16 位未定; ② **`state.f` 不能直接当屏幕旋转角**; ③ 文本描边 `c` 目前是 1px 八向近似; ④ 状态代表依赖运行时; ⑤ 列表项/玩家数据是 Lua 动态创建 |
| 思路 | ① 用 Lua 的 `ShowWnd`/状态值精确选子树; ② 读 `config/resource/*.txt` 补动态内容; ③ 继续确认 `f` 的真实用途 |

### 1.9 `.eft` 特效

| | 内容 |
|---|---|
| 已实现 | prop 级、**13 个 type 的 unit 逐字段解析**、**编辑器属性名映射 `props`**、资源字符串、贴图联络表、**结构预览卡**(含粗略粒子云) |
| 覆盖 | **754/754 prop 精确闭合、5036 个 unit、0 error**; 376 个文件 |
| 缺陷 | 类型 6/10/11 本作数据中未出现, 只做了字段翻译未实测 |
| 思路 | 若后续版本出现这些类型, 用同样的「恰好读完」校验补测 |

### 1.10 `.mtr` 材质

| | 内容 |
|---|---|
| 已实现 | 头部/deps(`gfx/*.inl`)、**extra 26 字节逐字段**(alphaBlend/src/dest/alpharef/zTest/zWrite/cull/diffuse/factor)、`mtr-info`、**AMS→GLB 自动应用 alphaMode/doubleSided** |
| 覆盖 | 12/12 个 `.mtr` 精确读完(0x2716 24B / 0x2717 26B) |
| 缺陷 | +126/+127 已识别为 specular/emissive 开关;+157/+158/+160 是 SMaterial 之外的模板遗留字段,未进入 GLB |
| 思路 | 若以后需要还原材质预览,再把 specular/emissive 开关映射到 glTF 扩展 |

### 1.11 旧版本包体(1.1.20 / 1.1.6.6)

> 版本细节见 [v1.1.20](modules/android/battleofpainter-v1-1-20.md)、
> [v1.1.6.6](modules/android/battleofpainter-v1-1-6-6.md)。

| | 1.1.20 | 1.1.6.6 |
|---|---:|---:|
| filelist 校验 | 391/391 | 117/117 |
| 解包文件 | 1661 | 551 |
| `.tex → PNG` | 489/489 | 140/140 |
| `.gui → PNG` | 159/159 | 49/49 |
| role GLB | 48/48 | 18/18 |
| fx GLB | 24/24 | 14/14 |
| `.ans/.anf/.eft/.mtr/.eqp` | 全通过 | 全通过 |

旧版兼容点:

- `.gui` 0x271B/0x271C 缺少 `s17/s26` 两个字符串,`more[]` 数量也不同;
  解析器已按版本分支,1.1.6.6 的 49 个 GUI 全部精确闭合。
- 旧版 `equip.txt` 用 `ID`/`szName`(不是 `nID`/`dName`);
  `skins` 已做字段别名,1.1.20 导出 39 条、1.1.6.6 导出 17 条。
- 旧版预览必须用**本版本自己的** `gui/tex` 转出的 atlas。1.1.6.6 的
  `tcs_jm002.tex` 在包内缺失(2 个 GUI 引用),1.1.20 的 `tcs_jm001.tex`
  缺失(1 个 GUI 引用);这是包内数据缺失,不是跨版本 atlas 混用可以修正的。
- **脚本层边界**:1.1.6.6 与《贪吃蛇乐园》200.0.0 没有任何 Lua 脚本,连
  `AsyncRequire("GameClientLoader")` 加载指纹都没有;客户端逻辑(包括 UI)
  全在 `libShellClient.so`。1.1.6.10 起才挂 Lua bootstrap,1.1.20 起 UI
  才真正脚本化。详见 [android/common.md](modules/android/common.md)。

## 2. 已知未解问题(按优先级)

### 2.1 `.eft` unit 布局(已解决)

之前只有 11/376, 根因有四个:

1. **子类先读自己的前缀, 最后才调基类** `CEffectUnitProp::ReadIndexData`;
   之前直接从基类读, 必然错位。
2. `CParticleSystemProp` 有 **3 个 TVector3(36B)**, 早期实现只跳过 12B。
3. `CRenderer::CreateEffectUnitProp` 的 Hex-Rays 伪代码把 switch 简化成了
   `if (type!=0) abort()`; 改看 Thumb 反汇编的 **TBB 跳转表**后,
   恢复了 13 个 type 的真实类映射(见 `eft.md` §3)。
4. prop 名扫描上限 40 字符, 漏掉了 `fx/loop14943896441494389754…` 这类长名;
   放宽到 200 后 prop 总数从 749 修正为 **754**。

最终校验: 376 个 `.eft`、**754/754 prop 精确闭合、5036 个 unit、0 error**。
字段顺序见 `docs/formats/eft.md` §3。

### 2.2 `.gui` 剩余字段

已标定:`state.k`、`src` 负宽/负高 = 翻转、`state.u[2..3]` = **平铺周期(texel)**,
`u1 bit4/5` = 从右/下反向平铺,
`u1 bit3` = 纯色块、`u5` 图集组 id、`SGroup`、
`i[0]` 字号 / `i[1]` 字间距 / `i[2]` 行间距 / `i[3]` LockFlag;
文本颜色来自 `SChildImageList.b`(ARGB),`c` 是底色/描边。
同矩形 `windows` 是运行时状态备选,预览默认取内容最多的一个,并保留透明点击层。

**文本对齐来自 `f[0]` style 位域**(不是 `i[3]`):`0x01000000` 水平居中、
`0x02000000` 右对齐、`0x04000000` 垂直居中、`0x08000000` 底部对齐。

仍缺:
- **`state.f` 的真实语义**(不能直接当屏幕旋转角;当前预览默认忽略);
- `u[0]`、`u[1]` 的 `0x40`、`u5` 高 16 位、`i[1]` 的精确语义;
- 文字描边 `c` 的精确 shader 参数(当前预览用 1px 八向近似);
- 状态代表仍依赖运行时值(段位/wifi 格数),需 Lua 才能精确;
- 动态列表项(皮肤格子、排行榜条目)由 Lua 创建。

### 2.3 动态内容(文本/列表项)

`equip.txt` 的皮肤表 + `iconconfig.txt` 的图标 + 字典, 已经够拼出"皮肤格子";
排行榜/好友列表这类需要吃 `config/*.txt` 的表格。思路: 做一个"表驱动 UI 填充器"
(给定 .gui + 表名, 按 sub 模板重复生成格子)。

### 2.4 46 号皮肤(p_046)

模型/贴图/骨骼/动作齐全(春节主题: 元宝帽、红包、灯笼、梅花), 但**全客户端未注册**:
无 `equip.txt` 行(`nID 26` 空号)、无 `0b040046` 字典键、无图标(2001–2035)、
不在 `male.eqp`/`propvalue`/`fragment`/`fusion` 里。名字只存在于宣传物料。

### 2.5 其他小项

| 项 | 状态 | 思路 |
|---|---|---|
| `etc/loading_1..4.tex` 的 JXR | 只抽出 `.jxr` | jxrlib 转码 |
| `.mtr` 的 +126/+127/+157/+158/+160 | 位于 SMaterial 拷贝范围外, 疑为编辑器保留 | 不影响混合/双面 |
| `.anf` 事件表(kc) | 已定位: 在 `.ans` 动画表, 568 clip 全部 kc=0 | `anf-table` 可导出; 非 0 时按 8B/事件保留 |
| 多材质分层 | 已确认不存在: `mats` 是配色变体(全部 tag=0、每 sub 单贴图) | 无需处理 |

## 3. 验证手段(结论可信度来源)

1. **恰好读完**:`.ams`/`.ans`/`.gui`/`.anf`/`.eft`/`.mtr` 都要求解析长度与段长完全相等 —— 这是最硬的判据;
   `.gui` 213/213、`.anf` 542/542、`.eft` **754/754 prop**、`.mtr` 12/12 都是这样得到的。
2. **数值交叉验证**:`.ans` 的 `C/B/D` 与 `bind.anf` 第 0 帧独立算出同一世界姿态, 97 根骨误差 < 0.06°。
3. **蒙皮自洽**: 静止姿态 `W · invBind ≈ I`(1e-4 量级)。
4. **第三方校验**: 所有 GLB 过 Khronos glTF-Validator(84 个文件 0 error), `pygltflib`/`trimesh` 可加载。
5. **渲染对比**: 软件光栅化把 GLB/`.gui` 画出来人工核对(见 `_agent-sandbox/gui_preview`、`skin_sheet.png`)。

## 4. 工具与产物

| 工具 | 作用 |
|---|---|
| `gammaray unpack/convert/pack/info` | GammaRay 前端统一入口 |
| `deltaxutils/gamma_formats.py tex-*` / `unpack` / `r-*` / `fl-*` | 贴图与容器 |
| `deltaxutils/gamma_formats.py eqp` / `skins` | 皮肤部件 + 权威皮肤表 |
| `deltaxutils/gamma_formats.py gui-info` / `gui-render` / `gui-batch` | UI 结构与预览 |
| `deltaxutils/gamma_formats.py eft-info` / `eft-tex` / `eft-preview` | 特效结构(逐 unit 字段/语义)/贴图表/结构预览卡 |
| `deltaxutils/gamma_formats.py mtr-info` | 材质 extra、alphaMode、doubleSided |
| `deltaxutils/gamma_models.py ams2glb` / `anf-info` / `anf-table` | 模型/骨架/动画(自动读 `.mtr`;kc 事件表) |
| `deltaxutils/skin_sheet.py` | 皮肤 3D 缩略图与皮肤页拼图 |

产物目录:

```
_agent-sandbox/
  png_all/          .tex 全量 PNG
  skin_all2/        78 个 role 模型(静态蒙皮)
  anim_all/         78 个模型 × 全部声明 clip(540 个动画)
  fx_models/        59 个特效模型
  gui_preview/      213 张 UI 预览 + 皮肤页
  temp-another/     临时脚本与证据
```
