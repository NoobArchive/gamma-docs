# .ans — Gamma 动画集

记录 4CC = `"ans\0"`,版本 `0x2715`,使用 `CCommonFileHead`。

## 结构

```
"ans\0" | version 0x2715 | depCount = 1 | deps[] | extraSize | index 数据
```

实测 `role/ani/bind.ans`(2597 字节):

```
dep tag "anf\0", nameCount = 6
  "../attack_1.anf"  (长度 14)
  "../bind.anf"      ...
  ... 共 6 个 .anf
```

依赖编码同 `CCommonFileHead`:

```
tag(4) + nameCount(u32) + nameCount × [ charCount(u32) + UTF16LE ]
```

随后是 `CAniGroup::OnReadIndexData`(@`0x2ba278`)读取的骨架/动画组/`SFigure` 索引,
与 `ams` 的 index 段一致(见 `ams.md`)。

## 与 ams / anf 的关系

```
.ams (CAniGroup)  ── 骨架 + 图元 + 内嵌动画
.ans (动画集)     ── dep -> N 个 .anf
.anf (动画帧)     ── 每帧 12+12+16+12 字节
```

`.ans` 是**动画集合**,把多个 `.anf` 组合成动作列表(如 `bind`、`attack_1`、`stand`)。

> 注意:`.ans` 的依赖表**不一定列出同目录所有 `.anf`**。例如
> `p_001_attack_1.anf` 存在,但 `p_001.ans` 只登记了 10 个 clip,没有 `attack_1`。
> `ams2glb --orphan-clips` 会额外补上同模型未登记的 `<model>_*.anf`
> (跳过 `p_022_a_*` 这类属于另一个骨架的前缀);默认 `--clips` 只导声明表。

## 共享骨架与 `bind.ans`(重要)

`.ans` **不是模型**,它只提供骨架/绑定/动画列表;模型是 `.ams`。
`role/ani` 里有些 `.ans` 没有同名 `.ams`,它们是共享或未使用的骨架:

| `.ans` | 用途 |
|---|---|
| `bind.ans` | **28 骨人形绑定骨架**,带 `attack_1/bind/fly_die/run_w/sprint/stand_rest1` 通用 clip;`p_*_a` 变体的回退骨架 |
| `box01.ans` | 箱子共享骨架(9 骨);`box02/03` 回退到它 |
| `p_001a.ans` | 人形骨架变体,当前没有同名模型(未使用/备用) |
| `obj_001_1.ans`、`obj_002/005/006.ans` | 道具骨架,本包中没有同名模型(备用) |
| `xx01_pb_001.ans` | 与 `fx/mod/xx01_pb_001a.ams` 不同名,当前未使用 |

模型查找骨架的顺序(工具实现):

1. 同名 `<model>.ans`;
2. 若模型名以 `_a` 结尾 → `bind.ans`;
3. 若模型名以 `box` 开头 → `box01.ans`;
4. 都没有 → 只导出静态网格(不写 skin/JOINTS_0)。

因此 `box02/03`、`p_004_a/p_015_a/p_023_a/p_024_a` 以前会丢骨架,现在会走共享回退;
`map01`、`obj_01..04` 本身 `skin=0`,静态导出是正确的。

## 骨架 / 绑定数据(extra 尾部)

`CAniGroup::OnReadIndexData`(@`0x2ba278`)的顺序:

```
u16 skeletonCount
  每骨: ReadString 名字
        u8 b1, u8 子骨数 b2, u8 b3;  u8×b3 子骨索引;  b2 × [ReadString + 48B]
u16 nAnf
  每个动画组: 4B + 2B + u16 keyCount + keyCount×8B
u16 figureCount
  figureCount × u16 (figure 索引)
skeletonCount × 52B  绑定数据
```

每骨 52 字节(`CAniGroup::_OnReadMainData` @`0x2b8b80` 读取):

```
f32×3  size        骨骼包围盒(不参与矩阵)
f32×3  B           局部平移(子相对父)
f32×4  C           四元数,表示**世界**绑定旋转
f32×3  D           世界逆绑定平移
```

引擎的用法(`_OnReadMainData` 里 `CQuaternion::operator CMatrix` + 存成 3×4):

- 世界绑定旋转 `Q_i = C_i^T`(等价四元数 `conj(C_i)`)
- 世界绑定位置 `t_i = -C_i^T · D_i`
- 逆绑定矩阵 `invBind_i = [C_i | D_i]`(`GetSkeletonMatrix` 的 a7 分支就是乘它)
- 局部平移就是 `B`,并且满足 `t_i = t_parent + C_parent^T · B_i`

实测(3 个模型 / 97 根骨):`t = -C^T·D` 与 `t_i = t_p + C_p^T·B_i`
最大误差 **0.0001**,层级可完整复现。

`glTF` 导出对应关系:

```
node.translation = B
node.rotation    = quat(C_parent · conj(C_i))
inverseBind      = [C_i | D_i]        // 直接取文件值,不用自己求逆
```

> 交叉验证:`p_046_bind.anf` 第 0 帧的局部旋转按 `world = parent · local` 连乘,
> 与 `conj(C_i)` 的角度差 < **0.06°**(97 根骨全部),说明上面的语义正确。

> 注意:`CAniCursorFile` 的 RIFF/`ACON`/`anih`/`fram` 块是 Windows ANI 光标格式,
> 与 `.ans` 无关。
