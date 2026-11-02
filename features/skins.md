# 皮肤表 / ID 编排

> 数据源:`config/resource/equip.txt`、`config/resource/fusion.txt`、`role/mod/*.ams`。
> 工具:`gamma_formats.py skins`。

## 1. 三套来源

| 来源 | 数量 | 说明 |
|---|---:|---|
| `equip` | 34 | 普通皮肤/角色;`nID` 1..35,缺 26 |
| `fusion` | 18 | 融合皮肤;`nID` 1001..1053 |
| `model` | 14 | 有 `.ams` 但未登记在 equip/fusion 的 `p_*` 模型 |

`model` 目前包含:

```
p_004_a, p_007, p_012, p_015_a, p_022_a, p_023_a, p_024_a, p_025_a,
p_034, p_035, p_036, p_037, p_039, p_046
```

不包括 `box*/map01/obj*/jingyanhezi`(道具/场景),也不包括 fusion 已收录的
`p_028*` / `l_p_*`。

## 2. ID 规律

### `equip.txt`

- `nID`:商店/配置行号,1..35;**26 是空号**,对应未登记的 `p_046`;
- `dName` / `dDescription`:字典键,后缀是**模型编号**:
  `p_001` → `0b040001`,`p_046` → (应为) `0b040046`;
- `nIconID`:2001..2035;
- `nSort`:展示排序 1..28;**999 = 特殊/隐藏排序**;
- `nShow`:`0` 表示商店默认不展示(未拥有/特殊皮肤);
- `nFragmentID` / `nFragmentCount`:碎片合成信息;
- `szModel` / `szAni`:模型/骨架路径;
- `szFx0..9`:跑动/展示特效。

旧版本列名不同,`skins` 已做别名:

| 版本 | ID 列 | 名字列 | 其它 |
|---|---|---|---|
| 1.3.3 | `nID` | `dName` | 完整字段 |
| 1.1.20 | `ID` | `szName` | `nSign`/`nType`,无 `nShow/nSort` |
| 1.1.6.6 | `ID` | `szName` | 仅 `szModel/szAni/nPrice/...` |

旧版没有 `nShow` 时默认 `show=true`;没有 `nSort` 时用 `ID`。

### `fusion.txt`

- `nID`:1001..1053,按 10 分组:
  - `1001..1003` → `p_028/p_028b/p_028c`(小天使/大天使/炽天使)
  - `1011..1013` → `l_p_001a/b/c`
  - `1021..1023` → `l_p_002a/b/c`
  - `1031..1033` → `l_p_003a/b/c`
  - `1041..1043` → `l_p_004a/b/c`
  - `1051..1053` → `l_p_005a/b/c`
- `dName`:`0b06xxxx`(如 `0b061011`);
- `nIconID`:3001..3018;
- `nSort`:1..6(按融合组);
- `nFragment1..5` / `nCount1..5`:多材料合成;
- `szModel` / `szAni` / `szFx*` 同 equip。

### 模型文件名

| 形式 | 含义 |
|---|---|
| `p_001` | 标准模型编号 1 |
| `p_004_a` | `p_004` 的变体(`_a` 后缀) |
| `p_028b` / `p_028c` | fusion 的 b/c 阶段 |
| `l_p_001a` | fusion 组 1 的 a 阶段 |

因此 **`id` 用模型 stem** 最稳定;`model_id` 再给数字编号(从 `p_NNN` 提取)。

## 3. 哪些皮肤默认隐藏

`nShow=0` 的 6 个(实测):

```
乱棍老法师, 音波豆丁, 缺爱小龙蛋, 内测玩赏家, 红烧狮子头, 圣诞老人
```

`nSort=999` 还包含 `财神驾到`(但 `nShow=1`),说明 999 只是“特殊排序”,
不等于隐藏。

## 4. 新 `skins.json` schema

`skins` 输出 66 条,字段:

| 字段 | 说明 |
|---|---|
| `order` | 输出顺序(equip → fusion → model) |
| `uid` | 唯一键:`equip:<nID>` / `fusion:<nID>` / `model:<stem>` |
| `id` | **模型 stem**(如 `p_001` / `p_004_a` / `l_p_001a`) |
| `legacy_id` | 原表 `nID`;model-only 为 `null` |
| `source` | `equip` / `fusion` / `model` |
| `registered` | 是否在 equip/fusion 表里注册 |
| `fusion` | 是否 fusion |
| `model_id` | 从 `p_NNN` 提取的数字 |
| `base_model` | 去掉变体后缀的基模型 |
| `variant` | `a/b/c` 等后缀 |
| `name` | 字典名;没有则用 `szMark` / stem |
| `dictionary_name` | 纯字典名 |
| `mark` | `szMark` |
| `key` | `dName` 字典键 |
| `description_key` / `description` | 描述键/文本 |
| `icon` | 图标 ID |
| `show` | `nShow == 1` |
| `sort` | `nSort`(999=特殊) |
| `price` / `acquire` / `propvalue` / `type` | 获取信息 |
| `fragment` | `[{id,count}]`;fusion 可能 5 项 |
| `pay` | `{type,index,count}` |
| `model` / `ani` | 模型/骨架路径 |
| `scale` | `nScale` |
| `materials` | `nMatrlIndex0..9` |
| `fx` | `szFx0..9` 中非空的路径 |

> `dName` 仍是本地化字典的权威名字来源;`id` 只是稳定标识,不替代 `key`。

## 5. 用法

```powershell
python gamma_formats.py skins config/resource/equip.txt `
    --config config --json skins.json

# 只要已注册的 equip/fusion
python gamma_formats.py skins config/resource/equip.txt --registered-only

# 自定义 fusion / 模型目录
python gamma_formats.py skins config/resource/equip.txt `
    --fusion config/resource/fusion.txt --models-root role/mod
```

导出的 `skins.json` 可直接喂给皮肤页/商店模拟器;`source=model` 的条目
默认 `show=false`,需要时自行打开。
