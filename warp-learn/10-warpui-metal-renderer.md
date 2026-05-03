# Metal Renderer — macOS GPU Rendering

> Source: `crates/warpui/src/platform/mac/rendering/metal/`
> How Warp turns a Scene into pixels on macOS using Apple's Metal GPU API.

---

## What Is Metal?

Apple's GPU API (like Vulkan for Linux, DirectX for Windows).
The `metal` Rust crate wraps the Objective-C Metal API.

---

## GPU Rendering From Absolute Zero

Your screen is a grid of pixels. The GPU's only job: decide the color of every pixel.

```
Screen (simplified 10×6):
  0 1 2 3 4 5 6 7 8 9
0 . . . . . . . . . .
1 . . . . . . . . . .     Each dot = one pixel = one RGBA color
2 . . . . . . . . . .
3 . . . . . . . . . .     A 1920×1080 screen = 2 million pixels
4 . . . . . . . . . .     GPU decides all their colors in parallel
5 . . . . . . . . . .
```

### Drawing one blue rect at (2,1) size 5×3:

**Step 1: Define the shape as triangles**
GPUs only understand triangles. A rect = 2 triangles = 1 "quad":

```
vertex 0 (0,0) ──── vertex 1 (1,0)     indices: [0,1,2, 2,3,1]
     │  ╲                │                       ─────── ───────
     │    ╲   tri 1      │                       tri 1   tri 2
     │ tri 2 ╲           │
vertex 2 (0,1) ──── vertex 3 (1,1)     This is the "unit quad" — reused for everything
```

**Step 2: Vertex shader — "where do the corners go?"**
GPU runs this 4 times (once per corner):

```
vertex_id=0: (0,0) × size(5,3) + origin(2,1) = (2,1)   top-left
vertex_id=1: (1,0) × size(5,3) + origin(2,1) = (7,1)   top-right
vertex_id=2: (0,1) × size(5,3) + origin(2,1) = (2,4)   bottom-left
vertex_id=3: (1,1) × size(5,3) + origin(2,1) = (7,4)   bottom-right
```

**Step 3: Rasterization — "which pixels are inside?" (automatic)**
GPU hardware fills in which pixels fall inside the triangles. No code needed.

```
  0 1 2 3 4 5 6 7 8 9
1 . . ? ? ? ? ? . . .    ? = pixels inside the quad
2 . . ? ? ? ? ? . . .    GPU identified 15 pixels
3 . . ? ? ? ? ? . . .
```

**Step 4: Fragment shader — "what color is each pixel?"**
GPU runs this once per `?` pixel (15 times, ALL in parallel):

```
pixel (4,2): deep inside → return solid blue
pixel (2,1): on corner   → distance_from_rect() = 0.3
                            alpha = 0.7 → semi-transparent (anti-aliased!)
pixel (8,2): outside      → alpha = 0 → invisible
```

**Step 5: Result on screen**

```
  0 1 2 3 4 5 6 7 8 9
1 . . ■ ■ ■ ■ ■ . . .
2 . . ■ ■ ■ ■ ■ . . .
3 . . ■ ■ ■ ■ ■ . . .
```

### Instancing — drawing 500 rects in ONE call

Instead of repeating 500 times, you say "draw this quad 500 times with different data":

```
vertices:  [(0,0), (1,0), (0,1), (1,1)]     ← ONE shared quad
uniforms:  [
  { origin:(2,1),   size:(5,3),  color:blue  },  ← rect #0
  { origin:(50,10), size:(20,8), color:red   },  ← rect #1
  ...499 more...
]
draw_instanced(6 indices, 500 instances)

GPU runs vertex shader 4 × 500 = 2000 times
  instance_id picks which rect's data
  vertex_id picks which corner
GPU runs fragment shader for ALL pixels in ALL 500 rects (in parallel)
```

---

## What Is a Shader?

A tiny program that **runs on the GPU, once per pixel/vertex, in massive parallel**.
CPU handles 8 things at once. GPU handles thousands.

```
Vertex shader:   "where does this shape go on screen?"
                  runs once per corner (4× per rect)

Fragment shader: "what color is this pixel?"
                  runs once per pixel (thousands in parallel)
```

---

## Three Primitives = Entire UI

```
Rect   → colored box (fill, gradient, border, corner radius, shadow, dashes)
Glyph  → one text character (from glyph atlas texture)
Image  → bitmap (photo, icon, SVG pre-rasterized)

Button     = Rect + Glyphs
Scrollbar  = Rect + Rect
Terminal   = Rect + thousands of Glyphs
Avatar     = Image with corner_radius
Cursor     = thin Rect
Circle     = Rect with corner_radius = width/2
Line       = 1px-tall Rect
```

---

## The Rendering Pipeline

```
Scene (from Presenter)
  │
  ├── layer[0].rects   → draw_rects()   → rect shader
  ├── layer[0].images  → draw_images()  → image shader
  ├── layer[0].glyphs  → draw_glyphs()  → glyph shader
  ├── layer[1]...      → same, drawn on top (scissor clipped)
  │
  └── present drawable → pixels on screen
```

---

## Key Files

```
renderer.rs            Rust code: sets up pipelines, sends data to GPU
shaders.metal          GPU code: vertex + fragment shaders (MSL)
shader_types.h         Shared C structs (Uniforms, PerRectUniforms, PerGlyphUniforms)
build.rs               Compiles shaders.metal → shaders.metallib (GPU bytecode)
                       Runs bindgen: shader_types.h → shader_types.rs
```

---

## How a Rect Gets Drawn

