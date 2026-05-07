# RtspServer

> A fork of [PHZ76/RtspServer](https://github.com/PHZ76/RtspServer) with added **G711U (µ-law)** audio support.
> Tested on Linux only. Windows platform is untested — please verify on your own.

## Introduction

A C++11 RTSP server and pusher supporting audio/video streaming over RTSP on both Linux and Windows platforms.

## Features

- **H264** and **H265** video codecs
- **G711A (A-law)**, **G711U (µ-law)**, and **AAC** audio codecs
- Simultaneous audio/video streaming
- Unicast (RTP_OVER_UDP, RTP_OVER_RTSP) and multicast
- Heartbeat detection (unicast)
- RTSP push (TCP)
- Digest Authentication
- Windows and Linux support

## Usage Examples

### RTSP Server (H264 Video)

```cpp
#include "xop/RtspServer.h"
#include "net/Timer.h"

// Create event loop and RTSP server
std::shared_ptr<xop::EventLoop> event_loop(new xop::EventLoop());
std::shared_ptr<xop::RtspServer> server = xop::RtspServer::Create(event_loop.get());
server->Start("0.0.0.0", 554);

// Create media session with H264 source
xop::MediaSession *session = xop::MediaSession::CreateNew("live");
session->AddSource(xop::channel_0, xop::H264Source::CreateNew());
xop::MediaSessionId session_id = server->AddSession(session);

// Push frame
AVFrame video_frame;
// ... fill video frame data ...
server->PushFrame(session_id, xop::channel_0, video_frame);
```

### RTSP Server (H265 Video)

```cpp
// Add H265 source instead
session->AddSource(xop::channel_0, xop::H265Source::CreateNew());
```

### RTSP Server (G711U Audio)

```cpp
// Add G711U audio source
session->AddSource(xop::channel_1, xop::G711USource::CreateNew());

// Push audio frame
AVFrame audio_frame;
audio_frame.type = AUDIO_FRAME;
audio_frame.size = audio_size;
audio_frame.timestamp = xop::G711USource::GetTimestamp();
audio_frame.buffer.reset(new uint8_t[audio_size]);
memcpy(audio_frame.buffer.get(), audio_data, audio_size);
server->PushFrame(session_id, xop::channel_1, audio_frame);
```

### RTSP Pusher

```cpp
#include "xop/RtspPusher.h"

xop::RtspPusher pusher(event_loop.get());
pusher.Open("rtsp://127.0.0.1:554/live", 3000);
pusher.PushFrame(xop::channel_0, video_frame);
```

See `example/` directory for complete sample code.

> **Frame format notes for H264/H265 pushing:**
> This project does not strip AnnexB start codes internally. The input frame data must satisfy one of the following:
> - **Option A (recommended)**: Pass pure NAL units without start codes, one NAL unit per `PushFrame` call (e.g., push SPS / PPS / IDR separately)
> - **Option B (convenient)**: Pass a complete AnnexB frame (may contain multiple NAL units), but **the leading start code** (`00 00 00 01` or `00 00 01`) **must be stripped**. Intermediate start codes within the data will be properly recognized as NAL delimiters by the receiver's decoder.
> 
> Both approaches have been verified on Linux.

## Build

### Linux

Three Makefiles are provided:

| Makefile | Purpose | Example |
|----------|---------|---------|
| `Makefile_example` | Build example programs locally (`rtsp_server`, `rtsp_pusher`, `rtsp_h264_file`) | `make -f Makefile_example` |
| `Makefile_static` | Cross-compile static library `librtspserver.a` | `make -f Makefile_static` |
| `Makefile_shared` | Cross-compile shared library `librtspserver.so` | `make -f Makefile_shared` |

> Note: `Makefile_static` and `Makefile_shared` use `aarch64-linux-` as the cross-compilation toolchain prefix. Modify `CROSS_COMPILE` in the Makefile according to your environment.

### Windows

This fork is only tested on Linux. Windows is untested — please verify on your own.

> The original project provides a VS2017 solution file `VS2017/rtsp.sln` for Windows users.

## Build Requirements

- Linux: gcc 4.8+
- Windows: VS2015+


## Directory Structure

```
├── example/           # Example programs
│   ├── rtsp_server.cpp
│   ├── rtsp_pusher.cpp
│   └── rtsp_h264_file.cpp
├── src/
│   ├── 3rdpart/       # Third-party (md5)
│   ├── net/           # Network layer (TCP, event loop, timers, etc.)
│   └── xop/           # RTSP protocol layer
│       ├── H264Source.cpp/.h
│       ├── H265Source.cpp/.h
│       ├── G711ASource.cpp/.h   # G711A (A-law)
│       ├── G711USource.cpp/.h   # G711U (µ-law)  ← added
│       ├── AACSource.cpp/.h
│       ├── RtspServer.cpp/.h
│       ├── RtspPusher.cpp/.h
│       └── RtpConnection.cpp/.h
├── VS2017/
├── Makefile_example
├── Makefile_static
└── Makefile_shared
```

## Credits

- Original author: PHZ76
- [websocketpp-md5](https://github.com/zaphoyd/websocketpp)
