# <span style="color: #88C0D0;">**Software Video Pipeline & Codec Benchmarking**</span>

### <span style="color: #81A1C1;">**Filename:** `overall.md`</span>

| <span style="color: #EBCB8B;">**Field**</span> | <span style="color: #EBCB8B;">**Details**</span> |
| :--- | :--- |
| <span style="color: #EBCB8B;">**Subproject**</span> | `1_sw_video_pipeline` (Stage 1 of Project Gentoo LLWVS) |
| <span style="color: #EBCB8B;">**Author**</span> | devilsu |
| <span style="color: #EBCB8B;">**Created**</span> | 2026-10-06 09:40 PDT |
| <span style="color: #EBCB8B;">**Modified**</span> | 2026-10-06 09:55 PDT |
| <span style="color: #EBCB8B;">**Version**</span> | v1.0.1 |
| <span style="color: #EBCB8B;">**Status**</span> | In Review 2026-10-06 09:55:00 |

---

## <span style="color: #88C0D0;">**Table of Contents**</span>
- [<span style="color: #88C0D0;">**1. Overview, ELI5 & Sources**</span>](#1-overview-eli5--sources)
  - [<span style="color: #81A1C1;">1.1 TL;DR</span>](#11-tldr)
  - [<span style="color: #81A1C1;">1.2 ELI5 (The Simple Analogy)</span>](#12-eli5-the-simple-analogy)
  - [<span style="color: #81A1C1;">1.3 Sources</span>](#13-sources)
- [<span style="color: #88C0D0;">**2. High-Level Architecture & Flowchart (ASCII)**</span>](#2-high-level-architecture--flowchart-ascii)
- [<span style="color: #88C0D0;">**3. Assumptions & Prerequisites**</span>](#3-assumptions--prerequisites)
  - [<span style="color: #81A1C1;">3.1 Environment & Dependencies</span>](#31-environment--dependencies)
  - [<span style="color: #81A1C1;">3.2 Architectural Assumptions</span>](#32-architectural-assumptions)
- [<span style="color: #88C0D0;">**4. Detailed Flow & Component Breakdown**</span>](#4-detailed-flow--component-breakdown)
  - [<span style="color: #81A1C1;">4.1 Component Roles & Responsibilities</span>](#41-component-roles--responsibilities)
  - [<span style="color: #81A1C1;">4.2 End-to-End Data Lifecycle</span>](#42-end-to-end-data-lifecycle)
- [<span style="color: #88C0D0;">**5. Step-by-Step Implementation Roadmap**</span>](#5-step-by-step-implementation-roadmap)
- [<span style="color: #88C0D0;">**6. Verification & Sign-Off Checklist**</span>](#6-verification--sign-off-checklist)
- [<span style="color: #88C0D0;">**7. Risks, Trade-offs & Mitigation**</span>](#7-risks-trade-offs--mitigation)

---

## <span style="color: #88C0D0;">**1. Overview, ELI5 & Sources**</span>

### <span style="color: #81A1C1;">**1.1 TL;DR**</span>
- Establishes a pure software video streaming and benchmarking pipeline to isolate <span style="color: #EBCB8B;">**video encoding and display latency**</span> prior to introducing SDR hardware.

- Evaluates multiple video codecs including <span style="color: #EBCB8B;">**H.264 (x264 zerolatency)**</span>, <span style="color: #EBCB8B;">**H.265 (x265)**</span>, and <span style="color: #EBCB8B;">**MJPEG**</span> across resolutions `720p` and `1080p` at both `30 fps` and `60 fps`.
  - Compares lightweight network transports including <span style="color: #EBCB8B;">**Raw UDP NALUs**</span>, <span style="color: #EBCB8B;">**RTP**</span>, and traditional <span style="color: #EBCB8B;">**MPEG-TS**</span>.
    - Measures packet overhead and parser jitter across local network loopbacks.

- Deploys an automated <span style="color: #EBCB8B;">**optical millisecond latency meter**</span> to establish an empirical glass-to-glass baseline for subsequent SDR stages.

### <span style="color: #81A1C1;">**1.2 ELI5 (The Simple Analogy)**</span>
> <span style="color: #B48EAD;">**The Simple Analogy:**</span>
>
> Imagine you want to test how fast you can send handwritten letters using carrier pigeons across town. Before you ever buy the pigeons or release them into the wind, you first time how fast you can write down the message and how fast your friend can read it when sitting at the same table.
>
> <br>
>
> In this stage, we test our camera, video compressor, and screen player on our local computer to see how much delay the software creates by itself. This ensures that when we later add our radio transmitters, we know exactly how much latency comes from the software and how much comes from the radio waves.

### <span style="color: #81A1C1;">**1.3 Sources**</span>
- [LamaBleu/Pluto-DATV-test Repository](https://github.com/LamaBleu/Pluto-DATV-test)

- [Master Repository Dashboard](../../readme.md)

---

## <span style="color: #88C0D0;">**2. High-Level Architecture & Flowchart (ASCII)**</span>

```text
+------------------------------------------------------------------------------------------------+
|                                    TRANSMITTER CAPTURE & ENCODER                               |
|                                                                                                |
|   +---------------------+   Raw V4L2 YUV    +---------------------+   H.264 / H.265 NALUs      |
|   |   USB UVC Camera    |------------------>|    Video Encoder    |------------------------+   |
|   |   (720p / 1080p)    |   MMAP Buffers    |   (x264 / x265)     |   Zero-Latency Tuned   |   |
|   +---------------------+                   +---------------------+                        |   |
|                                                        |                                   |   |
|                                                        | Codec Parameter Sweep             |   |
|                                                        v                                   |   |
|                                             +---------------------+                        |   |
|                                             |   Transport Mux     |<-----------------------+   |
|                                             |   (RTP / UDP / TS)  |                            |
|                                             +---------------------+                            |
+--------------------------------------------------------|---------------------------------------+
                                                         | Socket Stream (Localhost / LAN)
                                                         v
+------------------------------------------------------------------------------------------------+
|                                     RECEIVER DECODER & DISPLAY                                 |
|                                                                                                |
|   +---------------------+   Demuxed Bitstream +---------------------+   Raw Decoded YUV        |
|   |   Socket Receiver   |-------------------->|    Video Decoder    |----------------------+   |
|   |   (UDP / RTP Port)  |   Low Jitter FIFO   |   (FFmpeg / Gst)    |   Low-Latency Thread |   |
|   +---------------------+                     +---------------------+                      |   |
|                                                                                            |   |
|                                                                                            v   |
|   +---------------------+                     +---------------------+              +-------+   |
|   | Optical Benchmark   |                     | Video Latency Meter |              |  Zero-Buf |
|   | (OCR Milliseconds)  |<--------------------| (Millisecond Clock) |<-------------|  Renderer |
|   +---------------------+                     +---------------------+              +-----------+
+------------------------------------------------------------------------------------------------+
```

---

## <span style="color: #88C0D0;">**3. Assumptions & Prerequisites**</span>

### <span style="color: #81A1C1;">**3.1 Environment & Dependencies**</span>

| <span style="color: #EBCB8B;">**Dependency / Tool**</span> | <span style="color: #EBCB8B;">**Required Version**</span> | <span style="color: #EBCB8B;">**Purpose**</span> |
| :--- | :--- | :--- |
| <span style="color: #EBCB8B;">**Operating System**</span> | Ubuntu 22.04 / 24.04 LTS (x86_64) | Host development and benchmarking environment |
| <span style="color: #EBCB8B;">**Python Runtime**</span> | Python >= 3.10 | Automation test scripts and latency meters |
| <span style="color: #EBCB8B;">**FFmpeg / Libav**</span> | FFmpeg >= 4.4 / 6.x (`libavcodec`) | Reference video capture, encoding, and streaming |
| <span style="color: #EBCB8B;">**OpenCV Python**</span> | `opencv-python` >= 4.8 | Latency meter video rendering and OCR analysis |
| <span style="color: #EBCB8B;">**GStreamer Core**</span> | GStreamer 1.20+ (`gstreamer1.0-plugins-good`) | Alternative low-latency pipeline comparison |
| <span style="color: #EBCB8B;">**V4L2 Utilities**</span> | `v4l-utils` | USB camera capability querying and format inspection |

### <span style="color: #81A1C1;">**3.2 Architectural Assumptions**</span>
- <span style="color: #EBCB8B;">**Camera Interface**</span>: Standard USB UVC video camera providing hardware YUYV or MJPEG pixel formats up to `1920x1080` at minimum `30 fps`.

- <span style="color: #EBCB8B;">**Host CPU Capacity**</span>: Development PC has sufficient multi-core compute to encode 1080p60 H.264 in real time without CPU throttling.

- <span style="color: #EBCB8B;">**Zero-Buffer Target**</span>: Software receiver must configure display pipelines to use minimum buffer queues (`buffer-size=0`, `drop-on-latency=true`) to avoid artificial buffering latency.

---

## <span style="color: #88C0D0;">**4. Detailed Flow & Component Breakdown**</span>

### <span style="color: #81A1C1;">**4.1 Component Roles & Responsibilities**</span>
- <span style="color: #EBCB8B;">**Transmitter Streamer (`tx_streamer.py`)**</span>: Handles V4L2 device capture (`/dev/video0`), manages frame acquisition buffers, feeds raw frames into `x264` or `x265` with `tune=zerolatency`, and pushes encoded packets over UDP/RTP sockets.

- <span style="color: #EBCB8B;">**Receiver Player (`rx_player.py`)**</span>: Listens on dedicated UDP/RTP ports, handles depacketization, decodes incoming video slices immediately upon arrival, and renders frames to a low-latency surface.

- <span style="color: #EBCB8B;">**Screen Timer Generator (`screen_timer.py`)**</span>: Displays a millisecond-accurate counter clock on the host monitor that updates on every display refresh tick (e.g. `60 Hz` or `144 Hz`).

- <span style="color: #EBCB8B;">**Latency Analyzer (`measure_latency.py`)**</span>: Captures screen snapshots displaying both the live clock and the received webcam feed, performing OCR on the two timestamps to compute the glass-to-glass latency difference in milliseconds.

### <span style="color: #81A1C1;">**4.2 End-to-End Data Lifecycle**</span>

<span style="color: #EBCB8B;">**Step 1 - Camera Sensor Frame Ingestion**</span>
- Camera sensor captures optical photons and emits raw frames over the USB UVC bus.
- Kernel V4L2 subsystem maps incoming memory frames into user-space via zero-copy MMAP pointers.

<span style="color: #EBCB8B;">**Step 2 - Zero-Latency Video Compression**</span>
- The encoder receives raw frames and encodes without B-frames (`bframes=0`).
- Slices are emitted immediately with minimal intra-frame lookahead (`rc-lookahead=0`).

<span style="color: #EBCB8B;">**Step 3 - Packetization & Socket Transmission**</span>
- Encoded NAL units are packetized into UDP datagrams or RTP frames.
- Datagrams are pushed directly onto the local network stack without lingering in socket buffers.

<span style="color: #EBCB8B;">**Step 4 - Depacketization & Fast Decoding**</span>
- The receiver socket receives datagrams and passes NAL units directly to the decoder pipeline.
- Video decoder reconstructs YUV macroblocks without waiting for future frames.

<span style="color: #EBCB8B;">**Step 5 - Screen Rendering & Optical Latency Verification**</span>
- The decoded image renders to the display window.
- The optical latency analyzer records the difference between the source screen timer and the rendered stream.

---

## <span style="color: #88C0D0;">**5. Step-by-Step Implementation Roadmap**</span>

- [ ] <span style="color: #EBCB8B;">**Phase 1: Environment & Camera Ingestion Setup**</span>
  - [ ] Enumerate USB video devices using `v4l2-ctl --list-formats-ext`
  - [ ] Implement `tx_streamer.py` with configurable resolution (`720p`, `1080p`) and frame rate (`30 fps`, `60 fps`)
  - [ ] Implement `rx_player.py` with zero-buffer SDL/OpenGL rendering surface

- [ ] <span style="color: #EBCB8B;">**Phase 2: Codec & Transport Protocol Parameter Sweep**</span>
  - [ ] Configure and test H.264 with `libx264` (`tune=zerolatency`, `profile=baseline`, `bframes=0`)
  - [ ] Configure and test H.265 with `libx265` (`tune=zerolatency`)
  - [ ] Configure and test MJPEG streaming (`mjpeg`) as low-complexity baseline
  - [ ] Compare transport protocols: Raw UDP NALUs vs. RTP (`rtph264pay`) vs. MPEG-TS (`mpegts`)

- [ ] <span style="color: #EBCB8B;">**Phase 3: Optical Latency Measurement Harness**</span>
  - [ ] Create `screen_timer.py` generating high-contrast millisecond digital counter
  - [ ] Create `measure_latency.py` with automated frame capture and optical timestamp subtraction
  - [ ] Validate measurement reproducibility across at least 100 consecutive frames

- [ ] <span style="color: #EBCB8B;">**Phase 4: Automated Benchmark Execution & Sign-Off**</span>
  - [ ] Author `benchmark_codecs.sh` to execute the full matrix of resolutions and codecs
  - [ ] Log bit-exact measurements into `benchmark_results.json`
  - [ ] Compile Stage 1 retrospective and sign-off report in `summary.md`

---

## <span style="color: #88C0D0;">**6. Verification & Sign-Off Checklist**</span>

| <span style="color: #EBCB8B;">**Test / Acceptance Case**</span> | <span style="color: #EBCB8B;">**Target Criteria**</span> | <span style="color: #EBCB8B;">**Actual Result**</span> | <span style="color: #EBCB8B;">**Status**</span> |
| :--- | :--- | :--- | :--- |
| <span style="color: #EBCB8B;">**Camera Capture Stability**</span> | Sustained 1080p @ 30fps without dropped frames over 5 minutes | Pending | <span style="color: #4C566A;">**Planned**</span> |
| <span style="color: #EBCB8B;">**H.264 Zerolatency Tuning**</span> | Zero B-frames, zero lookahead delay verified via stream probe | Pending | <span style="color: #4C566A;">**Planned**</span> |
| <span style="color: #EBCB8B;">**Transport Comparison**</span> | Measure latency delta: Raw UDP vs RTP vs MPEG-TS | Pending | <span style="color: #4C566A;">**Planned**</span> |
| <span style="color: #EBCB8B;">**Glass-to-Glass Latency**</span> | Software glass-to-glass latency under 50 ms for 720p60 / 1080p30 | Pending | <span style="color: #4C566A;">**Planned**</span> |
| <span style="color: #EBCB8B;">**Automated Latency Logging**</span> | Bit-exact benchmark sweep results exported to JSON | Pending | <span style="color: #4C566A;">**Planned**</span> |

---

## <span style="color: #88C0D0;">**7. Risks, Trade-offs & Mitigation**</span>

| <span style="color: #EBCB8B;">**Identified Risk**</span> | <span style="color: #EBCB8B;">**Severity**</span> | <span style="color: #EBCB8B;">**Mitigation Strategy**</span> |
| :--- | :--- | :--- |
| <span style="color: #EBCB8B;">**UVC Driver Buffer Delays**</span> | Medium | Query driver buffer queue depths via `v4l2-ctl` and request minimum buffer allocations (`count=2` in `V4L2_BUF_TYPE_VIDEO_CAPTURE`). |
| <span style="color: #EBCB8B;">**Display Server V-Sync Lag**</span> | Medium | Use direct OpenGL or SDL with `vsync=0` and low-overhead compositor bypass to eliminate window-manager queuing. |
| <span style="color: #EBCB8B;">**CPU Spikes during 1080p60**</span> | Low | Set `preset=ultrafast` in `x264` and bind encoding worker threads to dedicated high-frequency CPU cores. |
