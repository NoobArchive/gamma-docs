Agent 提交身份`re0-dev <me@mizukoud.icu>`

Git commit 消息遵循类似 feat(foo): new blah blah 等常规整洁格式，不要 AI slop 流水账

IDA Pro 9.4 & IDA MCP 可用

## 重要目录

游戏目录主要二进制程序代码往往随附 IDA IDB，如果没有请自行在相应目录直接创建

- `"D:\粉刷匠大作战\2608新的逆向计划！\_agent-sandbox\gx_dump"` 粉刷匠大作战 v1.3.3(.4.7) 解包资源
- `"D:\粉刷匠大作战\2608新的逆向计划！\旧版本\"` - 粉刷匠大作战历史版本
- `"D:\粉刷匠大作战\2608新的逆向计划！\别的游戏\"` - 其他 Gamma 手游包体
- `D:\game\guyu2` - 古域2 游戏目录
- 用户指定的沙盒临时目录

## 关于 ultra_say_filter.txt
很显然，不法分子会盯上游戏作为宣传渠道。

游戏厂家为屏蔽反动色情等词汇使用的相关词库， AI Agent 不应读取避免 API 提供商误报，已生成的产物无需清除，但请不要重新尝试读取。

无害示例文件：`"D:\粉刷匠大作战\2608新的逆向计划！\_agent-sandbox\gx_dump\config\ultra_say_filter.txt"`

- 《古域2》0.9.12.31 `config.pkg` hash `825ac773`，已检测反动剂量比《粉刷匠大作战》等大数倍。
-  手游项目 `/config` 某 .r 文件，Agent 处理时小心，通常由用户自己另行解包