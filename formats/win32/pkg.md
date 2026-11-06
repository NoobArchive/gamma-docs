# `.pkg` — Windows 端 Gamma 资源包

> 适用:`D:\game\guyu2\data\*.pkg`(《古域2》现存的 Windows 端 Gamma 构建,
> 样本版本 `0.9.12.31`)。
> 结论:容器**不压缩**;v1 只是 16 字节位置相关 XOR;索引只有 **32-bit 路径哈希**,没有文件名字符串。
> 逆向入口:`GammaCommon2.dll`(32 位,基址 `0x10000000`)。
> 包列表/计数见 [`docs/modules/win32/guyu2.md`](../../modules/win32/guyu2.md)。

## 1. 总体布局

```
+0x00  Header (0x14)
       data region
+idx   Index   (count * 0x14)
       Dead table (deadCount * 0x08)
```

| 偏移 | 类型 | 字段 | 说明 |
|---:|---|---|---|
| 0x00 | `char[4]` | magic | `"pckg"` (`70 63 6B 67`) |
| 0x04 | `u32` | version | 0 = 明文;1 = 数据区 XOR |
| 0x08 | `u32` | count | 索引项数 |
| 0x0C | `u32` | indexOffset | 索引的**绝对文件偏移** |
| 0x10 | `u32` | deadCount | 尾部 dead/free 块表项数 |

加载器 `sub_1000A120` 只显式读取前 16 字节,随后直接 seek 到 `indexOffset` 读索引;
`deadCount` 与 dead 表对**客户端读取**没有作用;`.gmz` 补丁会直接改写
dead 表(见 [gmz.md](gmz.md)),但生成该表的原始打包器不在客户端里。

数据区通常从 `0x14` 开始(紧接 0x14 字节头),但**每个 entry 的 `dataOffset` 才是权威**,
不要假设连续。

## 2. 索引项

索引位于 `indexOffset`,共 `count` 条,每条 **0x14 字节**:

| 偏移 | 类型 | 字段 | 说明 |
|---:|---|---|---|
| 0x00 | `u32` | nameHash | 成员路径的 `GammaHash`(见 §5) |
| 0x04 | `u32` | headerSize | `CCommonFileHead` 总长度(含 deps + extra);0 = 裸文件 |
| 0x08 | `u32` | dataOffset | 数据在 `.pkg` 内的**绝对偏移** |
| 0x0C | `u32` | dataSize | 存储字节数(含文件头) |
| 0x10 | `u32` | reserved | 实测 91,244 条全为 0 |

索引顺序**不保证**按 hash 或偏移排序;运行时用 `std::map<u32, Entry>` 查,顺序无意义。

### `headerSize` 的验证

对本作 15 个包、91,244 条记录做全量统计:

- 22,591 条以 `CCommonFileHead` 4CC 开头(`tex/ams/ans/anf/mtr/gfx/gui/eft/map`);
  `headerSize` **22,591/22,591** 等于按 `CCommonFileHead` 解析出的
  `end + extraSize`;
- 68,653 条裸文件(`lua/xml/ttf/mp3/dds/...`)`headerSize == 0`;
- 没有反例。

因此 `headerSize` 就是“主数据起点”,读取端可用 `dataSize - headerSize` 得到主数据长度。

## 3. 数据区与 dead table

### 3.1 数据偏移

`dataOffset` 是绝对偏移,不是相对数据区。不同条目之间可以有**未被任何索引项引用的区间**;
这些区间里通常是旧版本文件残留(高熵贴图/旧索引),**不一定是 0**。

### 3.2 dead table

在 `indexOffset + count * 0x14` 之后,是 `deadCount` 条 8 字节记录:

```
u32 size;       // 死区字节数
u32 endOffset;  // 前一个 live 文件的结束偏移
```

因此死区范围是 `[endOffset, endOffset + size)`。实测:

- 大多数包 `deadCount * 8` 正好填满索引后剩余空间;
- 所有 `endOffset` 都能对上一个 live entry 的 `dataOffset + dataSize`;
- 死区既可能是两个 live 文件之间的空洞,也可能是**最后一个 live 文件之后的尾部**
  (`dictionary.pkg` 就是后者:实际没有中间空洞,但 `deadCount == 1`);
- 这些死区**不是空闲链表意义上的“空洞”**,内容可能仍是旧文件;
- 客户端加载器完全不读这张表,解包可以忽略。

实例:`dictionary.pkg` 只有 1 个 live 条目,`dataSize = 123,438`,
其后的 117,968 字节被 dead 表标为死区;但这段死区仍能解出连续的
UTF-16 词典文本(旧版本残留),只是从中间某条记录开始(旧文件头已被新数据覆盖)。
解包时必须按 `dataSize` 截断,不能把死区拼进成员文件。

