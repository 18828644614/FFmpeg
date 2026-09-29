---
type: topic
status: draft
created: 2026-09-29
updated: 2026-09-29
tags:
  - ffmpeg
  - cpp
  - windows
  - 组件
---

# FFmpeg组件概览

> 本章的目标不是背下所有 API，而是先建立一张可靠的地图：当你看到一个媒体文件、一个 FFmpeg 命令或一段 C++ 代码时，能说清楚它正在处理哪一层数据、哪个组件负责这件事，以及下一步应该查什么。

## 你学完本章应该能做什么

- 说清楚 FFmpeg 项目、命令行程序和 C/C++ 库之间的关系。
- 区分容器、流、编码器、数据包、帧这些容易混淆的词。
- 根据任务选择 `ffmpeg`、`ffprobe`、`ffplay` 或某个库。
- 画出“文件 -> 解封装 -> 数据包 -> 解码 -> 帧 -> 处理 -> 编码 -> 封装 -> 文件”的数据流。
- 认识 `AVFormatContext`、`AVStream`、`AVCodecParameters`、`AVPacket`、`AVFrame` 等对象的大致职责和所有权。
- 在 Windows 上用 MSVC、CMake 和 FFmpeg 共享库编译一个媒体信息查看器。
- 看到头文件、链接错误、DLL 错误、位数不匹配和输入文件错误时，知道先检查哪一层。

本章暂时不要求你写出完整播放器，也不要求你掌握 H.264、AAC 的内部压缩算法。解封装、解码、编码、滤镜和播放器会在后续章节展开：[[03-FFmpeg基础/媒体文件与容器]]、[[05-解封装与解码/解封装流程]]、[[05-解封装与解码/视频解码流程]]、[[06-编码与封装/转码流程]]。

## 章节大纲与学习顺序

请按表格从上到下学习。每一步先回答一个简单问题，再进入代码。

| 顺序 | 要回答的问题 | 本步掌握的内容 | 完成标志 |
|---|---|---|---|
| 1 | FFmpeg 到底是什么？ | 项目、工具、库的区别 | 能说出 `ffmpeg` 不是全部 FFmpeg |
| 2 | 一个 MP4 里面装了什么？ | 容器、流、编码、数据包、帧 | 能画出文件内部的层次 |
| 3 | 命令行工具分别做什么？ | `ffmpeg`、`ffprobe`、`ffplay` | 能用 `ffprobe` 查看流信息 |
| 4 | 每个库负责什么？ | `libavformat`、`libavcodec` 等 | 能把任务映射到库 |
| 5 | C++ 程序里有哪些对象？ | context、stream、packet、frame 和资源释放 | 能区分“借来的指针”和“自己拥有的对象” |
| 6 | Windows 为什么会编译失败或运行失败？ | include、`.lib`、`.dll`、PATH、架构 | 能定位三类常见错误 |
| 7 | 如何把地图变成程序？ | CMake + MSVC 媒体信息查看器 | 能编译并读取一个本地媒体文件 |
| 8 | 是否真的理解了？ | 对照输出、练习、检查清单 | 能不看答案解释完整数据流 |

## 前置知识

### 必须会的最少内容

你只需要具备下面这些基础，不需要先学完整个 C++：

- 会在 PowerShell 中切换目录、运行程序和查看环境变量。
- 知道 C++ 程序由源文件编译成 `.exe`，程序运行时还可能加载 `.dll`。
- 看得懂变量、函数、指针、`if`、循环、字符串和返回值。
- 知道路径中可能有空格，带空格的路径需要用双引号包起来。
- 知道“库”就是别人已经编译好的功能集合，头文件告诉编译器怎么调用它。

如果这些内容还不熟，先阅读 [[01-前置基础/C++基础]]、[[01-前置基础/计算机与音视频基础]] 和 [[02-Windows开发环境/Visual Studio与MSVC]]。本章中的每一个术语都会再次解释，所以不需要因为不会而停下来。

### 暂时不需要掌握的内容

本章先不要求你理解宏汇编、SIMD、GPU、线程模型、滤镜图内部实现或编码器数学原理。它们很重要，但不应该成为第一次认识 FFmpeg 的门槛。

## 一、先建立整体地图

### 1.1 FFmpeg 项目、程序和库不是同一个东西

“FFmpeg”这个名字通常同时指三层内容：

