# 目录封包 / filelist 生成

> 工具:`gammaray pack <目录> -o <输出> [--version ...] [--limit 204800]`
> 代码:`gammaray-py/src/deltaxutils/rpack.py`

## 1. 规则

01. 输入目录中除以下内容外,所有文件都参与打包:
    - 根目录 `etc/`(整个子树);
    - 根目录 `version.inf`;
    - 根目录 `filelist_*.r`;
    - 输出目录自身(如果在输入目录内)。
02. 版本号优先读根目录 `version.inf` 的 `[Data] Version = x.y.z`,
    其次读已有的 `filelist_*.r`;两者都没有时必须显式 `--version`。
    **filelist 名字必须和 `version.inf` 的 Version 一致**,否则游戏会去找
    `filelist_<version.inf>.r`,新 `.r` 全部对不上。
03. 打包完成后会把源目录的 `etc/` 和 `version.inf` **原样复制**到输出目录;
    它们不进入 `.r`/filelist,但输出目录可以直接当作 `assets/` 使用。
04. 按**直接父目录**分组;同一目录下按文件名(忽略大小写)排序。
05. 顺序累加文件,每个成员占 `4 + len(data)` 字节;累计不超过
    `--limit`(默认 **204800 = 200 KiB**)时合成一个逻辑包。
06. 单个文件自身 `4 + len` 超过 limit 时独占一个逻辑包。
07. 逻辑包的 `.r` 流 = `[u32 len][data]` 依次拼接;
    流长度超过 limit 时按 204800 字节切片(首片含记录头,后续片是裸数据)。
08. 每个分片写 `<dir>/<md5>.r`;filelist 行写 `<dir>/<md5>.z`、
    分片实际长度、压缩尺寸估计。
09. 成员行只在**该逻辑包所有分片之后**写一次,名字相对于包目录。
10. 输出目录中已有的其他 `filelist_*.r` 会被删除,避免游戏加载旧表。

## 2. filelist 的 `size` 到底是什么

`r-warn-probe` 已确认:

```text
col1(size) = 该 part 的 .r 文件实际字节数
           = 成员 payload 总和 + 4 × 成员数   (单分片时)
```

它不是“成员原始大小之和”,也不是“去掉记录头的数据大小”。
客户端 `CFileReader::CheckLocalBuffer` 用这个值校验本地 `.r` 大小。

因此:

- 只替换 `.r`、不更新 filelist 时,客户端会打印
  `CheckLocalBuffer: read error size!!!`;离线/无缓存时仍按 `.r` 记录头加载;
- 重新生成 filelist 时,`size` 必须等于实际 `.r` 长度,否则同样报警告。
- `fl-verify` 已增加 `size_mismatch` 统计;实测 `r-warn-probe` 为
  `ok=3 mismatch=0 size_mismatch=3`,官方 assets 为 `size_mismatch=0`。

> **排错提醒**:本次在线版本服实测中,清掉客户端本地缓存后同一份 APK
> 正常启动。该版本服只回应版本检查/serverlist,不做 filelist 与资源分发,
> 所以出现 `.z` 404 不一定是包本身的问题;先清缓存再判断。

`col2(csize)` 是服务器下载用的压缩尺寸。本地包不校验它;当前实现用
zlib 估算,官方 `.z` 压缩器尚未确认,数值可能与官方不同。

## 3. 重打包 gx_dump 对照 `_agent-sandbox/assets`

命令:

```powershell
python gammaray.py pack _agent-sandbox/gx_dump `
    -o _agent-sandbox/temp-another/repack_gx
```

版本号从 `gx_dump/version.inf` 自动读为 `1.3.3.4.7`;输出目录同时包含
`etc/`、`version.inf` 和 `filelist_1.3.3.4.7.r`。

结果:

```text
files=2565  packages=366  parts=855  data=147,484,160 B
filelist_1.3.3.4.7.r = 99,392 B
```

与官方 assets 对比(跳过 `/config` 顶层 `.r`,保留 `config/` 子目录):

| 项目 | 官方 assets | 本次 repack |
|---|---:|---:|
| filelist 成员 | 2515 | 2515 |
| 缺失成员 | — | 0 |
| 多出成员 | — | 0 |
| 成员内容差异 | — | **0** |
| 逻辑包 | 633 | 366 |
| 分片总数 | 1122 | 855 |

结论:

- **成员集合与每个成员的字节完全一致**;
- 差异全部来自**分组策略**:官方包的中位大小约 65 KiB,有 198 个单成员包;
  本工具按“同目录累计到 200 KiB”合并,因此包数更少、单包更大;
- 以“同一逻辑包内的成员对”衡量:
  官方 17,690 对,本工具 24,169 对,交集 11,780 对
  (官方召回 66.6%,本工具精确率 48.7%);完全相同的成员组 47 组;
- 按“成员集合完全相同”比较 365 个 repack 包(排除 config 顶层):
  **140 个与官方包字节完全一致**,225 个找不到相同成员集合的官方包
  (分组不同);没有任何“成员相同但字节不同”的包;
- 分片长度分布(1/2/3/6/7/28 片)与官方一致,说明 200 KiB 分片边界正确;
- 若要逐字节复刻官方 filelist,需要原服务器打包器的分组清单;
  仅凭目录和大小无法恢复官方分组。

## 4. 限制

- 不生成服务器 `.z` 压缩包,只生成客户端本地 `.r`;
- 不重打包/签名 APK;
- `csize` 是估算值;
- 官方分组不是单纯 200 KiB 贪心,无法保证包名/包划分与官方一致;
- 打包会读取所有输入字节(包括 `config` 下的文本表),但只做字节复制,
  不做内容解析或导出。
