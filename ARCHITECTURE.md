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
| `gammaray-py/src/deltaxutils/skin_sheet.py` | ~12 KB | GLB 软件光栅化、皮肤缩略图/皮肤页 |
| `gammaray-py/src/gammaray/` | 前端 | `unpack` / `convert` / `pack` / `info` 统一工作流 |

### 已能做的事

```
解包      unpack / fl-verify / r-unpack / fl-list
贴图      tex-info / tex-unpack / tex-batch / tex-pack
模型      ams2glb(蒙皮 + 动画 + 贴图 + mtr 材质)
骨架/动画  anf-info / anf-table / ans 绑定
UI        gui-info / gui-render / gui-batch
特效      eft-info / eft-tex / eft-preview
材质      mtr-info
皮肤      eqp / skins / skin_sheet
```

### 结构性问题

1. **没有统一的“转换”入口**:
   `out/original` 是混合目录(`.tex/.ams/.gui/.eft/.mtr/.eqp/...`),
   但转换要分别记 `tex-batch`、`ams2glb`、`gui-batch`、`eft-*`、`mtr-info` 的参数和顺序。
2. **两套 CLI 风格**:
   `gamma_formats.py` 和 `gamma_models.py` 各有一个 `main()`;
   公共逻辑(路径、依赖、错误处理)重复。
3. **`r-pack` 不是“回打包器”**:
   只写记录,不维护 filelist、不分片、不签名;多成员包顺序需手工保证。
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
                                 ├─ tex-batch ──► out/converted/tex
                                 ├─ ams2glb ───► out/converted/models
                                 ├─ gui-batch ─► out/converted/ui
                                 ├─ eft-* ─────► out/converted/fx
                                 └─ mtr-info ──► out/converted/mat
                                 │
                                 └─ r-pack ────► out/packed/*.r
```

**关键顺序**:先 `tex`,再 `models` / `ui`(后两者依赖 PNG)。

**打包边界**:`r-pack` 只负责记录布局;要替换游戏包,还需要
filelist 成员表 + 200 KiB 分片 + APK 重签名。这一层目前没有实现,
应在 `gammaray pack` 里拆成显式的 `--single` / `--replace` 两种模式,
而不是假装 `r-pack` 能一键回包。

## 3. 目标布局 - 旧设

当前决定继续维护 DeltaXUtils，GammaRay 仅作为高级前端

我们希望优化 DeltaXUtils 基础 CLI 使用体验和 GammaRay 前端在执行批量转换操作时与 DeltaXUtils 交互的参数组织方式，让 GammaRay 学会组织参数来让解包生活变得更容易。

目标是我们可以直接把 GammaRay 甩给新用户而不用苦揪 DeltaXUtils 当前令人抓狂的命令参数布局。

**并不是要彻底弃用 DeltaXUtils （gamma_formats.py等）**,我最初的不准确表述让 AI Agent 认为我们要把 DeltaXUtils 彻底干掉。

包名用 `gammaray`(目录名 `gammaray-py` 只是仓库目录,不能作为 Python 包名)。

APK 重签名暂时不用实现

## 4. 统一 CLI 契约

```powershell
# 解包
gammaray unpack <assets> -o out/original [--filelist ...] [--only config]

# 批量转换(核心新增)
gammaray convert <out/original> -o out/converted `
    [--only tex,models,ui,fx,mat] `
    [--texture-tool uv] [--jobs N]

# 打包
gammaray pack out/packed/one.r file.bin
gammaray pack out/packed/pkg.r --members f1 f2 f3   # 顺序敏感
# 替换包(未来)
gammaray pack --replace original.r --members ... --filelist ...
```

### `convert` 的输出约定

```
out/converted/
  tex/      保持原相对路径,如 role/tex/p_001_t01.png
  models/   role/mod/*.glb, fx/mod/*.glb
  ui/       gui/**/*.png
  fx/       cfg/*.png + *.json
  mat/      *.json
  skins.json / male_eqp.json
```

`convert` 内部顺序固定:tex → models/ui → fx/mat。
`--only` 控制子集;`--jobs` 可并行(先保证串行正确)。

## 5. 迁移计划(不破坏旧脚本)

1. **补文档**（已做）:
   `docs/formats/common.md`、`tex.md`、`package.md`;
   `FORMAT-NOTES.md` 改为迁移指针。
2. **建 `gammaray` 前端 + DeltaXUtils**（已做第一版）:
   `gammaray-py/src/deltaxutils/` 收纳原 `gamma_formats.py` / `gamma_models.py`
   与皮肤/打包工具;`gammaray/legacy.py` 直接导入 `deltaxutils`,
   前端只保留 `unpack` / `convert` / `pack` / `info` 工作流。
3. **逐步拆 DeltaXUtils 模块**:
   先拆 `common/package/tex`(依赖少),再拆 `ams/ans/anf/glb`,最后 `gui/eft/mtr`;
   前端不直接依赖大文件内部实现。
4. **旧 CLI 兼容**:
   `gamma_formats.py` / `gamma_models.py` 保留同名命令,内部转调 `gammaray.cli`,
   并在 `--help` 里提示新入口。
5. **测试/校验**:
   - `fl-verify` 636/636;
   - `.eft` 754/754 prop、`.mtr` 12/12;
   - `.gui` 213/213;
   - `ams2glb` role 78/78、fx 59/59;
   - 新增 `convert` 的 smoke test(小目录)。

## 6. 本轮的落地程度

- 文档:已补齐 `GETTING-STARTED.md`、`ARCHITECTURE.md`、`DOCS-AUDIT.md`、
  `common.md`、`tex.md`、`package.md`、`legacy-notes.md`;
- `FORMAT-NOTES.md`:改为“已迁移”指针(内容进入 docs/formats);
- `gammaray-py`:已实现可运行的 `unpack` / `convert` / `pack` / `info`;
  后端已迁入 `src/deltaxutils/`,`convert` 已实测 tex / models / ui / mat/fx。

> 不应把“所有格式都搬完”作为第一版目标;
> 第一版的价值是**一个入口 + 一条可重复的转换管线**。
