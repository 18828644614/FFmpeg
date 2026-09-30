---
type: topic
status: draft
created: 2026-09-29
updated: 2026-09-30
tags:
  - ffmpeg
  - cpp
  - windows
  - api
---

# API调用流程

本章先建立一张“程序怎样一步步请求 FFmpeg 工作”的地图，再通过一个可以编译运行的 C++ 小程序亲手走完“打开媒体文件、找到视频流、解码并统计视频帧、清理资源”的过程。第一次学习时不需要背住所有函数名；重点是知道每一步为什么存在、上一步交给下一步什么东西，以及哪里必须检查错误和释放资源。

## 本章学习范围与顺序

按下面的顺序学习。先完成前四步和实践程序，再回头读解码与输出的全景；这样不会一开始就被许多 API 名称淹没。

| 顺序 | 学什么 | 学完后你应该能做什么 | 后续深入页面 |
|---|---|---|---|
| 1 | 媒体文件、容器、流、包、帧 | 说清文件里有哪些轨道，以及压缩数据和解码画面的区别 | [[03-FFmpeg基础/媒体文件与容器]] |
| 2 | FFmpeg 各个库各负责什么 | 知道打开文件、解码、转格式大致该找哪个库 | [[03-FFmpeg基础/FFmpeg组件概览]] |
| 3 | Windows 工程里的头文件、.lib、.dll | 能区分编译、链接、运行三个阶段 | [[02-Windows开发环境/FFmpeg构建与安装]] |
| 4 | FFmpeg API 的通用调用模式 | 看懂“创建/打开 -> 使用 -> 收尾/释放” | [[04-C++调用FFmpeg/错误处理与资源生命周期]] |
| 5 | 打开媒体并读取流信息 | 调用 `avformat_open_input` 和 `avformat_find_stream_info` | [[04-C++调用FFmpeg/AVFormatContext]] |
| 6 | 读压缩数据包并选择目标流 | 理解 `av_read_frame`、`stream_index`、`av_packet_unref` | [[05-解封装与解码/解封装流程]] |
| 7 | 把数据包送入解码器并取回帧 | 理解 `avcodec_send_packet`、`avcodec_receive_frame`、缓冲和冲刷 | [[05-解封装与解码/视频解码流程]]、[[05-解封装与解码/音频解码流程]] |
| 8 | 认识输出、编码和封装的顺序 | 能区分只换容器（remux）与重新编码（transcode） | [[06-编码与封装/封装输出流程]]、[[06-编码与封装/转码流程]] |
| 9 | 错误检查、时间戳和资源生命周期 | 能判断错误发生在哪一层，并确保每个自有对象都被释放 | [[04-C++调用FFmpeg/错误处理与资源生命周期]]、[[03-FFmpeg基础/时间戳与时间基]] |

本章的可运行程序会详细做到第 1 至第 7 步中的“输入文件 + 视频解码”。编码器配置、音视频同步、滤镜、画面显示和最终写文件会先画出流程并说明入口，分别在后续章节深入。先理解一条数据怎样经过 FFmpeg，再逐个学习每个处理模块。

## 前置知识

### 先认识几种媒体对象

把一个媒体文件想成一个装着多条轨道的盒子：

- **容器（container）**：文件整体的包装格式，例如 MP4、MKV、MOV。容器可以同时装视频、音频、字幕和元数据。
- **流（stream）**：容器中的一条轨道。例如一条 H.264 视频流、一条 AAC 音频流。程序用 `AVStream` 表示它。
- **数据包（packet）**：从文件中读出的压缩数据块。它通常属于某一条流，带有流编号和时间信息，由 `AVPacket` 表示。
- **帧（frame）**：解码后的图像或一段音频采样，由 `AVFrame` 表示。压缩视频包和解码后的画面不是一回事。
- **解码器（decoder）**：按 H.264、AAC 等编码规则把压缩包还原成帧的程序。`AVCodecContext` 保存一次解码工作的状态。

别假设“一个包一定得到一帧”。一个压缩包可能没有可立即显示的画面，也可能让解码器产出多帧；有些解码器会暂存画面，直到收到更多数据或结束信号。

### C++ 读代码时需要会的几个词

- **函数参数**：调用函数时传进去的信息，例如输入文件名。
- **返回值**：函数执行后交回的结果。FFmpeg 的许多函数返回 `0` 或非负值表示成功，负值表示错误。
- **指针**：可以先把它理解成“某个对象在内存中的地址”。FFmpeg 用指针传递上下文、包和帧。
- **取地址符 `&`**：取得变量本身的位置。FFmpeg 需要创建对象并把新地址写回时，常见 `AVFormatContext**` 参数，所以调用处会写 `&format_context`。
- **枚举值**：一组有名字的选项，例如 `AVMEDIA_TYPE_VIDEO` 表示视频，`AVMEDIA_TYPE_AUDIO` 表示音频。
- **循环**：重复处理数据。媒体文件通常包含很多包，因此要循环读取，直到文件结束或发生错误。

