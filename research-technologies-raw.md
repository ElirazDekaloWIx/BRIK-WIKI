# BRIK — Integration Research Report
*A deep dive into technologies, APIs, and research techniques that can be integrated into the BRIK creative-AI platform to dramatically expand what designers, motion artists, VJs, typographers, and procedural creators can build from natural-language prompts.*

**Author:** Research pass, web-sourced (WebSearch + WebFetch)
**Date:** April 2026
**Scope:** WebGPU, assets, computer vision, generative media, audio, physics, typography, color, academic research, inference infrastructure.

All references are numbered [n] and collected at the end. Inline citations refer to the numbered bibliography.

---

## Executive summary

BRIK today already spans a rich creative-tech surface: 2D Canvas, WebGL via Three.js, p5.js, Hydra, Matter.js, GSAP. The single biggest structural upgrade available in the 2025–2026 window is **WebGPU**: as of early 2026, WebGPU finally ships by default in Chrome/Edge (all desktop platforms), Safari 26 on macOS Tahoe / iOS 26 / iPadOS 26 / visionOS 26, and Firefox 141+ on Windows with macOS support added in Firefox 145 [1][2][3][4]. WebGPU unlocks **compute shaders, 128MB storage buffers, and up to ~15–30× faster compute workloads** vs WebGL's texture-as-compute hacks [5][6][7][8]. This is a civilizational upgrade for creative apps, and the migration path is ready because **Three.js TSL (Three Shading Language)** compiles the same code to both WGSL (WebGPU) and GLSL (WebGL) with automatic fallback [9][10][11]. BRIK should treat WebGPU as the primary modality for new high-compute tools while keeping WebGL for the long tail of older devices.

Beyond the renderer, the report identifies six concrete integration vectors that would each step-change BRIK's capability:

1. **CC0 asset graph (Poly Haven + AmbientCG + Kenney + Quaternius + Pixabay + Freesound)** — let the assembler pull materials, HDRIs, models, photos, video, sounds under no-attribution licenses.
2. **In-browser vision (Transformers.js v3 + MediaPipe Tasks Vision + ONNX Runtime Web)** — background removal, pose/face/hand tracking, depth estimation, object detection all on-device with WebGPU acceleration and no server cost.
3. **Real-time generative image (fal.ai LCM / SDXL Lightning / FLUX Schnell)** — sub-250ms image generation over WebSocket, making "paint with AI" tools feasible.
4. **Unified inference gateway (fal.ai primary, Replicate fallback, OpenRouter for LLM)** — one SDK, hundreds of models, transparent commercial licensing.
5. **Modern color + typography stack (Culori + OKLCH + HarfBuzzJS + Fontsource + variable-font TSL shaders)** — brings BRIK into parity with Figma/Framer for brand/type work.
6. **Compute-shader library ecosystem (Rapier for physics, WebGPU reaction-diffusion, MLS-MPM fluids, Gaussian Splatting)** — enables simulation-driven tools that were previously impossible in the browser.

The **Ranked Top 20 integrations** list at the end gives the prioritized execution plan.

---

## Category 1 — WebGPU and next-gen graphics

### 1.1 State of WebGPU (2025–2026)

WebGPU reached **"critical mass"** in late 2025: by November 2025 it shipped by default in Chrome, Edge, Firefox, and Safari [1][2]. Key facts:

- **Chrome / Edge**: ships on Windows (D3D12), macOS, ChromeOS, Android 12+ (Qualcomm/ARM GPUs since Chrome 121); Chrome 144 Beta adds Intel Gen12+; broader Linux support in progress [1][2].
- **Safari**: Safari 26.0 enables WebGPU on macOS Tahoe 26, iOS 26, iPadOS 26, visionOS 26 — including **HDR canvas** and **WebXR + WebGPU on Vision Pro** as of Safari 26.2 [1][2].
- **Firefox**: 141 on Windows; 145 added macOS Apple Silicon; Linux/Android expected in 2026 [1].
- **caniuse** as of March 2026 reports ~84.7% global support [12].

Bottom line: WebGPU is safe to use as the default rendering path for modern devices, with WebGL fallback retained.

### 1.2 What WebGPU unlocks vs WebGL

| Capability | WebGL 2 | WebGPU |
|---|---|---|
| Compute shaders | No (hack via fragment shaders) | **Yes, native** [5][6] |
| Max storage buffer | Effectively ~texture size | **128 MB** per binding [6] |
| GPU atomics, workgroup memory | No | **Yes** (`atomicAdd`, shared memory, `storageBarrier()`) [6][13] |
| Typical compute speedup | 1× | **3–8× at ≥2048², up to 15–30× for ML workloads** [5][8] |
| Creative example | 10k particles @ 30ms | **1M particles @ <2ms** (~150× improvement) [14][15] |

Concrete 2025 demos include MLS-MPM fluid simulation at 100k particles [14][16], real-time curl-noise particle flow fields [14], reaction-diffusion Gray-Scott in compute shader [17][18], and million-particle galaxy simulations [19].

### 1.3 Library wrappers

**Three.js WebGPURenderer + TSL (Three Shading Language)** is the dominant pick. TSL is a node-based, JavaScript-authored shader system that **compiles to both WGSL and GLSL**, so the same shader runs on WebGPU or falls back to WebGL automatically [9][10][11][20]. It includes built-ins for PBR, post-processing (bloom, GTAO, FXAA, DoF, chromatic aberration, LUT3D, ACES/AgX tone mapping), variable-font-style conditional/loop constructs (`If()`, `Loop()`, `Switch()`), and a `compute(node, count, workgroupSize)` API with atomics [20]. A migration guide for late-2026 exists [21].

**Babylon.js 9** made WebGPU the **default renderer** in 2026 with ~2× performance over WebGL and rich compute-shader support (volumetric lighting, directional lights) [22][23]. Good choice for tools that need a full scene-graph/engine (gizmos, PBR, animation blending, physics bindings) rather than lower-level shader composition.

**Orillusion** is a pure-WebGPU engine (MIT) that positions itself around GPU physics (particles, fluids, soft bodies, rigid bodies, cloth) and PBR + SSR + HDR bloom [24][25]. Worth evaluating for simulation-heavy tools.

**Dedicated WebGPU libraries** for creative coding — MiniGPU, useGPU, WebGPU Fundamentals tutorials — are maturing but most production BRIK tools will ride on Three.js + TSL.

### 1.4 Implementation guidance for BRIK

1. **Add a renderer mode flag** to the assembler alongside `webgl`, `p5`, `hydra`, `canvas2d`: `webgpu` (Three + TSL).
2. **Default new tools to TSL** so the same code runs on both backends. The WebGPURenderer automatically falls back to WebGL when unsupported; this is the safest migration path [11].
3. **Introduce a `compute` primitive** in the assembler — a first-class concept for compute shaders. Use it to ship flagship BRIK templates: "million-particle flow field", "reaction-diffusion paint", "MLS-MPM fluid playground", "SDF sculpt tool".
4. **Feature-detect at runtime** via `navigator.gpu` and expose a "High-performance mode available" hint in the canvas toolbar when the user's browser supports WebGPU.
5. **Constrain compute-intensive templates** to WebGPU (with a friendly fallback screen in WebGL), because authoring equivalent fragment-shader hacks doubles engineering cost.

### 1.5 Academic / industry references

- Codrops: "WebGPU Fluid Simulations: High Performance & Real-Time Rendering" (2025) [14], "Particles, Progress, and Perseverance: A Journey into WebGPU Fluids" (2025) [16], "Reaction-Diffusion Compute Shader in WebGPU" (2024) [17].
- Chrome Dev: "From WebGL to WebGPU" [5].
- WebGPU Fundamentals: "WebGPU from WebGL" (migration), "Storage Buffers", "Compute Shader Basics" [6][7][13].
- webgpuexperts.com monthly roundups [26].

---

## Category 2 — Asset APIs (textures, HDRIs, models, images, video, audio, icons, fonts)

**Licensing sanity table** (most permissive first):

