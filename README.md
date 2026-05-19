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
* **Intermittent Depth Layering:** To maintain sharp contact shadows and react dynamically to ambient alterations (such as a moving light source or a creeping time-of-day cycle), a **Virtual Shadow Map (VSM)** layer is mapped over the cache pool. The VSM operates on a $128 \times 128$ pixel localized tile eviction matrix. Tiles remain completely frozen in VRAM until explicit lighting shifts invalidate their depth pages, reducing real-time shadow generation math to a $0\text{ ms}$ overhead cost across invariant frames.

### 2. The Live Pipeline (Proximity-Bounded Dynamic World)
The Live Pipeline handles responsive game loops, processing player models, dynamic actors, immediate particle systems, projectiles, and interactive structural environments that require frame-accurate updates.

* **Native High-Frequency Shading:** Operates on an un-cached forward or deferred pipeline executing at maximum hardware frame cycles ($240\text{+ Hz}$ target targets) with absolute minimum input latency.
* **Proximity Buffer Filtering:** The Live Pipeline extends outward from the camera frustum as a physical 3D bounding bubble. Any geometric asset or static background element crossing within a close radial proximity of the player is automatically elevated and transferred into the Live Pipeline.

---

## Architectural Vulnerabilities & Mitigations

A common failure mode of surface-cache architectures occurs when a player sharply rounds a corner, exposing un-cached surfaces and triggering an immense computational cost (a "Cache Miss" storm) that results in a severe frame drop. This framework natively mitigates this through its Dual-World composition:
