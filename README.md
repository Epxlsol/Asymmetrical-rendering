# Asymmetrical-rendering


This engine architecture decouples view independent, repetitive shading math from repetitive, view dependent camera rasterization through a Dual World Pipeline. By seperating scenes into asynchronous paths, the engine eliminates geometric and material processing bottlenecks, achieving drastically higher framerates without hugely sacrificed visual fidelity.


Theres 2 Pipelines:


Background: The background pipeline manages all static environment data (terrain, infrastructure, mountains) through an itemized Physical Texture Atlas Pool with Object Space Surface Caching.


Decoupled Evaluation: Lighting computations, material shaders, multi-bounce global illumination, and ambient occlusion are calculated in object space. Results are mapped to a 2D Parameterized Grid directly via individual mesh texture coordinates (UVs).
VRAM Volumetric Cap: To prevent memory allocation overflow, the architecture locks cache pages into a fixed Physical Texture Atlas Pool (clamped tightly to a smaller VRAM allocation. It scales uniformly with viewport resolution and visibility limits, never with global world dimensions.
Temporal Caching: Once object surfaces are evaluated inside VRAM, the graphics engine skips recalculating fragment shader data for multiple sequential frames.
Intermittent Depth Layering: To maintain sharp contact shadows and react dynamically to ambient alterations (such as a moving light source or slowly creeping time-of-day changes), a Virtual Shadow Map (VSM) layer is mapped over the cache pool. The VSM operates on a 128x128 pixel localized tile eviction matrix. Tiles remain completely frozen in VRAM until explicit lighting shifts invalidate their depth pages, reducing real-time shadow generation math to a 0 overhead cost across identical frames.


Live: The Live Pipeline handles responsive game loops, processing characters, dynamic actors, immediate particle systems, projectiles, and nearby structural environments that require frame-accurate updates.

Native High-Frequency Shading: Operates on an un-cached forward or deferred pipeline executing at maximum hardware frame cycles with no input latency.
Proximity Buffer Filtering: The Live Pipeline extends outward from the camera frustum as a physical 3D bounding bubble. Any geometric asset or static background element crossing within a close radial proximity of the player is automatically transferred into the Live Pipeline.


**Issues + Fixes**


A common failure mode of surface-cache architectures occurs when a player sharply rounds a corner, exposing un-cached surfaces and triggering an immense computational cost (a "Cache Miss" storm) that results in a severe frame drop.

This method addresses this through its Dual World Composition:

The Proximity Shield: When rounding a corner, the corner vertex boundaries, neighboring structures, and structural seams are already inside the player's Live Pipeline bubble. They are rendered with full, real-time native accuracy.

Asynchronous Amortization: Because the immediate foreground elements are fully drawn by the Live Pipeline, the distant background elements uncovered by the sweeping camera frustum have a multi frame buffer window to populate . The Background Pipeline streams missing tiles into the Physical Texture Atlas Pool asynchronously, time slicing the processing workload across 3 to 5 frames to completely level off computational spikes.


Spatiotemporal Compositing: Every frame, the GPU composites the pre lit Background theater and the high speed Live Pipeline data via depth sorting in the framebuffer. A stochastic dithered transition mask is applied along the perimeter edges where the Live Pipeline merges into the Background layer, providing a near seamless visual transition.


Startup Cache Initialization: Just as engines perform a shader pre-compilation step at startup to eliminate stuttering, this engine runs a localized initialization sweep across background meshes to pre-populate baseline ambient values directly into the object space atlas pool.

Optimized Hardware Footprint: High-frequency geometry streaming (LODs) drops background mesh vertices down to low overhead polygon proxies at a distance. The GPU is entirely freed from evaluating expensive multi bounce lighting per-pixel, allowing consumer hardware to drive massive, high-fidelity viewports at raw e-sports framerates.


**Pipeline Intersect & Compositing**

The transition and overlap between pipelines are handled at the hardware layer with two methods:

1. Overlap: 
   The Live Pipeline evaluates immediate player geometry and proximity assets at native engine speeds. The GPU uses standard Z-buffering/depth testing to instantly overlay the Live pipeline's Color Buffer onto the cached Background buffer stored in VRAM.

2. Dither Merge:
   To stop pop in from the Background pipeline transitioning into the Live pipeline, a screen space dither shader can be used. By changing pixel sampling between the two pipeline layers using a temporal noise mask the transition becomes smoother.