1. **FFmpeg 项目**：一组处理音视频的开源代码、库和工具。
2. **命令行程序**：你在终端运行的 `ffmpeg.exe`、`ffprobe.exe`、`ffplay.exe`。
3. **开发库**：你的 C++ 程序通过头文件和链接库调用的 `libavformat`、`libavcodec` 等。

可以把它们想成同一套厨房：命令行程序是已经装好的菜单，开发库是你自己拿来组合的食材和工具。命令行程序适合快速验证，开发库适合把能力嵌入自己的软件。

### 1.2 三个常用命令行程序

| 程序 | 作用 | 新手第一次怎么用 |
|---|---|---|
| `ffmpeg` | 读取、处理、编码并输出媒体 | `ffmpeg -i input.mp4 output.mkv` |
| `ffprobe` | 只查看媒体信息，尽量不改变文件 | `ffprobe -hide_banner input.mp4` |
| `ffplay` | 用来快速播放和观察媒体 | `ffplay input.mp4` |

`ffmpeg` 命令中的 `-i` 表示输入（input）。命令行工具内部也会调用下面介绍的库，所以它们是观察库行为的好入口。

### 1.3 从输入到输出的数据流

先看一条最重要的主线：

```text
输入文件或设备
       |
       v
解封装 demux（libavformat）  把容器拆成音频流、视频流和数据包
       |
       v
压缩数据包 AVPacket
       |
       v
解码 decode（libavcodec）    把压缩数据还原成可处理的数据
       |
       v
原始帧 AVFrame
       |
       +--> 视频转换/缩放（libswscale）
       +--> 音频重采样（libswresample）
       +--> 滤镜处理（libavfilter）
       |
       v
编码 encode（libavcodec）    把原始数据压缩成 H.264、AAC 等
       |
       v
封装 mux（libavformat）      把数据包写入 MP4、MKV 等容器
       |
       v
输出文件、网络或设备
```

这条线不是每个程序都完整执行。例如 `ffprobe` 主要停在解封装和读取元数据，媒体信息查看器也可以在看到流信息后结束。后续解码章节会把“数据包 -> 帧”这一步写成完整代码。

## 二、必须先分清的媒体术语

下面使用“一个带声音的 MP4”做例子。假设这个文件包含一条 H.264 视频流和一条 AAC 音频流。

| 术语 | 通俗解释 | 在例子中的样子 |
|---|---|---|
| 容器（container） | 装音视频数据的盒子，也保存时间、标题等信息 | MP4、MKV、MOV |
| 流（stream） | 盒子里的一条连续轨道 | 视频流、音频流、字幕流 |
| 编码器/解码器（codec） | 压缩和还原数据的规则与实现 | H.264、H.265、AAC、Opus |
| 数据包（packet） | 文件中携带压缩数据的一小段 | 一段 H.264 压缩字节 |
| 帧（frame） | 解码后可以显示或播放的一张图、一个音频采样块 | 一张视频画面、1024 个音频采样 |
| 解封装（demux） | 从容器中拆出各条流和数据包 | 从 MP4 取出视频包和音频包 |
| 封装（mux） | 把数据包按容器规则写入文件 | 把 H.264/AAC 写进 MP4 |
| 编码（encode） | 原始帧变成压缩数据包 | YUV 帧变成 H.264 包 |
| 解码（decode） | 压缩数据包还原成原始帧 | H.264 包变成 YUV 帧 |

### 2.1 容器不是编码器

`.mp4` 只说明“外面的盒子”是 MP4，不能说明里面一定是 H.264。MP4 也可能装 H.265、AV1、AAC 或其他组合。反过来，H.264 也可以放在 MP4、MKV、TS 等不同容器里。

判断一个文件时要分别问两个问题：

1. **容器是什么？** 例如 `mov,mp4,m4a,3gp,3g2,mj2`。
2. **每条流用什么编码？** 例如 `h264`、`aac`。

### 2.2 数据包和帧也不是一回事

数据包通常是压缩后的数据，解码器可能需要多个数据包才能输出一帧，也可能一次输入输出多帧。数据包带有时间戳和流编号，帧带有分辨率、像素格式、采样率等解码后属性。不要把“一个包等于一帧”写进自己的程序假设里。

## 三、各个 FFmpeg 组件负责什么

### 3.1 `libavutil`：所有组件共用的基础工具箱

它提供错误码、时间基、合理数、内存辅助、像素格式和采样格式枚举等基础能力。你会经常见到：