| Source | License | Attribution | Commercial | API |
|---|---|---|---|---|
| Poly Haven | CC0 | Not required [27][28] | Yes | `api.polyhaven.com` REST [29] |
| AmbientCG | CC0 | Not required [30] | Yes | `/api/v2/full_json` REST [30][31] |
| Kenney | CC0 | Not required [32] | Yes | No formal API — direct download [32] |
| Quaternius | CC0 | Not required [32] | Yes | No formal API |
| ShareTextures | CC0 | Not required [33] | Yes | No API |
| Pixabay | Pixabay License | Not required [34] | Yes | REST, 100 req/min [34][35] |
| Videvo | Multi-tier (RF / Attribution / CC 3.0) [36] | Depends | Yes | REST, 100 req/min [36] |
| Unsplash | Unsplash License (API requires attribution) [37][38] | **Required via API** | Yes | REST, 50/hr demo, 5000/hr prod [37] |
| Pexels | Pexels License | Not required [39] | Yes | REST, 200 req/hr, 20k/mo [39] |
| Freesound | CC0 / CC BY / CC BY-NC per sound [40][41] | Depends per sound | Mostly (filter NC) | REST v2 [40] |
| Sketchfab | CC BY by default, some CC BY-NC/-ND/-SA [42] | **Required** | Most yes | OAuth Download API [42] |
| Lucide | ISC | Not required [43][44] | Yes | CDN + NPM |
| Tabler Icons | MIT [43] | Not required | Yes | CDN + NPM |
| Iconify | MIT framework, per-set licenses [43] | Depends | Yes | REST + CDN |
| Noun Project | CC BY (free) or paid royalty-free [45][46] | CC BY requires credit | Yes | REST |
| Google Fonts | SIL OFL / Apache (per font) [47] | Not required in most | Yes | CSS2 API |
| Fontsource | Bundled OSS fonts (OFL/Apache) [48] | Per font | Yes | NPM packages |

### 2.1 Poly Haven (CC0)

**What**: 600+ HDRIs, 1000+ PBR textures, 300+ 3D models, all CC0 [27][28].
**Why for BRIK**: perfect seed content for "material picker", "HDRI lighting picker", "3D model prop library" nodes in the assembler. No attribution, no takedown risk.
**API**: Public REST at `api.polyhaven.com`; separate API Terms apply; encouraged for tool integration [29].
**Integration**: proxy the API server-side in Base44 to avoid CORS and to cache. Expose an asset picker that lets LLM prompts resolve ("weathered wood texture") to the best match via the `/assets` + tag filtering endpoints. Cache resized 2K/4K variants on BRIK CDN.

### 2.2 AmbientCG (CC0)

**What**: 3000+ PBR materials, HDRIs, also some 3D models, all CC0 [30].
**Why**: a broader PBR library than Poly Haven, with a sophisticated tag system (category, method: PBRPhotogrammetry/PBRProcedural, etc.).
**API**: `https://ambientcg.com/api/v2/full_json?include=imageData,downloadData&category=Wood` [30][31]. Tailorable include list keeps payloads small.
**Integration effort**: ~1 day for a Deno function wrapper; ~2 days for AI-tagged semantic search over the catalog.

### 2.3 Kenney / Quaternius / ShareTextures (CC0)

**What**: Kenney ships **40k+ CC0 game assets** (2D sprites, 3D voxels, UI kits, audio) [32]. Quaternius: stylized low-poly 3D (fantasy, sci-fi, nature, platformer) [32]. ShareTextures: 100+ 4K seamless PBR [33].
**Why**: gives BRIK a "stylized" aesthetic bucket distinct from Poly Haven's photoreal. Great for game-like, toylike, or iconographic tools.
**API**: none formal; bulk download + re-host on BRIK CDN. This is CC0-compliant and lets BRIK ship them in export bundles.

### 2.4 Sketchfab (CC BY default + commercial Download API)

**What**: 1M+ 3D models in glTF/GLB/USDZ; most CC BY; some CC BY-NC/-ND/-SA; paid models via Fab [42].
**Why**: massive breadth for "find me a 3D model of X" — artisan models, scans, characters, vehicles.
**API**: OAuth-authenticated Download API returning short-expiry signed URLs [42].
**Critical caveat**: CC BY requires BRIK to **display author + Sketchfab + license inline wherever the model appears**. The assembler must render this attribution automatically, and generated `.clay.html` exports must include it. Allow users to filter for CC0-only when they want attribution-free output.

### 2.5 Stock images + video

- **Unsplash**: Production rate 5000/hr after approval [37]. API **requires attribution display** and image-view tracking (hotlink). Good for "reference photo → stylize" flows.
- **Pexels**: 200 req/hr, 20k/mo, no attribution required [39]. Preferred default for BRIK.
- **Pixabay**: 100 req/min, 24h caching required, no attribution [34][35]. Covers photos + illustrations + vectors + video + sounds in one API.
- **Videvo**: 700k+ video/image assets, multi-tier license (some CC 3.0 require attribution, some RF do not) [36]. Read license per asset.

**Integration**: build a unified "stock media picker" node that queries Pexels first (no-attribution) and falls back to Pixabay/Unsplash/Videvo. Surface the license label in the UI so creators know what to expect.

### 2.6 Freesound (CC audio)

**What**: Millions of sounds under CC0, CC BY, or CC BY-NC [40][41]. API v2 supports filter-by-license. As of Feb 2026 Freesound added a per-sound **opt-out flag for Gen-AI training use** exposed via the API [40] — BRIK must respect this if we ever fine-tune a model on this data.
**Why**: feed audio-reactive BRIK tools. "Reactive visualizer → kick drum" becomes one click.
**Integration**: filter by `license=Creative Commons 0` for the default case; offer CC BY with auto-credit string in exports.

### 2.7 Icons

- **Lucide** (ISC): 1000+ svg, first-class Figma plugin, widely used; totally free commercial [43][44].
- **Tabler Icons** (MIT): 6000+ icons via jsdelivr CDN or `@tabler/icons` [43].
- **Iconify** (MIT framework): unifies 100+ icon sets with a single API (`api.iconify.design`) — aggregated 150k+ icons [43]. The per-set license varies; filter icons by license filter when generating for client projects.
- **Heroicons** (MIT, Tailwind team) — not in our search but worth pairing.
- **Noun Project**: 5M+ icons, CC BY by default or royalty-free via subscription/purchase [45][46]. Use only when Lucide/Tabler/Iconify don't cover the concept.

**Integration**: treat Iconify as the default icon-search backend (widest coverage, handles license filtering), with Lucide pinned for the "system" icons the BRIK UI itself uses.

### 2.8 Typography asset sources

- **Google Fonts CSS2 API** — all fonts open-source, commercial use allowed, weight ranges for variable fonts `family=Inter:wght@100..900` [47][49]. No attribution required in practice (specific fonts under SIL OFL or Apache).
- **Fontsource** — self-hosted OSS font NPM packages, 1500+ fonts [48]. Use when BRIK wants offline-first or wants to lock versions.
- **Adobe Fonts** — high quality but **licensing is per-subscription**, does not permit re-distribution in a platform like BRIK for others. Skip unless a B2B agreement.

### 2.9 3D model generators (tie to Category 4/9)

See Meshy / TripoSR under Category 9.

---

## Category 3 — Computer Vision & Segmentation APIs

Two philosophies for BRIK: **on-device (WebGPU in browser, no cost, private)** vs **API (heavier models, fixed cost per call)**. Prefer on-device.

### 3.1 Segment Anything 2 (SAM 2) — Meta, Apache-2.0

