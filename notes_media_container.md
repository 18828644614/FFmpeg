# Notes: 媒体文件与容器

## Existing Repository Evidence

- `03-FFmpeg基础/媒体文件与容器.md` 当前是空主题模板。
- `03-FFmpeg基础/阶段03-FFmpeg基础.md` 已将本主题放在 FFmpeg 命令行工具之后、时间戳与时间基之前。
- `03-FFmpeg基础/FFmpeg组件概览.md` 已解释 `AVFormatContext`、`AVStream`、`AVCodecParameters`、`AVPacket` 以及 Windows CMake 示例，本章应在此基础上聚焦容器和文件结构，不重复整篇组件总览。
- `03-FFmpeg基础/时间戳与时间基.md` 仍是模板，因此本章只做必要的时间戳铺垫，并链接到该主题作为深入阅读。

## Content Decisions

### Learning order

1. 文件、容器、流、编码的比喻和边界。
2. 数据包、帧、关键帧、交错写入和时间信息。
3. 封装、解封装、重封装、转码的区别。
4. MP4/MOV、MKV、MPEG-TS、WebM、AVI、WAV/FLAC 的定位和限制。
5. 容器级元数据、流级元数据、章节、字幕、附件和索引。
6. 用 `ffprobe` 和 `ffmpeg` 观察同一个文件。
7. 用 C++ 读取格式与流信息。
8. 用 C++ 在不解码、不重新编码的情况下完成 remux。

### Code safety notes

- 示例使用共享 FFmpeg 构建、CMake + MSVC、x64。
- 示例检查 `AV_NOPTS_VALUE`，并使用 `av_rescale_q` 将输入包时间戳换算到输出流时间基。
- 说明 `AVStream` 和 `AVCodecParameters` 是由格式上下文拥有的借用指针，不能直接 `delete`。
- 明确 remux 只适合目标容器支持的编码组合；不支持时应转码。

## Verification Checklist

- [x] 目标文件包含完整学习范围和顺序。
- [x] 每个首次出现的术语都用新手能懂的语言解释。
- [x] C++ 示例包含头文件、CMake、PowerShell 编译/运行命令和预期输出。
- [x] 常见错误覆盖扩展名误判、DLL、位数、时间戳、容器不支持编码、输入文件损坏和资源释放。
- [x] 练习题与阶段检查清单能验证“理解”和“动手”两类能力。
- [x] 内部 wikilink 指向知识库中现有文件。