本章代码里的指针和错误检查会逐段解释。若还不熟悉 C++ 指针、函数和循环，可先看 [[01-前置基础/C++基础]]。

## 先看完整流程地图

### 输入并解码

~~~text
媒体文件
  |
  v
打开输入
avformat_open_input
  |
  v
读取容器和流信息
avformat_find_stream_info
  |
  v
选择要处理的流
av_find_best_stream
  |
  v
为该流准备并打开解码器
avcodec_alloc_context3
avcodec_parameters_to_context / avcodec_open2
  |
  v
反复读压缩数据包
av_read_frame
  |
  +-- 不是目标流：释放这个包
  |
  +-- 是目标流：avcodec_send_packet
                    |
                    v
             avcodec_receive_frame
                    |
                    v
          得到零个、一个或多个 AVFrame
  |
  v
文件末尾后冲刷解码器，取出暂存帧
avcodec_send_packet(decoder, nullptr)
  |
  v
释放帧、包、解码上下文、输入上下文
~~~

### 如果还要输出文件

**重新编码（转码）**的大致方向是：

~~~text
输入文件 -> 解封装 -> 解码 -> 可选处理 -> 编码 -> 封装 -> 输出文件
~~~

**只换容器但不改变压缩内容（重新封装 / remux）**通常是：

~~~text
输入文件 -> 解封装 -> 调整流参数和时间戳 -> 封装 -> 输出文件
~~~

重新封装不经过解码器和编码器；转码会经过它们。不要看到“输入 -> 输出”就以为一定要解码再编码。

## FFmpeg 的 API 按什么分工

FFmpeg 是一组 C 语言库。C++ 程序可以调用它们，但要包含相应头文件，并在链接时加入相应库。

| 库 | 通俗职责 | 初学时常见对象和函数 |
|---|---|---|
| `libavformat` | 打开容器，读取流和包，写出容器 | `AVFormatContext`、`AVStream`、`avformat_open_input`、`av_read_frame` |
| `libavcodec` | 解码或编码压缩数据 | `AVCodec`、`AVCodecContext`、`AVPacket`、`AVFrame`、`avcodec_send_packet` |
| `libavutil` | 多个库共用的基础类型、时间和错误工具 | `AVRational`、`av_strerror`、`AVERROR` |
| `libswscale` | 转换视频尺寸或像素格式 | `SwsContext`、`sws_scale` |
| `libswresample` | 转换音频采样率、采样格式或声道布局 | `SwrContext`、`swr_convert` |
| `libavfilter` | 把裁剪、缩放、音量等处理连接成滤镜图 | `AVFilterGraph`、`AVFilterContext` |

这章的实验使用前三个库。视频画面怎么显示、音频怎么播放、怎么缩放和重采样都不属于这次实验的目标。

## API 调用的通用规律

大多数 FFmpeg 工作都能拆成下面五件事：

1. **创建或打开**：准备上下文或打开输入，例如 `avformat_open_input`。
2. **补齐配置**：告诉 FFmpeg 输入是什么，或从流参数复制解码器配置，例如 `avcodec_parameters_to_context`。
3. **正式初始化**：让组件准备开始工作，例如 `avcodec_open2`。
4. **循环处理数据**：读包、送包、取帧，直到结束或错误。
5. **冲刷并释放**：取出缓冲的数据，再按所有权规则释放自己创建的对象。

不是所有 API 都严格分成五个函数，但“先准备、再使用、最后收尾”这个思路很通用。

### 为什么 C++ 代码有 extern "C"

FFmpeg 的函数用 C 语言编译。C++ 编译器会用不同规则记录函数名称；`extern "C"` 告诉它，下面这些头文件里的函数要按 C 的规则连接。Windows/MSVC 项目一般这样包含 FFmpeg 头文件：

~~~cpp
extern "C" {
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libavutil/error.h>
}
~~~

如果漏掉 `extern "C"`，有时会在链接阶段看到找不到 FFmpeg 函数的错误。

### 为什么函数会收到 &format_context

先看声明和调用：

~~~cpp
AVFormatContext* format_context = nullptr;
avformat_open_input(&format_context, input_path, nullptr, nullptr);
~~~

`format_context` 是一个指针变量，起初还没有指向输入上下文。函数收到这个变量的位置后，才能把打开后的上下文地址写进去。因此调用时传 `&format_context`。这不是多余的符号，也不等同于把对象复制了一份。

### 如何判断函数是否成功

常见写法是保存返回值并立即检查：

~~~cpp
int result = avformat_open_input(&format_context, input_path, nullptr, nullptr);
if (result < 0) {
    // result 是 FFmpeg 的负数错误码
}
~~~

具体规则看对应函数的文档，但负数通常代表失败。用 `av_strerror` 可以把错误码翻译成人能看懂的文字。不是所有“看起来像零”的值都代表失败：`av_read_frame` 在读到末尾时返回 `AVERROR_EOF`，这是循环正常结束的信号。

