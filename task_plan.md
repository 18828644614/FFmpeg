# Task Plan: 补充时间戳与时间基

## Goal
为完全没有 FFmpeg 时间概念基础的读者，完成一篇可按顺序学习、可在 Windows + C++ + FFmpeg 环境中复现的“时间戳与时间基”章节。

## Phases
- [x] Phase 1: 检查知识库结构、现有模板和项目约定
- [x] Phase 2: 规划学习范围、顺序和实验路线
- [x] Phase 3: 编写主题正文、代码示例、排错和练习
- [x] Phase 4: 检查 Markdown、链接、命令和交付内容

## Key Questions
1. 完全新手需要按什么顺序学习时间、时间基、PTS、DTS 和 duration？
2. 哪些时间换算应该用可运行的 Windows + C++ + FFmpeg 实验验证？
3. 如何把“知道术语”推进到“能定位时间戳和同步错误”？

## Decisions Made
- 以 `03-FFmpeg基础/时间戳与时间基.md` 作为主要教材文件。
- 学习顺序采用“前置知识 -> 时间单位 -> time base -> PTS/DTS/duration -> FFmpeg 换算 API -> ffprobe -> C++ 查看与 remux -> seek/同步 -> 练习检查”。
- 示例优先使用 `av_rescale_q`、`av_packet_rescale_ts` 和 FFmpeg 格式 API，避免依赖尚未讲解的完整播放器架构。
- 保留现有知识库的 Obsidian wikilink 风格，并与媒体容器、解封装、编码和同步章节建立链接。

## Errors Encountered
- 技能目录名称与目录映射不同，已改用实际存在的 `.codex/skills` 路径读取技能文件。
- 当前 PowerShell 会话中未找到 `ffmpeg`、`ffprobe`、`cmake` 或 `cl`，因此完成了静态检查，未执行实际生成媒体、编译或运行。

## Status
**Complete** - 主题正文、代码、排错、练习和链接已完成；当前环境缺少 FFmpeg 工具链，因此只完成了静态检查。
