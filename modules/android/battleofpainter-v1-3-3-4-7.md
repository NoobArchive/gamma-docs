# 粉刷匠大作战 v1.3.3.4.7

最终版本包体,主要研究的游戏项目版本。

## 资源基准

- 全量解包:`_agent-sandbox/gx_dump`
- filelist:`filelist_1.3.3.4.7.r`,**636/636** 校验通过
- 格式/能力矩阵见 [`../../STATUS.md`](../../STATUS.md)

## 皮肤表与 `.eqp`

- 正式皮肤表是 `config/resource/equip.txt`(34 行)+ `fusion.txt`(18 行),
  字段为 `szModel` / `szAni` / `nMatrlIndex0..9` / `szFx0..9` / `nScale`;
- `role/dress/male.eqp` 仍随包发布(4,861B,29 条),但:
  - `libShellClient.so` 没有 `.eqp` / `DressingRoom` / `EquipmentGroup` /
    `role/dress/` 字符串;
  - `misc/**/*.lua` 没有 `eqp` / `CDressingRoom` 引用;
  - 它的 29 个名字是旧开发名(二哈、僵尸、兔女仆……),与 `equip.txt`
    的正式名(哈士奇、蹦迪僵尸、兔酱……)不一致,也不含后期融合皮肤。
- 结论:**该文件在本作是爆彩时代的遗留数据,运行时未使用**;
  1.1.20 与 1.3.3 的 `male.eqp` 哈希完全相同,后续新皮肤只更新
  `equip.txt` / `fusion.txt` / `role/mod/*.ams`。
  详见 [`../../formats/eqp.md`](../../formats/eqp.md)。

## UI 备注

- `.gui` 全部为 `0x271d`。
- 窗口 `f[1..4]` 是**有符号 int32**;`paihangbang` 等贴边窗口使用 `x=-1`,
  按 u32 读会被当成越界并重置到左上角。
- 文本主色 `b` / 描边 `c` 来自 `SChildImageList`;预览对 `c` 用 1px 八向近似。
