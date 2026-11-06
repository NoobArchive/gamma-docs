# `.gmz` — 更新增量包

> 位置:`D:\game\guyu2\patch\gycs-<from>-<to>.gmz`。
> 状态:**格式已解**,74/74 个样本全部精确解析。

## 1. 总体布局

```
+0x00  Header (0x1C)
+0x1C  LZMA props (propsSize, 本作恒为 5)
+..    raw LZMA1 stream (compressedSize)
```

| 偏移 | 类型 | 字段 | 说明 |
|---:|---|---|---|
| 0x00 | `char[4]` | magic | `"gmz\0"` (`67 6D 7A 00`) |
| 0x04 | `u32` | version | 本作恒为 `0x2711` |
| 0x08 | `u32` | dirCount | 目录表项数,本作恒为 7 |
| 0x0C | `u32` | fileCount | 被更新文件数 |
| 0x10 | `u32` | uncompressedSize | 解压后 manifest 字节数 |
| 0x14 | `u32` | compressedSize | raw LZMA1 流字节数(不含 props) |
| 0x18 | `u32` | propsSize | LZMA props 长度,本作恒为 5 |

校验:

```
fileSize == 0x1C + propsSize + compressedSize
```

74 个样本全部满足。

## 2. 压缩

`props[5]` 是 LZMA1 的 5 字节属性:

```
byte0       = (pb * 5 + lp) * 9 + lc
byte1..4    = dictionary size (LE)
```

本作所有样本都是:

```
5D 00 00 00 04
```

即 `lc=3, lp=0, pb=2, dict=0x04000000`(64 MiB)。

`compressedSize` 字节是**裸 LZMA1 流**(没有 `.lzma` 的 13 字节头,
也没有可靠的 end marker;必须用 `uncompressedSize` 限长解压)。

Python:

```python
import lzma
filt = {'id': lzma.FILTER_LZMA1, 'dict_size': 0x04000000,
        'lc': 3, 'lp': 0, 'pb': 2}
out = lzma.LZMADecompressor(format=lzma.FORMAT_RAW, filters=[filt]) \
          .decompress(stream, max_length=uncompressed_size)
```

## 3. 解压后的 manifest

### 3.1 目录表

先读 `dirCount` 个:

```
u32 charCount
charCount × UTF-16LE 字符
```

本作 74 个样本的目录表完全相同:

```
bin
bin/release
data
etc
log
patch
usr
```

这些路径相对**游戏根目录**;`Patcher.exe` 自身在 `bin/release`,
所以代码里会拼 `"../../"` 回到根。

### 3.2 文件表

随后读 `fileCount` 条文件记录:

```
u32 pathLen
u32 blockCount
u32 newFileSize
pathLen × UTF-16LE 字符          // 目标路径,如 data/config.pkg
blockCount × {
    u32 size
    u32 offset
    u8  data[size]
}
```

即:`blockCount` 是**二进制写入块数**;每块把 `data` 写到目标文件的
`offset` 处。`newFileSize` 是补丁完成后的文件大小。

### 3.3 应用语义

`sub_406757`(Patcher.exe @ `0x406757`):

1. 如果目标文件当前大小 != `newFileSize`,先 truncate/expand 到 `newFileSize`;
2. 对每个 block:`SetFilePointer(offset)` → `WriteFile(data, size)`;
3. `blockCount == 0` 时只做 resize,不写块(可用于截断/占位)。

`version.inf` 在 `sub_407627` 里有单独分支,但最终仍按同一套
`newFileSize + blocks` 写入。

## 4. 样本统计

74 个 `.gmz` 全部解析通过:

- 208 个目标文件记录;
- 136,890 个 block;
- 所有样本 `dirCount=7`、`props=5D 00 00 00 04`、version `0x2711`;
- 所有 `fileSize == 0x1C + 5 + compressedSize`。

目标文件汇总:

| 目标 | 补丁数 | 最终大小 |
|---|---:|---:|
| `data/config.pkg` | 72 | 88,527,989 |
| `data/role.pkg` | 3 | 178,303,293 |
| `data/misc.pkg` | 5 | 4,226,034 |
| `data/map.pkg` | 1 | 495,218,746 |
| `data/fx.pkg` | 1 | 70,897,443 |
| `bin/release/ShellClient.exe` | 4 | 7,458,816 |
| `bin/release/PatcherRes.dll` | 4 | 6,647,808 |
| 其余 Gamma DLL / Patcher / BugCommit / Launch.exe | 各 4 | — |
| `version.inf` | 74 | 26/28/29 |

`gui.pkg`、`font.pkg`、`mat.pkg`、`shader.pkg` 等没有出现在补丁目标里,
说明它们从基础包开始就没有被这条补丁链改过。

## 5. 与 `.pkg` 的关系

`.gmz` 不是 `.pkg` 容器,而是**按字节范围修改目标文件的补丁**:

- 目标可以是 `data/*.pkg`,也可以是 exe/dll/version.inf;
- block 数据是**目标文件的原始存储字节**:若目标是 version 1 的 `.pkg`,
  块内仍是按绝对偏移 XOR 后的数据,不是解密后的资源;
- `.pkg` 的内部索引、XOR、dead 表都由补丁字节直接改写;
- 因此 `.gmz` 里的 block 数据可能包含 `.pkg` 的索引项、资源头、依赖名等
  **原始字节**;扫描这些 block 可以补充 `.pkg` 的文件名线索
  (本轮用 `.gmz` block 把可读名从 12,873 提到 14,355,`shader.pkg` 5/5 全解)。

### `misc.pkg` 60 字节异常的来源

`gycs-0.9.12.20-0.9.12.21.gmz` 对 `data/misc.pkg` 的记录:

```
newFileSize = 4,226,034
block[0]: size=1,     offset=0x00000008
block[1]: size=3391,  offset=0x00406E76   // indexOffset + 429*20, dead 表起点
```

`0x406E76 + 3391 = 4,225,973`,而 `deadCount*8 = 424*8 = 3392`,
即补丁只写了 3391 字节(比 header 声明的 dead 表少 1 字节),
随后文件被 resize 到 4,226,034,于是 dead 表末尾留下
**1 字节缺失 + 60 字节旧数据**。这正好解释了解包时看到的
“`misc.pkg` 比 `indexOffset + count*20 + deadCount*8` 多 60 字节”。

客户端加载器不读 dead 表,所以不影响运行;解包按 `dataSize` 截断即可。

## 6. 代码位置(Patcher.exe)

| 地址 | 函数 | 作用 |
|---:|---|---|
| `0x407627` | `sub_407627` | 读 28 字节头、LZMA props、解压、遍历目录/文件 |
| `0x406757` | `sub_406757` | 按 `newFileSize` + block 写入目标文件 |
| `0x4188D0` | `sub_4188D0` | LZMA 解压包装 |
| `0x41A1A0` | `sub_41A1A0` | LZMA1 解码核心 |

## 7. 工具

`gammaray-py/src/deltaxutils/gmz_unpack.py`:

```powershell
# 查看 header / 目标文件 / block 数
python gmz_unpack.py info patch\gycs-0.9.12.30-0.9.12.31.gmz

# 导出解压后的 manifest + JSON(可选逐 block)
python gmz_unpack.py dump patch\gycs-0.9.12.30-0.9.12.31.gmz -o out\gmz --blocks
```

`dump` 只写 manifest 和 block 数据,不自动应用到游戏目录。

> `config.pkg` 的补丁 block 可能包含敏感词库文本;工具本身只做字节解析,
> 不会主动打印内容。不要对相关 block 做预览/引用。
