# .ams — Gamma 模型 / 骨架

**类**: `Gamma::CAniGroup`(`CAniGroup::GetFileType()` 返回 `"ams"`,即 `'a'|'m'<<8|'s'<<16`)。
记录 4CC = `"ams\0"`,版本 `0x2717`。

## 1. 容器头(`CCommonFileHead`,见 [common.md](common.md))

```
type "ams\0" | version 0x2717 | depCount | deps[] | extraSize
dep = tag(4) + nameCount(u32) + nameCount × [ charCount(u32) + UTF16LE ]
```

实测 `role/mod/box01.ams`:

```
mtr\0  [ "mat/emissive_ccw.mtr" ]
tex\0  [ "role/tex/box01_t01.tex", "role/tex/box01_t02.tex" ]
```

- `mtr` 依赖 = 材质(`.mtr`)
- `tex` 依赖 = 贴图(`.tex`)

## 1.5 piece 组 index 数据(`CPieceGroup::OnReadIndexData` @ `0x2f9da8`)

`.ams` 的 `extra` 段描述每个 piece 的**量化包围盒**与材质/贴图表:

```
u16 pieceClassCount
重复 pieceClassCount:
    u32 nameLen + UTF16LE                       piece 类名(如 "p")
    u16 pieceCount
    重复 pieceCount:
        u8  type
        u32 f16                                  (实测 0.0)
        f32 err                                  量化误差,决定顶点位宽档(见 §4)
        u8  bits                                 偏移精度位宽(实测 0)
        u8  profile                              (实测 1)
        f32 ×3 size                              包围盒**尺寸**
        f32 ×3 center                            包围盒**中心**
        f32 ×3, f32 ×3                           与上两者相同(旧版是拷贝)
        u8  matCount
        重复 matCount: u16 tag, u8 subs, 重复 subs: u8 n + u16[n]   // 贴图索引
```

即 `CPiece::ReadIndexData`(`0x2f8b60`,版本 `>0x2715`)读 `f16, err, bits, profile,
size, center, size, center`(共 58 字节),再 `ReadMaterial`。

每个 piece 的 `matCount` 项**不是分层,而是配色/皮肤变体**:每一项带一个贴图索引,
实测 `p_001` 的 10 张贴图是同一张 atlas 的 10 种配色(`t01` 红蓝、`t02` 洋红 …),
`box01` 的 2 张也是两种配色。运行时按玩家颜色选其中一项。

因此导出时每个 piece 取自己材质表里的**第 `--skin N` 项**(默认 0,即基础配色),
只嵌入真正用到的贴图;两个 piece 用同一张图时合并为一个 material。

**顶点真实位置**由引擎这样还原(见 `SaveInfo` 构造 与
`TVertexDecompress<8>::LoadVertex` @`0x2f2db4`):

```
pos = q * 0.25 + (center - size * 0.5)          // 即加上包围盒 min 角
```

`q` 是位流里的无符号量化值,`0.25` 就是"1/4 单位"精度;`SaveInfo` 里
`off = int(4*center) - int(4*size)/2`、再次 `*0.25` 得到同一个 min 角。
`bits=0` ⇒ `2^bits=1`,不额外缩放;`err` 用来选位宽档(≈1.0 → `<10,11,8>`)。

## 2. index 数据(`CAniGroup::OnReadIndexData` @ `0x2ba278`)

紧跟在头之后(`extraSize` 字节),按位/按序读取:

```
u16 skeletonCount
重复 skeletonCount:
    ReadString  (u32 charCount + UTF16LE)        骨骼名, 如 "Box"
    u8 ×3                                        (子骨骼数/标志/动画组数)
    [u8 ×n]                                      骨骼索引表
    [动画组 IAnimation]                           每项: ReadString + 12×float
u16 animationCount / IAnimation 表
u16 figureCount
SFigure[figureCount]                             每项 24 字节
```