- `AVRational`：用分子和分母表示一个分数，例如时间基 `1/90000`。
- `AV_NOPTS_VALUE`：表示某个时间戳未知，不能直接当成 0。
- `av_strerror`：把负数错误码转换成可读文字。
- `av_rescale_q`：在两个时间基之间换算整数时间值。

本章的 C++ 示例会直接用到 `av_strerror`、`AVRational` 和 `av_q2d`。

### 3.2 `libavformat`：容器和输入输出

它负责识别 MP4、MKV、TS 等容器，读取或写出流、数据包和元数据。

常见对象和函数：

- `AVFormatContext`：一次输入或输出任务的总上下文。
- `AVStream`：容器中的一条流。
- `AVCodecParameters`：这条流使用的编码类型、宽高、采样率等参数。
- `avformat_open_input`：打开输入。
- `avformat_find_stream_info`：读取足够多的信息来判断流属性。
- `av_read_frame`：逐个读取压缩数据包，后续解封装章节会用到。
- `avformat_close_input`：关闭输入并释放相关资源。

### 3.3 `libavcodec`：编码和解码

它负责把压缩数据包和原始帧互相转换。它知道 H.264、H.265、AAC、Opus 等编码格式的规则，也提供编码器和解码器的统一接口。

常见对象和函数：

- `AVCodec`：某个编码器或解码器的描述。
- `AVCodecContext`：一次编码或解码任务的可变上下文。
- `AVPacket`：通常携带压缩数据。
- `AVFrame`：通常携带解码后的原始数据。
- `avcodec_find_decoder` / `avcodec_find_encoder`：查找实现。
- `avcodec_send_packet`、`avcodec_receive_frame`：现代解码接口。
- `avcodec_send_frame`、`avcodec_receive_packet`：现代编码接口。

本章只通过 `avcodec_get_name` 读取编码名称，不开始真正解码。完整流程见 [[05-解封装与解码/视频解码流程]]。

### 3.4 `libswscale`：视频画面的格式和尺寸转换

它处理视频帧的缩放和像素格式转换。例如把 1920x1080 的 YUV420P 转成 1280x720 的 BGRA，供 Windows 窗口显示。它不负责读取 MP4，也不负责理解 H.264 压缩格式。

典型入口是 `sws_getContext` 和 `sws_scale`。需要显示画面时，常见路径是“`libavcodec` 解码 -> `libswscale` 转换 -> 窗口或渲染 API”。

### 3.5 `libswresample`：音频采样格式和采样率转换

它处理音频的重采样，例如把 48 kHz、浮点、立体声转换成 44.1 kHz、16 位整数、单声道，方便送给某个音频设备或编码器。它不负责播放声音，也不负责解码 AAC。

典型入口是 `swr_alloc_set_opts2`、`swr_init` 和 `swr_convert`。后续见 [[05-解封装与解码/音频重采样]]。

### 3.6 `libavfilter`：连接成图的音视频处理器

它把多个处理步骤连接成一个 filter graph（滤镜图）。例如：

```text
输入帧 -> crop 裁剪 -> scale 缩放 -> fps 改帧率 -> 输出帧
```

它接收和输出帧，本身通常不替代解码器或封装器。常见滤镜包括 `scale`、`crop`、`fps`、`volume`、`aresample`。后续见 [[07-音视频处理/滤镜图基础]]。

### 3.7 `libavdevice`：摄像头、麦克风等设备

它为操作系统设备提供输入输出适配。例如在 Windows 上读取摄像头或麦克风。设备名称和可用格式取决于操作系统，第一次学习文件处理时可以先忽略。

### 3.8 `libpostproc`：部分视频后处理

它提供较早期的视频后处理功能。很多普通应用不会直接调用它，初学阶段只需要知道它存在，不要把它和 `libavfilter` 混为一谈。

### 3.9 组件速查表

| 需求 | 首要组件 | 你会看到的对象或函数 |
|---|---|---|
| 打开 MP4、读取流 | `libavformat` | `AVFormatContext`、`AVStream` |
| 找到编码名称 | `libavcodec` | `AVCodecParameters`、`avcodec_get_name` |
| 解码视频或音频 | `libavcodec` | `AVCodecContext`、`avcodec_send_*` |
| 处理时间和错误 | `libavutil` | `AVRational`、`av_strerror` |
| 改视频尺寸或像素格式 | `libswscale` | `SwsContext`、`sws_scale` |
| 改音频采样率或格式 | `libswresample` | `SwrContext`、`swr_convert` |
| 串联裁剪、缩放、音量 | `libavfilter` | filter graph、`AVFilterContext` |
| 读摄像头或麦克风 | `libavdevice` | device input/output |

