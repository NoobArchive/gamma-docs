# 古域2

## 目录结构

```
├─bin
│  ├─release
│  └─release_test
├─data
├─etc
├─log
├─patch
└─usr
```
### /bin/release

``` title="/bin/release"
BugCommit.exe
d3dx9_27.dll
dbghelp.dll
DumpClient.dll
FR_info.cfg
GammaAudio.dll
GammaCommon2.dll
GammaEffect.dll
GammaGraphic.dll
GammaGUI.dll
GammaMp3.dll
GammaNetwork.dll
mfc80u.dll
Microsoft.VC80.CRT.manifest
Microsoft.VC80.MFC.manifest
msvcp80.dll
msvcr80.dll
Patcher.dat
Patcher.exe
PatcherRes.dat
PatcherRes.dll
ShellClient.exe
```

### /data

当前样本版本:`release_test/version.inf` = `0.9.12.31`(`release` 与
`release_test` 的 `ShellClient.exe` / `GammaCommon2.dll` 哈希相同)。

```
config.pkg      88,527,989   v1  10002 项
dictionary.pkg     241,454   v0      1 项
font.pkg        20,549,620   v0      2 项
fx.pkg          70,897,443   v0   5310 项
gui.pkg         27,135,916   v0    677 项
map.pkg        495,218,746   v0  66330 项
mapnpc.pkg       1,477,038   v0    172 项
mat.pkg              8,835   v0     67 项
misc.pkg         4,226,034   v1    429 项
music.pkg       35,143,259   v0     29 项
obj.pkg          8,214,240   v0    599 项
pet.pkg         15,211,022   v0   1091 项
role.pkg       178,303,293   v0   6165 项
shader.pkg          57,464   v1      5 项
sound.pkg       59,533,247   v0    365 项
```

共 15 个包、**91,244** 条记录。`.pkg` 是 Windows 端独有的容器,
手游使用的 `.r` + filelist 不是同一种东西;完整格式见
[`docs/formats/win32/pkg.md`](../../formats/win32/pkg.md)。

### /etc
```
directory.xml
gac_config.xml
```
directory.xml 和 gac_config.xml 在后续基于 Gamma 引擎的游戏中也得到使用。

直到《粉刷匠大作战》将 gac_config.xml 改名为 client_config.xml

## 资源包线索(2026-11)

- 加载器在 `GammaCommon2.dll`:`sub_1000A120` 读 `pckg` 头/索引,
  `sub_1000A9B0` 负责“虚拟路径 → 包名 + 成员路径”的拆分,
  `sub_1000D850` 做路径规范化并调用 `GammaHash`。
- `.pkg` 本身**不压缩**;`version==1` 的 `config/misc/shader` 只对数据区做
  16 字节位置相关 XOR,密钥 `E7 9B 9B 9C 4E D6 44 50 B3 53 8D F1 38 CC CE A9`,
  偏移按 `.pkg` 绝对文件偏移取模 16。索引和头部是明文。
- 索引项 20 字节:`nameHash / headerSize / dataOffset / dataSize / reserved`。
  `headerSize` 等于 `CCommonFileHead`(deps+extra)总长,裸文件为 0;
  本作 22,591 个带头文件全部精确对上,68,653 个裸文件为 0。
- **没有文件名表**:索引里只有 32-bit `GammaHash(小写化成员路径)`。
  同一个逻辑文件在包里叫 `包名/成员路径`,例如
  `font/msyh.ttf` → `font.pkg` 的 `msyh.ttf`;
  `misc/ShellClient/PlayerBasicInfo.lua` → `misc.pkg` 的
  `ShellClient/PlayerBasicInfo.lua`。
- 已用导出函数 `?GammaHash@Gamma@@YAIPBXI@Z` 对拍哈希实现(40 组随机字节串全一致),
  并用 `font.xml`、`msyh.ttf`、Lua 源路径等实际命中验证了路径拆分规则。
- `misc.pkg` 的尾部 dead table 有 60 字节未被头部 `deadCount` 覆盖;
  已确认来自 `0.9.12.20-0.9.12.21.gmz` 的 dead 表 block(写入 3391,
  `deadCount*8=3392`,少 1 字节),客户端不读该表,不影响解包。

## 补丁链

`patch/` 下有 74 个 `gycs-<from>-<to>.gmz`,是 0.9.10.15 → 0.9.12.31
的完整增量链。`.gmz` 不是 `.pkg` 容器,而是**按字节范围修改目标文件**:

