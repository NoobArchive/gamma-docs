# Gamma 工具上手指南

> 目标读者:拿到一个已解压的 Gamma APK,想把资源解包、转换、再打包回去。
> 数据基准:`V3-1.3.3`(`Rhythm Painters` 只是单机魔改版,不作为格式依据)。
>
> 2026-11 起后端已迁入 `gammaray-py/src/deltaxutils/`(原 `gamma_formats.py` /
> `gamma_models.py`)。新流程优先用 `gammaray`;下文旧 CLI 命令仍可作为
> DeltaXUtils 直接调用示例。

## 0. 准备

- Python 3.10+;
- 可选:`uv`(自动装 `pillow` / `numpy` / `texture2ddecoder`);
- 工具目录:`Gamma引擎相关/`;
- 下文用两个路径变量:

```powershell
$ROOT = "D:\粉刷匠大作战\2608新的逆向计划！"
$G    = "$ROOT\Gamma引擎相关"
$APK  = "$ROOT\V3-1.3.3\粉刷匠大作战_1.3.3"
```

APK 解压后资源在 `$APK\assets\`,里面有 `filelist_1.3.3.4.7.r`。

### 统一入口(实验)

`gammaray-py/` 提供同一套流程的入口;目前内部复用旧脚本,适合不想记多条命令时使用:

```powershell
cd "$G\gammaray-py"
python gammaray.py unpack "$APK\assets" -o "$ROOT\out\original" `
    --filelist filelist_1.3.3.4.7.r
uv run --with pillow --with texture2ddecoder --with numpy `
    python gammaray.py convert "$ROOT\out\original" -o "$ROOT\out\converted" `
    --only tex,models,ui,mat
# 需要动画时加 --clips(会读取 .ans 的全部 .anf)
python gammaray.py pack "$ROOT\out\packed\one.r" file.bin
```

下面仍按分步命令说明,便于排错和只跑某一类。

## 1. 解包 APK 资源 → `out/original`

```powershell
cd $G
python gamma_formats.py unpack "$APK\assets" "$ROOT\out\original" `
    --filelist filelist_1.3.3.4.7.r
```

- 输出保持资源相对路径,如 `role/mod/p_001.ams`、`gui/cfg/fsj_denglu.gui`;
- 解析失败的包会放进 `out/original/_unsupported/`,清单在 `_unsupported.txt`;
- 先校验一次包完整性:

```powershell
python gamma_formats.py fl-verify "$APK\assets" --filelist filelist_1.3.3.4.7.r
# ok=636 mismatch=0 missing=0
```

常用过滤:

```powershell
# 只解配置
python gamma_formats.py unpack "$APK\assets" "$ROOT\out\original" --only config
# 只解某类路径
python gamma_formats.py unpack "$APK\assets" "$ROOT\out\original" `
    --include "role/mod/*" --include "role/ani/*"
```

## 2. 转换 `out/original` → `out/converted`

**顺序很重要:先贴图,再模型/UI**(模型和 UI 都要引用 PNG)。

### 2.1 贴图(全部 `.tex` → PNG)

```powershell
uv run --with texture2ddecoder --with pillow python gamma_formats.py tex-batch `
    "$ROOT\out\original" "$ROOT\out\converted\tex"