## 四、C++ 中的核心对象和资源边界

FFmpeg 是 C API。它的很多对象都用指针表示，创建和释放通常必须调用 FFmpeg 提供的函数，不能随意 `delete`。

| 对象 | 所在层 | 通俗作用 | 常见创建/释放方式 |
|---|---|---|---|
| `AVFormatContext` | `libavformat` | 一次输入/输出任务 | `avformat_open_input` / `avformat_close_input` |
| `AVStream` | `libavformat` | 容器中的一条流 | 通常由 `AVFormatContext` 拥有，先不要单独释放 |
| `AVCodecParameters` | `libavcodec` | 流的编码和媒体参数 | 通常从 `AVStream::codecpar` 借用 |
| `AVCodecContext` | `libavcodec` | 一次编码/解码状态 | `avcodec_alloc_context3` / `avcodec_free_context` |
| `AVPacket` | `libavcodec` | 一段压缩数据 | `av_packet_alloc` / `av_packet_free` |
| `AVFrame` | `libavutil` | 一帧原始数据 | `av_frame_alloc` / `av_frame_free` |

### 4.1 “借来的指针”和“我拥有的指针”

下面两行看起来都只是指针，但含义不同：

```cpp
AVStream* stream = format->streams[index];
AVPacket* packet = av_packet_alloc();
```

- `stream` 指向 `format` 内部已经存在的对象。你可以读取它，但通常不负责释放它。
- `packet` 是你主动申请的对象。你必须在结束时调用 `av_packet_free(&packet)`。

这条规则是 C++ 调用 FFmpeg 时最容易出错的地方之一。后续章节会进一步介绍 RAII 封装和错误路径清理，见 [[04-C++调用FFmpeg/错误处理与资源生命周期]]。

### 4.2 旧教程为什么经常对不上

如果教程要求你调用下面的函数，先停下来核对版本：

- `av_register_all()`
- `avcodec_register_all()`
- `AVStream::codec`
- `av_init_packet()` 的旧式用法

现代 FFmpeg 已经不需要前两个全局注册函数，流的编码参数应从 `stream->codecpar` 读取，解码器上下文通常通过 `avcodec_parameters_to_context` 初始化。不要为了让旧代码通过编译而混用旧头文件和新库。

## 五、Windows + C++ 开发环境

### 5.1 你需要的四样东西

| 东西 | 作用 | 常见形式 |
|---|---|---|
| MSVC | 编译 C++ | Visual Studio 2022 的“使用 C++ 的桌面开发”工作负载 |
| CMake | 生成 Visual Studio 工程并管理路径 | CMake 3.20 或更高 |
| FFmpeg 开发包 | 头文件和链接库 | `include`、`lib`、`bin` |
| 一个测试媒体文件 | 让程序有输入 | 例如 `sample.mp4` |

本章示例假定 FFmpeg 放在 `C:\ffmpeg`，目录大致如下：

```text
C:\ffmpeg\
  include\libavformat\avformat.h
  include\libavcodec\avcodec.h
  include\libavutil\avutil.h
  lib\avformat.lib
  lib\avcodec.lib
  lib\avutil.lib
  bin\avformat-*.dll
  bin\avcodec-*.dll
  bin\avutil-*.dll
```

不同发行版的 DLL 文件名可能带版本号，这是正常的。第一次实验建议使用与 MSVC、x64 匹配的共享构建；静态链接需要额外的系统库和编译选项，留到工程化章节处理。

### 5.2 CMake 配置

