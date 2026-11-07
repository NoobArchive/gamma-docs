# 工具架构与 `gammaray-py` 设计

> 这份文档回答三个问题:
> 1. 现有工具的实际能力/边界;
> 2. 为什么“一条命令解包→转换→打包”目前不存在;
> 3. `gammaray-py` 应该长什么样、怎么迁移而不破坏旧脚本。

## 1. 现状盘点

| 文件 | 体量 | 内容 |
|---|---:|---|
| `gammaray-py/src/deltaxutils/gamma_formats.py` | ~112 KB | `CCommonFileHead`、`.tex`、`.r`/filelist、`.eqp`、`.gui`、`.eft`、`.mtr`、全部底层 CLI |
| `gammaray-py/src/deltaxutils/gamma_models.py` | ~44 KB | `.ams`/`.ans`/`.anf` 解析、GLB 写出、模型 CLI |
| `gammaray-py/src/deltaxutils/rpack.py` | 小 | 目录 → `.r` + `filelist`(200 KiB 分组/分片) |
| `gammaray-py/src/deltaxutils/texbuild.py` | 小 | PNG → raw BGRA `.tex` |
| `gammaray-py/src/deltaxutils/skin_sheet.py` | ~12 KB | GLB 软件光栅化、皮肤缩略图/皮肤页 |
| `gammaray-py/src/gammaray/` | 前端 | `unpack` / `convert` / `prepare` / `pack` / `info` |

### 已能做的事

```
解包      unpack / fl-verify / r-unpack / fl-list
封包      gammaray pack（目录→.r + filelist，200KiB 分组/分片）
预处理    gammaray prepare（PNG → raw BGRA .tex）
贴图      tex-info / tex-unpack / tex-batch / tex-pack
模型      ams2glb(蒙皮 + 动画 + 贴图 + mtr 材质)
骨架/动画  anf-info / anf-table / ans 绑定
UI        gui-info / gui-render / gui-batch
特效      eft-info / eft-tex / eft-preview
材质      mtr-info
皮肤      eqp / skins / skin_sheet
```

### 结构性问题

1. **转换入口已统一**:
   `gammaray convert` 按 tex → models/ui → fx/mat 的固定顺序调用,
   不再需要逐条记 `tex-batch` / `ams2glb` / `gui-batch` 的参数。
2. **两套 CLI 风格**:
   `gamma_formats.py` / `gamma_models.py` 仍保留底层命令;
   GammaRay 只做参数组织和错误处理,底层公共逻辑仍有重复。
3. **回打包器已补第一版**:
   `gammaray pack` / `rpack.py` 能按目录生成 `.r` + filelist,
   做 200 KiB 分组/分片;仍不生成服务器 `.z`、不重签 APK。
4. **文档漂移**:
   `FORMAT-NOTES.md` 原先与 `docs/` 脱节,数字/命令已旧;现改为迁移指针,
   `.tex`、`.r/filelist`、公共头三份正式文档已补到 `docs/formats/`。
5. **路径约定散落**:
   `--tex-root`、`--ani-root`、`--config`、`--mtr-root` 的默认推导各写各的。

## 2. “解包 → 转换 → 打包”的真实链路

```
APK/assets
  └─ unpack ───────────────► out/original
                                 │
                                 ├─ convert ─────► out/converted/{tex,models,ui,mat,fx}
                                 │
                                 └─ prepare ─────► out/datapre (PNG → .tex)
                                                       │
                                                       └─ pack ─► out/datarel
                                                                    *.r + filelist_<ver>.r
```

**关键顺序**:`unpack` →(`convert` 先 tex,再 models/ui)→ `prepare` → `pack`。

**打包边界**:`gammaray pack` 已能生成 `.r` + filelist 并做 200 KiB 分组/分片;
仍不生成服务器 `.z`、不重签名 APK。官方 filelist 的分组不是单纯的
200 KiB 贪心(中位包约 65 KiB),因此重打包的**成员内容一致、包划分不同**;
详见 [`formats/rpack.md`](formats/rpack.md)。

## 3. 目标

当前决定继续维护 DeltaXUtils，GammaRay 仅作为高级前端