```

输出保持相对目录:`out/converted/tex/role/tex/*.png`、`out/converted/tex/gui/tex/*.png`。
未安装 `texture2ddecoder` 时 ETC 贴图会落成 `.atf`,建议安装。

### 2.2 角色/特效模型 → GLB

```powershell
python gamma_models.py ams2glb `
    "$ROOT\out\original\role\mod" "$ROOT\out\converted\models\role" `
    --tex-root "$ROOT\out\converted\tex" `
    --ani-root "$ROOT\out\original\role\ani"

python gamma_models.py ams2glb `
    "$ROOT\out\original\fx\mod" "$ROOT\out\converted\models\fx" `
    --tex-root "$ROOT\out\converted\tex" `
    --ani-root "$ROOT\out\original\fx\ani"
```

- `--clips` 把 `.ans` 声明的全部 clip 写进同一个 GLB;`--orphan-clips` 才额外包含
  未登记的 `<model>_*.anf`;同时写 `<model>.anims.json`(declared / clips / orphans);
- `--skin N` 切换配色/皮肤变体;
- `--mtr-root` 默认从 AMS 目录向上找 `mat/`,也可显式指定;
- `.mtr` 的 `alphaMode` / `doubleSided` 会自动写进材质。

### 2.3 UI → PNG

```powershell
uv run --with pillow python gamma_formats.py gui-batch `
    "$ROOT\out\original\gui" "$ROOT\out\converted\ui" `
    --tex-root "$ROOT\out\converted\tex" `
    --config "$ROOT\out\original\config"
```

`--config` 用来加载 `config/dictionary/*.txt`(UI 文案)。常用开关:

```powershell
--all-windows       # 同矩形状态备选全部画(默认只画内容最多的一个)
--no-uv-flip        # 关闭负 UV 翻转(对照用)
--no-rotation       # 关闭 f 旋转
--no-wrap-tile      # 关闭 u[2]/u[3] 周期平铺(默认开启,仅调试用)
--hide-window NAME  # 隐藏动态模板窗口(如 jishazhe);
                    # SDK 登录页可加 W_text / B_ok 隐藏 IP 输入框
```

### 2.4 特效 / 材质 / 皮肤表

```powershell
# 单个特效的结构/JSON/预览卡
python gamma_formats.py eft-info "$ROOT\out\original\fx\cfg\benpao_1.eft" --units --json benpao_1.json
uv run --with pillow python gamma_formats.py eft-preview `
    "$ROOT\out\original\fx\cfg\benpao_1.eft" benpao_1.png `
    --tex-root "$ROOT\out\converted\tex"

# 材质
python gamma_formats.py mtr-info "$ROOT\out\original\mat\emissive_ccw.mtr" --json emissive.json

# 皮肤表(equip.txt) + eqp
python gamma_formats.py skins "$ROOT\out\original\config\resource\equip.txt" `
    --config "$ROOT\out\original\config" --json "$ROOT\out\converted\skins.json"
python gamma_formats.py eqp "$ROOT\out\original\role\dress\male.eqp" `
    --root "$ROOT\out\original" --json "$ROOT\out\converted\male_eqp.json"
```

### 2.5 动画表 kc

```powershell
python gamma_models.py anf-table "$ROOT\out\original\role\ani\p_001.ans" --json p_001_anf.json
```

> 这一版 568 个 clip 的 `kc` 全为 0,事件表为空。

## 3. 打包一个文件到 `.r`

最小用法:

```powershell
python gamma_formats.py r-pack "$ROOT\out\packed\my.r" "$ROOT\out\original\config\resource\equip.txt"
```

多个成员(顺序必须和原包 filelist 的成员顺序一致):

```powershell
python gamma_formats.py r-pack out.r f1.bin f2.bin f3.bin
```

### ⚠️ 重要边界

`r-pack` 只是**记录级打包**:

```
[u32 length][data][u32 length][data]...
```

它**不会**:

1. 自动恢复/更新 `filelist` 的成员名与 `col1/col2` 大小;
2. 按 200 KiB 自动分片;
3. 重新打包/签名 APK;
4. 帮你保持原包的成员顺序(多成员包必须自己保证)。

所以:

- 打包**单记录** `.r`(或成员顺序已知的小包)可以直接用;
- 替换游戏内包时,必须同步 filelist 和分片信息,否则客户端不会按预期加载;
- `.r` 本身没有加密,MP3/H.264 的高熵只是媒体压缩。

## 4. 命令速查

| 需求 | 命令 |
|---|---|
| 校验包 | `fl-verify` |
| 解包 | `unpack` |
| 列包/成员 | `fl-list` |
| 单包解包/打包 | `r-unpack` / `r-pack` |
| 贴图 | `tex-info` / `tex-unpack` / `tex-batch` / `tex-pack` |
| 模型 | `gamma_models.py ams2glb` |
| 动画 | `anf-info` / `anf-table` |
| UI | `gui-info` / `gui-render` / `gui-batch` |
| 特效 | `eft-info` / `eft-tex` / `eft-preview` |
| 材质 | `mtr-info` |
| 皮肤 | `eqp` / `skins` |
| 皮肤 3D 页 | `skin_sheet.py` |

## 5. 能力矩阵(工具到底能做什么)

| 输入 | 工具 | 输出 | 状态 |
|---|---|---|---|
| `.tex` | `tex-info` / `tex-unpack` / `tex-batch` | PNG / JXR / ATF / BIN | ✅ 788 PNG + 4 JXR |
| `.tex` | `tex-pack` | `.tex` | ✅ 原始 payload 回包 |
| `.r` + filelist | `unpack` / `fl-verify` / `r-unpack` | 原始成员文件 | ✅ 636/636 |
| 任意文件 | `r-pack` | 记录级 `.r` | ⚠️ 不更新 filelist/不分片 |
| `.ams` | `ams2glb` | GLB(蒙皮+动画+贴图+mtr) | ✅ role 78/78, fx 59/59 |
| `.ans` | `anf-info` / `anf-table` | 骨架/绑定/clip 表 | ✅ |
| `.anf` | `anf-info` / `ams2glb --anim/--clips` | 动画轨道/GLB 动画 | ✅ 542/542 |
| `.gui` | `gui-info` / `gui-render` / `gui-batch` | 窗口树/PNG | ✅ 213/213 |
| `.eft` | `eft-info` / `eft-tex` / `eft-preview` | JSON/贴图表/结构卡 | ✅ 754/754 prop |
| `.mtr` | `mtr-info` | JSON(alphaMode/doubleSided) | ✅ 12/12 |
| `.eqp` / `equip.txt` | `eqp` / `skins` | 部件/皮肤表 JSON/TXT | ✅ |
| `.ani` 光标 | — | — | ⬜ 仅旧笔记入口 |
| `.lua` / `.txt` / `.xml` | — | — | ⬜ 未研究 |
| `.264` | — | — | ⬜ 未转换 |

## 6. 常见问题

| 现象 | 处理 |
|---|---|
| `.tex` 输出 `.atf`/`.bin` | 安装 `texture2ddecoder` 后重跑 `tex-batch` |
| UI 没有文字 | `gui-batch` 要加 `--config out/original/config` |
| 模型没有贴图 | 先跑 `tex-batch`,并确认 `--tex-root out/converted/tex` |
| 模型没有骨骼/动画 | 加 `--ani-root out/original/role/ani`(或 `fx/ani`) |
| `r-pack` 后游戏不加载 | 检查成员顺序、filelist 大小、分片;`r-pack` 不是完整 APK 打包器 |
| 动态 HUD 模板出现在静态图里 | 用 `--hide-window NAME` 隐藏(如 `jishazhe`) |
