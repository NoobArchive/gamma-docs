# 公共容器与基础类型

> 适用于 `.tex` / `.ams` / `.ans` / `.anf` / `.gui` / `.eft` / `.mtr`。
> 旧 `FORMAT-NOTES.md` §0/§3 已合并到这里。

## 1. `CCommonFileHead`

所有 Gamma 资源文件都以同一个头开始(小端):

```
+0x00  u32 type            4CC 类型,如 "tex\0" / "ams\0" / "gui\0"
+0x04  u32 version         版本,如 0x2717 / 0x271b
+0x08  u32 depCount        依赖资源组数
       dep[depCount]:
         u32 tag           依赖类型 4CC:"tex\0" / "ams\0" / "anf\0" / "mtr\0" / "gfx\0"
         u32 nameCount
         nameCount × [u32 charCount + charCount × UTF-16LE 字符]
+..    u32 extraSize       索引/元数据字节数
       extra[extraSize]
       main[...]           剩余全部主数据
```

Python: `gamma_formats.CommonFileHead` 的 `parse()` / `build()`。
`gamma_formats.parse_file_head(path)` 之类的入口都可直接复用。

### 依赖 tag

| tag | 用途 |
|---|---|
| `tex` | 贴图 |
| `ams` | 模型 |
| `ans` | 骨架/动画集 |
| `anf` | 动画帧数据 |
| `mtr` | 材质 |
| `gfx` | 着色器模板(`.inl` / `.gfx`) |

依赖名是**相对资源根**的路径(反斜杠/正斜杠都有),不是文件名列表。
例如:

```
mod/box01.ams -> mtr=[mat/emissive_ccw.mtr],
                 tex=[role/tex/box01_t01.tex, role/tex/box01_t02.tex]
```

## 2. 文本编码

同一目录/包内可能混用:

- UTF-8(可带 BOM `EF BB BF`);
- UTF-16LE(BOM `FF FE`);
- GBK/GB2312(无 BOM)。

`gamma_formats.decode_text(bytes)` 的规则:BOM 优先 → UTF-8 → GBK。
配置表读取用 `read_text_any(path)`。

## 3. 4CC 类型速查

| 4CC | 版本 | 说明 | 文档 |
|---|---|---|---|
| `tex\0` | 0x2717 | 贴图 | [tex.md](tex.md) |
| `ams\0` | 0x2717 | 模型 | [ams.md](ams.md) |
| `ans\0` | 0x2715 | 骨架/绑定 | [ans.md](ans.md) |
| `anf\0` | 0x2715 | 动画帧 | [anf.md](anf.md) |
| `mtr\0` | 0x2716/17 | 材质 | [mtr.md](mtr.md) |
| `gui\0` | 0x271d | UI | [gui.md](gui.md) |
| `eft\0` | 0x271b | 特效 | [eft.md](eft.md) |

`.eqp` 不是标准 `CommonFileHead`(12 字节私有头),见 [male.eqp.md](male.eqp.md)。
`.r` 容器与 filelist 见 [package.md](package.md)。