**What**: open-vocabulary image + video segmentation with memory module for video tracking [50].
**License**: **Apache 2.0** for code + checkpoints (except the demo's Inter Font / Noto emoji under SIL-OFL) — fully commercial-usable [50][51].
**Browser**: Yes — `lucasgelfond/webgpu-sam2` runs the full SAM2 pipeline client-side in WebGPU via onnxruntime-web [52][53][54][55]. Input 1024², output 256² mask; encoder stage benefits most from WebGPU (~8× vs WASM) [55].
**Why for BRIK**: "click on subject in dropped image → get alpha-masked layer". A first-class node called "isolate subject" in the assembler. Also enables interactive rotoscoping of user video.
**Integration**: pre-compute image embedding once (~1–2 s), then masks are generated ~50 ms per click. Ship as a BRIK asset-processing utility that any tool can import.

### 3.2 MediaPipe Tasks Vision — Google

**What**: `@mediapipe/tasks-vision` package with pose (33 landmarks), face mesh (468 landmarks), hand (21 joints × 2 hands), gesture recognizer, image segmenter, object detector; 30+ FPS in browser [56][57][58].
**License**: Apache-2.0.
**Why for BRIK**: enables AR-filter-like tools, motion capture, face overlays. VJ and live-performance use cases.
**Integration**: lightweight, already bundled for web (`@mediapipe/tasks-vision` from NPM or CDN). Expose to tools as `webcam.pose`, `webcam.hands`, etc.

### 3.3 Depth Anything V2 / Apple Depth Pro

**What**: monocular depth estimation foundation models. V2 is NeurIPS 2024, more fine-grained than V1, competitive with SD-based depth models but faster [59][60].
**Browser**: ONNX versions published (`onnx-community/depth-anything-v2-small` / `large`) with **WebGPU backend** in Transformers.js v3 [59][61][62].
**License**: Depth Anything V2 — Apache 2.0 (small) / CC-BY-NC-4.0 (large) — so use the small/base variants for BRIK.
**Why**: unlocks "parallax from photo", "depth-driven displacement shader", "auto z-compositing" tools.
**Integration**: one-line via Transformers.js: `pipeline('depth-estimation', 'onnx-community/depth-anything-v2-small', { device: 'webgpu' })`.

### 3.4 YOLO v8 / v11 in browser

**What**: state-of-the-art real-time object detection; ONNX-exportable; runs in browser via onnxruntime-web on WebGPU/WASM [63][64][65][66].
**License**: Ultralytics offers AGPL-3.0 for community / commercial subscription — watch license. COCO-trained weights from the ultralytics repo are AGPL. For BRIK's commercial deployment use **YOLO-NAS (Apache-2.0)** or **DETR/RT-DETR** as license-safe equivalents.
**Use**: "detect objects in user video" → spawn AR markers, trigger effects.

### 3.5 Transformers.js v3

**Core browser ML runtime**. v3 (Oct 2024) introduced WebGPU support (up to 100× vs WASM), 1200+ pre-converted models, 25 examples [67][68]. Covers: image classification, object detection, segmentation, depth, ASR (Whisper), TTS, image-to-text, zero-shot image classification, text generation (small LLMs up to Phi-3 Mini, Llama-3.2 1B) [67][68].

**Recommended as BRIK's default in-browser ML runtime.** MIT license. Low integration cost — single module import.

### 3.6 API alternatives

- **Remove.bg**: ~$0.09/call first 500, then volume discounts [69][70].
- **Photoroom**: $0.02/call on Basic / $20 for 1000 / $49 for 2000 on Growth [69][70][71]. Ownership of output transfers to caller; commercial use allowed.
- **Replicate / fal.ai**: many SAM / remove-bg / depth models wrapped as pay-per-call.
- **Hugging Face Inference API**: $0.06–0.12 / min GPU-second on Serverless; good for one-offs, not for per-user latency-sensitive flows.

**BRIK recommendation**: default path is client-side (SAM2, Depth V2, MediaPipe). Fall back to Photoroom or fal.ai's BiRefNet/RMBG endpoints only for ultra-high-quality output on specific nodes (e.g., "studio-quality product cutout"). This keeps per-user infra cost at $0.

### 3.7 ControlNet / IP-Adapter

**What**: conditional image-generation plumbing that lets a user drive a diffusion model with edges, pose, depth, style reference, face reference [72][73].
**Where**: fal.ai FLUX general endpoint supports ControlNets + LoRAs + IP-Adapters in one URL; Replicate has a "control-net" collection with many specific variants [72].
**Why for BRIK**: the most creatively powerful generative lever — lets a user's sketch, pose, or reference image drive the output. Pair with SAM2 for mask-conditioned generation.

---

## Category 4 — Generative models (image, video)

### 4.1 Image — open-weights

| Model | License | Best-for | Via |
|---|---|---|---|
| SDXL (Stable Diffusion XL) | CreativeML Open RAIL-M [74] | Open-weights, anything | fal.ai, Replicate, Together |
| SD 3 / 3.5 | Stability Community License — free < $1M ARR [74][75] | Higher quality than SDXL | Stability API, Replicate |
| SDXL Turbo / Lightning | Stability NC for Turbo; Lightning distilled checkpoints mostly Apache-2.0 | **Real-time** (1–4 steps, <100ms) | fal.ai, self-host |
| FLUX.1 [Schnell] | **Apache-2.0** (!), free commercial [76] | Fastest open high-quality | fal.ai $0.003/MP [77] |
| FLUX.1 [Dev] | FLUX Dev Non-commercial license | Dev/research only | — |
| FLUX.1 [Pro] | Hosted-only (not open weights) | Prod quality | fal.ai |
| FLUX.2 [Pro] (2026) | Hosted-only, commercial included | Latest quality | fal.ai $0.03/MP [77] |

**FLUX.1 [Schnell] is the highest-value open-weights image model for BRIK** — Apache-2.0, sub-second generation, commercial-ready. Use it as the default image backend for any BRIK tool.

### 4.2 Image — proprietary APIs

| Provider | Model | Pricing per image | Commercial | Special |
|---|---|---|---|---|
| OpenAI | DALL·E 3 | $0.04 (1024²) / $0.08 (HD or 1792) / $0.12 (HD 1792) [78][79] | Full ownership to user [79] | Prompt adherence, safety heavy |
| OpenAI | GPT Image 1 / 1.5 | Metered (~$0.02–$0.19/image) [78] | Yes | Edit API, scribble, mask in |
| Google | Imagen 3 | Via Vertex / Gemini API | Yes | Photoreal focus |
| Google | Nano Banana (Gemini 2.5 Flash Image) | ~$0.039/image [80][81] | Yes | Cheap + multimodal |
| Google | Nano Banana Pro (Gemini 3 Pro Image) | $0.134 (1K/2K), $0.24 (4K); batch 50% off [80][81] | Yes | State-of-the-art Nov 2025 |
| Google | Gemini 3.1 Flash Image ("Nano Banana 2") | Flash-price, Pro-quality [80] | Yes | Aug 2026 direction |
| Black Forest Labs | FLUX.1 Pro / Ultra | $0.055/image on Replicate [76][77] | Yes | Best open-lineage quality |
| Ideogram | V3 | $0.04 / $0.025 turbo [82] | Yes | **Typography-in-image champion** |
| Stability | SD 3.5 Large | $0.065/image [74] | Yes (<$1M ARR free) | Open-weights hybrid |
| Midjourney | — | No public API | — | Skip for API integration |

### 4.3 Video

| Provider | Model | Price | Commercial |
|---|---|---|---|
| Runway | Gen-4 / Gen-3 Alpha | 10–12 credits/s (~$0.10–0.12/s); Gen-4 Turbo 5 credits/s [83][84] | Full ownership on paid [83] |
| Luma | Ray2 Std / Flash | Ray2 Std 32 credits/s, Flash 11 credits/s; API $0.32/MP [85] | Commercial on paid [85] |
| Google | Veo 3 | $0.40/s; Veo 3 Fast $0.15/s; Veo 3.1 Lite cheaper still [86] | Yes |
| MiniMax | Hailuo 02 / Pro | $0.08/s, 6s video ≈$0.48; 1080p $0.28 [87][88] | Commercial on paid [87] |
| Kling | Std / Pro | $6.99/mo Std; prepaid bundles for API [87] | Yes |
| OpenAI | Sora | **Discontinued April 2026** [86] | N/A |
| Krea | Realtime 14B | Real-time (~1 s first frame) [89][90] | Via plan |
| Black Forest Labs / FAL | Custom t2v | fal.ai per-second [91] | Yes |

**Strategic pick for BRIK**:
- **Image (real-time in tool)**: fal.ai FLUX Schnell + SDXL Lightning + LCM via WebSocket (~150–250 ms latency) [89][91][92]. This is the path to "paint-with-AI" and "slider drives re-gen" tools.
- **Image (hero quality)**: FLUX.1 Pro or Nano Banana Pro via fal.ai / OpenRouter / Google.
- **Video**: Veo 3 Fast and Hailuo 02 by default; add Runway Gen-4 Turbo for cinematic.
- **Typography-in-image**: Ideogram V3 as a specialized node.

### 4.4 Real-time generation (critical for BRIK "live" tools)

fal.ai's real-time inference uses **WebSockets** to bypass the queue, keeping the runner warm for subsequent frames [91][92][93]. Latency figures from fal's own docs:
- LCM (v1.5/XL): ~150 ms/image, ~3–5 fps sustained [91][92].
- SDXL Turbo / SD Turbo: <100 ms [91].
- SDXL Lightning (2/4/8-step): ~100–250 ms depending on steps [92].

Krea Realtime 14B for video streams first frame in ~1 s [90].

**This changes the product surface for BRIK**: enables "slider → regenerate in real time", "paint-brush that is a diffusion model", "camera feed → style transfer at 30 fps" (combine with ControlNet canny). A BRIK abstraction `fal.realtime.subscribe()` delivered via WebSocket would be the single highest-impact generative integration.

---

## Category 5 — Audio & music APIs

### 5.1 Web Audio API (baseline)

As of TPAC 2025, the Audio WG is advancing: Web Audio API 1.1 [94], AudioWorklet for off-main-thread processing [95], AudioContext interrupted state in origin trial, Performance.now() proposed for AudioWorkletGlobalScope [94][95]. Wasm Audio Worklets API via Emscripten lets BRIK ship C/C++ DSP (e.g., filters, reverbs) into worklets [96]. This is sufficient baseline — no API needed for most tasks.

### 5.2 Tone.js — scheduling / synthesis

MIT. Precise timing, polyphonic synths, effects chain, transport. The correct pick when a BRIK tool is music-first (sequencer, drum machine, step-tool). Integrates cleanly with Web Audio.

### 5.3 Meyda — real-time audio feature extraction

MIT; works over Web Audio API and via AudioWorklet [97][98]. Exposes RMS, spectral centroid, MFCCs, chroma, loudness, rolloff, spectral flatness. **The go-to for "audio-reactive" BRIK tools** — hook feature streams to shader uniforms or position parameters.

### 5.4 Essentia.js — MIR (music information retrieval)

AGPL-3.0 / commercial license; WASM port of the UPF Essentia library [99][100][101]. Deeper MIR than Meyda: onset detection, beat tracking, key estimation, mood. Works offline or real-time in browser [99]. For BRIK's "bpm-locked visuals" or "song-structure-aware animation", this is the strongest option.

### 5.5 TensorFlow.js Speech Commands / Whisper-tiny in browser

Transformers.js v3 ships Whisper variants that run in WebGPU — use for in-browser voice control of BRIK tools [67][68]. No server cost.

### 5.6 Music / SFX generation APIs

- **ElevenLabs Eleven Music** (Aug 2025): trained on **licensed** data (Merlin, Kobalt); commercial rights cleared; available in API [102][103][104]. Strongest legal posture.
- **ElevenLabs Sound Effects**: text-to-SFX API, commercial cleared.
- **Suno / Udio**: higher quality music to many ears but historic copyright suits; Udio has **no public API** [104]; Suno has limited API. Use only with user-supplied key.
- **Stable Audio 2**: open-weights research; commercial via Stability subscription.

**Integration recommendation**: Eleven Music for music-gen, ElevenLabs SFX for one-shots, and keep Suno optional via BYOK.

### 5.7 ASR

- **Deepgram Nova-3**: $0.46/hr PAYG (~$0.0043/min batch, $0.0077/min stream) [105][106].
- **AssemblyAI Universal-2/3**: $0.15/hr base batch; $0.45/hr streaming Pro [105][106]. **Roughly 3× cheaper than Deepgram at most volumes** per the comparison [105].
- **Whisper in browser** (via Transformers.js v3 WebGPU): free, ~20 fps on M-series; lower accuracy than Nova-3 but zero cost and private.

**Recommendation**: offer Whisper-WebGPU as default, with AssemblyAI as opt-in "premium accuracy".

---

## Category 6 — Physics & simulation

| Lib | Language | Dim | License | Notes |
|---|---|---|---|---|
| Matter.js | JS | 2D | MIT | Already in BRIK |
| **Rapier** | Rust→WASM | 2D+3D | Apache-2.0 | **5–8× faster than nphysics**, ~Box2D speed, browser-ready [107][108] |
| Cannon-es (fork of Cannon.js) | JS | 3D | MIT | Small, simple, predictable [109][110] |
| Ammo.js | C++→ASM.js/WASM (Bullet) | 3D | Zlib | Feature-rich but heavy, awkward API [109] |
| verlet-js | JS | 2D (Verlet) | MIT | Cloth, ragdoll, constraints [111][112] |
| Orillusion GPU physics | WebGPU | 2D+3D | MIT | GPU compute-based rigid, soft, fluid, cloth [24][25] |

### 6.1 Rapier — the strong addition

Rapier is the pick to add alongside Matter.js. 3D capabilities, deterministic, WASM binding `@dimforge/rapier2d-compat` or `rapier3d-compat`, same-order performance as native PhysX CPU [107][108]. The public `@dimforge/rapier` TS types make it assembler-friendly. One developer reports "thousands more active bodies" than cannon/ammo in real apps [107].

### 6.2 verlet-js — for cloth, hair, ropes

For fabric/fashion/industrial BRIK tools: verlet integration with distance + angular constraints, prebuilt `cloth()` and `tire()` helpers [111][112]. Complement, not replacement, for Rapier.

### 6.3 WebGPU-native sims (compute shader)

- **MLS-MPM fluids** — Material Point Method on compute shader at 100k particles [14][16]. Open-source demos.
- **Gray-Scott reaction-diffusion in compute** [17][18]. Endless aesthetic output; small shader code; could ship as a BRIK template.
- **SPH fluids, boids flocking, DLA, molecular dynamics** — comprehensive showcase repo `scttfrdmn/webgpu-compute-exploration` [14].
- **piellardj/water-webgpu** — 1M-particle water sim [14].
- **kishimisu/WebGPU-Fluid-Simulation** — Stam Real-Time Fluid Dynamics on WebGPU.
- **matsuoka-601/WebGPU-Ocean** — real-time 3D ocean [14].

**These become trivially embeddable BRIK templates** once the WebGPU renderer is exposed in the assembler.

---

## Category 7 — Typography (advanced)

### 7.1 HarfBuzzJS — font shaping in browser

**What**: WebAssembly build of HarfBuzz (world's canonical text-shaping engine used by Chrome, Firefox, Android, LibreOffice) [113][114][115].
**Why**: unlocks **proper Unicode shaping** — Arabic, Devanagari, Hangul contextual forms, ligatures, GSUB/GPOS features. Recent versions expose `face.getTableFeatureTags()`, `face.listNames()`, OpenType feature queries [115].
**For BRIK**: essential for serious typographic tools — especially for the Hebrew market (שאלתיות עברית) since the user is at Wix and will inevitably want right-to-left + niqqud + ligature support in type tools.
**Integration**: load WASM via CDN; shape a UTF-8 string + font → glyph IDs + positions; feed into canvas/WebGL/WebGPU glyph atlas.

### 7.2 opentype.js & fontkit — parsing / glyph manipulation

- **opentype.js**: read/write OpenType + TrueType; glyph `Path` objects with curve control [116][117].
- **fontkit**: advanced engine, supports variable fonts, AAT variations, subsetting, color emoji, text layout [116][118].

For BRIK:
- **opentype.js** is the "give me the outline of glyph 'A' so I can animate its points" library. Use for glyph-level drawing, type-as-path-particles, morphing animations.
- **fontkit** when you need variable-font axis interpolation done correctly (variationProcessor), subsetting for exports, or per-feature shaping without HarfBuzz.

### 7.3 Variable fonts via CSS

`font-variation-settings` is animatable, all modern browsers support it, and registered axes are `wght`, `wdth`, `slnt`, `ital`, `opsz` plus custom 4-letter uppercase axes (`'GRAD'`, `'CRSV'`, `'YOPQ'`...) [47][49][119][120]. BRIK should expose variable-font axes as sliders automatically — Fontsource registers all axes in its packages, making automatic UI feasible.

### 7.4 What BRIK should ship

1. **FontPicker node** — Google Fonts CSS2 API + Fontsource fallback; surface all variable axes as sliders.
2. **GlyphPath node** — opentype.js extracts SVG paths; route into any canvas/WebGL/WebGPU tool.
3. **Shaper node** — HarfBuzzJS for complex-script text; essential for Hebrew/Arabic/Indic users.
4. **Variable-axis animator** — GSAP-driven interpolation of `font-variation-settings`.
5. **Text-on-path / text-to-particles** — emerges naturally once GlyphPath is in.

---

## Category 8 — Color & perceptual

### 8.1 Culori — modern CSS color

**What**: supports all CSS Colors Level 4 spaces; used by Tailwind v4 and Radix UI [121][122]. OKLCH, OKLAB, LCH, LAB, P3, Rec2020.
**Why for BRIK**: the correct choice for modern design-system work. Perceptually uniform color interpolation gives much better gradient results than sRGB/HSL.

### 8.2 Chroma.js — still excellent for data viz & gradients

**What**: classic color manipulation library; 2025 added OKLAB/OKLCH support [121].
**Why**: built-in palette generators (`chroma.scale(['red','blue']).mode('oklab')`), categorical color scales, brewer palettes. Great for the "generate 5-color palette from hue" node.

### 8.3 CSS Color 4 + wide gamut

OKLCH is now universally supported in evergreen browsers [123][124]. Display-P3 available in CSS via `oklch(L C H)` or `color(display-p3 r g b)` [123][125]. Apple devices and modern high-end displays render ~25% more saturated colors [123].

**Recommendation**: BRIK's default internal color representation should be **OKLCH**, converted to sRGB for export when needed, or to P3 when the export target supports it (e.g., `.clay.html` embedded on P3 display).

### 8.4 WCAG 2 + APCA contrast

- `apca-w3` (npm, APACHE-2.0) [126] — APCA (perceptual, for WCAG 3) with reverse lookup: give it a background + desired contrast, it returns the foreground.
- WCAG 2 ratio checkers are trivial to implement; Culori has helpers.
- Atmos / Polypane-style checkers exist as reference [126][127].

**For BRIK**: expose a "check contrast" badge on any color swatch in any tool, showing both WCAG 2 ratio and APCA Lc. Designers expect this now.

---

## Category 9 — Academic research with implementation potential

### 9.1 Gaussian Splatting in the browser

**Paper**: Kerbl et al., "3D Gaussian Splatting for Real-Time Radiance Field Rendering", SIGGRAPH 2023 (arXiv:2308.04079). **New web-native 2025–2026 implementations**:

- **Visionary** (arXiv:2512.08478, Dec 2025) [128][129]: a WebGPU Gaussian Splatting engine that renders 3D/4D Gaussian splats + neural avatars + traditional meshes at 2–16 ms/frame on RTX-4090-class hardware. **Ships with a Three.js-compatible TypeScript API** — this is enormous for BRIK because the existing Three.js assembler mode can mount it directly.
- **WebSplatter** (arXiv:2602.03207, Feb 2026) [130][131]: wait-free hierarchical radix sort to bypass WebGPU's lack of global atomics; 1.2–4.5× faster than state-of-the-art web viewers.
- **antimatter15/splat**, **SuperSplat**, **gsplat-js**, **Luma AI three-gaussian-splatting** — older WebGL viewers.

**For BRIK**: add a `gaussian-splat` renderer mode. Let users upload `.splat` / `.ply` / `.ksplat` and build tools on top — guided camera paths, slice views, splat-field deformation. Pair with Luma's capture app on phones for UGC pipeline.

### 9.2 Text-to-3D

| Method | Year | arXiv | Compute | Viable in BRIK? |
|---|---|---|---|---|
| DreamFusion | 2022 | 2209.14988 | Multi-hour SDS | Via Replicate, not real-time |
| Magic3D | 2023 | 2211.10440 | ~40 min | Via Replicate |
| TripoSR | 2024 | 2403.02151 | **Sub-second on A100** | **Yes, via API** |
| Meshy | 2024–26 | — | Few minutes, commercial API | Yes, BYOK |
| Instant Mesh | 2024 | — | Seconds | Via Replicate |

**Recommendation for BRIK**: use **TripoSR** for image-to-3D-mesh generation via Replicate or self-host on Modal; **Meshy** via BYOK for text-to-3D at higher fidelity. Output GLB → drop into Three.js.

### 9.3 Procedural generation — Wave Function Collapse / Model Synthesis

Gumin's original WFC (2016) builds on Merrell's Model Synthesis (2007) [132][133][134]. Pure-JS ports abound (Chapelier, Coding Train) [132][133]. **In BRIK**: ship a WFC "tile-field generator" template — given a palette of input tiles with adjacency rules, generate infinite unique maps. Great for fashion print patterns, game levels, procedural architecture facades.

### 9.4 SDF ray marching + SDF modeling in browser

**reindernijhoff's WebGPU SDF Editor (Jan 2026)** [135] supports hierarchical SDF scene graphs with blend operations and blend radius. Codrops "Liquid Raymarching with TSL" is an active tutorial stream [135]. Iquilezles.org remains the bible for SDF primitives.

**For BRIK**: an "SDF-sculpt" renderer mode where tools manipulate a parametric SDF tree (union, smooth-union, subtract, twist, bend) and the renderer ray-marches it. Enables tools that literally cannot exist in polygon-based renderers.

### 9.5 Text-to-shader / NL-to-GLSL

- **AI Co-Artist** (arXiv:2512.08951, Nov 2025) — LLM-powered GLSL shader evolution framework [136]. Uses Code Llama / GPT-4 / Claude for interactive shader synthesis + fitness evaluation.
- **ShaderMatch benchmark** (ICSE 2025 / LLM4Code) — shader code completion benchmark over Shadertoy [137]. Reports even top LLMs fail on 31% of GLSL completions — this is BRIK's opportunity: a BRIK-specific fine-tune (or well-primed prompt library) could outperform raw GPT-4 because BRIK owns domain-specific context.
- **14islands "Exploring AI-powered art with simple text prompts"** — production case study [138].

**For BRIK**: the assembler's "Prompt → BRIK tool" path should include a tight loop for **TSL / GLSL generation with automatic compile-feedback** (compile the shader, if it errors, feed the error back to the LLM for correction — classic ReAct loop). Current LLMs benefit enormously from this tool-use pattern on low-resource languages like GLSL.

### 9.6 Differentiable rendering — Mitsuba 3

Mitsuba 3 (Realistic Graphics Lab, EPFL) is Python-first differentiable rendering via Dr.Jit JIT [139][140]. Not browser-native. Useful to BRIK as a **server-side node** for inverse-rendering tasks (fit a material to an image, optimize lighting to match a reference). Ship via Modal or Replicate with a Python handler.

### 9.7 Real-time style transfer

Tensor Flow.js ports of arbitrary style transfer exist from 2018 onwards [141][142]. With WebGPU backend the same models now run 3× faster [143]. Ship as a BRIK "webcam → stylized" tool.

### 9.8 Animation from sketch / vectorization

Adobe "Project VectorFusion", Google "DiffVG", "Text-to-SVG" research papers — worth tracking but none is a clean NPM drop-in yet. For now, simplest production path is to send image → LLM (GPT-4 vision) → generated SVG.

### 9.9 Flow fields & curve generation (art/design)

Perlin-noise-driven flow fields are trivial already in BRIK. Research to borrow from:
- Tyler Hobbs "Flow fields" essay (not academic but seminal).
- Curl noise ([Bridson et al. 2007]) — divergence-free flow; compute-shader friendly.
- "Differentiable vector graphics rasterization for editing and learning" (DiffVG, SIGGRAPH Asia 2020) — gradient descent on SVG parameters.

### 9.10 NeRF / implicit neural representations

Replaced in 2024–2026 by Gaussian Splatting for creative use. Keep NeRF as research memory; ship Gaussian Splatting first (see 9.1).

---

## Category 10 — Integration infrastructure

### 10.1 Unified inference providers

| Provider | Strength | Pricing style | Coverage | Real-time? |
|---|---|---|---|---|
| **fal.ai** | Latency champion; WebSocket real-time API [91][92][93] | Per-image / per-MP / per-second | 1000+ media models | **Yes, sub-second** |
| **Replicate** | Largest community model catalog (50k+ Cog models) | Per-second GPU | Broadest coverage | Async-first, webhook delivery [144] |
| **Together AI** | Best LLM token prices; OSS-first | Per-token / per-GPU-hour | Chat + image + vision + video [145] | No real-time video |
| **Modal** | "Infra as code" Python; great for custom ML | Per-second compute | Any model you deploy | Cold start ~1 s |
| **OpenRouter** | Unified LLM API across 300+ models | Passthrough + 5.5% fee [146] | Text LLMs only | N/A |
| **Hugging Face Inference Endpoints** | Official HF model hosting | Per-GPU-hour | Any HF model | No |

### 10.2 BRIK architectural recommendation

A **3-tier generative backend**:

1. **Tier 1 — In-browser (WebGPU + Transformers.js + MediaPipe + ONNX Runtime Web)**: default for anything that fits. Zero per-user cost, private, instant. Covers: segmentation, depth, object detection, pose, whisper-tiny, small LLMs, depth, background removal.

2. **Tier 2 — fal.ai real-time**: for anything that needs a larger model with <500 ms latency. LCM, FLUX Schnell, SDXL Lightning, background removal, speech-to-speech. Connect via fal's WebSocket client. This is where BRIK's "live" tools live.

3. **Tier 3 — Batch generation via Replicate or fal queue**: hero-quality image (FLUX Pro, Nano Banana Pro, Ideogram V3), all video (Veo 3, Runway Gen-4, Hailuo, Luma Ray2). Async with webhooks; surface progress in the UI.

LLM traffic (for tool prompt parsing, in-tool "ask BRIK" helpers, etc.) goes through **OpenRouter** for flexibility, or directly to Anthropic/OpenAI for the assembler's own pipeline.

### 10.3 Practical cost modeling (per active user per day)

Assume a power user runs 30 generations/day:
- 25× FLUX Schnell @ $0.003/MP × 1MP = $0.075
- 3× Nano Banana Pro 2K @ $0.134 = $0.402 (or $0.067 batch)
- 2× Veo 3 Fast 5s @ $0.15/s × 5 = $1.50

Total ≈ **$2.00/user/day for the heavy path**; **$0.08/user/day if real-time FLUX Schnell is default** and premium is opt-in.

---

## RANKED TOP 20 integrations for BRIK

*Criteria: creative leverage × integration cost × licensing safety × time-to-ship.*

1. **Three.js WebGPURenderer + TSL** — new assembler renderer mode; unlocks compute shaders, 1M-particle systems, reaction-diffusion, MLS-MPM fluids, faster everything. **Foundation for the next 3 years.** [9][10][11][20][21]
2. **Transformers.js v3 (WebGPU)** — in-browser ML runtime. One install, 1200+ models covering depth, segmentation, object detection, zero-shot CLIP, Whisper, small LLMs. Private, free, cache-friendly. [67][68]
3. **fal.ai real-time + queue API** — the single highest-impact generative integration. WebSocket for <250 ms image gen (FLUX Schnell, SDXL Lightning, LCM); queue for everything else. Commercial cleared. [76][77][91][92][93]
4. **Poly Haven + AmbientCG asset APIs** — CC0 PBR textures + HDRIs + models, no attribution, huge library. First-class "material/HDRI/model picker" nodes. [27][28][29][30][31]
5. **SAM 2 in browser (WebGPU via ONNX)** — Apache-2.0, runs client-side, one-click subject isolation. Gateway to "drop image → keep subject" flow in every BRIK tool. [50][51][52][53][54][55]
6. **MediaPipe Tasks Vision** — pose + face + hands + segmenter + gesture at 30 fps. VJ / AR-filter / motion-capture tools. Apache-2.0. [56][57][58]
7. **HarfBuzzJS + opentype.js + Fontsource + Google Fonts CSS2** — complete modern typography stack, including RTL + complex scripts. Critical for Hebrew market. [47][48][113][114][115][116][117]
8. **Rapier physics (Rust→WASM)** — 3D + 2D, Apache-2.0, 5–8× faster than cannon/nphysics. Enables physical-simulation tools comparable to native. [107][108]
9. **Pexels + Pixabay + Freesound APIs** — no-attribution stock photo/video/audio pickers. Ship "drop-in asset" UX into all canvas tools. [34][35][39][40][41]
10. **Ideogram V3 + OpenRouter / fal.ai bridge** — typography-in-image, $0.025–0.04/image. Poster, logo, brand-design tools. [82]
11. **Culori + OKLCH default + apca-w3** — modern perceptual color; aligns BRIK with Figma/Tailwind v4. WCAG + APCA contrast checks built in. [121][122][123][126]
12. **Depth Anything V2 (WebGPU via Transformers.js)** — monocular depth in 200 ms in-browser. Enables parallax-from-photo, displacement, composition tools. [59][60][61]
13. **Gaussian Splatting via Visionary (WebGPU)** — 2025–26's breakthrough 3D format. Rendered at 2–16 ms/frame in browser; plays with Three.js. Unlocks capture-to-canvas pipelines. [128][129][130][131]
14. **Iconify + Lucide + Tabler API** — 150k+ unified icons, MIT/ISC. "Icon" becomes a first-class token in every BRIK tool. [43][44]
15. **Veo 3 Fast + Runway Gen-4 Turbo** — dual-provider video generation with commercial licensing. Surface in BRIK's "make it a video" export. [83][86]
16. **Meyda + Essentia.js + Tone.js** — the audio-reactive stack. Meyda for real-time features, Essentia for beat/key/mood, Tone.js for scheduling. [97][99][100][101]
17. **ElevenLabs Eleven Music + SFX API** — licensed-data music-gen with commercial rights cleared. Safest provider for production BRIK tools that output audio. [102][103][104]
18. **ControlNet + IP-Adapter via fal.ai FLUX-general** — conditional gen from pose/edge/depth/reference. Pair with SAM2 + MediaPipe for full sketch-to-image or pose-to-image flows. [72]
19. **WebGPU compute-shader template library** — ship BRIK templates for MLS-MPM fluid, Gray-Scott reaction-diffusion, 1M-particle flow field, SDF ray-march sculpt, SPH, boids, DLA. Each is a marketable "signature BRIK tool". [14][16][17][18][135]
20. **TripoSR / Meshy text-to-3D** via Replicate — drops 3D into BRIK's world. Critical for industrial, architectural, fashion product tools. [147][148]

### Additional strong candidates (21–30)

- **Kenney + Quaternius stylized CC0 packs** — ship as BRIK-hosted library.
- **Sketchfab Download API** with compliant CC-BY attribution auto-display.
- **Modal** — for any custom BRIK server-side model (e.g., a BRIK-fine-tuned shader model).
- **Verlet-js** — cloth/hair/ropes for fashion-specific tools.
- **AssemblyAI streaming STT** — voice-controlled tools.
- **Orillusion** — alternative renderer mode for GPU-physics-forward tools.
- **Mitsuba 3 server-side** — differentiable rendering for inverse-material / inverse-lighting tools.
- **Wave Function Collapse pure-JS** — procedural pattern / tile-field generator template.
- **APCA + Polypane contrast API** — ship accessibility overlay on the canvas.
- **Sound.txt / ElevenLabs Sound Effects** — one-shot SFX from text prompts.

---

## How this maps to BRIK's assembler

The BRIK assembler today accepts a prompt, picks a canvas mode, and assembles a tool. The integrations above suggest **three new abstractions**:

1. **Renderer modes** (existing: `canvas2d`, `webgl`, `p5`, `hydra`, hybrids) + new: `webgpu`, `splat`, `sdf`.
2. **Asset providers** (new concept) — a unified interface for pulling assets from Poly Haven, AmbientCG, Sketchfab, Pexels, Freesound, Iconify, Google Fonts. Each exposes a `search(prompt) → [Asset]` method; the LLM resolves natural-language queries to concrete assets.
3. **Inference providers** (new concept) — a tier-aware router. Default tier 1 (in-browser), escalate to fal.ai real-time, escalate to fal/Replicate queue for hero quality. Each BRIK tool declares its quality/latency preference; the router picks the backend.

Making these three abstractions first-class in the assembler SDK lets every new integration slot in without a rewrite.

---

## Known risks and watch-items

- **Firefox WebGPU on Linux/Android** is still behind flag — budget fallback to WebGL for that slice until Mozilla ships, expected 2026 [1].
- **SAM 2 model weights are Apache-2.0** but some downstream fine-tunes are not — always use the official Meta checkpoints. [50][51]
- **Sketchfab CC BY attribution is mandatory** in-UI and in `.clay.html` exports — bake it into the export pipeline automatically.
- **Stable Diffusion 3/3.5** commercial threshold is $1M ARR — BRIK itself is above that, so either (a) pay Stability enterprise license for SD 3.x, or (b) stick to SDXL (CreativeML Open RAIL-M) and FLUX.1 Schnell (Apache-2.0) [74][75].
- **FLUX.1 Dev is non-commercial** — do not host on BRIK servers; use via fal.ai/Replicate which have their own commercial hosting deals, or use FLUX.1 Schnell (Apache) / FLUX.1 Pro (hosted commercial).
- **Ultralytics YOLO v8/v11 weights are AGPL-3.0** for free tier — replace with YOLO-NAS (Apache-2.0), RT-DETR, or DETR for license-clean deployments.
- **Freesound Gen-AI opt-out flag (Feb 2026)** — if BRIK ever fine-tunes on this data, respect the per-sound flag [40].
- **Suno / Udio ongoing label litigation** — do not ship as default music provider; BYOK only.
- **Sora shut down in April 2026** [86] — remove any Sora integration plans.
- **Imagen and Nano Banana Pro via Gemini API** — Google TOS require attribution that output may be filtered; understand the content-policy SLA if embedded in BRIK tools directly.

---

## References

1. "WebGPU Hits Critical Mass: All Major Browsers Now Ship It" — webgpu.com/news. https://www.webgpu.com/news/webgpu-hits-critical-mass-all-major-browsers-now-ship-it/
2. "WebGPU is now supported in major browsers" — web.dev. https://web.dev/blog/webgpu-supported-major-browsers
3. GPUWeb Implementation Status — GitHub. https://github.com/gpuweb/gpuweb/wiki/Implementation-Status
4. Overview of WebGPU — Chrome Developers. https://developer.chrome.com/docs/web-platform/webgpu/overview
5. From WebGL to WebGPU — Chrome for Developers blog. https://developer.chrome.com/blog/from-webgl-to-webgpu
6. WebGPU Storage Buffers — WebGPU Fundamentals. https://webgpufundamentals.org/webgpu/lessons/webgpu-storage-buffers.html
7. WebGPU Compute Shader Basics — WebGPU Fundamentals. https://webgpufundamentals.org/webgpu/lessons/webgpu-compute-shaders.html
8. "WebGPU vs WebGL: Performance Benchmarks for Client-Side Inference" — Sitepoint. https://www.sitepoint.com/webgpu-vs-webgl-inference-benchmarks/
9. Three.js TSL wiki — GitHub. https://github.com/mrdoob/three.js/wiki/Three.js-Shading-Language
10. "TSL: A Better Way to Write Shaders in Three.js" — Three.js Roadmap. https://threejsroadmap.com/blog/tsl-a-better-way-to-write-shaders-in-threejs
11. TSL docs — three.js. https://threejs.org/docs/pages/TSL.html
12. caniuse.com — WebGPU. https://caniuse.com/webgpu
13. WebGPU from WebGL — WebGPU Fundamentals. https://webgpufundamentals.org/webgpu/lessons/webgpu-from-webgl.html
14. "WebGPU Fluid Simulations: High Performance & Real-Time Rendering" — Codrops. https://tympanus.net/codrops/2025/02/26/webgpu-fluid-simulations-high-performance-real-time-rendering/
15. "WebGL vs. WebGPU Explained" — Three.js Roadmap. https://threejsroadmap.com/blog/webgl-vs-webgpu-explained
16. "Particles, Progress, and Perseverance: A Journey into WebGPU Fluids" — Codrops. https://tympanus.net/codrops/2025/01/29/particles-progress-and-perseverance-a-journey-into-webgpu-fluids/
17. "Reaction-Diffusion Compute Shader in WebGPU" — Codrops. https://tympanus.net/codrops/2024/05/01/reaction-diffusion-compute-shader-in-webgpu/
18. "WebGPU Unleashed — Reaction Diffusion". https://shi-yan.github.io/webgpuunleashed/Compute/reaction_diffusion.html
19. "Interactive Galaxy with WebGPU Compute Shaders" — Three.js Roadmap. https://threejsroadmap.com/blog/galaxy-simulation-webgpu-compute-shaders
20. TSL Getting Started — sbcode.net. https://sbcode.net/tsl/getting-started/
21. "Migrate Three.js to WebGPU (2026) — The Complete Checklist" — utsubo.com. https://www.utsubo.com/blog/webgpu-threejs-migration-guide
22. "Announcing Babylon.js 9.0" — Windows Developer Blog. https://blogs.windows.com/windowsdeveloper/2026/03/26/announcing-babylon-js-9-0/
23. Babylon.js WebGPU Support docs. https://doc.babylonjs.com/setup/support/webGPU
24. Orillusion — Next Generation WebGPU Engine. https://www.orillusion.com/en/
25. Orillusion GitHub. https://github.com/Orillusion/orillusion
26. "The Best of WebGPU in May 2025" — WebGPU Experts. https://www.webgpuexperts.com/best-webgpu-updates-may-2025/
27. Poly Haven License. https://polyhaven.com/license
28. Poly Haven FAQ. https://docs.polyhaven.com/en/faq
29. Poly Haven Public API. https://polyhaven.com/our-api
30. AmbientCG. https://ambientcg.com/
31. AmbientCG API `/full_json` docs. https://docs.ambientcg.com/api/v2/full_json/
32. Kenney + Quaternius — awesome-cc0 list. https://github.com/madjin/awesome-cc0
33. ShareTextures. https://www.sharetextures.com/
34. Pixabay API Documentation. https://pixabay.com/api/docs/
35. Pixabay Content License. https://pixabay.com/service/license-summary/
36. Videvo — via "Top Stock Video APIs" guide. https://www.plainlyvideos.com/blog/stock-video-api
37. Unsplash API Guidelines. https://help.unsplash.com/en/articles/2511245-unsplash-api-guidelines
38. Unsplash License. https://unsplash.com/license
39. Pexels API Documentation. https://www.pexels.com/api/documentation/
40. Freesound API FAQ. https://freesound.org/help/faq/
41. Freesound CC licenses discussion. https://freesound.org/forum/legal-help-and-attribution-questions/38715/
42. Sketchfab Download API Guidelines. https://sketchfab.com/developers/download-api/guidelines
43. Iconify Design. https://iconify.design/
44. Lucide. https://lucide.dev/
45. Noun Project API. https://thenounproject.com/api/
46. Noun Project license docs. https://help.thenounproject.com/hc/en-us/articles/200509798-What-licenses-do-you-offer-for-icons
47. Google Fonts CSS2 API. https://developers.google.com/fonts/docs/css2
48. Fontsource. https://fontsource.org/
49. MDN `font-variation-settings`. https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/font-variation-settings
50. SAM 2 GitHub (facebookresearch/sam2). https://github.com/facebookresearch/sam2
51. SAM 2 LICENSE (Apache-2.0). https://github.com/facebookresearch/sam2/blob/main/LICENSE
52. "Segment Anything 2, in WebGPU" — Lucas Gelfond. https://lucasgelfond.online/software/webgpu-sam2/
53. webgpu-sam2 GitHub. https://github.com/lucasgelfond/webgpu-sam2
54. "Image Segmentation in the Browser with SAM 2" — Geronimo on Medium. https://medium.com/@geronimo7/in-browser-image-segmentation-with-segment-anything-model-2-c72680170d92
55. Transformers.js SAM — Xenova on Hugging Face. https://huggingface.co/posts/Xenova/240458016943176
56. MediaPipe Tasks Vision npm. https://www.npmjs.com/package/@mediapipe/tasks-vision
57. MediaPipe Pose Landmarker Web Guide. https://ai.google.dev/edge/mediapipe/solutions/vision/pose_landmarker/web_js
58. MediaPipe Gesture Recognizer Web. https://ai.google.dev/edge/mediapipe/solutions/vision/gesture_recognizer/web_js
59. Depth-Anything-V2 GitHub (NeurIPS 2024). https://github.com/DepthAnything/Depth-Anything-V2
60. Depth-Anything-ONNX. https://github.com/fabio-sim/Depth-Anything-ONNX
61. onnx-community/depth-anything-v2-small on Hugging Face. https://huggingface.co/onnx-community/depth-anything-v2-small
62. DepthAnything on Browser — akbartus. https://github.com/akbartus/DepthAnything-on-Browser
63. YOLO Object Detection ONNX Runtime Web (WebGPU). https://github.com/nomi30701/yolo-object-detection-onnxruntime-web
64. "Run YOLO Model in the Browser with ONNX, WebAssembly, and Next.js" — PyImageSearch. https://pyimagesearch.com/2025/07/28/run-yolo-model-in-the-browser-with-onnx-webassembly-and-next-js/
65. YOLOv8 Browser Object Detection — akbartus. https://github.com/akbartus/Yolov8-Object-Detection-on-Browser
66. ONNX Runtime Web docs. https://onnxruntime.ai/docs/tutorials/web/
67. "Transformers.js v3: WebGPU Support, New Models & Tasks" — Hugging Face Blog. https://huggingface.co/blog/transformersjs-v3
68. Running models on WebGPU — Transformers.js docs. https://huggingface.co/docs/transformers.js/en/guides/webgpu
69. "Photoroom or Remove.bg" — Photoroom. https://www.photoroom.com/blog/photoroom-or-removebg
70. Photoroom API Pricing. https://www.photoroom.com/api/pricing
71. Photoroom API Basic Plan pricing docs. https://docs.photoroom.com/remove-background-api-basic-plan/pricing
72. FLUX.1 [dev] with ControlNets and LoRAs (image-to-image) — fal.ai. https://fal.ai/models/fal-ai/flux-general/image-to-image/api
73. IP-Adapter docs — diffusers. https://huggingface.co/docs/diffusers/main/en/using-diffusers/ip_adapter
74. Stability AI License. https://stability.ai/license
75. Stability AI Community License update news. https://stability.ai/news-updates/license-update
76. FLUX.1 [schnell] on Hugging Face (Apache-2.0). https://huggingface.co/black-forest-labs/FLUX.1-schnell
77. FLUX family on fal.ai. https://fal.ai/flux
78. OpenAI DALL·E & GPT Image Pricing Calculator — Costgoat. https://costgoat.com/pricing/openai-images
79. OpenAI API Pricing. https://openai.com/api/pricing/
80. "Nano Banana 2 (Gemini 3.1 Flash Image Preview) — API Pricing" — OpenRouter. https://openrouter.ai/google/gemini-3.1-flash-image-preview
81. "Nano Banana Pro API Price: Complete 2025 Pricing Guide" — AI Free API. https://www.aifreeapi.com/en/posts/nano-banana-pro-api-price
82. Ideogram API Pricing. https://ideogram.ai/features/api-pricing
83. Runway API Pricing & Costs docs. https://docs.dev.runwayml.com/guides/pricing/
84. Runway AI Plans. https://runwayml.com/pricing
85. Luma Dream Machine API pricing FAQ. https://lumaai-help.freshdesk.com/support/solutions/articles/151000210176-what-are-your-prices-for-api-
86. "Google's Veo 3.1 Lite Cuts API Costs in Half as OpenAI's Sora Exits the Market" — Decrypt. https://decrypt.co/363077/google-veo-3-1-lite-cuts-api-costs-half-openai-sora
87. "Hailuo 02: 1080p AI Video at $0.28" — UC Strategies. https://ucstrategies.com/news/hailuo-02-1080p-ai-video-at-0-28-specs-benchmarks-pricing-2026/
88. Hailuo-02 [pro] model on fal.ai. https://fal.ai/models/fal-ai/minimax/hailuo-02/pro/text-to-video
89. "Krea Realtime 14B: Real-Time, Long-Form AI Video Generation" — Krea blog. https://www.krea.ai/blog/krea-realtime-14b
90. Krea API features. https://www.krea.ai/features/api
91. "Real-Time Inference" — fal docs. https://fal.ai/docs/documentation/model-apis/inference/real-time
92. "Building Applications with Real-Time Stable Diffusion APIs" — fal blog. https://blog.fal.ai/building-applications-with-real-time-stable-diffusion-apis/
93. fal.ai Real-Time Models. https://docs.fal.ai/model-apis/real-time
94. W3C TPAC 2025: Audio WG update. https://www.w3.org/2025/11/TPAC/demo-audio-wg-update.html
95. MDN — AudioWorklet. https://developer.mozilla.org/en-US/docs/Web/API/AudioWorklet
96. Wasm Audio Worklets API — Emscripten docs. https://emscripten.org/docs/api_reference/wasm_audio_worklets.html
97. Meyda. https://meyda.js.org/
98. Meyda GitHub. https://github.com/meyda/meyda
99. Essentia.js. https://mtg.github.io/essentia.js/
100. Essentia.js ISMIR paper. https://program.ismir2020.net/static/final_papers/260.pdf
101. "Audio and Music Analysis on the Web using Essentia.js" — TISMIR. https://transactions.ismir.net/articles/10.5334/tismir.111
102. "ElevenLabs launches an AI music generator, which it claims is cleared for commercial use" — TechCrunch. https://techcrunch.com/2025/08/05/elevenlabs-launches-an-ai-music-generator-which-it-claims-is-cleared-for-commercial-use/
103. "Eleven Music, now available in the API" — ElevenLabs blog. https://elevenlabs.io/blog/eleven-music-now-available-in-the-api
104. "Suno vs Udio vs ElevenLabs Music: The 2026 AI Music Generator Showdown" — AI Magicx Blog. https://www.aimagicx.com/blog/suno-vs-udio-vs-elevenlabs-music-comparison-2026
105. "AssemblyAI vs Deepgram: API Pricing for High Volume" — Brass Transcripts. https://brasstranscripts.com/blog/assemblyai-vs-deepgram-pricing-high-volume-comparison
106. Deepgram Pricing. https://deepgram.com/pricing
107. "Announcing the Rapier physics engine" — Dimforge. https://dimforge.com/blog/2020/08/25/announcing-the-rapier-physics-engine/
108. Rapier GitHub. https://github.com/dimforge/rapier
109. "Preferred physics engine (cannon.js, ammo.js, DIY...)" — Three.js forum. https://discourse.threejs.org/t/preferred-physics-engine-cannon-js-ammo-js-diy/1565
110. Cannon-es npm / GitHub (fork of Cannon.js). https://github.com/pmndrs/cannon-es
111. verlet-js GitHub. https://github.com/subprotocol/verlet-js
112. "An Introduction to Verlet.js" — SitePoint. https://www.sitepoint.com/an-introduction-to-verlet-js/
113. HarfBuzzJS GitHub. https://github.com/harfbuzz/harfbuzzjs
114. HarfBuzz Manual — OpenType features. https://harfbuzz.github.io/shaping-opentype-features.html
115. harfbuzzjs on npm. https://www.npmjs.com/package/harfbuzzjs
116. opentype.js. https://opentype.js.org/
117. opentype.js GitHub. https://github.com/opentypejs/opentype.js/
118. fontkit on npm. https://www.npmjs.com/package/fontkit
119. "Animating variable fonts with CSS" — Val Head. https://valhead.com/2020/11/15/animating-variable-fonts-with-css/
120. MDN — Variable fonts guide. https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Fonts/Variable_fonts
121. "culori vs chroma-js vs tinycolor2: Color Manipulation in JavaScript" — PkgPulse. https://www.pkgpulse.com/blog/culori-vs-chroma-js-vs-tinycolor2-color-manipulation-javascript-2026
122. Culori homepage. https://culorijs.org/
123. "OKLCH in CSS: why we moved from RGB and HSL" — Evil Martians. https://evilmartians.com/chronicles/oklch-in-css-why-quit-rgb-hsl
124. Migrate to HD CSS color — Chrome. https://developer.chrome.com/docs/css-ui/migrate-hd-color
125. "Add wide gamut P3 and alpha transparency to your color picker in HTML" — WebKit. https://webkit.org/blog/16900/p3-and-alpha-color-pickers/
126. apca-w3 on npm. https://www.npmjs.com/package/apca-w3
127. APCA Contrast Calculator. https://apcacontrast.com/
128. "Visionary: The World Model Carrier Built on WebGPU-Powered Gaussian Splatting Platform" — arXiv:2512.08478. https://arxiv.org/abs/2512.08478
129. Visionary HTML. https://arxiv.org/html/2512.08478v1
130. "WebSplatter: Enabling Cross-Device Efficient Gaussian Splatting in Web Browsers via WebGPU" — arXiv:2602.03207. https://arxiv.org/abs/2602.03207
131. WebSplatter HTML. https://arxiv.org/html/2602.03207v1
132. WaveFunctionCollapse — mxgmn. https://github.com/mxgmn/WaveFunctionCollapse
133. "Wave Function Collapse" — Coding Train. https://thecodingtrain.com/challenges/171-wave-function-collapse/
134. Model Synthesis (Wikipedia). https://en.wikipedia.org/wiki/Model_synthesis
135. "WebGPU SDF Editor: Real-Time Signed Distance Field Modeling" — reindernijhoff.net (2026). https://reindernijhoff.net/2026/01/webgpu-sdf-editor-real-time-signed-distance-field-modeling/
136. "AI Co-Artist: A LLM-Powered Framework for Interactive GLSL Shader Animation Evolution" — arXiv:2512.08951. https://arxiv.org/html/2512.08951
137. "Evaluating Language Models for Computer Graphics Code Completion" (LLM4Code 2025, ICSE 2025). https://conf.researchr.org/details/icse-2025/llm4code-2025-papers/13
138. "Exploring AI-powered art with simple text prompts" — 14islands journal. https://www.14islands.com/journal/ai-generated-glsl-shaders
139. Mitsuba 3. https://www.mitsuba-renderer.org/
140. Mitsuba 3 GitHub. https://github.com/mitsuba-renderer/mitsuba3
141. Arbitrary Image Stylization TFJS — reiinakano. https://github.com/reiinakano/arbitrary-image-stylization-tfjs
142. "Porting Arbitrary Style Transfer to the Browser" — Magenta. https://magenta.tensorflow.org/blog/2018/12/20/style-transfer-js/
143. "AI in the Browser: TensorFlow.js, WebGPU, and the Future of On-Device Inference" — KAIRI on Medium. https://medium.com/kairi-ai/ai-in-the-browser-tensorflow-js-webgpu-and-the-future-of-on-device-inference-42b4cc33ea26
144. Replicate Webhooks docs. https://replicate.com/docs/topics/webhooks
145. Together AI pricing docs. https://www.together.ai/pricing
146. OpenRouter Pricing. https://openrouter.ai/pricing
147. TripoSR repository / Stability AI. https://github.com/VAST-AI-Research/TriposSR
148. Meshy.ai home. https://www.meshy.ai/
149. DreamFusion (arXiv:2209.14988). https://arxiv.org/abs/2209.14988
150. Magic3D (arXiv:2211.10440). https://arxiv.org/abs/2211.10440
151. "How Replicate transformed their webhooks infrastructure using Svix" — Svix. https://www.svix.com/customers/replicate/
152. Modal pricing. https://modal.com/pricing
153. Hugging Face Inference Endpoints pricing. https://huggingface.co/docs/inference-endpoints/en/pricing
154. fal.ai vs Replicate comparison — getdeploying. https://getdeploying.com/fal-ai-vs-replicate
155. fal.ai LCM model on fal.ai. https://fal.ai/models/fal-ai/fast-lcm-diffusion/api
156. fal.ai SDXL Lightning model. https://fal.ai/models/fal-ai/fast-lightning-sdxl/api
157. antimatter15/splat WebGL Gaussian Splat Viewer. https://github.com/antimatter15/splat
158. SuperSplat. https://superspl.at
159. `@texel/color` modern library. https://github.com/texel-org/color
160. WCAG / APCA Web Standards update (Dec 2025). https://web-standards.dev/news/2025/12/apca-wcag-contrast-checker/
161. Hydra Video Synth docs. https://hydra.ojack.xyz/docs/
162. Hydra GitHub. https://github.com/hydra-synth/hydra
163. "Real-time Style Transfer Mirror with ml5.js" — Paperspace. https://blog.paperspace.com/creating-your-own-style-transfer-mirror/
164. "Tendrils: Emergent WebGL Particle Visuals" — webgpu.com showcase. https://www.webgpu.com/showcase/tendrils-emergent-webgl-particle-visuals/
165. "Particle Life simulation in browser using WebGPU" — lisyarus blog. https://lisyarus.github.io/blog/posts/particle-life-simulation-in-browser-using-webgpu.html
166. `piellardj/water-webgpu` — 1M-particle water sim. https://github.com/piellardj/water-webgpu
167. `kishimisu/WebGPU-Fluid-Simulation`. https://github.com/kishimisu/WebGPU-Fluid-Simulation
168. `matsuoka-601/WebGPU-Ocean`. https://github.com/matsuoka-601/WebGPU-Ocean
169. `scttfrdmn/webgpu-compute-exploration` — SPH, fractals, boids, DLA. https://github.com/scttfrdmn/webgpu-compute-exploration
170. "How to Create a Liquid Raymarching Scene Using Three.js Shading Language" — Codrops. https://tympanus.net/codrops/2024/07/15/how-to-create-a-liquid-raymarching-scene-using-three-js-shading-language/

---

*End of report.*
