# 爆彩童话 v1.1.0.2(九游)

## 包结构

- 路径:`别的游戏/爆彩童话-1.1.0.2-九游/assets`
- 资源包:2,536 个 `.r` + `filelist_1.1.0.2.r`
- filelist:**2163/2163** 校验通过
- 解包:17,716 个文件 / 约 240 MB
- 沙盒产物:`_agent-sandbox-snake261103/bc_*`

详细探测见
[`bc_probe.md`](../../../../_agent-sandbox-snake261103/bc_probe.md)。

## 格式版本

| 类型 | 版本 | 覆盖 |
|---|---|---|
| `.tex` | 0x2717 | 1565/1565 |
| `.ams` | 0x2716 / 0x2715 | 0x2716 全通过;10 个 0x2715 失败 |
| `.ans` | 0x2715 / 0x2713 | 175/175 |
| `.anf` | 0x2715 / 0x2711 / 0x2714 | 1593/1593 |
| `.gui` | 0x271d / 0x271c / 0x271b | 523/523 精确闭合 |
| `.mtr` | 0x2717 / 0x2716 | 11/11 |
| `.eft` | 0x271a / 0x2719 / 0x2718 | 667/667 |

## 产物

```text
bc_unpack/     解包
bc_tex/        1565 张贴图 (PNG/JXR)
bc_ui/         523 张 UI PNG
bc_models/     122 个 role GLB
bc_fxmodels/   219 个 fx GLB
bc_eqp_*.json  male / female / npc 部件表
```

## 备注

- 本作比古域2小得多,但仍有 `.rgn/.skc/.tsk/.sce/.map` 等专属格式未覆盖;
- 2 个 `.tex` 是嵌入式 JXR,当前只抽出 `.jxr`;
- 10 个 0x2715 `.ams` 需要补静态/特效模型 piece 布局。