新建一个目录，例如 `component-overview`，在其中放入下面的 `CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.20)

project(ffmpeg_component_overview LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Pass a Windows path with forward slashes, for example C:/ffmpeg.
set(FFMPEG_ROOT "" CACHE PATH "FFmpeg installation root")

if(NOT EXISTS "${FFMPEG_ROOT}/include/libavformat/avformat.h")
    message(FATAL_ERROR "FFmpeg headers were not found under ${FFMPEG_ROOT}/include")
endif()

if(NOT EXISTS "${FFMPEG_ROOT}/lib/avformat.lib")
    message(FATAL_ERROR "avformat.lib was not found under ${FFMPEG_ROOT}/lib")
endif()

add_executable(ffmpeg_component_overview main.cpp)

target_include_directories(ffmpeg_component_overview PRIVATE
    "${FFMPEG_ROOT}/include"
)

target_link_directories(ffmpeg_component_overview PRIVATE
    "${FFMPEG_ROOT}/lib"
)

target_link_libraries(ffmpeg_component_overview PRIVATE
    avformat
    avcodec
    avutil
)

if(MSVC)
    target_compile_options(ffmpeg_component_overview PRIVATE /W4)
endif()
```

这里的三种路径分别解决三种不同问题：

- `include`：编译器寻找 `.h` 头文件。
- `lib`：链接器寻找 `.lib` 导入库，生成 `.exe` 时使用。
- `bin`：程序运行时寻找 `.dll` 动态库。

### 5.3 Windows 中的 `main.cpp` 示例

这个程序只读取容器和流信息，不解码画面，因此非常适合作为第一个组件实验。保存为 `main.cpp`：

```cpp
#include <cstdint>
#include <iomanip>
#include <iostream>
#include <string>

extern "C" {
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/avutil.h>
#include <libavutil/error.h>
#include <libavutil/mathematics.h>
}

static std::string ffmpeg_error(int error_code) {
    char buffer[AV_ERROR_MAX_STRING_SIZE] = {};
    av_strerror(error_code, buffer, sizeof(buffer));
    return std::string(buffer);
}

static void print_duration(const char* label, std::int64_t value, AVRational time_base) {
    if (value == AV_NOPTS_VALUE || time_base.den == 0) {
        std::cout << label << ": unknown\n";
        return;
    }

    std::cout << label << ": " << std::fixed << std::setprecision(3)
              << value * av_q2d(time_base) << " seconds\n";
}

int main(int argc, char* argv[]) {
    if (argc != 2) {
        std::cerr << "Usage: ffmpeg_component_overview.exe <input-media>\n";
        return 2;
    }

    const std::string input_path = argv[1];
    AVFormatContext* format_context = nullptr;

    int result = avformat_open_input(
        &format_context,
        input_path.c_str(),
        nullptr,
        nullptr
    );
    if (result < 0) {
        std::cerr << "Could not open input: " << input_path << "\n"
                  << ffmpeg_error(result) << "\n";
        return 1;
    }

    result = avformat_find_stream_info(format_context, nullptr);
    if (result < 0) {
        std::cerr << "Could not read stream information: "
                  << ffmpeg_error(result) << "\n";
        avformat_close_input(&format_context);
        return 1;
    }

    std::cout << "FFmpeg version: " << av_version_info() << "\n";
    std::cout << "Input: " << input_path << "\n";
    if (format_context->iformat != nullptr) {
        std::cout << "Container: " << format_context->iformat->name;
        if (format_context->iformat->long_name != nullptr) {
            std::cout << " (" << format_context->iformat->long_name << ")";
        }
        std::cout << "\n";
    }
    print_duration("Format duration", format_context->duration,
                   AVRational{1, AV_TIME_BASE});

    std::cout << "Stream count: " << format_context->nb_streams << "\n";
    for (unsigned int index = 0; index < format_context->nb_streams; ++index) {
        const AVStream* stream = format_context->streams[index];
        const AVCodecParameters* parameters = stream->codecpar;
        const char* media_type = av_get_media_type_string(parameters->codec_type);

        if (media_type == nullptr) {
            media_type = "unknown";
        }

        std::cout << "\nStream #" << index << "\n";
        std::cout << "  Type: " << media_type << "\n";
        std::cout << "  Codec: " << avcodec_get_name(parameters->codec_id) << "\n";
        std::cout << "  Time base: " << stream->time_base.num << "/"
                  << stream->time_base.den << " seconds per tick\n";
        print_duration("  Duration", stream->duration, stream->time_base);

        if (parameters->codec_type == AVMEDIA_TYPE_VIDEO) {
            std::cout << "  Size: " << parameters->width << "x"
                      << parameters->height << "\n";
        } else if (parameters->codec_type == AVMEDIA_TYPE_AUDIO) {
            std::cout << "  Sample rate: " << parameters->sample_rate
                      << " Hz\n";
        }
    }

    // av_dump_format writes a human-readable summary to stderr.
    av_dump_format(format_context, 0, input_path.c_str(), 0);
    avformat_close_input(&format_context);
    return 0;
}
```

