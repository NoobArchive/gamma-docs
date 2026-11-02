# .eft — Gamma 特效

记录 4CC = `"eft\0"`,版本 `0x271b`(实测 376/376 全部为此版本),使用 `CCommonFileHead`。
**没有 main 段**,内容全在 `extra`(1952 KB 总量)。

## 1. 依赖

```
dep "ams"  特效用的模型(fx/mod/*.ams)     —— 最多 5 个
dep "ans"  特效用的骨骼动画(fx/ani/*.ans)
dep "tex"  粒子贴图(fx/tex/*.tex)          —— 最多 31 个
```

实测:53 个不同模型、14 个动画、425 张贴图被引用;
`fx\tex\glow_001.tex` 被 122 个特效复用,是最常见的粒子贴图。

## 2. 结构(`CEffectGroup::OnReadIndexData` @`0x2c6578`)

```
u16 propCount
重复 propCount:
    ReadString                      prop 名(以 "fx/" 开头)
    CEffectProp::OnReadIndexData ...
```

`CEffectProp::OnReadIndexData`(@`0x2c5480`,版本 0x271b 走进全部分支):

```
[ver>0x2711] u32 dur(毫秒)
             u32 nStr + nStr × ReadString        // 触发条件/标签
             ReadString                          // 拼接名(如 "all")
f32 ×3 pos
f32 ×3 rot
[ver>0x2713] u8  b92
[ver>0x2719] u8  b93, u8 b94
u16 unitCount
unitCount × (u16 type + 该 type 的 unit 数据)
```

- prop 名 = **动作状态名**:`fx/loop`(311 次)、`fx/fly_die`、`fx/attack_1`、
  `fx/run_w`、`fx/cast`、`fx/rb1`、`fx/end` …… 和角色动画 clip 名一致;
  也有程序拼出来的长名(如 `fx/loop14943896441494389754…`);
- `dur` 常见值:1000ms(351)、2000ms(99)、500ms(65)、850ms(40)、6000ms(31);
- 一个 `.eft` 内 prop 数:1 个占多数,其余 2~12 个。

> prop 名字符串长度上限不能卡在 40:本作有 50+ 字符的拼接名,
> 卡 40 会漏掉真正的 prop 起点(`_eft_prop_marks` 现为 200)。

## 3. unit(发射器)

`u16 type` 之后由 `CRenderer::CreateEffectUnitProp`(@`0x30348c`)的 TBB 跳转表分派。
**type → 类**的映射(直接从跳转表字节读出,13 个 case):

| type | 类 | 本作 unit 数 |
|---:|---|---:|
| 0 | `CParticleSystemProp` | 668 |
| 1 | `CSpriteProp` | 2431 |
| 2 | `CPolyTrailProp` | 219 |
| 3 | `CCameraShakeProp` | 12 |
| 4 | `CCameraFilterProp` | 8 |
| 5 | `CModelConsoleProp` | 462 |
| 6 | `CDynamicLightProp` | 0 |
| 7 | `CNullProp` | 26 |
| 8 | `C3DAudioProp` | 29 |
| 9 | `CModelMaterialProp` | 797 |
| 10 | `CPolyChainProp` | 0 |
| 11 | `CModelAnimationProp` | 0 |
| 12 | `CGeometryProp` | 384 |

每个子类的 `ReadIndexData` 都是**先读自己的前缀,最后调用基类**
`CEffectUnitProp::ReadIndexData`(@`0x2c9140`)。
下表偏移是类成员偏移(便于和反编译对照),`ver` 是前缀里的第一个 u32
(不是文件版本,是**该 unit 的功能版本**,决定后续条件字段)。

### 3.1 子类前缀

```
type 0 CParticleSystem
  u32 ver
  u32 a180, a188
  f32 f192, f196
  TVector3 v200                     // 最小发射间隔
  TVector3 v212, v224, v236         // 每次发射粒子数 / 最小尺寸
  u32 a252, a256, a260, a264
  f32 f268, f272
  u32 a276,a284,a288,a296,a300,a304,a312,a292,a316,a280,a324,a328
  u8  f336
  ver≠0: u32 a184, a332
  ver>1: u32 a308
  ver>2: u32 a320
  ver>3: u32 a248

type 1 CSprite
  u32 ver
  u32 a204, a208
  f32 f252
  TVector3 v180                     // 调节类型
  TVector3 v192                     // 曲线参数
  f32 f212, f216
  u32 a220, a224, a228
  u8  f232
  ver>0: u8 f233 ; ver>1: u8 f234 ; ver>2: u8 f235
  ver>3: u32 a324
  ver>4: 8B r236 + 8B r244

type 2 CPolyTrail
  u32 ver
  u8 f177,f178,f179,f180,f181
  12B r184
  f32 f196, f200
  u32 a208, a212, a216
  u8  f220
  ver>0: u32 a224 ; ver>1: a228 ; ver>2: a232 ; ver>3: a204

type 3 CCameraShake
  u32 ver
  u32 a180, a184
  f32 f188, f192
  ver≠0: u32 a196

type 4 CCameraFilter
  u32 ver
  u32 a180, a188, a184
  f32 f196, f200, f204
  ver≠0: u8 f208, u32 a192, u8 f209, u8 f210

type 5 CModelConsole
  u32 ver
  string s240
  u16 u232
  u16 u244, u246
  u32 a248, a252
  u32 a192, a196, a200
  string s208
  string s236
  12B r180
  7 × (u16, u8)
  ver>1: u8 f288

type 7 CNull
  u32 ver
  12B r180
  u8 f196
  ver≠0: u32 a192, u8 f197
  ver==0: 2B(跳过) + u8 f197
  ver>1: u8 f198

type 8 C3DAudio
  u32 ver
  u32 a196
  f32 f200, f204
  ver≠0: u8 f208
  string s192

type 9 CModelMaterial
  u32 ver
  string applyClass
  u8 shaderType
  ver>5: 12B, 否则 ver≠0: 8B
  2<=ver<=5: string
  3<=ver<=5: 8B

type 12 CGeometry
  u32 ver
  u8 f199, f195, f196
  u32 a208, a212, a200, a204
  u8 f197, f198
  u8 f272, f273, f274, f275
  u32 a180
  u8 f184
  u32 a188
  u8 f192, f193, f194
  12B r216
  f32 f228, f232, f236, f240, f244, f248
  u16 u252, u254
  u32 a256, a260, a264, a268
  ver≠0: u32 a276
```