- 28 字节头(`gmz\0` / `0x2711` / dirCount / fileCount / usize / csize / propsSize);
- 5 字节 LZMA1 props(`5D 00 00 00 04`,64 MiB 字典)+ 裸 LZMA1 流;
- manifest:7 个固定目录 + 每个文件 `newFileSize` + 若干 `(offset,size,data)` 块;
- 目标包括 `data/config.pkg`、`data/map.pkg`、`data/role.pkg`、
  `data/misc.pkg`、`data/fx.pkg`、ShellClient/Patcher/各 Gamma DLL、
  `version.inf` 等,共 208 个文件记录、136,890 个块。

`misc.pkg` 尾部多出的 60 字节也来自补丁:`0.9.12.20-0.9.12.21.gmz`
在 dead 表起点只写了 3391 字节(`deadCount*8=3392`),随后把文件
resize 到 4,226,034,留下了 1 字节缺失 + 60 字节旧数据。

包内容(按 4CC/魔数统计,详见解包 manifest):

| 包 | 主要类型 |
|---|---|
| `config.pkg` | 360 Lua 字节码、108 XML、231 UTF-16 文本、9,303 个私有二进制配置表 |
| `misc.pkg` | 422 Lua 字节码、3 XML、4 其它 |
| `fx.pkg` | 2,206 `.eft`、2,355 `.tex`、475 `.ams`、152 `.anf`、116 `.ans` |
| `gui.pkg` | 581 `.gui`、59 `.tex`、35 个 RIFF 光标/动画 |
| `map.pkg` | 576 `.map`、3,499 `.ams`、4,029 `.tex`、235 `.anf`、214 `.ans`、57,777 个裸地形块 |
| `role.pkg` | 818 `.ams`、2,983 `.anf`、404 `.ans`、1,954 `.tex` |
| `obj.pkg` | 211 `.ams`、160 `.anf`、73 `.ans`、155 `.tex` |
| `pet.pkg` | 91 `.ams`、854 `.anf`、31 `.ans`、115 `.tex` |
| `sound.pkg` | 362 WAV、1 MP3 |
| `music.pkg` | 29 MP3 |
| 其余 | `font.pkg` 2 TTF、`dictionary.pkg` 1 UTF-16 词典、`shader.pkg` 5 `.gfx`、`mat.pkg` 67 `.mtr`、`mapnpc.pkg` 172 个模型/动画/贴图 |

`config.pkg` 里还有 9,303 个私有二进制配置表(常见首 dword `0x1B` /
`0xFFFFFFFF` / `0x2C`),以及 `map.pkg` 的 57,777 个 `0x0C` 开头裸地形块;
它们不是 `CCommonFileHead`,本轮未展开。

## 格式转换探测(2026-11)

目录:`_agent-sandbox-guyu261103/guyu2_pkg_unpack`(91,260 文件 / ~1 GB)。
详细记录见 [`gammaray-py/docs/guyu2/GAMMA_TOOL_PROBE.md`](../../../gammaray-py/docs/guyu2/GAMMA_TOOL_PROBE.md),
版本矩阵与字段见 [`gammaray-py/docs/guyu2/formats/`](../../../gammaray-py/docs/guyu2/formats/version-matrix.md)。

- `.ans` 863/863、`.anf` 4474/4474、`.eft` 2207/2207 可解析;
- `.tex`:旧版头在 main 开头(24B),`>=0x2712` 每级有 u32 长度,
  压缩时是块 RLE;**8700/8700 转为 PNG**
  (`out_tex_all`,DXT1 5400 / DXT5 3209 / RGBA32 58 / RGB24 23 /
  A4R4G4B4 8 / RGB565 2);
- `.ams`:旧 piece 主数据头 + `GetVertexSize` 算 stride;
  **5117/5117 转为 GLB**(`out_glb_all`);
- `.gui` 0x2714:UTF-16 字符串、44B state、根节点 windows;
  **581/581 精确闭合并渲染**(`out_ui`);
- `.mtr` 0x2712/0x2713:**67/67 精确闭合**,alphaMode/doubleSided 已用于 GLB;
- `.map/.rgn/.skc/.tsk/.sce` 与 2 个 `0x7000` 地图块仍未覆盖。

古域2 现在具备完整的“纹理 + 模型 + UI + 材质”转换链路,但
`.map/.rgn` 地形和本地化词典(`dictionary.pkg`)还不在本轮范围内。
