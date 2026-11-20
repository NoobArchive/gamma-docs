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
  `col1` = **该分片 `.r` 的实际字节数**(不是 payload 大小);
  `col2` = 网络压缩后的 `.z` 字节数。
  单分片时 `col1 = 成员 payload 总和 + 4 × 成员数`。
- `col1` 为空/负数:成员行,紧跟在一段连续分片行之后,是该逻辑包内的文件
  (按出现顺序对应拼接头内第 0、1… 条记录)。

> 只替换 `.r` 而不更新 `col1` 时,客户端会打印
> `CFileReader::CheckLocalBuffer: read error size!!!`,
> 但仍能按 `.r` 自己的记录头加载。实测见
> [`r-warn-probe/REPORT.md`](../../../_agent-sandbox/r-warn-probe/REPORT.md)。

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
# 推荐:目录封包 + 自动生成 filelist(200 KiB 分组/分片)
python gammaray.py pack <目录> -o out/datarel --version 1.3.3.4.7

python gamma_formats.py fl-list   assets/filelist_1.3.3.4.7.r
python gamma_formats.py fl-verify <assets_dir> --filelist filelist_1.3.3.4.7.r
python gamma_formats.py unpack    <assets_dir> out/original --filelist filelist_1.3.3.4.7.r
python gamma_formats.py r-unpack  a.r b.r outdir [--names n1 n2]
python gamma_formats.py r-pack    out.r file1 [file2 ...]
python gamma_formats.py r-types   a.r
```

当前全量验证:`ok=636 mismatch=0 missing=0 size_mismatch=0`。
`fl-verify` 现在同时校验每个分片 `.r` 的实际长度与 filelist `col1`;
只替换 `.r` 不更新 filelist 时会显示 `size_mismatch`。
目录封包的规则、与官方 assets 的对比见
[`rpack.md`](rpack.md)。

### `pack` / `r-pack` 能做什么、不能做什么

`gammaray pack` 会按目录生成完整的 `.r` + `filelist_<version>.r`:
同目录累计 ≤200 KiB 合成一个逻辑包,大文件按 200 KiB 分片;
根目录 `etc/`、`version.inf`、`filelist_*.r` 被排除。

它**不会**:

- 生成服务器下载用的 `.z`(`csize` 只是 zlib 估算);
- 重打包/签名 APK;
- 复刻官方服务器打包器的分组(官方中位包约 65 KiB,
  不是单纯的 200 KiB 贪心);
- 恢复官方包名(用内容+成员列表生成 MD5 名)。

`r-pack` 仍然只是**记录级打包器**:把多个文件按 `[u32 len][data]`
写成一个 `.r`,不写 filelist、不分片。

`.r` 记录本身没有加密;高熵内容(MP3/H.264)只是媒体压缩,不是加密。

## 5. Android 热更新缓存

客户端的热更新缓存目录(示例
`.../files/cache`)把下载的 `.r` 以 **percent-encoded URL** 命名:

```text
http%A3%F2%F2res.fsj.mqgamer.com%F2fenshuajiang%F2data%F2config%F2<hash>.r
```

- `%F2` 是 GBK 全角斜杠 `／`(A3F2)被拆开后的残留,实际用作 `/`;
- 逻辑路径取 `/data/` 之后的部分,例如 `config/<hash>.r`;
- 缓存根里通常还有 `filelist_<ver>.r`、`media.inf` 等。

`gammaray unpack <cache>` 会自动识别这种目录,展开到临时 assets 视图后
按 filelist 解包;`--base <APK assets>` 可先铺基础资源再叠加缓存。
缓存可能只包含热更新文件,缺少的包会写进 `_unsupported.txt`。