我们希望优化 DeltaXUtils 基础 CLI 使用体验和 GammaRay 前端在执行批量转换操作时与 DeltaXUtils 交互的参数组织方式，让 GammaRay 学会组织参数来让解包生活变得更容易。

目标是我们可以直接把 GammaRay 甩给新用户而不用苦揪 DeltaXUtils 当前令人抓狂的命令参数布局。

**并不是要彻底弃用 DeltaXUtils （gamma_formats.py等）**,我最初的不准确表述让 AI Agent 认为我们要把 DeltaXUtils 彻底干掉。

包名用 `gammaray`(目录名 `gammaray-py` 只是仓库目录,不能作为 Python 包名)。

APK 重签名暂时不用实现

## 4. 统一 CLI 契约

```powershell
# 解包:目录或 APK;自动找 filelist;可 -c 顺带转换
gammaray unpack <目录或.apk> [-o out/original] [--filelist ...] [-c [out/converted]]

# 转换:默认 src=out/original, out=out/converted
gammaray convert [out/original] [-o out/converted] [--only tex,models,ui,mat,fx]

# 预处理:PNG → raw BGRA .tex
gammaray prepare <目录> [-o out/datapre-时间戳] [--tex-format bgra]

# 目录封包 + filelist(200KiB 分组/分片)
gammaray pack <目录> [-o out/datarel-时间戳] [--version 1.3.3.4.7] [--limit 204800]
```

### `convert` 的输出约定

```
out/converted/
  tex/      保持原相对路径,如 role/tex/p_001_t01.png
  models/   role/mod/*.glb, fx/mod/*.glb
  ui/       gui/**/*.png
  fx/       cfg/*.json
  mat/      *.json
```

`convert` 内部顺序固定:tex → models/ui → fx/mat。
`--only` 控制子集;当前串行执行,`--jobs` 尚未实现。

## 5. 迁移计划(不破坏旧脚本)

1. **补文档**（已做）:
   `docs/formats/common.md`、`tex.md`、`package.md`;
   `FORMAT-NOTES.md` 改为迁移指针。
2. **建 `gammaray` 前端 + DeltaXUtils**（已做）:
   `gammaray-py/src/deltaxutils/` 收纳原 `gamma_formats.py` / `gamma_models.py`
   与 `rpack.py` / `texbuild.py` / 皮肤工具;`gammaray/legacy.py` 直接导入 `deltaxutils`,
   前端提供 `unpack` / `convert` / `prepare` / `pack` / `info` 工作流。
3. **逐步拆 DeltaXUtils 模块**:
   已拆出 `rpack.py`(目录封包/filelist)与 `texbuild.py`(PNG→.tex);
   `common/package/tex` 与 `ams/ans/anf/glb` 仍在大文件里,后续再拆。
4. **旧 CLI 兼容**:
   `gamma_formats.py` / `gamma_models.py` 保留全部旧命令,互不转调;
   GammaRay 只组合参数,底层行为仍可直接用旧命令复现。
5. **测试/校验**:
   - `fl-verify` 636/636;
   - `.eft` 754/754 prop、`.mtr` 12/12;
   - `.gui` 213/213;
   - `ams2glb` role 78/78、fx 59/59;
   - `pack` 重打 gx_dump:成员 2515/2515、内容差异 0;
   - `prepare` PNG→TEX→PNG 像素差异 0;
   - `unpack` 目录 / APK / `-c` 均实测通过。

## 6. 本轮的落地程度

- 文档:已补齐 `GETTING-STARTED.md`、`ARCHITECTURE.md`、`DOCS-AUDIT.md`、
  `common.md`、`tex.md`、`package.md`、`rpack.md`、`legacy-notes.md`;
- `FORMAT-NOTES.md`:改为“已迁移”指针(内容进入 docs/formats);
- `gammaray-py`:已实现 `unpack`(目录/APK/`-c`)、`convert`、`prepare`、
  `pack`(目录+filelist)、`pack-records`、`info`;
- 后端已迁入 `src/deltaxutils/`,新增 `rpack.py` / `texbuild.py`。

> 不应把“所有格式都搬完”作为第一版目标;
> 第一版的价值是**一个入口 + 一条可重复的转换管线**。