代码中的几个关键点：

- `avformat_open_input` 只负责打开并识别输入，不等于已经拿到了所有流信息。
- `avformat_find_stream_info` 让 FFmpeg 读取更多数据来完善流信息。
- `format_context->streams[index]` 是容器内部的流指针，示例只读取它。
- `stream->codecpar` 是流的编码参数，示例用它判断音视频类型、编码名称和尺寸。
- `AV_NOPTS_VALUE` 表示未知时长，代码先检查再换算。
- `avformat_close_input` 是本示例唯一需要主动释放的输入上下文。

## 六、编译、运行和观察结果

### 6.1 用 Visual Studio 生成并编译

在 PowerShell 中进入 `component-overview` 目录：

```powershell
cmake -S . -B build -G "Visual Studio 17 2022" -A x64 -DFFMPEG_ROOT="C:/ffmpeg"
cmake --build build --config Release
```

如果你的 FFmpeg 根目录不同，把 `C:/ffmpeg` 换成实际路径。`-A x64` 表示生成 64 位程序，必须和 FFmpeg 的开发包架构一致。

### 6.2 让 Windows 找到 DLL

当前 PowerShell 窗口临时设置 PATH：

```powershell
$env:Path = "C:\ffmpeg\bin;$env:Path"
```

然后运行：

```powershell
.\build\Release\ffmpeg_component_overview.exe "C:\media\sample.mp4"
```

也可以把 `C:\ffmpeg\bin` 中程序需要的 DLL 复制到 `.exe` 同一目录。学习时优先使用临时 PATH，方便切换不同版本。

### 6.3 先用命令行交叉验证

在运行 C++ 程序前，先确认输入文件本身正常：

```powershell
ffmpeg -hide_banner -version
ffprobe -hide_banner -show_format -show_streams "C:\media\sample.mp4"
ffplay "C:\media\sample.mp4"
```

如果 `ffprobe` 能读而 C++ 程序不能读，优先检查 C++ 的 DLL、链接版本和路径。如果两者都不能读，先检查文件路径、文件权限和文件本身。

### 6.4 你应该看到什么

不同文件和 FFmpeg 版本的数字会不同，但结构应该类似：

```text
FFmpeg version: 7.x.x
Input: C:\media\sample.mp4
Container: mov,mp4,m4a,3gp,3g2,mj2 (QuickTime / MOV)
Format duration: 12.480 seconds
Stream count: 2

Stream #0
  Type: video
  Codec: h264
  Time base: 1/90000 seconds per tick
  Duration: 12.480 seconds
  Size: 1920x1080

Stream #1
  Type: audio
  Codec: aac
  Time base: 1/48000 seconds per tick
  Duration: 12.459 seconds
  Sample rate: 48000 Hz
```

最后的 `av_dump_format` 还会向终端输出一份更详细的摘要。音视频时长不完全相同很常见，不要因为几毫秒差异就认为程序错了。

### 6.5 如何读懂时间基

如果视频流的时间基是 `1/90000`，一个时间单位就是 `1/90000` 秒；如果某个时间戳是 `900000`，对应秒数就是：

```text
900000 * (1 / 90000) = 10 秒
```

代码中的 `av_q2d` 把 `AVRational` 转成小数，后续章节会用 `av_rescale_q` 在不同时间基之间做精确的整数换算。详见 [[03-FFmpeg基础/时间戳与时间基]]。

## 七、先用命令行认识组件

下面每个命令都可以看成对某一层组件的观察。

### 7.1 查看整体版本和构建信息

```powershell
ffmpeg -hide_banner -version
```

重点观察：版本号、配置选项、是否包含某些编码器或设备支持。`--enable-gpl`、`--enable-nonfree` 等构建选项还会影响分发许可，做产品时要记录你使用的构建来源和许可证。

### 7.2 只查看流信息

```powershell
ffprobe -v error -show_entries "format=format_name,duration:stream=index,codec_type,codec_name,width,height,sample_rate" -of json "C:\media\sample.mp4"
```

把输出分成两层看：`format` 是容器级信息，`stream` 是每条流的信息。JSON 只是输出格式，不代表媒体文件本身是 JSON。

### 7.3 做一个不输出文件的处理

