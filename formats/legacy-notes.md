# 历史笔记 / 未整理线索

> 这些内容来自早期 `FORMAT-NOTES.md`,尚未进入正式格式文档;保留供继续逆向。

## 1. `.ani` 光标资源

`Gamma::CAniCursorFile` 使用 Windows ANI(RIFF)风格 4CC 块:

```
RIFF / LIST / ACON / INFO / INAM / IART / anih / fram / seq  / rate / icon
```

反编译入口:

- `CAniCursorFile::Load` @`0x265e5c`
- `TryReadChunk` @`0x265e94`
- `ReadChunk<EChunkIDType>` 模板按 4CC 分派

## 2. 包与哈希

- 文件名是哈希(`<dir>/<md5>.z`),但实测客户端**没有做哈希校验**;
- 保证 `.r` 记录布局、filelist 成员顺序/大小、分片拼接正确即可;
- `.r` 没有加密;高熵只是 MP3/H.264 等媒体本身的压缩。
