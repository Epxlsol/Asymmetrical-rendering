# Asymmetrical-Rendering Architecture

This engine architecture decouples **low-frequency, view-independent global illumination math** from **high-frequency, view-dependent camera rasterization** through a Three-Tier Asymmetrical Pipeline. By dividing scene elements into independent execution paths based on update frequency, the engine eliminates geometric and material processing bottlenecks, achieving drastically higher framerates without severe compromises to visual fidelity.

---

## The Three-Tier Pipeline Layout

Instead of forcing the entire scene layout through a single, heavily congested frame loop, the architecture splits processing tasks across three synchronized pipelines running at independent update rates.

[ MULTI-FREQUENCY DOMAIN RUNTIME ]

STATIC BACKGROUND  --->  Low-Frequency (1–10 Hz)   --->  Object-Space Surface Cache

DEFORMABLE WORLD    --->  Mid-Frequency (30–60 Hz)  --->  Volumetric Probe / Vertex Pass

LIVE PROXIMITY     --->  High-Frequency (240+ Hz)  --->  Native Screen-Space Shading

### 1. The Static Background Pipeline (Low-Frequency Tier)
The Background Pipeline manages all invariant environment data (such as terrain, infrastructure, and mountains) using an itemized **Physical Texture Atlas Pool** driven by Object-Space Surface Caching.

* **Decoupled Diffuse Evaluation:** Heavy multi-bounce global illumination (GI), ambient occlusion, and matte material math are calculated entirely in object-space. Results are mapped directly to a 2D Parameterized Grid via individual mesh texture coordinates (UVs).
* **VRAM Volumetric Cap:** To prevent memory allocation overflow, the architecture locks cache pages into a fixed Physical Texture Atlas Pool (clamped tightly to a fixed 512MB–1GB VRAM budget). This allocation footprint scales uniformly with viewport resolution and visibility limits using Virtual Texturing—never with global world dimensions.
* **Temporal Caching:** Once object surfaces are evaluated inside VRAM, the graphics engine skips recalculating fragment shader data for multiple sequential frames.
* **Intermittent Depth Layering:** To maintain sharp contact shadows and react dynamically to ambient alterations (such as a creeping time-of-day cycle), a **Virtual Shadow Map (VSM)** layer is mapped over the cache pool. The VSM operates on a 128x128 pixel localized tile eviction matrix. Tiles remain completely frozen in VRAM until explicit lighting shifts invalidate their depth pages, reducing real-time shadow generation math to a 0 ms overhead cost across invariant frames.

### 2. The Deformable Environment Pipeline (Mid-Frequency Tier)
To prevent "Cache Invalidation Storms" caused by moving or destructible background elements (such as swaying foliage, blowing flags, rotating windmills, or crumbling walls), these assets are explicitly banned from the static texture atlas pool.

* **Volumetric Probe Interpellation:** Deformable assets are routed through a separate pipeline where vertex movement calculations are handled cheaply on the GPU, but their shading data is drawn from a low-resolution, asynchronously updated **Global Light Probe / Voxel Volume**. 
* **Dynamic Decoupling:** Lighting updates for an animating background asset can be safely time-sliced and updated at 30 Hz or 60 Hz, entirely decoupled from the main camera's fast refresh rate.

### 3. The Live Proximity Pipeline (High-Frequency Tier)
The Live Pipeline handles highly responsive game loops, processing player models, dynamic actors, immediate particle systems, projectiles, and sharp, view-dependent optics.

* **Native High-Frequency Shading:** Operates on an un-cached forward or deferred pipeline executing at maximum hardware frame cycles (240+ Hz target refresh rates) with absolute minimum input latency.
* **View-Dependent Specular Overlay:** Sharp specular reflections, metallic highlights, and gloss—which change dynamically based on the camera angle—are calculated natively in screen-space (via SSR or ray-traced reflections) and overlaid seamlessly over the cached background diffuse data during final composition.
* **Proximity Buffer Filtering:** The Live Pipeline extends outward from the camera frustum as a physical 3D bounding bubble. Any geometric asset or static background element crossing within a close radial proximity of the player is automatically elevated and transferred into the Live Pipeline.

---

## Architectural Vulnerabilities & Mitigations