`SFigure` 与动画帧数据由 `CAniGroup::_OnReadMainData`(@`0x2b8b80`)读取:
每帧 `12B 平移 + 12B 缩放 + 16B 四元数 + 12B`;倍率 = `renderer[+4] * (1/64)`。

## 3. main 数据 —— 单 piece(`CStaticPiece`)

`role/mod/*.ams` 的 main 由一个或多个 **piece 顺序拼接**而成(不是 per-vertex 交错):

```
piece*:
  u16   (通常 0)
  u16   vertexCount
  u8    packedVertices[vertexCount * packed]   // packed = 8/9/10/11/12
  u8    skin[vertexCount * 8]                  // 蒙皮块(可能不存在)
  u32   indexCount
  u8/u16 indices[]                             // vertexCount > 255 ? u16 : u8
```

- 多 piece 例:`p_001.ams` = 2 件(435 + 8 顶点),`p_034.ams` = 2 件(packed=9 + 8);
- 解析时递归尝试 `(packed, skin)` 组合,要求**恰好消费完整个 main**,并校验 UV∈[0,1]、法线单位长;
- 合并时把后续 piece 的索引整体偏移,并把该 piece 的包围盒 `min` 角加到顶点上
  (见 §1.5;不加会把模型整体推到正八分区,动画时炸开);
- 蒙皮块是独立块,不能按 `stride=16` 交错读,否则出现尖刺;
- `CStaticPiece::ReadMainData` @ `0x2f8f54`,`WriteMainData` @ `0x2eaac4`,
  `OnReadVertex` @ `0x2f9024`。

### 顶点变体(`TVertexDecompress<1..8>` 的 `CTinyVertex`)

| packed | 位宽 | 位置 bit | 法线 bit | UV bit |
|---|---|---|---|---|
| 8 | 10,11 | 0/10/20 | 30..41 | u 42(11), v 53(11) |
| 9 | 12,12 | 0/12/24 | 36..47 | u 48(12), v 60(12) |
| 10 | 12,16 | 0/12/24 | 36..47 | u 48(16), v 64(16) |
| 11 | 14,16 | 0/14/28 | 42..53 | u 54(16), v 70(16) |
| 12 | 16,16 | 0/16/32 | 48..59 | u 60(16), v 76(16) |

位置 ×0.25、UV ×0.0005;法线统一 12 bit。本次游戏绝大多数模型是 **packed=8 + skin=8**。

## 4. 顶点解码(以 packed=8 的 `<10,11,1>` 为例)

每个 packed 顶点是**小端位流**(`TBitSet::GetBit`,`0x2ec4c8`);其它 packed 尺寸的位偏移见上表。

| 字段 | bit | 换算 |
|---|---|---|
| pos.x | 0..9 | `* 0.25` |
| pos.y | 10..19 | `* 0.25` |
| pos.z | 20..29 | `* 0.25` |
| normal | 30..41 | `TTinyNormal<12>::Decompress1` |
| uv.u | 42..52 | `* 0.0005` |
| uv.v | 53..63 | `* 0.0005` |

`TTinyNormal<12>`(`0x2ec5c0`):

```
v = n & 0x3FF; m = isqrt(v)
a = m * 0.050671
b = (v - m*m) * pi / (2*m)
x = sin(a)*sin(b); y = cos(a); z = sin(a)*cos(b)
if n & 0x800: x = -x
if n & 0x400: y = -y
if v == 0: (0, ±1, 0)
```

索引: `vertexCount>255` 用 u16,否则 u8。

### 验证

- `box01.ams`: 664 顶点 / 1536 索引(512 三角形),`packed=8 + skin=8`;
  解出 262 个唯一位置,最大三角形边 / 包围盒对角 = **0.38**(无尖刺)。
