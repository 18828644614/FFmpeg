# Notes: 计算机与音视频基础

## Existing Repository Evidence

- `01-前置基础/计算机与音视频基础.md` 是空主题模板，包含“章节大纲、前置知识、核心概念、API / 工具 / 配置、实践与实验、常见问题、参考资料”等栏目。
- `01-前置基础/阶段01-前置基础.md` 当前只列出四个主题入口，尚未给出每个主题的学习顺序和阶段验收标准。
- `00-索引/主索引.md` 已有 12 个学习阶段和 Mermaid 主线关系，但阶段 01 目前只有总体复选框。
- 其他阶段主题页大多仍是模板，因此本章需要自包含地解释术语，并通过 wikilink 指向后续主题作为延伸阅读。

## Content Decisions

### Learning order

1. 先建立计算机的“数据、程序、内存、文件、进程”模型。
2. 再补 C++ 开发中会遇到的编译、链接、指针、资源释放和错误码。
3. 进入数字音频：采样、采样率、量化、位深、声道、PCM、帧和码率。
4. 进入数字视频：像素、分辨率、帧率、扫描方式、颜色空间、像素格式和码率。
5. 解释 codec、container、stream、packet、frame、mux、demux、decode、encode、transcode。
6. 解释时间戳、时间基、PTS、DTS、duration、音视频同步。
7. 解释 FFmpeg 组件、C API 对象、平面数据、linesize、内存生命周期。
8. 用 Windows + CMake + C++ + FFmpeg 做媒体信息查看、时间换算和 PCM 实验。

### Example API choices

- Media inspection: `avformat_open_input`, `avformat_find_stream_info`, `av_dump_format`, `avformat_close_input`.
- Stream metadata: `AVFormatContext`, `AVStream`, `AVCodecParameters`, `avcodec_get_name`, `av_q2d`, `av_rescale_q`.
- Error display: `av_strerror`.
- Avoid deprecated APIs and avoid decoding in the first example; decoding is covered in `05-解封装与解码`.

## Verification Checklist

- [x] Markdown headings are complete and readable for beginners.
- [x] Code blocks specify language and include Windows execution commands.
- [x] Every new technical term is explained before it is used as an assumption.
- [x] Common failures cover DLL search path, include/lib mismatch, input path quoting, no stream info, timestamps, and cleanup.
- [x] Wikilinks point to existing files.

## Environment Check

- The current PowerShell session does not have `ffmpeg`, `ffprobe`, `cmake`, or `cl` on PATH.
- The document therefore includes reproducible commands but does not claim that the examples were compiled in this session.
