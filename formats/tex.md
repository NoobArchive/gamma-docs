# `.tex` — Gamma 贴图

版本 `0x2717`,公共头见 [common.md](common.md)。当前工具可解全部 **792 个 `.tex`**:
其中 **788 个转成 PNG**,`etc/loading_1..4.tex` 是 JPEG-XR,只能抽出 `.jxr`
(需要 jxrlib/texconv 再转 PNG)。

## 1. 索引块(extra,36 字节 = 9 × u32)

```
[0] format        ETextureFormat 枚举
[1] storage       0 = 原始像素/块数据; 4 = ATF 容器
[2] widthPow2     2 的幂宽度
[3] heightPow2
[4] cube/depth    通常 0
[5] mipCount
[6] width         实际宽度
[7] height        实际高度
[8] reserved      0
```

实测:

| 文件 | 索引块 | 含义 |
|---|---|---|
| `etc/loading_0.tex` | `[1,4,256,256,0,1,256,256,0]` | 8bpp 压缩, ATF, 256×256, 1 级 |
| `etc/gui/tex/bc_jieya.tex` | `[3,0,512,512,0,10,512,512,0]` | 32bpp RGBA, 原始, 512×512, 10 级 mip |

## 2. 主数据

- `storage==4`:ATF 容器,payload 以 `ATF` 开头。Android 包内基本是该平台的
  ETC1/ETC2 块:
  - ATF5(8bpp)= **ETC1 颜色 + ETC1 alpha**,按 `w × 2h` 排布(上半颜色、下半 alpha);
  - ATF3(4bpp)= **ETC1/ETC2 RGB**,`w × h`。
  - 解码后是 BGRA,需要交换 R/B。
  - `etc/loading_0.tex` 是唯一的 DXT5 变体。
- `storage==0`:原始像素/块数据。`version>0x2711` 时**每级 mip 前有 `u32` 长度前缀**,
  解码时要跳过 4 字节,否则图像会左移 1px。
- `format∈{0,1,6,7,13,14}` 是压缩家族;其余按 bpp 表处理。

### `ETextureFormat` bpp 表

```
idx:  0 1 2  3  4  5  6 7 8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28
bpp:  4 8 24 32 16 16  4 8 24 32 16 16 16  4  8  8 16 16 32 32 32 32 64 64 128 0 16 32 32
```

## 3. ATF 容器里的 `format`(不要和 `.tex` 的 `ETextureFormat` 混淆)

Gamma 的 ATF 头(10 字节):

```
+0x00  "ATF"
+0x03  u24 big-endian bodyLen
+0x06  u8  format
+0x07  u8  log2(width)
+0x08  u8  log2(height)
+0x09  u8  levelCount
+0x0A  body[...]
```

ATF 头里的 `format` 描述的是 **ATF body 的编码**:

| ATF `format` | body 里实际是什么 | 工具行为 |
|---:|---|---|
| 3 | 块压缩颜色(DXT1/ETC1/ETC2) | **解 PNG** |
| 5 | 块压缩颜色 + alpha(DXT5/ETC2A8) | **解 PNG** |
| 0 / 1 | **JPEG-XR codestream**(不是原始 RGBA,也不是 DXT) | 只能抽 `.jxr`;需要 jxrlib/texconv 再转 PNG |
| 2 / 4 / 0x0c / 0x0d | LZMA 包装的块/JXR | 抽 `.jxr` 或写 `.atf` |

也就是说:

- **ETC2A8 / ETC2 / DXT5 / RAW 都能直接转 PNG**;
- `loading_1..4.tex` 是唯一 4 个 **body 真的是 JPEG-XR** 的贴图,所以工具输出 `.jxr`;
- `format=0/1` 这一行不是“把 raw RGBA 转成 JXR”,而是“文件里本来就是 JXR,工具只能原样抽出来”。

块数据位置用 `u24 == level0 期望字节数` 判断:

- `+16` 匹配 → 平台块(Android=ETC2),数据在 `+19`;
- `+10` 匹配 → 旧 DXT 变体,数据在 `+13`。

## 4. 工具

```powershell
python gamma_formats.py tex-info   a.tex
python gamma_formats.py tex-unpack a.tex outdir        # .png / .jxr / .atf / .bin
python gamma_formats.py tex-batch  <src_dir> <out_dir> # 递归全部 .tex
python gamma_formats.py tex-pack   payload.bin out.tex --format 3 --width 512 --height 512 --mips 10
```

- ETC2 解码依赖 `texture2ddecoder`:
  `uv run --with texture2ddecoder python gamma_formats.py tex-batch ...`;
  未安装时 ATF 会落成 `.atf`(原始/DXT/JPEG-XR 仍可纯标准库处理)。
- `tex-unpack` 输出 `.meta.json`(宽高/格式/mip),便于回打包。
- `tex-pack` 需要已经解码好的原始 payload;不会从 PNG 自动压缩成 ETC2。