## 按顺序拆解输入与解码 API

### 第一步：打开输入

~~~cpp
AVFormatContext* format_context = nullptr;

int result = avformat_open_input(
    &format_context, // 函数会把打开后的上下文写到这里
    input_path,      // 本地路径或支持的输入地址
    nullptr,         // nullptr 表示让 FFmpeg 自动识别格式
    nullptr          // 本次不传额外选项
);
~~~

成功后，`format_context` 包含这次输入任务的总体信息。这个函数主要负责打开输入和识别容器；不要把它理解成“所有流信息已经读取完成”。

### 第二步：读取流信息

~~~cpp
result = avformat_find_stream_info(format_context, nullptr);
~~~

有些容器的流信息需要先读取一部分数据才能确定。这个函数会尝试收集这些信息，之后再检查 `format_context->nb_streams` 和 `format_context->streams`。读取网络输入时它可能花较长时间；本章先使用本地文件。

### 第三步：选择一条流和对应解码器

一个文件可能有多个视频轨道，例如不同语言或不同清晰度。可以先让 FFmpeg 选择一条适合的流：

~~~cpp
const AVCodec* decoder = nullptr;
int video_stream_index = av_find_best_stream(
    format_context,
    AVMEDIA_TYPE_VIDEO,
    -1,       // 不指定某一条固定编号
    -1,       // 暂不要求与另一条流关联
    &decoder, // FFmpeg 会在这里提供匹配的解码器
    0
);
~~~

返回值是流编号，不是视频宽度，也不是包的编号。若返回负数，说明没有找到合适的视频流或发生错误。音频解码时把 `AVMEDIA_TYPE_VIDEO` 改为 `AVMEDIA_TYPE_AUDIO`。

### 第四步：准备并打开解码器

流的 `codecpar` 保存容器提供的编码参数；解码器的 `AVCodecContext` 保存实际解码时使用的状态。先申请上下文，再把参数复制进去，最后打开解码器：

~~~cpp
AVCodecContext* decoder_context = avcodec_alloc_context3(decoder);
avcodec_parameters_to_context(
    decoder_context,
    format_context->streams[video_stream_index]->codecpar
);
avcodec_open2(decoder_context, decoder, nullptr);
~~~

正式程序必须检查每一步的返回值以及内存分配结果。上下文是自己申请的对象，工作完成后要用 `avcodec_free_context` 释放，不能用 C++ 的 `delete`。

### 第五步：读取压缩包

`av_read_frame` 每次从容器中取一个包，通常不会保证取出的包属于视频。包上的 `stream_index` 表示它属于哪条流。

~~~cpp
while ((result = av_read_frame(format_context, packet)) >= 0) {
    if (packet->stream_index == video_stream_index) {
        // 只把视频包交给这个视频解码器
    }

    av_packet_unref(packet);
}
~~~

每次处理完一个包都要调用 `av_packet_unref`，包括不属于目标流的包。它会释放或归还包所持有的数据，但保留 `AVPacket` 结构本身供下一次读取复用。

### 第六步：向解码器送包并取帧

现代 FFmpeg 解码器使用一对接口：

- `avcodec_send_packet`：把一个压缩包交给解码器。
- `avcodec_receive_frame`：取出当前已经准备好的解码帧。

收到一个包后要持续取帧，直到 `avcodec_receive_frame` 返回 `AVERROR(EAGAIN)`。这个错误的意思是“现在没有更多输出，请再提供输入”，在这个位置它是正常流程，不表示文件坏了。

~~~cpp
result = avcodec_send_packet(decoder_context, packet);
if (result >= 0) {
    while (true) {
        result = avcodec_receive_frame(decoder_context, frame);

        if (result == AVERROR(EAGAIN)) {
            break; // 需要更多输入包
        }
        if (result == AVERROR_EOF) {
            break; // 解码器已经结束
        }
        if (result < 0) {
            // 真正的解码错误
            break;
        }

        // 这里拿到一帧解码结果
        av_frame_unref(frame);
    }
}
~~~

送入解码器的包和取出的帧是两种不同对象。不要用 `packet->size` 当作画面大小，也不要用包数量推断解码帧数量。

**关于送包返回 EAGAIN：**这表示解码器要求调用者先取出待处理的帧。先调用 `avcodec_receive_frame` 取到它返回 `EAGAIN` 为止，再把同一个尚未释放的包重新发送。正确的收发循环会保留这个包，不会在重试前调用 `av_packet_unref`。可运行示例包含了这一步。

### 第七步：文件结束时冲刷解码器

读到 `AVERROR_EOF` 只表示容器没有更多包。解码器内部可能还暂存着 B 帧或其他延迟输出，因此还要明确告诉解码器“不再有新包”：

~~~cpp
avcodec_send_packet(decoder_context, nullptr);
~~~

随后继续调用 `avcodec_receive_frame`，直到它返回 `AVERROR_EOF`。空指针是解码接口约定的“冲刷”信号，不是空媒体文件。忘记冲刷可能会漏掉最后几帧。编码时也有类似收尾：发送空帧，再取出编码器缓冲的包。