### 3.2 基类(所有 type 最后都走这里)

```
[ver>0x2712] u32 id
u32 texCount + texCount × u32          // 只有存在 tex 依赖时才读这些索引
u32 n + n × f32                        // this+104
u32 n + 跳过 4n
u32 rows + rows × (u32 m + m × f32)    // this+128
u32 rows + rows × (u32 m + m × f32)    // this+152
u32 n + n × TVector3(12B)              // this+116
颜色:
  ver<=0x2717: u32 n + n × u32
  ver>0x2717:  n × (u32 m + m × u32)
字节数组:
  ver<=0x2715: u32 n + n 字节
  ver>0x2715:  n × (u32 m + m 字节)
u32 ×4 (this+16/20/24/28)
u16 (+88), u8 (+90), u8 (+91)
[ver>0x2716] u16 (+92)
[ver>0x2718] quat 4×f32 (+72)
[ver>0x271A] u32 (+68)
ReadString ×3 (this+32/36/64)
u32 k + k × ReadString (this+40 的 set)
```

`ReadLength<T>` 只读**一个** T(不是数组);引擎会把它乘上
`renderer.scale / 64`。工具里保留文件原始值,便于做数值对照。

### 3.3 语义属性(`props`)

各子类的 `GetBaseProp`/`GetAdvancedProp` 会把字段注册成编辑器属性,属性名是
**UTF-32LE 字符串**(如 `0x4D2720 = "接点偏移和缩放"`)。现在 `unit_details` 每个 unit
多一个 `props` 字典,把这些偏移翻成中文名,例如 CParticleSystem:

```
挂接点偏移和缩放 / 绕速度方向旋转 / 旋转同步
最大发射间隔 / 最小发射间隔 / 每次发射粒子数
最小尺寸 / 最大尺寸 / 发射平面法线
最小速度 / 最大速度 / 阻尼系数 / 最小角速度 / 最大角速度
最小生命周期 / 最大生命周期 / 发射器最小半径 / 发射器最大半径
矩形长宽比 / 长宽密度比 / 角点平分 / 跟随方式 / 混合模式
```

CSprite 的 `props` 含 `调节类型 / 曲线参数 / 动作名 / 朝向 / 旋转同步 /
最大发射间隔 / 圆周起始`;CGeometry 含 `UV模式 / UV方式 / 贴图拉伸 / 几何体类型 /
轴向上方最小/最大 / 圆周最大分段` 等。原始 `aXXX/fXXX/vXXX` 字段仍然保留。

### 3.4 验证

对 376 个 `.eft`、**754 个 prop、5036 个 unit** 做「恰好读完」校验:

- 每个 prop 的 unit 段必须与下一个 prop 的起点(最后一个到 `extra` 末尾)完全重合;
- 结果:**754/754 prop 精确闭合,0 error**;
- unit 类型分布见 §3 表。

## 4. 工具

```powershell
# 结构/依赖/prop/unit 列表(可导出 JSON;unit_details 是逐字段结果)
python gamma_formats.py eft-info <x.eft> [--json out.json] [--units]

# 结构预览卡:每个 prop 一张卡,含贴图、粗略粒子云和语义属性
uv run --with pillow python gamma_formats.py eft-preview <x.eft> <out.png> --tex-root <png_all>
# 把该特效用到的贴图拼成联络表(需要 Pillow)
uv run --with pillow python gamma_formats.py eft-tex <x.eft> <out.png> --tex-root <png_all>
```

`eft-info` 输出示例:

```
[fx/fly] dur=1000ms units=1 ... exact=True types=[CParticleSystem×1]
```

`eft-tex` 输出示例:`fx/cfg/benpao_1.eft` → 22 张贴图(速度线、光斑、月牙、魔法阵、闪电…),
可以直接看出这个特效长什么样。

## 5. 与模型的关系

特效引用的 `fx/mod/*.ams` 就是普通模型,`gamma_models.py` 可以直接导出 GLB:

```powershell
python gamma_models.py ams2glb <fx/mod> <outdir> --ani-root <fx/ani> --tex-root <png_all>
```

实测 59 个里 **59 个**都能导出(2026-10 复测;早期报的 11 个 piece 布局问题已随
`parse_ams` 修正而消失)。`.mtr` 也会自动带上:特效模型多为 `alphablend_ccw_2.mtr`
(alphaMode=BLEND、双面)。
