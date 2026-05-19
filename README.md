# Asymmetrical-Rendering Architecture

This engine architecture decouples **view-independent, high-frequency shading math** from **view-dependent, real-time camera rasterization** through a Dual-World Pipeline. By dividing scene elements into asynchronous execution paths, the engine eliminates geometric and material processing bottlenecks, achieving drastically higher framerates without severe compromises to visual fidelity.

---

## The Dual-World Pipeline Layout

Instead of forcing the entire scene layout through a single, heavily congested pipeline, the architecture splits processing tasks across two synchronized engines running at independent update rates.

### 1. The Background Pipeline (Object-Space Static World)
The Background Pipeline manages all static environment data (terrain, infrastructure, mountains) using an itemized **Physical Texture Atlas Pool** driven by Object-Space Surface Caching.

* **Decoupled Shading Evaluation:** Lighting computations, material shaders, multi-bounce global illumination (GI), and ambient occlusion are calculated entirely in object-space. Results are mapped directly to a 2D Parameterized Grid via individual mesh texture coordinates (UVs).
* **VRAM Volumetric Cap:** To prevent memory allocation overflow, the architecture locks cache pages into a fixed Physical Texture Atlas Pool (clamped tightly to a smaller, fixed VRAM budget). This allocation footprint scales uniformly with viewport resolution and visibility limits—never with global world dimensions.
* **Temporal Caching:** Once object surfaces are evaluated inside VRAM, the graphics engine skips recalculating fragment shader data for multiple sequential frames.
* **Intermittent Depth Layering:** To maintain sharp contact shadows and react dynamically to ambient alterations (such as a moving light source or a creeping time-of-day cycle), a **Virtual Shadow Map (VSM)** layer is mapped over the cache pool. The VSM operates on a 128x128 pixel localized tile eviction matrix. Tiles remain completely frozen in VRAM until explicit lighting shifts invalidate their depth pages, reducing real-time shadow generation math to a 0 ms overhead cost across invariant frames.

### 2. The Live Pipeline (Proximity-Bounded Dynamic World)
The Live Pipeline handles responsive game loops, processing player models, dynamic actors, immediate particle systems, projectiles, and interactive structural environments that require frame-accurate updates.

* **Native High-Frequency Shading:** Operates on an un-cached forward or deferred pipeline executing at maximum hardware frame cycles (240+ Hz target refresh rates) with absolute minimum input latency.
* **Proximity Buffer Filtering:** The Live Pipeline extends outward from the camera frustum as a physical 3D bounding bubble. Any geometric asset or static background element crossing within a close radial proximity of the player is automatically elevated and transferred into the Live Pipeline.

---

## Architectural Vulnerabilities & Mitigations

A common failure mode of surface-cache architectures occurs when a player sharply rounds a corner, exposing un-cached surfaces and triggering an immense computational cost (a "Cache Miss" storm) that results in a severe frame drop. This framework natively mitigates this through its Dual-World composition:

[ EXTENDED VIEWPORT ]
|---> Foreground (Within Proximity Bubble) ---> Render via Live Pipeline (100% Real-Time Precision)
|---> Midground/Background (Frustum Entry) ---> Cached Virtual Texture Atlas (Amortized over 3-5 frames)

* **The Proximity Shield:** When rounding a corner, the corner vertex boundaries, neighboring structures, and structural seams are *already inside* the player's local Live Pipeline bubble. They are rendered with full, real-time native accuracy, shielding the player from visual holes.
* **Asynchronous Amortization:** Because the immediate foreground elements are fully handled by the Live Pipeline, the distant background elements uncovered by the sweeping camera frustum have a multi-frame buffer window to populate. The Background Pipeline streams missing tiles into the Physical Texture Atlas Pool asynchronously, time-slicing the processing workload across 3 to 5 frames to completely level off computational spikes.
* **Startup Cache Initialization:** Just as engines perform a shader pre-compilation step at startup to eliminate runtime stuttering, this engine runs a localized initialization sweep across background meshes during the loading sequence to pre-populate baseline ambient values directly into the object-space atlas pool.
* **Optimized Hardware Footprint:** High-frequency geometry streaming (Level of Detail / LOD systems) drops background mesh vertices down to low-overhead polygon proxies at a distance. The GPU is entirely freed from evaluating expensive multi-bounce lighting per-pixel, allowing consumer hardware to drive massive viewports at raw e-sports framerates.

---

## Pipeline Intersect & Compositing

The transition, layout sorting, and overlap between the two distinct processing layers are handled directly at the hardware layer using two primary methods:

### 1. Hardware Depth Overlap
The Live Pipeline evaluates immediate player geometry and proximity assets at native engine speeds. The GPU uses standard hardware Z-buffering and depth-testing to instantly overlay the Live Pipeline's Color Buffer onto the cached Background buffer stored in VRAM, eliminating sorting artifacts.

### 2. Spatiotemporal Dither Merge
To prevent harsh popping artifacts as the Background pipeline data transitions into the high-frequency Live pipeline, a screen-space dither shader is deployed. By changing pixel sampling criteria between the two pipeline layers using a temporal noise mask, the spatial transition boundary becomes visually seamless.
