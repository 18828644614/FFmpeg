# Task Plan: 补充媒体文件与容器

## Goal
为完全新手补充一篇可按顺序学习的“媒体文件与容器”章节，覆盖概念、Windows + C++ + FFmpeg 实验、运行方法、排错、练习和阶段检查清单。

## Phases
- [x] Phase 1: 检查知识库结构、目标文件和相邻章节
- [x] Phase 2: 规划学习范围、概念顺序和实验路线
- [x] Phase 3: 编写主题正文、C++ 示例和命令行实验
- [x] Phase 4: 检查 Markdown、代码、命令和内部链接
- [x] Phase 5: 交付并说明验证边界

## Key Questions
1. 新手如何区分文件、容器、流、编码、数据包和帧？
2. 哪些容器层概念需要在进入解封装/编码章节前掌握？
3. 如何用 Windows + C++ + FFmpeg 先读取信息，再完成一次不重新编码的重封装？

## Decisions Made
- 以 `03-FFmpeg基础/媒体文件与容器.md` 为唯一主要教材文件。
- 学习顺序采用“文件和盒子 -> 流和编码 -> 数据包与时间 -> 常见容器 -> 元数据与索引 -> 命令行观察 -> C++ 查看 -> C++ 重封装”。
- C++ 示例优先使用现代 FFmpeg API：`avformat_open_input`、`avformat_find_stream_info`、`av_read_frame`、`avformat_alloc_output_context2`、`av_interleaved_write_frame`。
- 先做媒体信息查看，再做不重新编码的 remux；完整解码、编码和播放器线程模型留给后续章节。

## Errors Encountered
- 技能别名目录与实际目录名不同，已改用 `C:\Users\Administrator\.agents\skills` 下实际存在的技能文件。
- 当前工作树包含其他任务的未提交修改，本任务不覆盖、不回滚这些文件。
- 当前 PowerShell 会话未确认已安装 `ffmpeg`、`ffprobe`、CMake 或 MSVC，因此文档会提供运行命令，但不宣称本次已完成实际编译。

## Status
**Complete** - 主题正文、命令行实验、两个 C++ 示例、排错、练习和阶段检查清单已写入目标文件；静态检查已完成，实际编译受当前环境缺少 FFmpeg/MSVC/CMake 限制。