```
1. Rust (renderer.rs):
   for each rect in layer.rects:
     pack PerRectUniforms { origin, size, color, border, radius, shadow... }
   upload all uniforms to GPU buffer
   encoder.draw_indexed_primitives_instanced(500 rects)

2. GPU vertex shader (shaders.metal):
   for each rect, for each of 4 corners:
     scale unit quad (0,0)-(1,1) to actual rect position
     convert pixel coords → screen coords (-1..+1)
     pass all rect properties to fragment shader

3. GPU rasterizer (hardware, automatic):
   fills in all pixels inside the quad

4. GPU fragment shader (shaders.metal):
   for EACH pixel (in parallel):
     compute distance from rounded rect edge (SDF)
     if drop_shadow → gaussian blur math
     else → blend background color + border color
     apply corner rounding via distance field
     return final RGBA color

5. Result → framebuffer → drawable.present() → screen
```

---

## Signed Distance Field (SDF) — The Key Math

```
distance_from_rect(pixel_pos, center, corner, radius)

Returns:
  negative → pixel is INSIDE the shape
  zero     → pixel is ON the edge
  positive → pixel is OUTSIDE the shape

        outside (+2)
          ┌──────────────┐
          │  edge (0)    │
          │  ┌────────┐  │
          │  │ inside  │  │
          │  │  (-5)   │  │
          │  └────────┘  │
          └──────────────┘

Used for:
  corner rounding:  color.a *= 1.0 - saturate(distance + 0.5)
  borders:          inner_distance vs outer_distance
  anti-aliasing:    smooth transition at ±0.5px
```

---

## Glyph Rendering — Text on Screen

```
1. For each character in Scene.glyphs:
   - Look up in glyph cache (texture atlas on GPU)
   - If not cached: rasterize via Core Text → upload to atlas
   - Pack PerGlyphUniforms { position, UV coords, color, fade, is_emoji }

2. GPU glyph shader:
   - vertex: position the glyph quad, compute fade alpha
   - fragment: sample glyph bitmap from atlas texture
     - Regular text: use input color, multiply by sampled alpha
     - Emoji: use sampled color directly (full color bitmap)
     - Apply fade effect (text fading at edges)
```

Glyph atlas = one big GPU texture with many characters packed in:
```
┌──────────────────────┐
│ A B C D E F G H I J  │
│ K L M N O P Q R S T  │
│ a b c d e f g h i j  │
│ 😀 🎉 ... (emoji)    │
└──────────────────────┘
Each glyph has UV coordinates pointing to its region.
```

---

## Metal API Objects (Rust ↔ Metal)

```
Rust (metal crate)              Apple Metal
──────────────────              ───────────
metal::Device                   MTLDevice (the GPU)
metal::CommandQueue             MTLCommandQueue (work queue)
metal::CommandBuffer            MTLCommandBuffer (one frame's commands)
metal::RenderCommandEncoder     MTLRenderCommandEncoder (records draw calls)
metal::MetalDrawableRef         CAMetalDrawable (the canvas / framebuffer)
metal::RenderPipelineState      MTLRenderPipelineState (compiled shader pair)
metal::Buffer                   MTLBuffer (data on GPU)
metal::Texture                  MTLTexture (image on GPU)
```

---

## Why Shaders Are Written in MSL, Not Rust

```
Rust (CPU)                          MSL shader (GPU)
──────────                          ────────────────
"Set up pipeline"                   "For each pixel, compute color"
"Upload data to GPU"                Runs on thousands of GPU cores
"Say: draw 500 rects"  ──────→     in parallel

GPU has its own instruction set. Can't run Rust.
Like SQL runs in a database — you send it, the engine runs it.
```

---

## Command Encoder — The GPU To-Do List Writer

The encoder doesn't execute anything. It **records commands**. The GPU executes them later in one batch.

```
Rust (encoder):                           GPU (later):
──────────────                            ────────────
"use rect pipeline"
"bind quad vertices at slot 0"
"bind rect uniforms at slot 1"
"bind viewport at slot 2"
"draw 500 instanced quads"     ──commit──→ executes everything
"use glyph pipeline"                       in parallel
"bind glyph uniforms"
"draw 3000 instanced quads"
```

Batching = fast. One submit instead of thousands of individual draws.

---

## Shader Input Annotations — How Data Flows to GPU

```
Rust (encoder):                              Shader (GPU reads):
──────────────                               ──────────────────
encoder.set_vertex_buffer(0, quad_vertices)  → [[buffer(0)]] = vertices
encoder.set_vertex_buffer(1, rect_uniforms)  → [[buffer(1)]] = per-rect data
encoder.set_vertex_bytes(2, viewport)        → [[buffer(2)]] = global uniforms

                                             [[vertex_id]]   = which corner (0-3)
                                             [[instance_id]]  = which rect (0-499)
                                             (auto from GPU, not set by Rust)
```

Buffer slots are numbered mailboxes. Rust puts data in, shader reads from same number.

---

## Frame Lifecycle

```
1. window.nextDrawable()      → get framebuffer to draw into
2. command_queue.new_buffer() → create command buffer
3. buffer.new_encoder()       → create render encoder
4. for each layer:
     set scissor rect (clipping)
     draw_rects(layer)        → encode rect draw calls
     draw_images(layer)       → encode image draw calls
     draw_glyphs(layer)       → encode glyph draw calls
5. encoder.end_encoding()     → finalize commands
6. buffer.commit()            → send to GPU (GPU starts working)
7. buffer.wait_until_completed() → wait for GPU
8. drawable.present()         → show on screen
```