A common failure mode of surface-cache architectures occurs when a player sharply rounds a corner, exposing un-cached surfaces and triggering an immense computational cost (a "Cache Miss" storm) that results in a severe frame drop. This framework natively mitigates this through its Multi-Frequency composition:


[ EXTENDED VIEWPORT ]
|---> Foreground (Within Proximity Bubble) ---> Render via Live Pipeline (100% Real-Time Precision)
|---> Midground/Background (Frustum Entry) ---> Cached Virtual Texture Atlas (Amortized over 3-5 frames)


* **The Proximity Shield:** When rounding a corner, the corner vertex boundaries, neighboring structures, and structural seams are *already inside* the player's local Live Pipeline bubble. They are rendered with full, real-time native accuracy, shielding the player from visual holes.
* **Asynchronous Amortization:** Because the immediate foreground elements are fully handled by the Live Pipeline, the distant background elements uncovered by the sweeping camera frustum have a multi-frame buffer window to populate. The Background Pipeline streams missing tiles into the Physical Texture Atlas Pool asynchronously, time-slicing the processing workload across 3 to 5 frames to completely level off computational spikes.
* **Amortized Stream with Stochastic Fallbacks (The Occlusion Shield):** If a player sharply rounds a corner and exposes a massive, un-cached distant mountain range, the textures cannot bake instantly. To completely eliminate visual holes or "black shapes" during the 3–5 frame asynchronous bake window, the engine deploys a **Stochastic Low-Res Fallback Pass**. For those initial 3 frames, the distant assets bypass the atlas and render via a dirt-cheap, un-cached vertex-color shader or a low-frequency, screen-space global illumination probe fallback. This maintains visual continuity and color matching at a fraction of the cost, cleanly bridging the gap until the high-fidelity atlas tiles stream in.
* **Startup Cache Initialization:** Just as engines perform a shader pre-compilation step at startup to eliminate runtime stuttering, this engine runs a localized initialization sweep across background meshes during the loading sequence to pre-populate baseline ambient values directly into the object-space atlas pool.
* **Optimized Hardware Footprint:** High-frequency geometry streaming (Level of Detail / LOD systems) drops background mesh vertices down to low-overhead polygon proxies at a distance. The GPU is entirely freed from evaluating expensive multi-bounce lighting per-pixel, reducing the screen-space environment fragment shader pass to a single **hardware bilinear texture fetch (approx. 4 instructions per pixel)**.

---

## Pipeline Intersect & Compositing

The transition, layout sorting, and overlap between the distinct processing layers are handled directly at the hardware layer using three primary methods:

### 1. Hardware Depth Overlap
The Live Pipeline evaluates immediate player geometry and proximity assets at native engine speeds. The GPU uses standard hardware Z-buffering and depth-testing to instantly overlay the Live Pipeline's Color Buffer onto the cached Background buffer stored in VRAM, eliminating sorting artifacts.

### 2. Spatiotemporal Dither Merge
To prevent harsh popping artifacts as the Background pipeline data transitions into the high-frequency Live pipeline, a screen-space dither shader is deployed. By changing pixel sampling criteria between the two pipeline layers using a temporal noise mask, the spatial transition boundary becomes visually seamless.

### 3. Spatiotemporal Alignment via Motion Vector Re-projection
Because the Static Background Pipeline evaluates lighting at a lower frequency (10–30 Hz) than the Live Proximity Pipeline (240+ Hz camera loop), high-speed camera movement would traditionally cause the background texture to lag, judder, or smear behind the geometry. 

To maintain pixel-perfect alignment, the engine implements **Velocity Vector Re-projection**:
[ FRAME N-1 CACHE ] ---> [ Apply Camera Delta Matrix ] ---> [ Temporally Re-projected Frame N ]
|
(No Shading Math Required)


Every frame, the vertex shader outputs screen-space motion vectors calculating the exact pixel delta between the current frame ($N$) and the previous frame ($N-1$). When the player rotates the camera rapidly, the engine samples the lower-frequency background cache but uses the motion vectors to warp and re-project the texture coordinates to perfectly match the new camera matrix. This math occurs completely via lightweight texture coordinate offsets, achieving crystal-clear background stability during movement without forcing the object-space lighting thread to re-render.
