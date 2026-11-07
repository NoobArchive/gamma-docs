# .eqp — 装备/皮肤注册表

> 格式来自 `misc/ShellCommonLua/equip/CEquipmentGroup.lua` 的
> `CEquipmentGroup.Parse` / `LoadEquipment`(爆彩童话 2015 版)。
> 当前解析器:`gamma_formats.py` 的 `parse_eqp` / `eqp` 命令。
>
> 结论:**爆彩童话真正在用;粉刷匠大作战从 1.1.6.6 起就没有任何运行时引用。**

## 1. 两种游戏的实际使用情况

### 爆彩童话 1.1.0.2(在用)

启动时 `misc/ShellClientLua/CGameClient.lua` 无条件执行:

```lua
CDressingRoomClient:Instance():LoadAllFromURL( g_App:GetDataPath().."role/dress/", false );
```

`CDressingRoom.s_EquipGroups = { "npc", "male", "female" }`,依次加载
`role/dress/npc.eqp`、`male.eqp`、`female.eqp`。

`m_EquipmentPackages[group][equipName]` 的用途:

| 调用点 | 作用 |
|---|---|
| `CPlayer.lua:72` | `SetAnsFileName(GetAnsFileName(Equip[1], Equip[2], Equip[3]))` |
| `CNpc.lua:148` / `CHero.lua:140` | NPC/英雄按 `"npc"` 组取 `.ans` |
| `CDressingRoomClient:PutOnAll` | `ClearPiece` → 逐个 `AddPiece(mesh.ams, pieceClass, matIndex)` → `SetAni(.ans)` → 特效/缩放 |
| `LoadAllHeroModel` | 预加载 `npc.eqp` 的 `yingxiong` 组 |
| `CSelectFighter` / `CHeroPanelInfo` / `CCharacterPreviewWnd` | 选人、英雄面板、预览换装 |

也就是说:爆彩的“装备名 → 骨架 + 多个部件 + 特效 + 缩放 + 材质/贴图变体”
全部由 `.eqp` 决定,是**角色渲染的核心注册表**。

样本规模:

| 文件 | 装备数 | 分组 | ams/ans/eft 依赖 |
|---|---:|---|---|
| `male.eqp` | 18 | body/foot/head 各 6 | 6 / 1 / 1 |
| `female.eqp` | 18 | body/foot/head 各 6 | 6 / 1 / 1 |
| `npc.eqp` | 292 | boss 29, emeng 7, hero 26, npc 19, obj 33, tszdnpc 19, yingxiong 34, yingxiong_zhanshi 29, zdnpc 96 | 118 / 112 / 42 |

### 粉刷匠大作战(未使用)

| 版本 | `role/dress/male.eqp` | 二进制是否引用 | Lua 是否引用 | 真正用的表 |
|---|---|---|---|---|
| 1.1.6.6 | 2,982B, 18 装备 | 否 | 无 Lua | `config/resource/equip.txt` |
| 1.1.20 | 4,861B, 29 装备 | 否 | 否 | `equip.txt` |
| 1.3.3.4.7 | 与 1.1.20 **哈希相同** | 否 | 否 | `equip.txt` + `fusion.txt` |

证据:

- 各版本 `libShellClient.so` 里搜不到 `.eqp` / `eqp` / `DressingRoom` /
  `EquipmentGroup` / `role/dress/`,但能搜到 `equip.txt`、`role/mod/`、`role/ani/`;
- 1.3.3 的 `misc/ShellClientLua`、`ShellCommonLua` 没有任何 `eqp` /
  `CDressingRoom` / `CEquipmentGroup` 引用;
- 1.3.3 的 `equip.txt` 有 34 行、`fusion.txt` 18 行,字段直接是
  `szModel` / `szAni` / `nMatrlIndex0..9` / `szFx0..9` / `nScale`,
  每套皮肤一个模型,不再需要多部件注册表;
- `male.eqp` 里的名字是旧开发名(二哈、僵尸、兔女仆、厨子张飞、大法师、
  女粉刷匠、孙悟空……),与 1.3.3 `equip.txt` 的正式名
  (哈士奇、蹦迪僵尸、兔酱、张飞砍咸鱼、乱棍老法师、节奏小粉酱、
  齐天大圣……)对不上;且 eqp 只有 29 项,不含 1.3.3 后期的 18 个融合皮肤。

结论:`male.eqp` 是爆彩时代/粉刷匠早期工程留下的**静态遗留数据**,
粉刷匠从首个公开包开始就没有装载它;后续新皮肤只更新
`equip.txt` / `fusion.txt` / `role/mod/*.ams`。

## 2. 文件结构

标准 `CCommonFileHead`(version = `8`),4 组依赖:

```text
deps[0] = ams   部件网格
deps[1] = ans   骨架/动画
deps[2] = eft   特效
deps[3] = tex   皮肤贴图
```

extra 数据:

```text
u32 skinTypeCount
per skin type:
    u32 textureSetCount
    per set:
        u32 + UTF-16 setName
        u32 skinCount
        per skin:
            u32 + UTF-16 className
            u32 textureIndex          -> deps[tex]

u32 equipmentCount
per equipment:
    u32 + UTF-16 equipName
    u32 + UTF-16 equipGroup           // body/foot/head/npc/hero/boss...
    u32 aniIndex                      -> deps[ans]
    u16 figureID
    f32 scale
    u32 effectIndex                   -> deps[eft]
    u32 + UTF-16 effectName           // 逻辑名,如 "fx/loop"
    skinTypeCount × (u32 + UTF-16 decorateSetName)
    u32 meshPartCount
    per part:
        u32 pieceIndex                -> deps[ams]
        u8  materialIndex
        u32 + UTF-16 pieceClassName   // 与 .ams extra 的 piece class 对应
    u32 hideCount × (u32 + UTF-16 hideType)
    u8  transparency
    u8  hideSkin
    u32 renderFlag
    u32 + UTF-16 stateEffectFileName
```

古早的 `parse_eqp_names` / `parse_eqp_skins` 只是按字符串模式扫描;
现在是完整解析,5 个样本全部“恰好读完”:

```text
bc male   2706/2706
bc female 3054/3054
bc npc   52134/52134
fsj 1.1.6.6 male 2982/2982
fsj 1.1.20/1.3.3 male 4861/4861
```

`skinTypeCount = 3`,但所有样本的 `textureSetCount = 0`,
`decorate_sets` 也都是空串 —— 贴图变体机制存在但这两作没启用。

## 3. 工具

```powershell
# 完整解析:依赖数、分组统计、每条装备的 .ans/.ams/scale/材质
python gamma_formats.py eqp <x.eqp> [--json out.json] [--txt out.txt] ^
    [--root <解包根目录>] [--limit 40]
```

JSON 里字段与 Lua 一一对应:

```text
version / exact / deps.ams|ans|eft|tex
skin_types{setName: {className: texName}}
equipments[]:
  name, group, ani, figure_id, scale
  effect, effect_name, decorate_sets[]
  mesh_parts[]: {piece, piece_class, material_index}
  hide[], transparency, hide_skin, render_flag, state_effect
```

## 4. 与 `.ams` 的关系

爆彩的装备是**多部件**:同一套外观可由多个 `.ams` + piece class 组成,例如
`npc.eqp` 的“企鹅老大boss”:

```text
bh_013.ams : zhijijunshenti
bh_013.ams : zhijijunwuqi
bh_013.ams : zhujijunxiashen
```

粉刷匠的 `equip.txt`/`fusion.txt` 则是一套皮肤一行、一个 `szModel` + 一个
`szAni`,这也是 `.eqp` 完全没有必要继续存在的原因。
