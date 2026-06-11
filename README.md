<div align="center">

# RyoCam 亮像

**面向 Windows 的系统级虚拟摄像头 · GPU 实时人像处理 · Rust 原生高性能**

**A system-level virtual camera for Windows · GPU-accelerated real-time portrait processing · built in native Rust**

[中文介绍](#中文介绍) · [English](#english)

</div>

---

# 中文介绍

## 一句话简介

**RyoCam（亮像）** 是一款用 Rust 编写的 Windows 桌面虚拟摄像头软件。它把你的物理摄像头画面经过 GPU 实时处理（智能抠图、背景替换、美颜、自动取景、身份融合超级美颜），再以**系统级虚拟摄像头**的身份输出，可被 OBS、Zoom、腾讯会议、钉钉、浏览器以及 Windows「相机」应用直接调用——无需任何插件。

> 全程**本地处理、纯离线运行**，画面不经过任何云端服务器，兼顾隐私与低延迟。

---

## 核心功能

### 1. 系统级虚拟摄像头（Windows 11）

- 基于 Windows 11 官方的 `MFCreateVirtualCamera` 接口，注册为 **Media Foundation 系统媒体源**，对所有相机消费端（OBS / Zoom / 浏览器 / 系统相机）一视同仁地可见，名为「RyoCam Virtual Camera」。
- 处理后的画面通过一条**无锁共享内存环形缓冲**从主程序传给运行在 Windows 帧服务器（Frame Server）中的 COM 源 DLL，延迟极低。
- **多分辨率协商**：向消费端暴露多档分辨率（而非仅 720P），适配不同会议软件与录制软件的需求。
- **会话级生命周期**：程序退出即自动注销虚拟摄像头；若程序异常崩溃，源端会在数秒后自动回落到测试色条，不会让消费端卡死黑屏。
- 一键「安装驱动（管理员）」：自动把源 DLL 部署到帧服务器可读的 `%ProgramData%` 路径并注册 CLSID，并在更新 DLL 后重启帧服务器。

### 2. GPU 实时抠图与背景处理

- **人像分割**采用 ONNX Runtime + **DirectML** 硬件加速推理（MODNet 等模型），在独显/核显上运行，不占满 CPU。
- **背景模式齐全**：背景虚化、图片替换、纯色填充、绿幕、直通透明（straight-alpha，可供 OBS 二次合成）。
- **抠图范围可选**：「仅人物 / 标准 / 含道具」三档，兼顾「连同手持物体一起保留」与「贴脸干净抠除」等不同场景。
- **柔边遮罩精修流水线**：双线性上采样 → 运动自适应时域平滑（EMA）→ 形态学边缘收缩 → 羽化 → 联合双边（导向）边缘细化，全程保持**软边过渡**，发丝与边缘自然不锯齿，并带边缘去色溢出（despill）。
- 抠图、精修与合成均在 **GPU（wgpu/DirectX）** 上完成，端到端 GPU 流水线，避免 CPU↔GPU 反复搬运。

### 3. 美颜

- 提供风格预设（**美眉 / 帅哥 / 全参数**）一键切换，也可进入全参数模式精细调节。
- 磨皮、亮肤等常规参数之外，特别加入**「轮廓」**参数：增强鼻梁、脸颊、眼窝等部位的立体感（明暗塑形），而非简单提亮发灰。
- 美颜与背景替换可**同时生效**，互不冲突。

### 4. 自动取景（人脸追踪）

- 基于 YuNet 人脸检测 + 关键点，**自动追踪人脸并居中**到输出画面。
- 即使你移动到输入画面边缘，也能持续跟随并保持构图居中；带**边缘感知缩放**与最大裁切上限，避免越界与过度放大。
- 提供「特写」等取景模式，适合单人口播、直播场景。

### 5. 超级美颜（身份融合）

这是 RyoCam 的进阶亮点——不是简单贴滤镜，而是基于深度模型的**身份融合**：

- **ArcFace** 提取人脸 512 维身份向量，**inswapper** 完成换脸级身份融合，**GFPGAN** 负责换脸后的高频细节重建（皮肤、睫毛、发丝），输出更清晰自然。
- 通过 **Slerp（球面插值）** 在「你的身份」与「模板身份」之间平滑融合，由「相似度」滑块控制融合强度，保证身份连贯而非生硬拼贴。
- **色彩迁移**让融合区域与原始肤色无缝衔接。
- **One Euro 滤波器**对关键点做自适应时域平滑，显著消除面部整体抖动/果冻感。
- **内置 AI 虚拟偶像模板**（女爱豆 / 男爱豆，均为 AI 生成的虚拟人脸，非真人），并提供**缩略图画廊**直观选择；也支持用你自己的照片生成模板。

> 身份融合所需的换脸模型（arcface / inswapper / gfpgan）因许可与体积原因不随安装包分发，由用户自行下载放入 `models/`，详见发布说明。

### 6. 内测授权（加油码）

- 首次安装享 **7 天试用**，每输入一个有效「加油码」延长 **30 天**；**同一台机器重复输入相同加油码不会重复延期**。
- 到期后软件**完全锁定**并提示输入加油码，不会泄露任何可用功能。

### 7. 内置版本更新

- 启动时从 GitHub 公开仓库拉取一份**经 Ed25519 签名**的更新清单（被污染的镜像无法推送恶意版本）。
- **多镜像竞速**（raw / jsDelivr / fastly）+ 短超时，缓解 GitHub 在国内的访问速度问题。
- 应用内**一键下载 + SHA-256 校验**，由独立的更新器 `RyoCam-apply` 在主程序退出后替换文件并自动重启。
- 支持「更新 / 稍后 / 忽略此版本」，并展示 What's New（中英双语更新说明）。

### 8. 其它

- **中英双语界面**，启动时按操作系统语言自动切换。
- 设置持久化（`%APPDATA%/RyoCam/settings.json`）与预设。
- 实时遥测：输出帧率、端到端延迟、丢帧统计。

---

## 性能优势

- **原生 Rust，零 GC 停顿**：无垃圾回收引发的卡顿，内存安全且贴近硬件性能；发布版启用 `thin-LTO + codegen-units=1 + opt-level=3` 深度优化。
- **端到端 GPU 流水线**：抠图推理（DirectML）、遮罩精修、背景合成均在 GPU 上完成，CPU 占用低，遮罩精修较早期实现提速约 2 倍。
- **自动选卡**：DXGI 枚举显卡并自动优选独显（NVIDIA / AMD → Intel 核显 → CPU 回退），用户无需手动配置。
- **异步多线程架构**：推理线程、身份融合线程与输出循环**解耦**，重负载（分割 / 换脸 / 增强）不再阻塞出图，避免帧在队列里堆积导致的累积丢帧。
- **可调推理尺寸**（256 / 384 / 512）：在画质与速度之间自由权衡；较小尺寸往往更稳定、抠图边缘抖动更小。
- **重复帧过滤**：识别并丢弃 Media Foundation 速率转换器产生的逐字节重复帧，避免无意义的重复计算。
- **低延迟输出**：虚拟摄像头侧采用无锁共享内存环形缓冲，主程序到帧服务器的传输开销极小。

---

**实时处理流水线：**

```text
采集 → RGBA 归一化 → 人像分割 → 遮罩上采样/精修 → 时域平滑
     → （可选）自动取景 → （可选）身份融合 + GFPGAN 增强
     → 背景合成 → 预览 + 虚拟摄像头输出
```

---

## 系统要求

- **操作系统**：Windows 11（build 22000+）可使用系统虚拟摄像头；Windows 10 可运行处理与预览功能。
- **显卡**：支持 DirectX 12 / DirectML 的独显或核显（建议独显以获得最佳帧率）；不支持时可回落 CPU。
- **运行依赖**：发布包已内置所需运行库（含 `DirectML.dll`、静态链接的 ONNX Runtime）。

---

## 隐私

RyoCam **完全在本地运行**，不上传任何画面或身份数据；授权与更新检查也仅在你主动联网时进行，可纯离线长期使用。

---
---

# English

## In one line

**RyoCam** is a Windows desktop virtual-camera app written in Rust. It runs your physical camera feed through a real-time GPU pipeline (smart matting, background replacement, beauty, auto-framing, and identity-fusion "super beauty") and presents the result as a **system-level virtual camera** that OBS, Zoom, Teams, browsers, and the Windows Camera app can use directly — no plugins required.

> Everything runs **locally and fully offline**: frames never leave your machine, which is good for both privacy and latency.

---

## Core features

### 1. System-level virtual camera (Windows 11)

- Built on the official `MFCreateVirtualCamera` API and registered as a **Media Foundation system media source** ("RyoCam Virtual Camera"), visible to every capture consumer (OBS / Zoom / browsers / the Camera app).
- Processed frames travel from the app to a COM source DLL hosted in the Windows **Frame Server** over a **lock-free shared-memory ring**, keeping latency very low.
- **Multi-resolution negotiation**: exposes several resolutions (not just 720p) to fit different conferencing and recording apps.
- **Session-scoped lifetime**: the virtual camera is removed automatically on exit; if the app crashes, the source falls back to a test pattern after a few seconds instead of freezing consumers.
- One-click **"Install driver (admin)"** deploys the source DLL to a Frame-Server-readable `%ProgramData%` path, registers the CLSID, and restarts the Frame Server after DLL updates.

### 2. GPU real-time matting & backgrounds

- **Person segmentation** via ONNX Runtime + **DirectML** hardware acceleration (MODNet-class models), running on the GPU instead of saturating the CPU.
- **Full set of background modes**: blur, image replacement, solid color, green screen, and straight-alpha passthrough (for downstream compositing in OBS).
- **Selectable matting scope** — *Person only / Standard / Include props* — covering both "keep the object I'm holding" and "clean cut right at the face".
- **Soft-mask refinement pipeline**: bilinear upscale → motion-adaptive temporal EMA → morphological edge shift → feather → joint-bilateral (guided) edge refinement, staying soft end-to-end for natural, non-jagged hair edges, plus edge despill.
- Matting, refinement, and compositing all run **on the GPU (wgpu/DirectX)** as an end-to-end GPU pipeline, avoiding repeated CPU↔GPU copies.

### 3. Beauty

- Style presets (**美眉 / 帅哥 / full-parameters**) for one-click looks, with a full manual mode for fine control.
- Beyond smoothing/brightening, a dedicated **"contour"** parameter adds facial dimensionality (nose bridge, cheeks, eye sockets) through light/shadow shaping rather than flat brightening.
- Beauty and background replacement work **simultaneously**.

### 4. Auto-framing (face tracking)

- Uses YuNet face detection + landmarks to **track and center the face** in the output.
- Keeps following and recentering even when you move to the edge of the input, with **edge-aware zoom** and a maximum-crop cap to avoid overscaling.
- Offers framing modes such as close-up, ideal for solo presenting and streaming.

### 5. Super beauty (identity fusion)

RyoCam's advanced highlight — not a sticker filter, but a deep-model **identity fusion**:

- **ArcFace** extracts a 512-D identity vector, **inswapper** performs swap-grade identity fusion, and **GFPGAN** reconstructs high-frequency detail (skin, eyelashes, hair) afterward for a sharper, more natural result.
- **Slerp (spherical interpolation)** blends *your* identity with the *template* identity, controlled by a "similarity" slider, keeping identity coherent rather than a hard paste.
- **Color transfer** seamlessly matches the fused region to your original skin tone.
- A **One Euro filter** adaptively smooths landmarks over time, largely eliminating whole-face jitter / jelly wobble.
- Ships with **built-in AI virtual-idol templates** (female / male idol — AI-generated faces, not real people) shown in a **thumbnail gallery**; you can also build templates from your own photos.

> The fusion models (arcface / inswapper / gfpgan) are not bundled (license + size) — download them into `models/` yourself; see the release notes.

### 6. Beta licensing ("fuel code")

- **7-day trial** on first install; each valid "fuel code" adds **30 days**; **re-entering the same code on the same machine never re-extends**.
- On expiry the app **locks down completely** and prompts for a code, exposing no functionality.

### 7. Built-in updates

- On startup it fetches an **Ed25519-signed** update manifest from a public GitHub repo (so a poisoned mirror can't push a malicious build).
- **Mirror racing** (raw / jsDelivr / fastly) with short timeouts mitigates slow GitHub access in some regions.
- In-app **one-click download + SHA-256 verification**; a separate `RyoCam-apply` helper replaces files after the app exits and relaunches.
- **Update / Later / Skip-this-version** options plus a bilingual What's New.

### 8. Extras

- **Bilingual UI** (Chinese / English), auto-selected by OS locale on startup.
- Settings persistence (`%APPDATA%/RyoCam/settings.json`) and presets.
- Live telemetry: output FPS, end-to-end latency, dropped-frame stats.

---

## Performance advantages

- **Native Rust, zero GC pauses**: no garbage-collection stutter, memory-safe and close to the metal; release builds use `thin-LTO + codegen-units=1 + opt-level=3`.
- **End-to-end GPU pipeline**: inference (DirectML), mask refinement, and compositing all on the GPU; mask refine is ~2× faster than the early implementation.
- **Automatic GPU selection**: DXGI enumerates adapters and prefers a discrete GPU (NVIDIA / AMD → Intel iGPU → CPU fallback) with no manual config.
- **Asynchronous multi-threaded architecture**: inference and fusion threads are **decoupled** from the output loop, so heavy work (segmentation / swap / enhance) never blocks frame delivery or causes queued-frame buildup and cascading drops.
- **Adjustable inference size** (256 / 384 / 512): trade quality vs. speed freely; smaller sizes are often more stable with less edge jitter.
- **Duplicate-frame filtering**: detects and drops byte-identical frames emitted by the Media Foundation rate converter, avoiding wasted recomputation.
- **Low-latency output**: a lock-free shared-memory ring keeps app→Frame-Server transfer overhead minimal.

---

**Real-time pipeline:**

```text
capture → RGBA normalize → segmentation → mask upscale/refine → temporal smoothing
        → (optional) auto-framing → (optional) identity fusion + GFPGAN enhance
        → background composite → preview + virtual-camera sink
```

---

## Requirements

- **OS**: Windows 11 (build 22000+) for the system virtual camera; Windows 10 can still run processing and preview.
- **GPU**: a DirectX 12 / DirectML-capable discrete or integrated GPU (discrete recommended for best FPS); CPU fallback otherwise.
- **Runtime**: the release bundles what it needs (`DirectML.dll`, statically linked ONNX Runtime).

---

## Privacy

RyoCam runs **entirely locally** and uploads no video or identity data. Licensing and update checks only happen when you go online, and the app works fully offline.
