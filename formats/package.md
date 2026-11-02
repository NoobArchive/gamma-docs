# `.r` 资源包 / filelist

## 1. 包结构

`Gamma::CPackage::OnLoaded`(@`0x26fd3c`):

```
record*:
  u32 length
  u8  data[length]
```

连续记录、**不加密、不压缩**。`CPackage::GetFileBuffer(i)` 返回
`buf + offset + 4`,长度 `u32(offset)`。每条记录 = 一个成员文件的原始字节。

`gamma_formats.parse_package(data)` / `build_package(blobs)` 是解析/写出端。

## 2. filelist

filelist 本体也是一个 `.r`(单记录),内容是 **TSV**(`CTabFile`,`\r\n` 分行、`\t` 分列):

```
<name> <TAB> <col1> <TAB> <col2> \r\n
```

- `col1 >= 0`:包分片行,通常名为 `<dir>/<md5>.z`。
  `col1` = 分片本地字节数,`col2` = 网络压缩字节数。
- `col1` 为空/负数:成员行,紧跟在一段连续分片行之后,是该逻辑包内的文件
  (按出现顺序对应拼接头内第 0、1… 条记录)。

本地文件名 = 包名结尾 `.z` → `.r`;分片从 APK 的 `assets/<name>` 读取。

## 3. 分组与分段(重要)

**连续多个分片行属于同一个逻辑包**,分片数可以 >1;成员行只在其所属包的全部分片后出现。
`CPackage::OnLoaded` 会把所有 part 按顺序拼接,再按 `[u32 len][data]` 解析。
所以工具必须先拼接再解析。

- 每片磁盘存储 ≤ **0xC8000 = 204800 字节(200 KiB)**。
- **首片** = 普通 `.r`,以 `[u32 总长][data...]` 开头;本片只含前 204796 字节。
- **后续片** = 无文件头的裸数据,按顺序接在上一片之后。
- 拼接后长度 == 首片声明的总长。

实例:

```
music/0a53430c….z  204800  首片,头 u32 = 256487
music/2f3741d3.z    51691  无头续片
成员: fenshuajianglianxi.mp3
204796 + 51691 = 256487 ✓
```

## 4. 工具与边界

```powershell
python gamma_formats.py fl-list   assets/filelist_1.3.3.4.7.r
python gamma_formats.py fl-verify <assets_dir> --filelist filelist_1.3.3.4.7.r
python gamma_formats.py unpack    <assets_dir> out/original --filelist filelist_1.3.3.4.7.r
python gamma_formats.py r-unpack  a.r b.r outdir [--names n1 n2]
python gamma_formats.py r-pack    out.r file1 [file2 ...]
python gamma_formats.py r-types   a.r
```

当前全量验证:`ok=636 mismatch=0 missing=0`。

### `r-pack` 能做什么、不能做什么

`r-pack` 只是**记录级打包器**:把多个文件按 `[u32 len][data]` 写成一个 `.r`。

它**不会**自动:

- 恢复 filelist 的成员名/顺序;
- 更新 filelist 的 `col1/col2`(大小);
- 做 200 KiB 分片;
- 重新签名 APK。

因此:

- 打包一个**单记录** `.r`(或成员顺序已知的多记录包)可以直接用;
- 要替换游戏内包,必须保持原包的**成员顺序**,并在必要时同步修改 filelist 与分片;
- 不要期待 `r-pack` 生成一个可直接替换 APK 的完整包。

`.r` 记录本身没有加密;高熵内容(MP3/H.264)只是媒体压缩,不是加密。