### 第八步：关闭和释放

输入上下文、解码器上下文、包和帧的释放函数不同：

| 对象 | 谁创建或拥有 | 完成后怎么处理 |
|---|---|---|
| `AVFormatContext` | `avformat_open_input` 打开输入 | `avformat_close_input(&format_context)` |
| `AVStream`、`codecpar` | 属于输入上下文 | 通常只读取，不单独释放 |
| `AVCodecContext` | `avcodec_alloc_context3` | `avcodec_free_context(&decoder_context)` |
| `AVPacket` | `av_packet_alloc` | `av_packet_free(&packet)`；每包内容用 `av_packet_unref` 归还 |
| `AVFrame` | `av_frame_alloc` | `av_frame_free(&frame)`；每帧内容用 `av_frame_unref` 归还 |

“解引用内容”和“释放对象”不是一回事：`av_packet_unref` 清掉当前包的数据；`av_packet_free` 连包结构本身也释放。后面的 [[04-C++调用FFmpeg/错误处理与资源生命周期]] 会详细介绍所有权和 C++ RAII。

## Windows + C++ + FFmpeg 实践：统计视频帧数

这个程序会：

1. 打开一个本地媒体文件并读取流信息；
2. 找到 FFmpeg 认为合适的视频流和解码器；
3. 逐包读取，只把目标视频流的包交给解码器；
4. 不保存图像，只统计收到的帧数；
5. 文件读完后冲刷解码器，并自动释放资源。

“帧数”是解码器实际输出的帧数。因为可变帧率、输入是否完整等原因，它不一定等于“文件时长乘以标称帧率”。

### 准备目录

安装或准备一个与 MSVC、x64 匹配的 FFmpeg **开发包**，目录包含头文件、导入库和 DLL。以下假设它放在 `C:/ffmpeg`：

~~~text
C:/ffmpeg/
  include/libavformat/avformat.h
  include/libavcodec/avcodec.h
  include/libavutil/avutil.h
  lib/avformat.lib
  lib/avcodec.lib
  lib/avutil.lib
  bin/（运行需要的 FFmpeg DLL）
~~~

只有 `ffmpeg.exe` 不够：编译 C++ 程序还需要头文件和 `.lib`。MSVC 和 FFmpeg 的位数要匹配。有关下载和安装方式见 [[02-Windows开发环境/FFmpeg构建与安装]]。

新建工作目录，例如 `C:/work/api-flow`，在里面创建 `CMakeLists.txt` 和 `main.cpp`。

### CMakeLists.txt

~~~cmake
cmake_minimum_required(VERSION 3.20)

