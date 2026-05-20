# GPU-Driven Asynchronous GI Pipeline (v3.0)
## Technical Design: Layered Compositing & Asynchronous Scheduling

### 1. Abstract
This architecture defines a hardware-accelerated, layered rendering pipeline. By decoupling the rendering process into three distinct layers (LIVE, DYNAMIC, and BACKGROUND) that operate on independent GPU queues, we achieve high-fidelity dynamic lighting with resilience against frame-time spikes and resource bottlenecks.

### 2. The Three-Tier Layered Model
The frame is assembled using a compositing approach. Each layer maintains its own update lifecycle, allowing the engine to function even when background data is "stale."

| Tier | Function | Hardware Queue | Update Frequency |
| :--- | :--- | :--- | :--- |
| **LIVE** | Direct Raster, Player Interaction | Graphics | Every Frame (16.6ms) |
| **DYNAMIC** | Temporal Reprojection/Glue | Compute (Async) | Every 1-2 Frames |
| **BACKGROUND** | Voxelization, GI Ray-tracing/Bounces | Compute (Async) | Variable (Staggered) |


### 3. Layered Compositing & Stale Data Strategy
Unlike traditional synchronous pipelines, our compositor follows a **Non-Blocking Composition Policy**:
* **Frame-Independent Assembly:** If the `BACKGROUND` layer is not ready for the current frame, the compositor reuses the previous valid buffer.
* **Resilience:** This prevents the `LIVE` layer (interactables/flashlight) from stalling, ensuring the game maintains high responsiveness even during heavy indirect lighting calculations.
* **Dynamic Glue:** The `DYNAMIC` layer uses Motion Vectors to reproject "stale" `BACKGROUND` data, effectively warping it to align with current camera transformations to prevent "snapping."

### 4. AI-Accelerated Denoising
We utilize Tensor Cores to bridge the gap between low-frequency `BACKGROUND` updates and high-frequency `LIVE` rendering:
* **Tensor Inference Pass:** Raw, noisy radiance data from the `BACKGROUND` tier is processed by AI-denoisers in the `DYNAMIC` tier.
* **Temporal Reconstruction:** The AI utilizes current motion vectors and historical frame data to fill in missing light information, providing a stable, high-fidelity result even at lower refresh intervals.


### 5. Architectural Constraints & Mitigations
| Constraint | Mitigation Strategy |
| :--- | :--- |
| **Sync Drift** | Global Timeline Semaphores ensure the compositor only samples matching timestamps. |
| **Bandwidth Contention** | Priority-Based Memory Scheduling: LIVE layer receives reserved bandwidth; BACKGROUND is throttled under load. |
| **VRAM Footprint** | Sparse Resource Aliasing: DYNAMIC and BACKGROUND tiers share memory buffers. |
| **Light Bleed** | Disocclusion detection triggers an immediate high-priority BACKGROUND update pass. |

### 6. Summary
By shifting from an atomic rendering model to an **Asynchronous Layered Composition** model, we transform GI from a performance-draining bottleneck into a background service. This system provides a path to real-time, interactive, and high-fidelity worlds that scale gracefully across hardware tiers.

---
