# .anf — Gamma 动画帧数据

记录 4CC = `"anf\0"`,版本 `0x2715`。骨架来自同名 `.ans`(见 `ans.md`),`anf` 只存
**每根骨的关键帧**。

## 文件布局

```
CCommonFileHead (depCount=0, extra=0)
main:
  f32  transScale        // 平移长度的倍率
  f32  scaleScale        // 缩放的倍率
  boneCount × [u16 nScale, u16 nRot, u16 nTrans]     // 所有骨头先集中存放
  然后按骨顺序:
    nScale × [u8 time][u16 × axisCount]              // axisCount = 3 或 1
    nRot   × [u8 time][u32 axis][u16 angle]
    nTrans × [u8 time][u32 axis][u16 length]
```

- 关键帧是**逐个交错**存放的(`time[i]` 紧跟 `key[i]`),不是先读完时间表。
- `time` 是帧号(0..255)。
- **时间轴 = 30 fps**(与游戏内动作速度一致);`--fps` 可覆盖,但默认 30。
  例如 47 帧 ≈ 1567ms。
- `axis` 用 `TTinyNormal<32>::Decompress1` 解成单位向量:
  ```
  v = n & 0x3FFFFFFF; m = isqrt(v)
  a = m * 0.000047938
  b = (v - m*m)*pi/(2*m)
  x = sin(a)sin(b); y = cos(a); z = sin(a)cos(b)
  if n & 0x80000000: x = -x
  if n & 0x40000000: y = -y
  ```
- 旋转:`angle = u16 * pi / 65535`,四元数 = `(sin(angle)*axis, cos(angle))`。
- 平移:`pos = axis * (u16 * transScale / 65535)`。
- 缩放:axisCount=3 时 `xyz = u16 * scaleScale / 65535`;=1 时三轴取同一值。

## 实例

`box01_stand.anf`:9 骨、每骨 32 个旋转 key、无平移/缩放。
`p_046_bind.anf`:59 骨、每骨 39 个旋转 key。

## 关键语义(反编译 `IAnimation::GetFrameData` @`0x2b9d08`)

- **旋转是绝对局部旋转**:解码出的四元数直接就是 `CFixedSkeleton.quat`,
  引擎不再乘绑定姿态;缺 rotation 通道时回退**单位四元数**(不是绑定旋转)。
- **平移**:解码出的平移**直接替换**局部平移;缺通道时回退 `.ans` 里的 `B`(绑定局部平移)。
- **缩放**:缺通道时回退 `1.0`。
- 若传入 `SFigureUnit*`(附加 figure),只做 `trans = anim + figure.trans`、
  `scale = anim.scale * figure.scale`,旋转仍取动画值。

`TAnimation<u16>::GetFrameData`(@`0x2b...`)里这段回退逻辑可见:
`count==0` 时 translation 取 `bone[16..24]`(即 `.ans` 的 B),rotation 写 `(0,0,0,1)`。

> 因此动画 clip 的旋转是完整姿态,`bind.anf` 第 0 帧 = 绑定姿态;
> 导出时对没有 rotation 通道的骨要显式写单位四元数(已实现)。

## 解析覆盖率与两种变体

`role/ani/*.anf` 共 **542** 个,全部可解析(542/542)。有两处必须按“恰好读完 main”反推:

1. **骨骼表长度 ≠ 骨架骨数**:部分 clip 只动画前 N 根骨。
   例:`p_001_attack_1` 只有 28 项(`p_001.ans` 有 29 根),`p_002_attack_1` 23 项(26 根)。
   缺失的骨在引擎里就是 `nS=nR=nT=0`,即「单位旋转 + 绑定平移 + 缩放 1」;
2. **时间戳位宽**:帧数 > 255 的 clip 用 **u16** 时间戳(如 `l_p_003b_stand_rest1`,282 帧);
   ≤255 用 u8。

解析时先按骨架骨数试,失败则遍历 `(时间位宽 1/2) × (axis 3/1) × (骨数 1..N)`,
用长度方程 `8 + 6*nb + Σ(nS*(tw+2*axis)+nR*(tw+6)+nT*(tw+6)) == len(main)` 精确判定,
再把结果补齐到骨架骨数。

## 动画表事件(kc)

每个 `.ans` 尾部的动画表,在 52B/骨的绑定数据**之前**,每条 clip 的布局是:

```
u32 a              // 实测 1
u16 b              // 实测 14..47,疑为帧数/时长相关
u16 kc             // 事件数
kc × 8B events     // 事件数据(本作全部为空)
```

用 `anf-table` 扫描 93 个 `.ans`、**568 个 clip**:`kc` **全部为 0**,事件表为空。
因此本作没有可利用的动画事件;若后续版本出现非 0,事件按 8 字节原样保留。

## 工具

```powershell
python gamma_models.py anf-info <x.anf> --ans <x.ans>
python gamma_models.py anf-table <x.ans> [--json out.json]   # kc/事件表
python gamma_models.py ams2glb <x.ams> <x.glb> --ani-root <role/ani> --anim <x_run.anf>
```
