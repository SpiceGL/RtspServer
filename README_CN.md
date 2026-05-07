# RtspServer

> 基于 [PHZ76/RtspServer](https://github.com/PHZ76/RtspServer) 的 fork，新增 **G711U (µ-law)** 音频格式支持。
> 仅在 Linux 下测试验证，Windows 平台请使用者自行验证。

## 项目介绍

C++11 实现的 RTSP 服务器和推流器，支持在 Linux 和 Windows 平台上进行音视频的 RTSP 转发与推流。

## 功能特性

- 支持 **H264**、**H265** 视频格式
- 支持 **G711A (A-law)**、**G711U (µ-law)**、**AAC** 音频格式
- 支持同时传输音视频
- 支持单播（RTP_OVER_UDP、RTP_OVER_RTSP）和组播
- 支持心跳检测（单播）
- 支持 RTSP 推流（TCP）
- 支持摘要认证（Digest Authentication）
- 支持 Windows 和 Linux 平台

## 使用示例

### RTSP 服务器（H264 视频推流）

```cpp
#include "xop/RtspServer.h"
#include "net/Timer.h"

// 创建事件循环和 RTSP 服务器
std::shared_ptr<xop::EventLoop> event_loop(new xop::EventLoop());
std::shared_ptr<xop::RtspServer> server = xop::RtspServer::Create(event_loop.get());
server->Start("0.0.0.0", 554);

// 创建媒体会话，添加 H264 源
xop::MediaSession *session = xop::MediaSession::CreateNew("live");
session->AddSource(xop::channel_0, xop::H264Source::CreateNew());
xop::MediaSessionId session_id = server->AddSession(session);

// 推流
AVFrame video_frame;
// ... 填充视频帧数据 ...
server->PushFrame(session_id, xop::channel_0, video_frame);
```

### RTSP 服务器（H265 视频推流）

```cpp
// 添加 H265 源
session->AddSource(xop::channel_0, xop::H265Source::CreateNew());
```

### RTSP 服务器（G711U 音频推流）

```cpp
// 添加 G711U 音频源
session->AddSource(xop::channel_1, xop::G711USource::CreateNew());

// 推流音频帧
AVFrame audio_frame;
audio_frame.type = AUDIO_FRAME;
audio_frame.size = audio_size;
audio_frame.timestamp = xop::G711USource::GetTimestamp();
audio_frame.buffer.reset(new uint8_t[audio_size]);
memcpy(audio_frame.buffer.get(), audio_data, audio_size);
server->PushFrame(session_id, xop::channel_1, audio_frame);
```

### RTSP 推流器

```cpp
#include "xop/RtspPusher.h"

xop::RtspPusher pusher(event_loop.get());
pusher.Open("rtsp://127.0.0.1:554/live", 3000);
pusher.PushFrame(xop::channel_0, video_frame);
```

详细示例请参考 `example/` 目录下的源码。

> **关于 H264/H265 推流的帧格式说明：**
> 本项目核心并未内置 AnnexB 起始码剥离逻辑，传入的 H264/H265 帧数据需要满足以下要求之一：
> - **方式一（推荐）**：传入不带起始码的纯 NAL 单元，每个 `PushFrame` 只传一个 NAL 单元（如 SPS / PPS / IDR 分开推）
> - **方式二（方便）**：传入 AnnexB 格式的一整帧数据（可包含多个 NAL 单元），但**必须去掉数据开头的第一个起始码**（`00 00 00 01` 或 `00 00 01`），数据中后续的中间起始码会自动被接收端解码器识别为 NAL 分隔符
> 
> 两种方式均已在 Linux 下验证通过。

## 编译

### Linux

项目提供了三个 Makefile，按需选用：

| Makefile | 用途 | 示例 |
|----------|------|------|
| `Makefile_example` | 本地编译示例程序（`rtsp_server`、`rtsp_pusher`、`rtsp_h264_file`） | `make -f Makefile_example` |
| `Makefile_static` | 交叉编译生成静态库 `librtspserver.a` | `make -f Makefile_static` |
| `Makefile_shared` | 交叉编译生成动态库 `librtspserver.so` | `make -f Makefile_shared` |

> 注意：`Makefile_static` 和 `Makefile_shared` 中的交叉编译工具链路径为 `aarch64-linux-`，请根据实际环境修改 `CROSS_COMPILE` 变量。

### Windows

本 fork 仅对 Linux 平台做了测试验证，Windows 下未做测试，请使用者自行验证。

> 原项目提供了 VS2017 工程文件 `VS2017/rtsp.sln`，Windows 用户可尝试直接打开编译。

## 编译环境

- Linux: gcc 4.8+
- Windows: VS2015+

## 目录结构

```
├── example/           # 示例程序
│   ├── rtsp_server.cpp      # RTSP 服务器示例
│   ├── rtsp_pusher.cpp      # RTSP 推流器示例
│   └── rtsp_h264_file.cpp   # H264 文件推流示例
├── src/
│   ├── 3rdpart/       # 第三方库（md5）
│   ├── net/           # 网络层（TCP、事件循环、定时器等）
│   └── xop/           # RTSP 协议层
│       ├── H264Source.cpp/.h    # H264 视频源
│       ├── H265Source.cpp/.h    # H265 视频源
│       ├── G711ASource.cpp/.h   # G711A (A-law) 音频源
│       ├── G711USource.cpp/.h   # G711U (µ-law) 音频源  ← 新增
│       ├── AACSource.cpp/.h     # AAC 音频源
│       ├── RtspServer.cpp/.h    # RTSP 服务器
│       ├── RtspPusher.cpp/.h    # RTSP 推流器
│       └── RtpConnection.cpp/.h # RTP 连接管理
├── VS2017/                 # VS2017 工程文件
├── Makefile_example        # 本地编译示例程序
├── Makefile_static         # 交叉编译静态库
└── Makefile_shared         # 交叉编译动态库
```

## 致谢

- 原项目作者：PHZ76
- [websocketpp-md5](https://github.com/zaphoyd/websocketpp)