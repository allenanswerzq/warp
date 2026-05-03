# GPU Rendering Landscape — Why So Many APIs and Languages

> Background knowledge to understand why Warp has Metal + wgpu + MSL + WGSL

---

## The Problem

Every GPU vendor (Apple, NVIDIA, AMD, Intel) has different hardware.
Every OS talks to GPUs differently. So we ended up with many APIs.

---

## GPU APIs (how your code talks to the GPU)

```
API          OS              Who made it     Era
───          ──              ───────────     ───
OpenGL       everywhere      Khronos Group   1992 — old, simple, deprecated
DirectX 11   Windows         Microsoft       2009 — Windows gaming
Metal        macOS/iOS       Apple           2014 — Apple only, fast
Vulkan       Win/Linux/And   Khronos Group   2016 — modern, verbose, fast
DirectX 12   Windows         Microsoft       2015 — modern, verbose, fast
WebGL        browsers        Khronos Group   2011 — OpenGL for web
WebGPU       browsers        W3C             2023 — modern GPU for web
```

### Why so many?

- **Apple** refused to support Vulkan and made **Metal** instead
- **Microsoft** has their own thing: **DirectX**
- **Vulkan** is the open standard but Apple doesn't support it
- **Web** has its own constraints (sandboxed, no direct GPU access)

---

## Shader Languages (code that runs ON the GPU)

```
Language     Used by          Looks like     Compiled to
────────     ───────          ──────────     ───────────
GLSL         OpenGL/Vulkan    C-like         SPIR-V bytecode
HLSL         DirectX          C-like         DXIL bytecode
MSL          Metal            C++-like       metallib bytecode
WGSL         WebGPU/wgpu      Rust-like      SPIR-V → driver
SPIR-V       Vulkan           binary IR      (intermediate format)
```

### Why can't GPUs just run one language?

Each GPU vendor compiles shader code to their specific hardware instructions.
The shader language is just a starting point — the GPU driver compiles it
further to the actual chip's machine code:

```
Your shader code (WGSL/MSL/GLSL)
       │
       ▼
Compiler (in GPU driver or build tool)
       │
       ▼
GPU-specific machine code (different for NVIDIA vs AMD vs Apple Silicon)
```

---

## How wgpu Solves the Fragmentation

wgpu is a **Rust crate that wraps all the native APIs** behind one interface:

```
Your Rust code
     │
   wgpu (one API)
     │
     ├── macOS     → translates to Metal calls
     ├── Windows   → translates to DirectX 12 (or Vulkan)
     ├── Linux     → translates to Vulkan
     └── Browser   → translates to WebGPU
```

For shaders, wgpu uses **WGSL** and translates it:

```
Your WGSL shader code
     │
   naga (wgpu's shader compiler)
     │
     ├── macOS     → converts to MSL → Metal compiles it
     ├── Windows   → converts to HLSL → DirectX compiles it
     ├── Linux     → converts to SPIR-V → Vulkan driver compiles it
     └── Browser   → sends WGSL directly → WebGPU compiles it
```

**You write WGSL once → naga translates → every GPU can run it.**

---

## Why Warp Has BOTH Metal and wgpu

```
macOS:           Metal directly (fastest, started here)
Linux/Win/WASM:  wgpu (cross-platform, added later)

Metal (MSL)  ←── Warp started as Mac-only, native Metal was best
wgpu (WGSL)  ←── when expanding to other platforms, wgpu covers all
```

Eventually they may unify on wgpu for everything
(the `experimental-wgpu-renderer` feature flag on macOS suggests this).

---

## Quick Comparison

```
                  Metal          wgpu             OpenGL
                  ─────          ────             ──────
Performance       best on Mac    great everywhere  ok, outdated
Platforms         Apple only     everywhere        everywhere (deprecated)
Shader language   MSL            WGSL              GLSL
Complexity        medium         medium            low (but limited)
Modern features   yes            yes               no
Who uses it       Apple apps     Bevy, Warp, Zed   legacy apps

                  Vulkan         DirectX 12
                  ──────         ──────────
Performance       best on Linux  best on Windows
Platforms         Linux/Win/And  Windows only
Shader language   GLSL/SPIR-V    HLSL
Complexity        very high      very high
Modern features   yes            yes
Who uses it       game engines   game engines
```

---

## The Analogy

```
GPU API (runs on cpu) = the delivery service (FedEx, UPS, DHL)
  → different service per country, same job: deliver packages

Shader language (runs on gpu) = the language you write the shipping label in
  → the delivery service translates it to local language

wgpu = a universal shipping broker
  → you give it one label (WGSL), it picks the right service per country
```

---

## In Warp's Codebase

```
crates/warpui/src/
├── platform/mac/rendering/metal/
│   ├── shaders/shaders.metal       ← MSL (Apple's shader language)
│   └── renderer.rs                 ← Metal API calls (Apple only)
│
└── rendering/wgpu/
    ├── shaders/
    │   ├── rect_shader.wgsl        ← WGSL (cross-platform)
    │   ├── glyph_shader.wgsl
    │   └── image_shader.wgsl
    └── renderer.rs                 ← wgpu API calls (everywhere else)

Both do the EXACT same thing:
  Scene.rects  → rect shader   → colored boxes on screen
  Scene.glyphs → glyph shader  → text characters on screen
  Scene.images → image shader  → images on screen

Just different APIs + shader languages underneath.
```

### How it fits in the landscape

                    GRAPHICS (drawing)        COMPUTE (math)
                    ──────────────────        ──────────────
NVIDIA only         -                         CUDA
Apple only          Metal (graphics)          Metal Compute
Cross-platform      Vulkan, OpenGL            Vulkan Compute, OpenCL
Cross-platform Rust wgpu (graphics)           wgpu (compute)
Web                 WebGPU (graphics)         WebGPU (compute)