project(api_flow LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(FFMPEG_ROOT "" CACHE PATH "FFmpeg development package root")

if(NOT EXISTS "${FFMPEG_ROOT}/include/libavformat/avformat.h")
    message(FATAL_ERROR "FFmpeg headers were not found under FFMPEG_ROOT/include")
endif()

foreach(required_library avformat avcodec avutil)
    if(NOT EXISTS "${FFMPEG_ROOT}/lib/${required_library}.lib")
        message(FATAL_ERROR "Missing FFmpeg import library: ${required_library}.lib")
    endif()
endforeach()

add_executable(api_flow main.cpp)

target_include_directories(api_flow PRIVATE
    "${FFMPEG_ROOT}/include"
)

target_link_directories(api_flow PRIVATE
    "${FFMPEG_ROOT}/lib"
)

target_link_libraries(api_flow PRIVATE
    avformat
    avcodec
    avutil
)

if(MSVC)
    target_compile_options(api_flow PRIVATE /W4)
endif()
~~~

`include` 供编译器找头文件，`lib` 供链接器找 `.lib`，`bin` 中的 DLL 则在程序启动时由 Windows 查找。这三个目录解决的是三个不同阶段的问题。

### main.cpp

~~~cpp
#include <cerrno>
#include <cstdint>
#include <iostream>

extern "C" {
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/avutil.h>
#include <libavutil/error.h>
}

static void print_ffmpeg_error(const char* operation, int error_code) {
    char buffer[AV_ERROR_MAX_STRING_SIZE] = {};
    av_strerror(error_code, buffer, sizeof(buffer));
    std::cerr << operation << " failed: " << buffer << std::endl;
}

// 取出当前能拿到的所有帧。
// 返回 AVERROR(EAGAIN) 表示需要更多包，AVERROR_EOF 表示解码结束。
static int receive_available_frames(
    AVCodecContext* decoder,
    AVFrame* frame,
    std::uint64_t& frame_count
) {
    while (true) {
        int result = avcodec_receive_frame(decoder, frame);

        if (result == AVERROR(EAGAIN) || result == AVERROR_EOF) {
            return result;
        }
        if (result < 0) {
            return result;
        }

        ++frame_count;
        if (frame_count == 1) {
            std::cout << "First decoded frame: "
                      << frame->width << "x" << frame->height
                      << std::endl;
        }

        // 归还这一帧的数据，frame 结构可以继续接收下一帧。
        av_frame_unref(frame);
    }
}

// 送入一个包，并尽可能取出由它产生的帧。
static int decode_packet(
    AVCodecContext* decoder,
    const AVPacket* packet,
    AVFrame* frame,
    std::uint64_t& frame_count
) {
    int result = avcodec_send_packet(decoder, packet);

    if (result == AVERROR(EAGAIN)) {
        // 解码器要求先取走待处理帧；packet 仍保留，取完后重试。
        result = receive_available_frames(decoder, frame, frame_count);
        if (result != AVERROR(EAGAIN)) {
            return result;
        }
        result = avcodec_send_packet(decoder, packet);
    }

    if (result < 0) {
        return result;
    }

    result = receive_available_frames(decoder, frame, frame_count);
    if (result == AVERROR(EAGAIN)) {
        return 0; // 正常：等待后续输入
    }
    return result;
}

// 这个小守卫会在 main 返回时调用 FFmpeg 对应的释放函数。
// AVStream 和 codecpar 属于 format_context，不在这里单独释放。
struct Resources {
    AVFormatContext* format_context = nullptr;
    AVCodecContext* decoder_context = nullptr;
    AVPacket* packet = nullptr;
    AVFrame* frame = nullptr;

    ~Resources() {
        av_frame_free(&frame);
        av_packet_free(&packet);
        avcodec_free_context(&decoder_context);
        avformat_close_input(&format_context);
    }
};

int main(int argc, char* argv[]) {
    if (argc != 2) {
        std::cerr << "Usage: api_flow.exe <input-media>" << std::endl;
        return 2;
    }

    const char* input_path = argv[1];
    Resources resources;
    const AVCodec* decoder = nullptr;
    std::uint64_t frame_count = 0;

    int result = avformat_open_input(
        &resources.format_context,
        input_path,
        nullptr,
        nullptr
    );
    if (result < 0) {
        print_ffmpeg_error("avformat_open_input", result);
        return 1;
    }

    result = avformat_find_stream_info(resources.format_context, nullptr);
    if (result < 0) {
        print_ffmpeg_error("avformat_find_stream_info", result);
        return 1;
    }

    std::cout << "FFmpeg version: " << av_version_info() << std::endl;
    av_dump_format(resources.format_context, 0, input_path, 0);

    const int video_stream_index = av_find_best_stream(
        resources.format_context,
        AVMEDIA_TYPE_VIDEO,
        -1,
        -1,
        &decoder,
        0
    );
    if (video_stream_index < 0) {
        print_ffmpeg_error("av_find_best_stream (video)", video_stream_index);
        return 1;
    }
    if (decoder == nullptr) {
        std::cerr << "No decoder was found for the selected video stream."
                  << std::endl;
        return 1;
    }

    resources.decoder_context = avcodec_alloc_context3(decoder);
    if (resources.decoder_context == nullptr) {
        std::cerr << "Could not allocate a decoder context." << std::endl;
        return 1;
    }

    result = avcodec_parameters_to_context(
        resources.decoder_context,
        resources.format_context->streams[video_stream_index]->codecpar
    );
    if (result < 0) {
        print_ffmpeg_error("avcodec_parameters_to_context", result);
        return 1;
    }

    result = avcodec_open2(resources.decoder_context, decoder, nullptr);
    if (result < 0) {
        print_ffmpeg_error("avcodec_open2", result);
        return 1;
    }

    resources.packet = av_packet_alloc();
    resources.frame = av_frame_alloc();
    if (resources.packet == nullptr || resources.frame == nullptr) {
        std::cerr << "Could not allocate a packet or frame." << std::endl;
        return 1;
    }

    // 每次成功读取一个包后，都要处理它并归还包数据。
    while ((result = av_read_frame(
                resources.format_context,
                resources.packet
            )) >= 0) {
        if (resources.packet->stream_index == video_stream_index) {
            result = decode_packet(
                resources.decoder_context,
                resources.packet,
                resources.frame,
                frame_count
            );
            av_packet_unref(resources.packet);

            if (result < 0) {
                print_ffmpeg_error("decoding video packet", result);
                return 1;
            }
        } else {
            // 文件中的音频、字幕等包不是这个视频解码器的输入。
            av_packet_unref(resources.packet);
        }
    }

    if (result != AVERROR_EOF) {
        print_ffmpeg_error("av_read_frame", result);
        return 1;
    }

    // 容器读完不代表解码器已吐出所有缓存帧，发送空包开始冲刷。
    result = avcodec_send_packet(resources.decoder_context, nullptr);
    if (result == AVERROR(EAGAIN)) {
        result = receive_available_frames(
            resources.decoder_context,
            resources.frame,
            frame_count
        );
        if (result == AVERROR(EAGAIN)) {
            result = avcodec_send_packet(resources.decoder_context, nullptr);
        }
    }
    if (result < 0 && result != AVERROR_EOF) {
        print_ffmpeg_error("flushing decoder", result);
        return 1;
    }

    if (result != AVERROR_EOF) {
        while (true) {
            result = receive_available_frames(
                resources.decoder_context,
                resources.frame,
                frame_count
            );

            if (result == AVERROR_EOF) {
                break;
            }
            if (result == AVERROR(EAGAIN)) {
                std::cerr << "Decoder requested more input during flush."
                          << std::endl;
                return 1;
            }
            if (result < 0) {
                print_ffmpeg_error("receiving flushed frame", result);
                return 1;
            }
        }
    }

    std::cout << "Video stream index: " << video_stream_index << std::endl;
    std::cout << "Decoded video frames: " << frame_count << std::endl;
    return 0;
}
~~~

### 逐段理解程序

1. `Resources` 保存程序自己创建的四个对象。函数返回时析构函数会按顺序释放它们，所以中途报错 `return` 也不会漏掉已经创建的对象。这是 C++ 的自动清理习惯，称为 RAII；先记住“资源跟着对象离开作用域时释放”即可。
2. `avformat_open_input` 打开文件；`avformat_find_stream_info` 收集流信息；`av_dump_format` 把识别结果写到终端，方便对照。
3. `av_find_best_stream` 返回视频流编号，并找出可用解码器。编号之后用于判断每个包属于哪条流。
4. `avcodec_alloc_context3` 申请解码器状态；`avcodec_parameters_to_context` 复制流参数；`avcodec_open2` 初始化解码器。三个步骤各有作用，不能只申请上下文就开始解码。
5. `av_read_frame` 逐个读包。只把视频流的包送入视频解码器；音频和字幕包仍然要 `av_packet_unref`，只是这次不处理。
6. `decode_packet` 先送包，再反复收帧。遇到“请先接收输出”的 `EAGAIN` 时会先收帧，再用仍然有效的同一个包重试。
7. `av_read_frame` 返回 `AVERROR_EOF` 后，程序发送空包并取出延迟帧。`receive_available_frames` 返回 `AVERROR(EAGAIN)` 表示这一轮输出已取完；冲刷结束时应收到 `AVERROR_EOF`。
8. `av_frame_unref` 和 `av_packet_unref` 归还每一轮数据；`Resources` 的析构函数最终释放结构和上下文。

本程序只解码视频并统计帧数，没有保存图片，也没有播放声音。要将帧保存为图片，需要增加像素格式转换或图像编码步骤；完整播放器还要处理音频、时钟和音视频同步。

### 编译和运行

先安装 Visual Studio 2022 的“使用 C++ 的桌面开发”组件、CMake，以及同一套 FFmpeg 开发文件。下面命令在 PowerShell 中执行，并假设：

- `C:/work/api-flow` 里有 `CMakeLists.txt` 和 `main.cpp`；
- FFmpeg 开发包放在 `C:/ffmpeg`；
- 测试文件放在 `C:/media/sample.mp4`；
- MSVC、FFmpeg 和 CMake 都是 x64。

~~~powershell
cd C:/work/api-flow

cmake -S . -B build -G "Visual Studio 17 2022" -A x64 -DFFMPEG_ROOT="C:/ffmpeg"
cmake --build build --config Release

# 当前 PowerShell 会话临时增加 DLL 搜索目录
$env:Path = "C:/ffmpeg/bin;$env:Path"

./build/Release/api_flow.exe "C:/media/sample.mp4"
~~~

如果 FFmpeg 不在 `C:/ffmpeg`，把 `FFMPEG_ROOT` 和 DLL 目录换成实际位置。路径中有空格时要加双引号。为减少 Windows 窄字符路径编码问题，第一次练习请把示例程序和媒体文件放在纯英文路径；中文路径支持可以之后单独练习。

正常运行时，会看到 FFmpeg 版本、输入容器和流摘要、第一张解码画面的尺寸，以及类似下面的帧数：

~~~text
FFmpeg version: 7.x.x
First decoded frame: 1920x1080
Video stream index: 0
Decoded video frames: 360
~~~

实际版本、宽高、流编号和帧数取决于你使用的文件。`av_dump_format` 的详细内容通常写到标准错误流，因此终端里可能与 `std::cout` 的行交错显示，这是正常的。

## 输出方向：认识顺序即可

本章实践只读和解码；读完后应能认出转码程序后续需要哪些步骤：

1. 用 `avformat_alloc_output_context2` 创建输出容器上下文。
2. 查找编码器，用 `avcodec_alloc_context3` 创建编码上下文，设置编码参数，再调用 `avcodec_open2`。
3. 用 `avcodec_parameters_from_context` 把编码器参数写入输出流。
4. 如需创建普通文件，调用 `avio_open` 打开输出文件。
5. 调用 `avformat_write_header` 写容器头。
6. 将输入帧送给编码器：`avcodec_send_frame`；循环用 `avcodec_receive_packet` 取编码包。
7. 用 `av_packet_rescale_ts` 把编码包时间戳换算到输出流的时间基，设置 `stream_index`，再调用 `av_interleaved_write_frame` 写包。
8. 冲刷编码器，写出剩余包；调用 `av_write_trailer` 完成容器。
9. 关闭 `AVIOContext`，释放编码上下文和输入、输出上下文。

只重新封装时不需要创建解码器和编码器，但仍需要建立输出流、写头、搬运压缩包、处理时间戳、写尾。详细实现见 [[06-编码与封装/封装输出流程]] 和 [[06-编码与封装/转码流程]]。

## 时间戳在流程中的位置

每个流可以用自己的 `time_base` 表示时间。`AVPacket::pts`、`AVFrame::pts` 等值不是天然的“秒”，必须结合对应时间基理解。输出文件的流时间基还可能与输入不同，因此写包前通常要做时间基换算。

把时间戳直接当秒打印、把视频时间基套给音频、或忘记在写出时重新缩放，都可能造成播放速度异常、音画不同步或时长错误。先阅读 [[03-FFmpeg基础/时间戳与时间基]]；本章统计帧数的示例不改写时间戳。

## 旧教程和版本差异

尽量使用现代 send/receive 解码接口。遇到旧教程中的下列内容，要先核对它针对的 FFmpeg 版本：

- `av_register_all()`、`avcodec_register_all()`：现代 FFmpeg 不需要应用程序调用全局注册函数。
- `AVStream::codec`：旧式成员；新代码从 `AVStream::codecpar` 读取编码参数，再用 `avcodec_parameters_to_context` 初始化解码上下文。
- 旧式“每次读一个包就直接调用解码函数”的接口：现代解码使用 `avcodec_send_packet` 和 `avcodec_receive_frame`，编码使用 `avcodec_send_frame` 和 `avcodec_receive_packet`。

FFmpeg 版本更新时，少数结构字段和函数可能变化。编译时使用的头文件、链接时的 `.lib` 和运行时加载的 DLL 应来自同一套构建；不要把不同版本的三者混在一起。遇到接口差异先看当前安装包附带的头文件和官方 API 文档，再判断能否采用教程里的写法。

## 常见错误与排查

先区分错误发生在**编译、链接、启动还是 FFmpeg 运行时**。这四个阶段的原因不同。

| 现象 | 通常发生在哪一步 | 常见原因和排查方法 |
|---|---|---|
| `C1083: Cannot open include file` | 编译 | `include` 路径没传给编译器；确认 `C:/ffmpeg/include/libavformat/avformat.h` 存在 |
| `LNK1104: cannot open file 'avformat.lib'` | 链接 | `.lib` 搜索目录不对、包中没有 MSVC 导入库，或误拿了只有运行程序的发行包 |
| `LNK2019` 等未解析符号 | 链接 | 忘记链接 `avformat`、`avcodec` 或 `avutil`；头文件与 `.lib` 不配套；混用了 MinGW/MSVC 构建 |
| 启动时提示缺少 `avcodec-*.dll` | 启动 | Windows 没找到运行时 DLL；把对应 `bin` 临时加进 PATH，或按部署章节处理 |
| `0xc000007b` | 启动 | 程序和 DLL 的 x86/x64 位数不匹配，或 DLL 版本/运行时不兼容 |
| `No such file or directory` | FFmpeg 运行时 | 输入路径错、当前目录与预想不同，或路径编码有问题；先用英文路径并检查文件是否存在 |
| `Invalid data found when processing input` | FFmpeg 运行时 | 文件损坏、内容并非媒体文件，或当前构建不能读取对应输入格式 |
| `avformat_open_input` 失败 | 输入打开 | 路径不存在、文件不可访问、容器无法识别；显示 `av_strerror` 的文字并先用 `ffprobe` 检查 |
| `avformat_find_stream_info` 失败 | 输入探测 | 输入不完整、可读数据不足或流信息无法识别；换一个确定有效的本地文件复核 |
| 找不到视频流 | 流选择 | 文件可能只有音频/字幕，或视频流无法识别；先看 `av_dump_format`，不要误判为编译问题 |
| `avcodec_open2` 失败 | 解码器初始化 | 缺少该解码器、参数不兼容，或库/DLL 混用；检查解码器名称和加载的构建 |
| `avcodec_send_packet` 或 `avcodec_receive_frame` 报错 | 解码循环 | 包送给了错误的解码器、`EAGAIN` 没按收发顺序处理，或文件数据有问题 |
| 解码帧数比预计少 | 解码/输入 | 没有冲刷解码器、文件不完整、可变帧率，或估算方法不适用 |
| 程序编译成功但运行失败 | 启动/运行时 | 编译和运行是不同阶段；优先检查 DLL、路径和文件，不要认为编译通过就代表运行依赖齐全 |

建议用以下命令分别验证 FFmpeg、测试媒体和输入信息：

~~~powershell
ffmpeg -hide_banner -version
ffprobe -hide_banner -show_streams "C:/media/sample.mp4"
Test-Path "C:/media/sample.mp4"
~~~

若 `ffprobe` 能打开文件而 C++ 程序不能，重点检查程序加载的 DLL 是否与头文件、`.lib` 同版本，以及输入路径是否一致。

## 练习题

### 练习 1：用自己的话重画数据流

不看上面的流程图，把“打开文件 -> 选视频流 -> 读包 -> 解码 -> 冲刷 -> 释放”的步骤画出来，并在每一步旁写出负责的 FFmpeg 函数。

**检查点：**能说明 `av_read_frame` 给出压缩包，`avcodec_receive_frame` 得到解码帧；能说明读到文件末尾后还需冲刷解码器。

### 练习 2：区分对象

在示例中标出哪些指针指向输入上下文内部的对象，哪些是程序自己申请的。

**提示：**`format_context->streams[index]` 和 `stream->codecpar` 属于输入上下文；`avcodec_alloc_context3`、`av_packet_alloc`、`av_frame_alloc` 创建的对象需要由程序配对释放。

### 练习 3：观察不同流

用 `ffprobe` 查看一个同时有视频和音频的文件。记下流编号、流类型、编码名称、分辨率或采样率，再和 `av_dump_format` 的输出对照。

**检查点：**不要假设视频一定是流 0；程序用 `av_find_best_stream` 选择视频，并用包的 `stream_index` 做匹配。

### 练习 4：让程序处理音频流

把程序的目标类型改成 `AVMEDIA_TYPE_AUDIO`，统计解码出的音频帧数。打印音频帧的 `nb_samples`，观察它与视频帧“宽 x 高”的信息为什么不同。

**检查点：**音频帧含有采样数据；它不是一张有宽、高的图像。声道布局和采样格式后续再细学。

### 练习 5：理解错误路径

分别不传参数、传不存在的文件、传一个只有音频的文件。记录三种情况下程序在哪里结束、打印什么信息。

**检查点：**缺少参数由程序自己检查；打不开文件由 `avformat_open_input` 报错；只有音频的文件会在选择视频流时失败。这些错误发生在不同步骤。

### 练习 6：修改帧计数程序

额外统计成功读取了多少个输入包，再与解码帧数比较。解释为什么两个数字通常不同。

**检查点：**输入包可能来自音频或字幕流；一个视频包可能产生零帧或多帧；解码器也可能暂存输出。

## 阶段检查清单

完成本章后，逐项确认自己能做到：

- [ ] 不看笔记，用通俗语言区分容器、流、数据包和帧。
- [ ] 说出 `libavformat` 负责读容器，`libavcodec` 负责编解码，`libavutil` 提供基础工具。
- [ ] 解释为什么 C++ 中的 FFmpeg 头文件通常放在 `extern "C"` 块里。
- [ ] 说明 `avformat_open_input` 和 `avformat_find_stream_info` 是先后不同的两步。
- [ ] 从 `AVFormatContext` 找到流，并说明 `stream_index` 如何标识包所属的流。
- [ ] 说出 `avcodec_parameters_to_context`、`avcodec_open2` 在开始解码前各做什么。
- [ ] 解释 `avcodec_send_packet` 和 `avcodec_receive_frame` 的配合方式。
- [ ] 分辨“需要更多输入”的 `AVERROR(EAGAIN)` 和读到末尾的 `AVERROR_EOF`。
- [ ] 解释为什么读到文件末尾后还要冲刷解码器。
- [ ] 为自己创建的输入上下文、解码上下文、包和帧调用对应的 FFmpeg 释放函数。
- [ ] 在 Windows 上区分头文件目录、`.lib` 目录和 DLL 目录，并能分别排查编译、链接和启动错误。
- [ ] 能用 CMake 编译示例，并用本地媒体文件观察解码帧数。
- [ ] 说出 remux 和 transcode 的区别，并知道输出 API 后续在哪些章节学习。

如果其中几项还做不到，回到对应顺序的小节复习，再完成一次练习。下一步建议阅读 [[04-C++调用FFmpeg/AVFormatContext]]，逐个理解本章已经用到的输入上下文、流和编码参数。

## 参考资料

- [[03-FFmpeg基础/FFmpeg组件概览]]：库的职责和常见对象。
- [[04-C++调用FFmpeg/AVFormatContext]]：输入容器上下文和流。
- [[04-C++调用FFmpeg/AVCodecContext]]：编解码上下文。
- [[04-C++调用FFmpeg/AVFrame与AVPacket]]：包和帧的对象差异。
- [[04-C++调用FFmpeg/错误处理与资源生命周期]]：错误处理、所有权和释放。
- [[02-Windows开发环境/FFmpeg构建与安装]]：Windows 下的 FFmpeg 开发文件准备。
- [FFmpeg 官方文档](https://ffmpeg.org/documentation.html)：按当前安装版本核对 API。
- [FFmpeg 官方 API 文档](https://ffmpeg.org/doxygen/trunk/)：查找函数、结构体和参数说明。

