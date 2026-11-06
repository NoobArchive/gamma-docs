# .eqp — 皮肤/装备部件清单

> 《爆彩童话》中得到大量使用，具有 `male.eqp`, `female.eqp`, `npc.eqp`
> 《粉刷匠大作战》中该文件可能只是《爆彩童话》代码重复利用，并且不区分 male/female/npc，并且后续新皮肤实装过程未使用，实际上该文件可能已被弃用。

> **注意: 粉刷匠项目中 `.eqp` 里内嵌的 29 个名字不是正式皮肤名**,更像开发期备注/预留文案。
> 正式名字请看 `config/resource/equip.txt`(`dName` 字典键 + `szMark` 字面名)
> 和 `config/dictionary/*.txt`,用 `python gamma_formats.py skins ...` 导出。

记录 4CC 不是标准 `CCommonFileHead`,而是**名称列表**。实测 粉刷匠大作战 v1.3.3.4.7 `role/dress/male.eqp`(4865 字节):

```
+0x00  12 字节头: FF FF FF FF | u32(8) | u32(4)
+0x0C  "ams\0"
+0x10  u32 count            (male = 29)
       重复 count 次:
           u32 charCount
           UTF16LE name
```

`male.eqp` 解析结果(29 项):

```
role/mod/p_025.ams, role/mod/p_027.ams, ... , role/mod/obj_001.ams
```

## 皮肤名表(后半段)

`.ams` / `.ans` 依赖列表之后还有一个 **29 项的皮肤表**:

```
u32 count (=29)
重复 count 次:
    u32 nameLen + UTF-16   皮肤名(如 "二哈"、"兔女仆"、"厨子张飞")
    u32 classLen + UTF-16  部件类名("body" / "obj")
    ~59 字节元数据(首字段是 0,1,2… 序号,含 1.0f 缩放等)
```

条目长度随名字长度变化(实测 79/80 字节),所以按「名字 + 类名」模式扫描,不依赖固定 stride。

实测 29 个皮肤(与 29 个部件模型按顺序 1:1 对应):

```
 0 二哈        1 僵尸        2 兔女仆      3 冲浪小子    4 厨子张飞
 5 大法师      6 女粉刷匠    7 孙悟空      8 小黄人灯泡  9 弹簧鸡
10 忍者       11 恐龙       12 柴犬       13 死神       14 熊猫
15 盒子猫     16 章鱼小八   17 篮球小子   18 粉刷匠     19 粉刷匠_黄色
20 绿帽侠     21 羊驼       22 蜘蛛侠     23 雪人       24 音波小子
25 食人花     26 龙宝宝     27 经验盒子(obj)           28 金币(obj)
```

配对已被画面验证(`p_001`=粉刷匠、`p_001_02`=粉刷匠_黄色、`jingyanhezi`=经验盒子、
`obj_001`=金币)。完整名单见 `python gamma_formats.py eqp` 或 JSON 的 `skins` 数组。

## 用途

`.eqp` 是**皮肤/角色装备的部件索引**:列出一组 `.ams` 模型,游戏按皮肤把部件组合成角色。
与 `ams` 的依赖合并即可得到完整皮肤链路:

```
male.eqp
  -> role/mod/p_001.ams
       -> mtr=[mat/emissive_ccw.mtr] -> gfx/skeletal.inl
       -> tex=[role/tex/p_001_t01.tex, ...]
```

工具:

```powershell
# 直接列出名字(可同时写 JSON / TXT;--root 用来检查文件是否存在)
python gamma_formats.py eqp role/dress/male.eqp ^
    --json male_eqp_names.json --txt male_eqp_names.txt --root <解包根目录>

# 全量依赖图(.eqp + .ams)
python gamma_formats.py deps <role_dir> role_deps.json
```

`eqp` 输出示例(29/29 全部定位到文件):

```
  0  role/mod/p_025.ams  [ok]
  ...
 27  role/mod/jingyanhezi.ams  [ok]
 28  role/mod/obj_001.ams  [ok]
```

`role_deps.json` 里也已经包含这份名单:

```json
{"file": "dress\\male.eqp", "type": "eqp",
 "deps": [{"tag": "ams", "names": ["role/mod/p_025.ams", "..."]}]}
```

