# `.gmz` - 更新增量包

> 位置:`D:\game\guyu2\patch\gycs-<from>-<to>.gmz`。
> 状态:本轮只做了入口级确认,**未**完成格式解析。

## 已确认

- 文件头 4CC = `"gmz\0"` (`67 6D 7A 00`),随后 `u32 version = 0x2711`;
- 头部之后还有若干 `u32` 字段(样本中可见 `7, 2, ...`),但尚未逐字段标定;
- 正体是**高熵数据**,用 `.pkg` 的 16 字节 XOR 密钥按同样方式解**不能**得到
  可读文本/`CCommonFileHead`,因此不是 `.pkg` 的简单 XOR 变体;
- `Patcher.exe` 与 `ShellClient.exe` 都有对 4CC `"gmz\0"` 的比较;
- `patch/patch.inf` 是补丁索引(每节 `HASH` / `Version` / `HTTP_URL` / `BT_URL`),
  与 `.gmz` 文件一一对应。

## 与 `.pkg` 的关系

`.gmz` 是补丁链上的增量包,不直接是 `.pkg` 容器;它可能包含“目标文件 +
新数据/差异”,由 `Patcher` 合并进 `data/*.pkg`。`.pkg` 尾部的 dead table
(见 [pkg.md](pkg.md) §3)很可能就是给这种原地更新复用空间用的,但尚未证实。

## 后续

- 从 `Patcher.exe` / `ShellClient.exe` 的 `"gmz\0"` 分支入手,定位头部字段、
  压缩算法(若为 LZ4/LZO/zlib)和成员表;
- 确认 `.gmz` 内是否存有目标路径/哈希,它可能是恢复 `.pkg` 文件名的关键线索。