```powershell
ffmpeg -hide_banner -i "C:\media\sample.mp4" -f null -
```

这个命令读取输入并把结果丢到 `null`，适合确认 FFmpeg 是否能打开、解码和处理文件。最后的 `-` 表示使用标准输出；在 Windows PowerShell 中不要把它误解成文件名。

## 八、常见错误和排查顺序

遇到问题时按“路径 -> 版本 -> 架构 -> 运行时 -> 输入文件”的顺序检查，通常比盲目修改代码快。

| 现象 | 常见原因 | 排查和修复 |
|---|---|---|
| `fatal error C1083: Cannot open include file: 'libavformat/avformat.h'` | 头文件目录没有加入编译器 | 检查 `FFMPEG_ROOT/include` 是否存在，并确认 CMake 指向的是开发包根目录 |
| `LNK1104: cannot open file 'avformat.lib'` | `.lib` 目录错误，或开发包不匹配 | 检查 `FFMPEG_ROOT/lib/avformat.lib`，不要只下载运行时 DLL |
| 启动时提示缺少 `avformat-*.dll` | Windows 找不到运行时 DLL | 在当前 PowerShell 设置 `C:\ffmpeg\bin` 到 PATH，或复制 DLL 到 exe 旁边 |
| `0xc000007b` | 32/64 位混用，或 DLL 与 exe 架构不同 | `cmake` 使用 `-A x64`，并确保 FFmpeg 是 x64；不要混用 x86 DLL |
| 程序打印 `Usage` | 没有传入输入文件参数 | 从终端运行，并在路径有空格时使用双引号 |
| `No such file or directory` | 路径拼写错误、当前目录不对或文件不存在 | 用 `Test-Path "C:\media\sample.mp4"` 检查 |
| `ffmpeg` 不是内部或外部命令 | FFmpeg 的 `bin` 目录没有加入 PATH | 用 `Get-Command ffmpeg` 检查；临时执行 `$env:Path = "C:\ffmpeg\bin;$env:Path"` |
| `ffplay` 找不到 | 当前 FFmpeg 发行版没有包含播放器程序 | 用 `ffprobe` 查看或用 `ffmpeg -f null -` 验证，不要把它当成库链接错误 |
| `Invalid data found when processing input` | 文件损坏、不是媒体文件、协议或构建不支持 | 先用 `ffprobe` 验证；不要只根据扩展名判断格式 |
| `Could not read stream information` | 文件截断、输入不可读或网络输入不完整 | 先换成本地完整文件；网络选项放到网络章节学习 |
| 时长显示 `unknown` | 文件没有可靠的 duration 或时间基无效 | 这是合法情况，使用时间戳逐步处理，不要把未知当成 0 |
| 教程中的 `av_register_all` 找不到 | 教程版本太旧 | 删除全局注册调用，使用当前 FFmpeg API |
| 链接时出现大量未解析符号 | 把静态库当共享库使用，或库版本/编译器不匹配 | 第一次实验使用同一发行版的共享开发包；静态链接留到工程化章节 |
| `ffprobe` 可以运行，C++ 程序不行 | 命令行程序和 C++ 使用的 DLL 不是同一套 | 检查 PATH 顺序、CMake 的 include/lib 路径和 exe 旁边的 DLL |

## 九、练习题

### 练习 1：组件归类

把下面任务分别归到一个主要组件：读取 MP4、H.264 解码、YUV 转 BGRA、48 kHz 转 44.1 kHz、裁剪视频、读取摄像头。

**参考答案**：依次是 `libavformat`、`libavcodec`、`libswscale`、`libswresample`、`libavfilter`、`libavdevice`。

### 练习 2：容器和编码器

用 `ffprobe` 找一个 `.mp4` 文件，写下 `format_name`、视频 `codec_name` 和音频 `codec_name`。再回答：“把文件扩展名改成 `.mkv` 会不会自动改变编码器？”

**检查点**：不会。改扩展名不会重新封装，更不会改变容器内部结构；应使用 FFmpeg 命令真正转换或重新封装。

### 练习 3：修改示例输出

修改 `main.cpp`，只打印视频流；再修改为只打印音频流。不要删除遍历所有流的逻辑，只在循环中增加判断。

### 练习 4：时间基换算

选择一个视频流，打印 `stream->time_base` 和 `stream->duration`，手算时长，再和程序输出比较。解释为什么音频和视频的时间基可能不同。