本作实例(`deadCount` 与死区数):

| 包 | deadCount | 备注 |
|---|---:|---|
| `config.pkg` | 729 | 729 条全部对上 |
| `map.pkg` | 1208 | 1208 条全部对上 |
| `role.pkg` | 92 | 92 条全部对上 |
| `gui.pkg` | 11 | 11 条全部对上 |
| `fx.pkg` | 52 | 52 条全部对上 |
| `misc.pkg` | 424 | 实际可解析 431 对 + 4 字节,共多出 60 字节 |
| 其余 | 0/1/2 | — |

`misc.pkg` 的 60 字节是唯一异常,原因已从 `.gmz` 补丁链确认:
`gycs-0.9.12.20-0.9.12.21.gmz` 对 `misc.pkg` 的 dead 表起点写入了
**3391** 字节(`deadCount*8 = 3392`,少 1 字节),随后把文件 resize 到
4,226,034;于是 dead 表末尾留下 1 字节缺失 + 60 字节旧数据。
所以按头部解析会看到 `431 * 8 + 4` 字节,但只有前 424 对(或 3391 字节)
是补丁写入的新表,后面是旧的残留。客户端不读该表,不影响解包。多数包满足:

```
fileSize == indexOffset + count * 20 + deadCount * 8
```

`misc.pkg` 多 60 字节。

## 4. 版本 1:位置相关 XOR

`version == 1` 时,**只有数据区**被 XOR;头部和索引是明文。

16 字节密钥(`sub_10001B10` @ `0x10001B10`):

```
E7 9B 9B 9C 4E D6 44 50 B3 53 8D F1 38 CC CE A9
```

解密第 `i` 个字节:

```
plain[i] = cipher[i] ^ KEY[(dataOffset + i) & 0xF]
```

注意密钥索引是**绝对文件偏移**,不是文件内相对偏移。例:`shader.pkg` 第一条
`dataOffset = 0x14`,所以从 `KEY[4]` 开始;解出首 4 字节正好是 `"gfx\0"`。

> 为什么索引不加密?加载器在读完 16 字节头、索引之后才把 `version` 写进
> `pkg+4`;之后的数据读取路径 `sub_1000A610` 才按 `pkg+4` 调用 XOR。

版本 1 仍**没有压缩**。解出的 payload 是原始 Gamma 文件、Lua 字节码、XML、
TTF、MP3 等;高熵内容只是原始媒体/编译后的 shader,不是二次加密。

本作实际版本:

| 包 | version | 处理 |
|---|---:|---|
| `config.pkg` | 1 | XOR |
| `misc.pkg` | 1 | XOR |
| `shader.pkg` | 1 | XOR |
| 其余 12 个 | 0 | 直接读 |

## 5. 文件名 = 32-bit 哈希

### 5.1 没有名字表

索引项只有 `nameHash`,没有 `char*`/UTF-16 名字。打包后无法从 `.pkg` 本身
反查名字;运行时也只按哈希查。要得到可读文件名,只能从外部线索恢复:

- XML 里的 `FileName=` / `file=` / `groupPath=`;
- `.ams/.ans/.anf/.mtr/.map` 的 `CCommonFileHead` 依赖名;
- Lua 字节码里的源路径(`D:\guyu\src/game/lua/...`);
- Lua 工程文件(`.luaproj` 的 `<Compile Include="...">`)等;
- `.gmz` 补丁的 block 数据里也可能有旧版本/新版本资源的头与依赖名
  (本轮用它把可读名从 12,873 提到 14,355,`shader.pkg` 5/5 全解)。

成员路径可以是中文,例如 `map.pkg` 里的
`tile/五龙/五龙植物/ani/l_f008_01.ans`;哈希只对 ASCII 字母做小写化,
中文原样参与 `GammaHash`。

### 5.2 路径规范化

运行时入口 `sub_1000A9B0` 把虚拟路径拆成 `包名/成员名`。该函数有“按第一个
分隔符”和“按最后一个分隔符”两条分支(由 file manager 的 `+38` 标志选择);
本作 `.pkg` 实测走**第一段 = 包名**这条:

```
font/msyh.ttf        -> font.pkg        + msyh.ttf
misc/ShellClient/x.lua -> misc.pkg      + ShellClient/x.lua
config/skill/skillclass.xml -> config.pkg + skill/skillclass.xml
role/tex/w_m304_t01.tex -> role.pkg     + tex/w_m304_t01.tex
```

包名是**第一段**;成员名是去掉第一段后的相对路径。哈希只对**成员名**做:

