# Task Plan: 补充 FFmpeg 组件概览

## Goal
为完全新手补充一篇可按顺序学习的 FFmpeg 组件概览，覆盖概念、Windows + C++ + FFmpeg 实验、排错、练习和阶段检查，并同步阶段 03 入口。

## Phases
- [x] Phase 1: 检查知识库结构、模板和相邻章节
- [x] Phase 2: 规划组件、数据流和实验范围
- [x] Phase 3: 编写主题正文和可运行示例
- [x] Phase 4: 更新阶段 03 入口
- [x] Phase 5: 检查链接、代码块、命令和交付内容

## Key Questions
1. 新手应先建立哪些术语和数据流模型？
2. 哪些 FFmpeg 库和工具需要在本章掌握，哪些应留到后续章节？
3. 怎样让 Windows + C++ 示例能验证组件职责，而不提前进入完整解码器实现？

## Decisions Made
- 先讲“工具与库的区别”，再讲媒体数据流和 C API 对象。
- 组件范围覆盖 `ffmpeg`、`ffprobe`、`ffplay` 以及 `libavformat`、`libavcodec`、`libavutil`、`libswscale`、`libswresample`、`libavfilter`、`libavdevice`、`libpostproc`。
- 首个 C++ 实验只做媒体信息读取，使用 `avformat_open_input`、`avformat_find_stream_info`、`av_dump_format` 和流元数据，避免在本章重复解码章节。
- Windows 先采用共享 FFmpeg 构建，使用 CMake + MSVC，运行时通过 `PATH` 找到 DLL。

## Errors Encountered
- 技能目录别名与实际路径不一致：已改用实际存在的 `C:\Users\Administrator\.codex\skills` 路径读取技能。
- 工作区已有未提交的 `task_plan.md`、`notes.md` 和其他文件：未覆盖，使用本任务单独文件记录过程。

## Status
**Complete** - 正文、C++ 示例、阶段入口和静态检查均已完成。
