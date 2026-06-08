# 🌆 CityCams — Public City-Camera Platform

Two production systems built on the same **~200 public city-camera** foundation:

| | System | What it does |
|---|---|---|
| 🔴 | **CityCams Live** | A single-process **GStreamer** engine that broadcasts ~200 cameras **24/7 to YouTube + Kick** with **gapless** switching and **live chat control**. |
| 🎬 | **CityCams Shorts** | An automated **computer-vision pipeline** that mines those cameras for interesting moments and publishes **vertical short-form video** hands-free. |

![Python](https://img.shields.io/badge/Python-3.10-3776AB?logo=python&logoColor=white)
![GStreamer](https://img.shields.io/badge/GStreamer-1.20-FF6600)
![YOLO](https://img.shields.io/badge/CV-YOLOv8-00FFFF)
![Egress](https://img.shields.io/badge/egress-YouTube%20%2B%20Kick%20%2B%20TikTok-red)
![Uptime](https://img.shields.io/badge/uptime-24%2F7-brightgreen)

> **Architecture & engineering showcase.** Production source is private (it carries
> infrastructure and credentials). Documented here: the design, the hard problems, and
> representative code. _Full source available on request._

## ▶️ Live demos
- **Live stream — YouTube:** `<YOUR_YOUTUBE_LIVE_URL>`
- **Live stream — Kick:** `<YOUR_KICK_URL>`
- **Shorts channel:** `<YOUR_SHORTS_CHANNEL_URL>`

> _Replace with your links — the broadcast is on right now._

---

## Why both systems share one foundation

Cities publish hundreds of public street cameras as **heterogeneous, frequently-flaky HLS
streams** — different CDNs, codecs, TLS trust quirks, short-lived URL tokens, and geo-blocks.
A shared layer normalizes all of that (URL resolution, per-source headers, a curated camera
pool, a geo-aware request mesh). On top of it sit two very different consumers: a **real-time
streaming engine** and a **batch CV/ML video factory**. Same cameras, opposite workloads.

---

# 🔴 System 1 — CityCams Live (real-time streaming)

A single **Python + GStreamer** process that streams ~200 cameras 24/7 to **YouTube (RTMP)
and Kick (RTMPS) simultaneously**, on a **3-vCPU** box at **~1.6 cores**, **28+ days with no
encoder restart**, switchable from chat.

### Architecture
```
          chat commands (YouTube pytchat + Kick Pusher-ws)
                              │
                              ▼
   ~200 HLS cams      ┌──────────────┐  atomic source swap   ┌───────────────┐
   (1 active,         │   Source     │ ───────────────────►  │ Pusher thread │
    1 pre-warm)  ───► │ ffmpeg  -re  │      latest frame      │  25 fps CFR   │
                      │ (real-time)  │ ─────────────────────► │ (dup last     │
                      └──────────────┘                        │ frame on stall)│
                                                              └──────┬────────┘
                                                                     ▼ frame every 40 ms
                       appsrc ─► x264 (HARD CBR, bframes=0) ─► flvmux ─► ffmpeg (egress)
                                                                     ▼
                                                          MediaMTX (local RTMP hub)
                                                       ┌─────────────┴───────────┐
                                                       ▼                         ▼
                                                YouTube (RTMP)             Kick (RTMPS)
```

### Hard problems solved
- **One encode, two platforms, never restart.** A continuously-fed `appsrc` decouples ingest
  from encode; the pusher guarantees a frame every 40 ms (duplicating the last one on stall) so
  x264 runs in a steady state for weeks. Hard-CBR + `bframes=0` keep the bitstream clean.
- **Gapless switching without rebuilding the pipeline.** The active camera is just a reference
  the pusher reads from; switching swaps that reference atomically (old source retired off the
  hot path). No element churn, no renegotiation → **~25–40 s switches, zero discontinuity.**
- **Audio that survives YouTube's transcoder.** Glitches traced to non-monotonic AAC DTS snapped
  to the video grid (YouTube drops them, Kick tolerated). Fix: re-encode egress audio to clean,
  monotonic **48 kHz** AAC (`aresample=async=1`), copy video untouched.
- **An audio bed that can't die.** The looping music ffmpeg is wrapped in a **bash supervisor**
  so the pipe's write-end never closes (a silent audio-EOF once killed sound with no crash to recover from).
- **Surviving a lossy intercontinental path.** TCP CUBIC collapsed throughput over time
  (`videoIngestionStarved`); fixed with **BBR** + relaying the single feed through a closer region.
- **Self-healing.** systemd watchdog (pinged only while frames advance) + slate-on-stall +
  a relay watchdog that restarts *only* the egress when it wedges. `health.json` heartbeat feeds a control panel.

### Core idea (simplified)
```python
# A single long-lived encoder reads from whatever PUSH["src"] points to.
# Switching cameras = swapping that reference. The pusher always feeds a frame at a
# fixed cadence — duplicating the last good one on stall — so the encoder never breaks.
PUSH = {"src": slate_source}

def switch_camera(new_source):                 # from the chat-command handler
    new_source.start()                         # warm up: begin real-time decode
    if new_source.wait_until_ready(timeout=10):
        PUSH["src"] = new_source               # ← atomic swap; no pipeline rebuild
        retire_async(previous_source)          # tear down old source off the hot path
    else:
        new_source.stop()                      # dead camera → skip, stay on air

def pusher_loop(appsrc):                        # own thread, 25 fps CFR
    pts, last = 0, slate_frame
    while running:
        last = PUSH["src"].latest or last       # never block the encoder
        push_frame(appsrc, last, pts)           # constant 40 ms timestamp
        pts += FRAME_DURATION_NS
        sleep_until_next_tick()
```

---

# 🎬 System 2 — CityCams Shorts (computer-vision automation)

A **scheduled, hands-off pipeline** that turns the same cameras into vertical short-form videos
and auto-publishes them — running **CPU-only (no GPU)**.

### Pipeline
```
 slot scheduler (≈6×/day)
        │
        ▼
 pick city ──► resolve HLS URL ──► single-frame probe (Referer)  ──► dead / boring? → skip
 (TR pool +        (normalizer)        (cheap pre-screen)
  US sources)
        │ alive & promising
        ▼
 record short clip ──► YOLO gate (person / vehicle, FAIL-CLOSED) ──► no detection / error → reject
        │ pass
        ▼
 vertical render: smart crop + Turkish overlay + synthesized voice/weather line + music bed
        │
        ▼
 auto-publish:  YouTube Shorts   +   TikTok (via Telegram handoff)
```

### Hard problems solved
- **Fail-closed content gating.** A clip is uploaded **only** if YOLO positively detects something
  worth showing; any error rejects it. The system never publishes unscreened footage — important
  for quality *and* privacy on public cameras.
- **Cheap-first screening.** Before spending CPU on a full record→detect→render cycle, a single
  decoded frame (fetched with each camera's required `Referer`) screens out dead or empty cameras —
  most rejections cost almost nothing.
- **Sourcing through a geo-aware mesh.** ~200 cameras span many TR cities plus US feeds. Some
  origins geo-block foreign IPs, so *only* those requests route through a region-correct forward
  proxy, while the heavy work (YOLO, encode, upload) stays on the main box. A resolver layer hides
  per-city CDN/TLS/token differences.
- **Truly unattended publishing.** A slot-scheduled daemon renders vertical Shorts with burned-in
  Turkish overlays and a synthesized voice line, uploads to YouTube Shorts, and mirrors to TikTok
  via a Telegram handoff — no human in the loop.
- **CPU-only ML on a budget.** YOLOv8-nano on CPU (PyTorch CPU build), kept within a tight resource
  envelope so it coexists with other workloads without a GPU.

### Core idea (simplified)
```python
# Fail-closed gate: upload ONLY on a positive YOLO detection. Any error → reject.
def passes_gate(clip) -> bool:
    try:
        dets = yolo.detect(sample_frames(clip), classes=WANTED)   # person / vehicle / ...
    except Exception:
        log.warning("YOLO failed → reject (fail-closed)")
        return False                          # error means NO upload, ever
    return enough_signal(dets) and motion_ok(clip)
```

---

## 🧱 Shared infrastructure
- **Camera resolver layer** — normalizes ~200 heterogeneous municipal HLS sources (per-source
  headers/Referer, short-lived URL-token regeneration, TLS trust workarounds, HTTP-only proxying).
- **Curated camera pool** — health-tracked catalog with weekly auto-refresh and decay protection.
- **Geo-aware request mesh** — region-correct egress so geo-blocked origins still resolve.

## 🛠️ Tech stack
**Common:** Python (asyncio + threads), ffmpeg, systemd, MediaMTX
**Live:** GStreamer 1.20 (`appsrc`, software x264), pytchat (YouTube chat), Pusher WebSocket (Kick), BBR TCP
**Shorts:** PyTorch (CPU) + Ultralytics **YOLOv8**, OpenCV, edge-TTS, YouTube Data API, Telegram Bot API

## 📊 Results
| | Live | Shorts |
|---|---|---|
| Cameras | ~200 (1 live + 1 pre-warm) | ~200 across many cities + US |
| Output | 720p · ~2.8 Mbps · 25 fps H.264 | vertical Shorts (YouTube + TikTok) |
| Compute | ~1.6 / 3 vCPU, no GPU | CPU-only YOLOv8n |
| Reliability | 28+ days, 0 encoder restarts | fail-closed, unattended, scheduled |
| Interaction | live chat commands (2 platforms) | fully autonomous |

## 🔒 What's intentionally *not* in this repo
Production specifics are omitted on purpose: server addresses, stream keys, the camera catalog and
how it's sourced, and exact tuning values. This repo documents **how the systems are built**, not
the keys to run them. Happy to walk through the full implementation in an interview.

---

*Designed, built, deployed, and operated solo — real-time media infrastructure + CV/ML automation, 24/7.*