- `box03.ams`: 1081 顶点 / 1268 三角形,689 个唯一位置,maxEdge/diag = **0.35**。
- `jingyanhezi.ams`: 20 顶点 / 12 三角形,去重后正好 **8 个立方角点**,
  bbox 64×64×64,maxEdge/diag = **0.82**(面对角线,正常)。

> 旧版按“16 字节交错”读取会把蒙皮块当作顶点,正是 Blender 里长尖刺的来源。

## 5. 工具

```powershell
python gamma_models.py ams2glb <in.ams | dir> <out.glb | outdir>
python gamma_models.py ams2glb <dir> <outdir> --tex-root <png_root> \
    --ani-root <role/ani> [--flip-v]
python gamma_models.py ams2glb <x.ams> <x.glb> --ani-root <role/ani> \
    --anim <x_run.anf> [--fps 30] [--skin 0]
python gamma_models.py ams2glb <x.ams> <x.glb> --ani-root <role/ani> --clips
```

- `--tex-root`:把 `ams` 的 `tex` 依赖对应 PNG 嵌入 GLB(`baseColorTexture`);
- `--ani-root`:`role/ani` 目录。加载同名 `.ans` 的骨架(如 `p_046.ans` 59 骨),
  写入 `skins` + `JOINTS_0`/`WEIGHTS_0`(蒙皮块每顶点 `[4×u8 权重][4×u8 骨索引]`);
- `--anim`:把一个 `.anf` clip 写成 glTF animation(旋转/平移/缩放通道);
- `--clips`:把该模型 `.ans` 声明的全部 clip 写进同一个 GLB(多 `animations[]`,
  Blender 里会变成多个 Action);clip 名取声明名(如 `stand`、`run_w`);
- `--orphan-clips`:额外包含同模型未登记在 `.ans` 的 `<model>_*.anf`
  (如废案 `p_001_attack_1.anf`);默认不包含;
- `--no-anim-manifest`:默认会在 GLB 旁写 `<model>.anims.json`,
  记录 `declared`(ans 声明)、`clips`(实际导入)、`orphans`(未登记及 `included` 状态);
- `--fps`:动画时间轴帧率(默认 30);
- `--skin`:配色/皮肤变体序号(取 piece 材质表第 N 项,默认 0);
- `--flip-v`:贴图上下颠倒时输出 `1-v`;
- 当前覆盖 **78 / 78** 个 `role/mod` 模型(单/多 piece、packed=8/9、skin=8);
- 多 piece 导出为多个 primitive,每个 piece 的顶点都已加上自己的包围盒 `min` 角;
- 关节 rest 变换 = `translation = B`、`rotation = C_parent · conj(C_bone)`(需归一化);
- `inverseBindMatrices` **直接取 `.ans` 里存的 `[R | D]`**(见 `ans.md`),
  与引擎 `GetSkeletonMatrix` 的 a7 分支完全一致;
- 静止姿态下 `W · invBind` 实测偏差 < 1e-4(即绑定姿态自洽)。

## 回写(GLB → `.ams`)缺口

目前只做 **`.ams` → GLB**,没有反方向。要回写至少还缺:

1. **piece 主数据布局**:旧版 0x2713/0x2714 的 per-piece 头已解,
   但 0x2716/0x2717 的 stride/蒙皮/索引与前缀仍需逐字段确认;
2. **顶点量化**:位宽 8/9/10/11/12 的打包、`TTinyNormal`、UV 缩放
   需要把 float 逆量化回整数并处理越界;
3. **piece 包围盒/材质表**:`.ams` extra 的 size/center/bits/mats
   需要从 GLB 的 primitive/材质反推;
4. **动画**:`.anf` 的 30fps 时间轴、四元数轴角、平移/缩放轨道
   需要从 glTF 动画采样并量化;
5. **蒙皮**:joint/weight 的 4×u8 打包顺序与 `.ans` 骨索引绑定。

因此当前 `prepare` 只处理贴图,不处理模型;`pack` 只负责把现有文件封包。