```
1. 取 UTF-16 字符串,最多 1023 个字符;
2. ASCII 'A'..'Z' 转小写;反斜杠 '\' 转 '/';
3. 按 UTF-16LE 取字节(不含结尾 NUL);
4. GammaHash(bytes)
```

所以大小写不敏感,`/` 与 `\` 等价。

### 5.3 `GammaHash` 算法

`Gamma::GammaHash` 导出在 `0x10001210`,核心是 `sub_10001000`:
12 字节一块的 Jenkins/lookup3 风格混合(不是标准 lookup3,参数/轮序以本实现为准),初值:

```
a = 0x9E3779B9
b = 0
c = 0x9E3779B9
```

每 12 字节读取 `w0/w1/w2`(小端),执行 9 步混合;尾部按剩余 1..11 字节
累加到 `a/b/c`,再做同样的 final mix。参考实现(已用 DLL 导出函数对拍,
随机 40 组字节串全部一致):

```python
import struct

M = 0xFFFFFFFF

def gamma_hash(data: bytes) -> int:
    n = len(data)
    a, b, c = 0x9E3779B9, 0, 0x9E3779B9
    i = 0
    while i + 12 <= n:
        w0, w1, w2 = struct.unpack_from('<III', data, i)
        c = (c + w1) & M
        b = (b + w2) & M
        t1 = ((b >> 13) ^ ((a + w0 - b - c) & M)) & M
        c1 = ((t1 << 8) ^ ((c - b - t1) & M)) & M
        b1 = ((c1 >> 13) ^ ((b - c1 - t1) & M)) & M
        a1 = ((b1 >> 12) ^ ((t1 - b1 - c1) & M)) & M
        c2 = ((a1 << 16) ^ ((c1 - b1 - a1) & M)) & M
        b2 = ((c2 >> 5) ^ ((b1 - c2 - a1) & M)) & M
        a2 = ((b2 >> 3) ^ ((a1 - b2 - c2) & M)) & M
        c3 = ((a2 << 10) ^ ((c2 - b2 - a2) & M)) & M
        b3 = ((c3 >> 15) ^ ((b2 - c3 - a2) & M)) & M
        a, b, c = a2, b3, c3
        i += 12

    rem = n - i
    tail = data[i:]
    v15 = (n + b) & M
    if rem >= 11: v15 = (v15 + (tail[10] << 24)) & M
    if rem >= 10: v15 = (v15 + (tail[9] << 16)) & M
    if rem >= 9:  v15 = (v15 + (tail[8] << 8)) & M
    if rem >= 8:  c = (c + (tail[7] << 24)) & M
    if rem >= 7:  c = (c + (tail[6] << 16)) & M
    if rem >= 6:  c = (c + (tail[5] << 8)) & M
    if rem >= 5:  c = (c + tail[4]) & M
    if rem >= 4:  a = (a + (tail[3] << 24)) & M
    if rem >= 3:  a = (a + (tail[2] << 16)) & M
    if rem >= 2:  a = (a + (tail[1] << 8)) & M
    if rem >= 1:  a = (a + tail[0]) & M

    v16 = ((v15 >> 13) ^ ((a - v15 - c) & M)) & M
    v17 = ((v16 << 8) ^ ((c - v15 - v16) & M)) & M
    v18 = ((v17 >> 13) ^ ((v15 - v17 - v16) & M)) & M
    v19 = ((v18 >> 12) ^ ((v16 - v18 - v17) & M)) & M
    v20 = ((v19 << 16) ^ ((v17 - v18 - v19) & M)) & M
    v21 = ((v20 >> 5) ^ ((v18 - v20 - v19) & M)) & M
    v22 = ((v21 >> 3) ^ ((v19 - v21 - v20) & M)) & M
    x = ((v22 << 10) ^ ((v20 - v21 - v22) & M)) & M
    return ((x >> 15) ^ ((v21 - x - v22) & M)) & M

def hash_path(path: str) -> int:
    out = []
    for ch in path:
        o = ord(ch)
        if 65 <= o <= 90:
            o += 32
        elif o == 92:
            o = 47
        out.append(o)
    raw = b''.join(struct.pack('<H', o) for o in out)
    return gamma_hash(raw)