### 练习 5：故意制造 DLL 错误

在新的 PowerShell 窗口中不设置 FFmpeg 的 PATH，运行程序并记录错误；再设置 PATH 后运行。把“编译成功”和“运行成功”分别写进问题记录。

### 练习 6：画数据流

不用看本章的图，手写下面这条链：MP4 文件、`libavformat`、`AVPacket`、`libavcodec`、`AVFrame`、`libswscale`、输出窗口。为每个箭头写一句“数据发生了什么变化”。

### 练习 7：比较 `ffprobe` 和 C++ 输出

选同一个文件，把 `ffprobe` 的流信息和 C++ 程序输出逐项对照。找出至少一个 C++ 当前没有打印的字段，并说明它属于容器层还是流层。

### 练习 8：阅读后续 API

预习 [[04-C++调用FFmpeg/AVFormatContext]] 和 [[04-C++调用FFmpeg/AVFrame与AVPacket]]，回答：为什么 `AVPacket` 和 `AVFrame` 都需要清理？为什么 `AVStream` 通常不用单独清理？

## 十、阶段检查清单

完成一项就勾选一项。不会只凭“看过”勾选，尽量用口头解释或实际命令证明。

- [ ] 我能用自己的话解释 FFmpeg 项目、`ffmpeg.exe` 和 `libavformat` 的区别。
- [ ] 我能区分容器、流、编码器、数据包和帧。
- [ ] 我能说出 `ffmpeg`、`ffprobe`、`ffplay` 各自适合的任务。
- [ ] 我能把打开 MP4、H.264 解码、视频缩放、音频重采样和滤镜处理映射到正确的库。
- [ ] 我能解释 `libavformat` 和 `libavcodec` 的先后关系。
- [ ] 我能解释 `AVFormatContext`、`AVStream` 和 `AVCodecParameters` 的关系。
- [ ] 我知道哪些 FFmpeg 指针是借用的，哪些是自己申请后必须释放的。
- [ ] 我能解释 `.h`、`.lib` 和 `.dll` 分别在 Windows 的哪一步发挥作用。
- [ ] 我能用 CMake 生成 x64 的 Visual Studio 工程并编译示例。
- [ ] 我能通过 PATH 或复制 DLL 解决运行时缺少 DLL 的问题。
- [ ] 我能运行示例，读懂容器、流数量、编码名称、时长和时间基。
- [ ] 我能解释 `AV_NOPTS_VALUE` 为什么不能直接当作 0 秒。
- [ ] 我能画出从输入文件到输出文件的主数据流。
- [ ] 我完成了至少三道练习，并把一个错误记录到 [[12-问题记录与复盘/问题记录模板]]。

如果有三项以上无法完成，回到对应的小节重新做一次命令或代码实验。完成清单后再进入 [[03-FFmpeg基础/FFmpeg命令行工具]]、[[03-FFmpeg基础/媒体文件与容器]] 和 [[03-FFmpeg基础/时间戳与时间基]]。

## 十一、与后续章节的关系

```text
本章：组件地图和第一次信息读取
  |
  +--> 命令行工具：熟悉 ffmpeg/ffprobe 参数
  +--> 媒体文件与容器：深入 MP4、MKV、流和封装
  +--> 时间戳与时间基：深入 PTS、DTS、duration 和同步
  +--> C++ 调用 FFmpeg：学习对象、错误码和资源生命周期
          |
          +--> 解封装与解码：AVPacket -> AVFrame
          +--> 编码与封装：AVFrame -> AVPacket -> 输出容器
```

本章的示例故意停在“读取信息”。这是一个明确的学习边界：先确认你能识别组件和数据层，再进入会分配更多对象、处理更多错误路径的解码代码。

## 参考资料

- FFmpeg 官方文档：<https://ffmpeg.org/documentation.html>
- FFmpeg 官方 `ffmpeg` 文档：<https://ffmpeg.org/ffmpeg.html>
- FFmpeg 官方 `ffprobe` 文档：<https://ffmpeg.org/ffprobe.html>
- FFmpeg Doxygen API 文档：<https://ffmpeg.org/doxygen/trunk/index.html>
- 知识库中的 [[02-Windows开发环境/FFmpeg构建与安装]]、[[04-C++调用FFmpeg/API调用流程]]、[[12-问题记录与复盘/FFmpeg术语表]]
