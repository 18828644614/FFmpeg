# Notes: FFmpeg 组件概览

## Repository Evidence

- 目标文件 `03-FFmpeg基础/FFmpeg组件概览.md` 是空主题模板。
- `03-FFmpeg基础/阶段03-FFmpeg基础.md` 只列出了四个主题入口，缺少目标、顺序和检查标准。
- `02-Windows开发环境/FFmpeg构建与安装.md`、`04-C++调用FFmpeg/API调用流程.md` 目前仍是模板，因此本章需要自包含地说明第一次 C++ 实验所需的最少配置。

## Beginner Learning Sequence

1. 工具与库的区别，以及 FFmpeg 在音视频程序中的位置。
2. 容器、流、编码器、数据包、帧和解封装/封装。
3. `ffmpeg`、`ffprobe`、`ffplay` 的职责和最小命令。
4. 从输入到输出的数据流，以及各个库在其中的位置。
5. C API 的核心对象和所有权边界。
6. Windows 的头文件、导入库、DLL、架构匹配和 CMake 配置。
7. 运行媒体信息查看器，观察格式、流、编码器、时间基和时长。
8. 通过练习和检查清单确认能解释数据流和定位常见链接/运行错误。

## Code Design

- 使用当前 API，不使用 `av_register_all`、`avcodec_register_all`、`AVStream.codec` 等旧教程内容。
- 示例只读取容器和流元数据，不分配解码器上下文或处理音视频帧。
- 失败路径统一打印 `av_strerror` 的可读错误并调用 `avformat_close_input`。

## Verification

- 主题页包含 32 个成对的 Markdown 代码围栏，覆盖文本图、CMake、C++、PowerShell 和示例输出。
- 主题页和阶段入口中的 Wikilink 均解析到当前知识库中的文件。
- `git diff --check` 未报告新增内容的空白错误。
- 未在当前机器上运行 C++ 编译：工作区没有确认可用的 FFmpeg 头文件、`.lib` 和 `.dll` 安装，正文已给出运行前置条件和命令。
