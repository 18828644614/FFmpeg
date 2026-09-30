# Notes: 时间戳与时间基

## Existing Repository Evidence

- `03-FFmpeg基础/时间戳与时间基.md` 原来是只含栏目标题的空模板。
- `03-FFmpeg基础/阶段03-FFmpeg基础.md` 已把本主题放在容器、流和命令行之后，适合作为本章前置顺序。
- `03-FFmpeg基础/媒体文件与容器.md` 已解释包、帧和重封装，本章可以专注时间概念。
- `05-解封装与解码/音视频同步基础.md`、`06-编码与封装/时间戳写出.md` 和 `08-播放器开发/音视频同步实现.md` 是后续延伸章节。

## Content Decisions

### Learning order

1. 先补时间单位、整数除法和 `AVRational`。
2. 再解释 time base、PTS、DTS、duration、start_time 和帧率的区别。
3. 说明 `AVStream::time_base`、`AVCodecContext::time_base` 和 `AV_TIME_BASE` 的边界。
4. 用 `ffprobe` 查看真实包和帧时间戳。
5. 用 Windows + MSVC + FFmpeg 写时间换算、包查看和 remux 示例。
6. 最后补 seek、负时间戳、音频采样时间和同步直觉。

### Example API choices

- Time conversion: `av_q2d`, `av_rescale_q`, `av_rescale_q_rnd`, `av_packet_rescale_ts`.
- Packet inspection: `avformat_open_input`, `avformat_find_stream_info`, `av_read_frame`, `av_packet_unref`, `avformat_close_input`.
- Remux: `avformat_alloc_output_context2`, `avcodec_parameters_copy`, `avformat_write_header`, `av_interleaved_write_frame`, `av_write_trailer`.
- Error display: `av_strerror`.
- Avoid decoding in the first examples; complete decoding and synchronization are covered by later chapters.

## Verification Checklist

- [x] Learning range and order are stated before the detailed explanation.
- [x] Code blocks specify language and include Windows execution commands.
- [x] PTS, DTS, duration, time base, frame rate and `AV_NOPTS_VALUE` are explained separately.
- [x] Examples include pure conversion, packet inspection and remux timestamp rescaling.
- [x] Common failures cover wrong units, wrong direction, rounding, seek, DLL, include/lib and architecture issues.
- [x] Wikilinks used by the chapter point to existing files.

## Environment Check

- The current PowerShell session does not have `ffmpeg`, `ffprobe`, `cmake`, or `cl` on PATH.
- The document includes reproducible commands but does not claim that media generation, compilation or execution succeeded in this session.
