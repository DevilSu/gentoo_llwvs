# <span style="color: #88C0D0;">**Software Video Pipeline & Codec Benchmarking**</span>

### <span style="color: #81A1C1;">**Filename:** `overall.md`</span>

| <span style="color: #EBCB8B;">**Field**</span> | <span style="color: #EBCB8B;">**Details**</span> |
| :--- | :--- |
| <span style="color: #EBCB8B;">**Subproject**</span> | `1_sw_video_pipeline` (Stage 1 of Project Gentoo LLWVS) |
| <span style="color: #EBCB8B;">**Author**</span> | devilsu |
| <span style="color: #EBCB8B;">**TX Hardware Target**</span> | **Orange Pi 5** (Rockchip RK3588, accessed via SSH over LAN/Wi-Fi) |
| <span style="color: #EBCB8B;">**RX Hardware Target**</span> | **Linux PC Host Workstation** (Display, video decoding & latency meter) |
| <span style="color: #EBCB8B;">**Transport Link**</span> | **Local Network Socket Stream (UDP / RTP over Wi-Fi or Ethernet)** |
| <span style="color: #EBCB8B;">**Camera Hardware**</span> | USB UVC Webcam connected directly to Orange Pi 5 (1080p @ 30/60 fps) |
| <span style="color: #EBCB8B;">**Created**</span> | 2026-10-06 09:40 PDT |
| <span style="color: #EBCB8B;">**Modified**</span> | 2026-10-06 12:50 PDT |
| <span style="color: #EBCB8B;">**Version**</span> | v1.2.0 |
| <span style="color: #EBCB8B;">**Status**</span> | <span style="color: #A3BE8C;">**Approved 2026-10-06 12:50:00**</span> |

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
  - [<span style="color: #81A1C1;">4.3 Approved Architectural Decisions</span>](#43-approved-architectural-decisions)
- [<span style="color: #88C0D0;">**5. Step-by-Step Implementation Roadmap**</span>](#5-step-by-step-implementation-roadmap)
- [<span style="color: #88C0D0;">**6. Verification & Sign-Off Checklist**</span>](#6-verification--sign-off-checklist)
- [<span style="color: #88C0D0;">**7. Risks, Trade-offs & Mitigation**</span>](#7-risks-trade-offs--mitigation)

---

## <span style="color: #88C0D0;">**1. Overview, ELI5 & Sources**</span>

### <span style="color: #81A1C1;">**1.1 TL;DR**</span>
- Establishes a distributed software video streaming and benchmarking pipeline between the <span style="color: #EBCB8B;">**Orange Pi 5 transmitter (TX)**</span> and the <span style="color: #EBCB8B;">**Linux PC receiver (RX)**</span> across a local Wi-Fi / Ethernet network.
- Connects to and controls the Orange Pi 5 natively via <span style="color: #EBCB8B;">**SSH**</span>, ingesting camera frames on ARM64 and streaming compressed video over <span style="color: #EBCB8B;">**UDP / RTP sockets**</span> directly to the PC receiver.
- Swaps the PlutoSDR RF link with lightweight <span style="color: #EBCB8B;">**UDP network transport**</span> to isolate video encoding, network encapsulation, and display latency prior to introducing software-defined radio hardware in Stage 2.
- Evaluates multiple video codecs including <span style="color: #EBCB8B;">**H.264 (x264 zerolatency)**</span>, <span style="color: #EBCB8B;">**H.265 (x265)**</span>, and <span style="color: #EBCB8B;">**MJPEG**</span> across resolutions `720p` and `1080p` at both `30 fps` and `60 fps`.
- Deploys an automated <span style="color: #EBCB8B;">**optical millisecond latency meter**</span> to establish an empirical glass-to-glass baseline for subsequent SDR stages.

### <span style="color: #81A1C1;">**1.2 ELI5 (The Simple Analogy)**</span>
> <span style="color: #B48EAD;">**The Simple Analogy:**</span>
>
> Imagine you want to test how fast you can write down handwritten letters and send them across the house. Before you buy expensive radio transmitters and antennas, you write the letters on your small portable notepad (Orange Pi 5) and toss them through your local home network (Wi-Fi/Ethernet) directly to your desk computer (PC receiver).
>
> <br>
>
> By using high-speed UDP (throwing letters instantly without waiting for return receipts), we measure exactly how many milliseconds it takes for the camera to see a number, compress it, send it across the network, and draw it on your computer screen. This gives us our clean baseline before replacing the network cable with radio waves in Stage 2.

### <span style="color: #81A1C1;">**1.3 Sources**</span>
- [LamaBleu/Pluto-DATV-test Repository](https://github.com/LamaBleu/Pluto-DATV-test)
- [Stage 1 Proposal Artifact](file:///home/devilsu/.gemini/antigravity-cli/brain/30ad34aa-efb7-4ab9-a0f8-87671b3d2f09/stage1_sw_video_pipeline_proposal.md)
- [Master Repository Dashboard](../../readme.md)

---

## <span style="color: #88C0D0;">**2. High-Level Architecture & Flowchart (ASCII)**</span>

```text
+----------------------------------------------------------------------------------------------------+
|                         ORANGE PI 5 (TX NODE) — ARM64 / ROCKCHIP RK3588                             |
|                               (Managed & Deployed via SSH)                                         |
|                                                                                                    |
|   +---------------------+   Raw V4L2 YUV    +---------------------+   H.264 / H.265 NALUs          |
|   |   USB UVC Camera    |------------------>|    Video Encoder    |----------------------------+   |
|   |   (720p / 1080p)    |   MMAP Buffers    |   (x264 / x265)     |   Zero-Latency Tuned       |   |
|   +---------------------+                   +---------------------+   (rc-lookahead=0, bframes=0)  |
|                                                                                                    v
|                                                                             +--------------------+ |
|                                                                             |   UDP Socket TX    | |
|                                                                             |   (Raw UDP / RTP)  | |
|                                                                             +---------|----------+ |
+---------------------------------------------------------------------------------------|------------+
                                                                                        |
                                       LOCAL NETWORK LINK                               | Low-overhead UDP
                                  (Wi-Fi / Gigabit Ethernet)                            | Socket Datagrams
                                                                                        v
+----------------------------------------------------------------------------------------------------+
|                         LINUX PC HOST WORKSTATION (RX NODE & LATENCY METER)                         |
|                                                                                                    |
|                                                                             +--------------------+ |
|                                                                             |   UDP Socket RX    | |
|                                                                             |   (Zero-Buf FIFO)  | |
|                                                                             +---------|----------+ |
|                                                                                       |            |
|   +---------------------+   Raw Decoded YUV +---------------------+   Demuxed NALUs   v            |
|   |  Zero-Buf Renderer  |<------------------|    Video Decoder    |<------------------+            |
|   |  (SDL2 / OpenGL)    |   Low-Latency     |   (FFmpeg / Gst)    |   Immediate Slice Feed     |
|   +----------|----------+   Display Thread  +---------------------+                                |
|              |                                                                                     |
|              v                                                                                     |
|   +---------------------+   Side-by-Side    +---------------------+                                |
|   | Video Latency Meter |------------------>| Optical Benchmark   |                                |
|   | (Millisecond Clock) |   Display Sync    | (OCR Latency Diff)  |                                |
|   +---------------------+                   +---------------------+                                |
+----------------------------------------------------------------------------------------------------+
```

---

## <span style="color: #88C0D0;">**3. Assumptions & Prerequisites**</span>

### <span style="color: #81A1C1;">**3.1 Environment & Dependencies**</span>

| <span style="color: #EBCB8B;">**Dependency / Tool**</span> | <span style="color: #EBCB8B;">**Node**</span> | <span style="color: #EBCB8B;">**Required Version**</span> | <span style="color: #EBCB8B;">**Purpose**</span> |
| :--- | :--- | :--- | :--- |
| <span style="color: #EBCB8B;">**Orange Pi 5 OS**</span> | TX Node | Ubuntu 22.04 / Debian 11/12 (ARM64) | Target single board computer OS |
| <span style="color: #EBCB8B;">**OpenSSH Server / Client**</span> | Both | OpenSSH >= 8.9 | Remote control, code deployment, and telemetry |
| <span style="color: #EBCB8B;">**V4L2 Utilities**</span> | TX Node | `v4l-utils` | USB camera capability querying and format inspection |
| <span style="color: #EBCB8B;">**FFmpeg / Libav**</span> | Both | FFmpeg >= 4.4 / 6.x | Video capture, encoding, streaming, and decoding |
| <span style="color: #EBCB8B;">**Python Runtime**</span> | Both | Python >= 3.10 | Automation test scripts and latency meters |
| <span style="color: #EBCB8B;">**OpenCV Python**</span> | RX Node | `opencv-python` >= 4.8 | Latency meter video rendering and OCR analysis |
| <span style="color: #EBCB8B;">**Network Connectivity**</span> | Both | Wi-Fi (802.11ac/ax) or GbE | Low-latency local network socket transmission |

### <span style="color: #81A1C1;">**3.2 Architectural Assumptions**</span>
- <span style="color: #EBCB8B;">**Camera Interface**</span>: Standard USB UVC video camera connected directly to Orange Pi 5 USB 3.0 port, providing YUYV or MJPEG pixel formats up to `1920x1080` at `30/60 fps`.
- <span style="color: #EBCB8B;">**Transport Protocol (Why UDP?)**</span>: UDP (User Datagram Protocol) is strictly required for real-time FPV video. TCP enforces retransmissions and stream stalls (head-of-line blocking) when packets drop, introducing 200–1500 ms latency spikes. Raw UDP / RTP drops corrupt slices instead of stalling the pipeline, keeping glass-to-glass latency fixed and minimal.
- <span style="color: #EBCB8B;">**SSH Control Plane**</span>: The host PC orchestrates TX capture and streaming on the Orange Pi 5 via automated SSH commands (`deploy_tx_opi5.sh`).

---

## <span style="color: #88C0D0;">**4. Detailed Flow & Component Breakdown**</span>

### <span style="color: #81A1C1;">**4.1 Component Roles & Responsibilities**</span>
- <span style="color: #EBCB8B;">**Orange Pi 5 Deployment Helper (`deploy_tx_opi5.sh`)**</span>: Synchronizes streamer code to the Orange Pi 5 over SSH/rsync, checks camera connectivity (`/dev/video*`), and starts/stops the TX streamer process remotely.
- <span style="color: #EBCB8B;">**Transmitter Streamer (`tx_streamer.py`)**</span>: Runs on Orange Pi 5. Handles V4L2 device capture, feeds raw frames into `x264` or `x265` with `tune=zerolatency`, and pushes encoded packets over UDP/RTP sockets directly to the receiver PC IP.
- <span style="color: #EBCB8B;">**Receiver Player (`rx_player.py`)**</span>: Runs on Linux PC. Listens on dedicated UDP/RTP ports, decodes incoming video slices immediately upon arrival, and renders frames to a zero-buffer window.
- <span style="color: #EBCB8B;">**Screen Timer Generator (`screen_timer.py`)**</span>: Displays a millisecond-accurate counter clock on the host monitor updating on every display refresh tick (e.g. `60 Hz` or `144 Hz`).
- <span style="color: #EBCB8B;">**Latency Analyzer (`measure_latency.py`)**</span>: Captures screen snapshots displaying both the live clock and the received webcam feed, performing OCR on the two timestamps to compute glass-to-glass latency in milliseconds:
  `Latency (ms) = Live Timestamp - Decoded Timestamp`.

### <span style="color: #81A1C1;">**4.2 End-to-End Data Lifecycle**</span>

<span style="color: #EBCB8B;">**Step 1 - Camera Sensor Frame Ingestion on Orange Pi 5**</span>
- Camera sensor captures optical photons and emits raw frames over USB UVC to the Orange Pi 5.
- Kernel V4L2 maps incoming memory frames into user-space via MMAP pointers.

<span style="color: #EBCB8B;">**Step 2 - Zero-Latency Video Compression on Orange Pi 5**</span>
- The encoder receives raw frames and encodes without B-frames (`bframes=0`).
- Slices are emitted immediately with minimal intra-frame lookahead (`rc-lookahead=0`).

<span style="color: #EBCB8B;">**Step 3 - UDP Socket Transmission Across Local Network**</span>
- Encoded NAL units are packetized into UDP datagrams or RTP packets.
- Datagrams are transmitted across the Wi-Fi / Ethernet link without queuing in socket buffers.

<span style="color: #EBCB8B;">**Step 4 - Depacketization & Fast Decoding on Linux PC**</span>
- PC socket receives UDP datagrams and feeds NAL units directly to the decoder.
- Video decoder reconstructs YUV macroblocks without waiting for future frames.

<span style="color: #EBCB8B;">**Step 5 - Screen Rendering & Optical Latency Verification**</span>
- Decoded video renders side-by-side with the live screen timer.
- Optical latency analyzer captures frames and measures empirical latency: `Latency (ms) = Live - Decoded`.

### <span style="color: #81A1C1;">**4.3 Approved Architectural Decisions**</span>
- <span style="color: #EBCB8B;">**Video Streaming Engine (Option C — Hybrid Pipeline)**</span>:
  - Streaming core runs direct high-performance GStreamer / FFmpeg pipelines to eliminate Python GIL overhead and buffer copies.
  - Python scripts ([`tx_streamer.py`](../../sw/subprojects/1_sw_video_pipeline/tx_streamer.py), [`rx_player.py`](../../sw/subprojects/1_sw_video_pipeline/rx_player.py)) manage CLI arguments, network socket setup, and telemetry logging.
- <span style="color: #EBCB8B;">**Optical Latency Meter Strategy (Option A — Fixed-Font Template Matching)**</span>:
  - [`screen_timer.py`](../../sw/common/latency_meter/screen_timer.py) renders high-contrast digital millisecond timestamps on the display.
  - [`measure_latency.py`](../../sw/common/latency_meter/measure_latency.py) performs normalized 2D cross-correlation (`cv2.matchTemplate`) against 10 pre-saved digit templates (0–9).
  - Sub-1.5 ms execution per frame, zero external OCR dependencies (pure OpenCV/NumPy), and zero misrecognition errors under motion blur.
- <span style="color: #EBCB8B;">**Benchmark Sweep Matrix (Option A — Core Matrix)**</span>:
  - Sweeps `720p @ 30/60 fps` and `1080p @ 30/60 fps` across H.264, H.265, and MJPEG.
  - Evaluates Raw UDP, RTP, and MPEG-TS socket transports under fixed FPV target bitrates (2–4 Mbps) with `tune=zerolatency`.

---

## <span style="color: #88C0D0;">**5. Step-by-Step Implementation Roadmap**</span>

- [ ] <span style="color: #EBCB8B;">**Phase 1.1: Camera Enumeration & Latency Meter Setup**</span>
  - Connect to Orange Pi 5 via SSH (`deploy_tx_opi5.sh`) and verify V4L2 USB camera detection (`/dev/video*`).
  - Enumerate camera hardware capabilities on Orange Pi 5 and populate `camera_capabilities.json`.
  - Implement and verify PC host latency measurement harness (`screen_timer.py` and `measure_latency.py`).

- [ ] <span style="color: #EBCB8B;">**Phase 1.2: Orange Pi 5 Zero-Latency TX Streamer Implementation**</span>
  - Implement `tx_streamer.py` and automated SSH synchronization script `deploy_tx_opi5.sh`.
  - Configure zero-latency encoding (H.264, H.265, MJPEG) with `tune=zerolatency`, `rc-lookahead=0`, `bframes=0`.
  - Stream encoded video payloads over UDP socket directly to the receiver PC host IP.

- [ ] <span style="color: #EBCB8B;">**Phase 1.3: Low-Latency RX Player Implementation**</span>
  - Implement `rx_player.py` on Linux PC host to ingest incoming UDP socket video packets.
  - Configure zero-buffer rendering surface with synchronized side-by-side display alignment.

- [ ] <span style="color: #EBCB8B;">**Phase 1.4: Distributed Benchmark Sweep & Sign-Off**</span>
  - Author and execute `benchmark_codecs.sh` orchestrating automated multi-codec runs across the network link.
  - Record bit-exact measurements across all matrix configurations in `benchmark_results.json`.
  - Author Stage 1 sign-off report in `summary.md`.

---

## <span style="color: #88C0D0;">**6. Verification & Sign-Off Checklist**</span>

| <span style="color: #EBCB8B;">**Test / Acceptance Case**</span> | <span style="color: #EBCB8B;">**Target Criteria**</span> | <span style="color: #EBCB8B;">**Actual Result**</span> | <span style="color: #EBCB8B;">**Status**</span> |
| :--- | :--- | :--- | :--- |
| <span style="color: #EBCB8B;">**Stream Continuity**</span> | Zero frame drops or stalls over 5 minutes continuous stream across network | Pending | <span style="color: #4C566A;">**Planned**</span> |
| <span style="color: #EBCB8B;">**ARM64 Encoding / Transmission**</span> | Continuous real-time encoding on Orange Pi 5 without pipeline drops | Pending | <span style="color: #4C566A;">**Planned**</span> |
| <span style="color: #EBCB8B;">**Transport Verification**</span> | Raw UDP stream exhibits sub-5 ms network transit over local LAN | Pending | <span style="color: #4C566A;">**Planned**</span> |
| <span style="color: #EBCB8B;">**Glass-to-Glass Latency**</span> | Glass-to-glass latency under 60 ms for 720p60 and under 85 ms for 1080p30 | Pending | <span style="color: #4C566A;">**Planned**</span> |
| <span style="color: #EBCB8B;">**Automated Latency Logging**</span> | Bit-exact benchmark sweep results exported to JSON | Pending | <span style="color: #4C566A;">**Planned**</span> |

---

## <span style="color: #88C0D0;">**7. Risks, Trade-offs & Mitigation**</span>

| <span style="color: #EBCB8B;">**Identified Risk**</span> | <span style="color: #EBCB8B;">**Severity**</span> | <span style="color: #EBCB8B;">**Mitigation Strategy**</span> |
| :--- | :--- | :--- |
| <span style="color: #EBCB8B;">**Orange Pi 5 CPU Load during Software Encoding**</span> | Medium | Use `preset=ultrafast`, zero B-frames, and limit threads; if software x264 exceeds CPU budget on 1080p60, fall back to 720p60 or evaluate initial MPP hardware acceleration path. |
| <span style="color: #EBCB8B;">**Wi-Fi Packet Jitter / Packet Loss**</span> | Medium | Use 5 GHz Wi-Fi band or wired Gigabit Ethernet connection for benchmark baselining; configure socket send/receive buffer sizes appropriately. |
| <span style="color: #EBCB8B;">**UVC Driver Buffer Delays on SBC**</span> | Low | Query driver buffer queue depths via `v4l2-ctl` and request minimum buffer allocations (`count=2` in MMAP). |
