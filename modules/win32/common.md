# 共同模块: Windows 端游
古域2 等 Gamma 游戏

诸多迹象证明《粉刷匠大作战》等手游在内部开发亦编译了 Windows 端游，但如今已彻底遗失。

## 特有格式

### `.pkg` — 资源包

- 位置:`data/*.pkg`;
- 头:`pckg` + version + count + indexOffset + deadCount(0x14 字节);
- 索引在文件尾,每条 0x14 字节:
  `nameHash / headerSize / dataOffset / dataSize / reserved`;
- `version==0` 明文;`version==1` 只对数据区做 16 字节位置相关 XOR,
  密钥和偏移规则见 [pkg.md](../../formats/win32/pkg.md);
- **不压缩**,索引里**没有文件名**,只有 `GammaHash(小写成员路径)`;
- 加载器:`GammaCommon2.dll` 的 `sub_1000A120` / `sub_1000A9B0` /
  `sub_1000D850` / `sub_10001B10`。

`config.pkg`、`misc.pkg`、`shader.pkg` 是 version 1,其余 12 个为 version 0。
完整规格、哈希参考实现、读取伪代码见
[`docs/formats/win32/pkg.md`](../../formats/win32/pkg.md)。

### `.gmz` — 更新增量包

`patch/*.gmz`,文件头为 `gmz\0` + version `0x2711`;本轮只确认了它与
`.pkg` 不是同一容器、且不能用 `.pkg` 的 XOR 密钥直接解,完整格式待补
(见 [gmz.md](../../formats/win32/gmz.md))。

谨慎预览 `config.pkg`，可能仍然具有《粉刷匠大作战》的 ultra_say_fliter.txt 屏蔽词库

（粉刷匠大作战使用的屏蔽词库本就继承自 joyegame 的旧网游项目）