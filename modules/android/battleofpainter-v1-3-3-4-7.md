# 粉刷匠大作战 v1.3.3.4.7

最终版本包体,主要研究的游戏项目版本。

## 资源基准

- 全量解包:`_agent-sandbox/gx_dump`
- filelist:`filelist_1.3.3.4.7.r`,**636/636** 校验通过
- 格式/能力矩阵见 [`../../STATUS.md`](../../STATUS.md)

## UI 备注

- `.gui` 全部为 `0x271d`。
- 窗口 `f[1..4]` 是**有符号 int32**;`paihangbang` 等贴边窗口使用 `x=-1`,
  按 u32 读会被当成越界并重置到左上角。
- 文本主色 `b` / 描边 `c` 来自 `SChildImageList`;预览对 `c` 用 1px 八向近似。