```

### 5.4 实例(均已实际命中索引)

| 成员路径 | 包 | hash |
|---|---|---:|
| `msyh.ttf` | `font.pkg` | `062c63f8` |
| `simkai.ttf` | `font.pkg` | `1c6a4d8f` |
| `font.xml` | `config.pkg` | `8a3222e0` |
| `ShellClient/PlayerBasicInfo.lua` | `misc.pkg` | `001a23cf` |
| `tex/w_m304_t01.tex` | `role.pkg` | `d6170318` |
| `skeletal.gfx` | `shader.pkg` | `85ea3336` |
| `default.mtr` | `mat.pkg` | `b35d56b5` |

> 注意 `misc.pkg` 的 Lua 成员名通常**不带 `lua/` 前缀**:
> 源文件里的调试路径是 `D:\guyu\src/game/lua/ShellClient/...`,
> 但包内成员是 `ShellClient/...`。

由于哈希对 ASCII 大小写不敏感,从 XML/Lua 引用恢复出的名字大小写
不一定等于打包时的原始大小写;这不影响命中索引。

端到端验证:`font.pkg` 只有两条记录,TTF `name` 表分别是
`Microsoft YaHei` / `KaiTi_GB2312`;它们的索引哈希正好等于
`hash_path("msyh.ttf")` / `hash_path("simkai.ttf")`,
而 `config.pkg` 的 `font.xml` 又同时引用了这两个文件名。

`shader.pkg` 的 5 条记录也已全部恢复:
`terrain.gfx` / `water.gfx` / `blurdepth.gfx` / `skeletal.gfx` / `depth2tex.gfx`
(其中 `skeletal.gfx` 来自 `.mtr` 依赖,其余来自 `.gmz` patch block 中的
shader 资源头/字符串)。

### 5.5 运行时查找与哈希碰撞

`sub_1000A9B0` 先按成员路径哈希查 `std::map`;查不到时会把路径追加一个
十六进制数字(`0..f`)再查(`sub_10009C30`),用于同名/哈希碰撞的变体。
本作每个包内的 hash 都唯一(跨包可以有同名成员),没有观察到实际使用后缀的样本。

## 6. 读取伪代码

```python
if data[:4] != b'pckg':
    raise ValueError
version, count, idxoff, dead_count = struct.unpack_from('<IIII', data, 4)

for i in range(count):
    h, hsize, off, size, reserved = struct.unpack_from('<IIIII', data, idxoff + i*20)
    blob = data[off:off+size]
    if version == 1:
        blob = bytes(b ^ KEY[(off + j) & 0xF] for j, b in enumerate(blob))
    assert hsize <= size
    yield h, hsize, blob
```

完整解包还要处理:

- `off + size <= len(data)` 边界检查;
- 有名字线索时按 `成员路径` 写盘,否则用 `<hash:08x>.<推断后缀>`;
- `_manifest.json` 记录 `hash/offset/size/headerSize/reserved/name`;
- v1 的 XOR 基于**绝对偏移**,复制/移动文件后不能直接沿用旧偏移解密。

## 7. 与 `.r` 的区别

| | `.r` / filelist(手游) | `.pkg`(Windows 古域2) |
|---|---|---|
| 名字 | filelist 外部给出 `<dir>/<md5>.z` | 索引内只有 32-bit 路径哈希 |
| 记录 | `[u32 len][data]` 顺序记录 | 头部 + 数据区 + 尾部索引 |
| 索引位置 | 无独立索引 | 文件尾 `indexOffset`(索引后接 dead 表) |
| 压缩 | 无 | 无 |
| 加密 | 无 | v1 数据区 16 字节位置 XOR |
| 分片 | 有(200 KiB) | 无 |
| 随机访问 | 需解析记录 | 索引直接给 offset/size |

## 8. 代码位置

`GammaCommon2.dll`:

| 地址 | 函数 | 作用 |
|---:|---|---|
| `0x1000A120` | `sub_1000A120` | 加载 `.pkg`:读头、映射、读索引、填 map |
| `0x1000A9B0` | `sub_1000A9B0` | 虚拟路径 → `包.pkg` + 成员哈希查找 |
| `0x1000D850` | `sub_1000D850` | 路径规范化 + 调 `GammaHash` |
| `0x10001000` | `sub_10001000` | `GammaHash` 核心 |
| `0x10001B10` | `sub_10001B10` | 位置相关 XOR |
| `0x1000A610` | `sub_1000A610` | 按需读取/解密数据 |

## 9. 未解 / 待补

- dead table 的**打包器**语义(客户端与 `.gmz` 的写入行为已确认,但生成
  dead 表的原始打包器不在客户端里);
- 全量可读文件名的恢复(`.pkg` 本身不存名字,只能靠引用/工程文件反推;
  本轮已恢复到 14,355/91,244);
- `version > 1` 的行为(本作未出现);
- 哈希碰撞后缀的实际样本(本作未出现;运行时逻辑已确认是追加单个
  十六进制数字 `0..f` 重试)。

`.gmz` 已解,见 [gmz.md](gmz.md);它按字节范围修改目标文件,可同时更新
`data/*.pkg`、exe/dll 和 `version.inf`。
