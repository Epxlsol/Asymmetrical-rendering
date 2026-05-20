# GPU-Driven Asynchronous GI Pipeline
## Design Proposal: The Three-Tier Scheduling Model

### 1. Abstract
This proposal outlines a decoupled rendering architecture designed to offload heavy Global Illumination (GI) computations from the main graphics thread. By utilizing a "Three-Tier" task scheduling model, we can leverage the asynchronous capabilities of modern GPUs to achieve high-fidelity dynamic lighting without compromising frame-time consistency.

### 2. Problem Statement
Current VoxelGI/SDFGI implementations are constrained by:
* **Synchronous Bottlenecks:** Main-thread "baking" causes frame-time stutters.
* **Scene-Tree Coupling:** Rigid reliance on existing nodes limits dynamic object interaction.
* **8-Node Limit:** Inefficient scaling for complex, open-world environments.

### 3. The Three-Tier Architecture
We propose splitting the rendering pipeline into independent task tiers:

| Tier | Responsibility | Queue | Update Frequency |
| :--- | :--- | :--- | :--- |
| **LIVE** | Direct lighting, G-Buffer, Rasterization | Graphics | Every Frame (16.6ms) |
| **DYNAMIC** | Motion Vectors, Reprojection, Temporal Filtering | Compute (Async) | Every 1-2 Frames |
| **BACKGROUND** | Voxelization, GI Ray-tracing/Bounces, Denoising | Compute (Async) | Every 4-8 Frames |

### 4. Implementation Logic
* **LIVE Tier:** Operates as the standard Forward+ cluster loop. It remains the source of truth for direct visual output, ensuring 0ms latency for player-facing input (e.g., flashlights).
* **DYNAMIC Tier:** Acts as the "glue." It uses motion vectors to reproject data from previous frames, ensuring the indirect lighting remains glued to surfaces even during camera movement.
* **BACKGROUND Tier:** The heavy-lifting engine. It performs voxelization and cone tracing on an asynchronous `RenderingDevice` compute queue. It ignores the current frame's millisecond budget, focusing solely on high-fidelity light simulation.

### 5. Synchronization & Data Flow
To prevent stalls, we utilize a **Triple-Buffering** strategy for GI textures:
1.  **Read/Write Swap:** While the Graphics Queue samples from the "Last Valid GI Buffer," the Async Compute Queue writes new data to the "Next GI Buffer."
2.  **Fences/Semaphores:** The renderer utilizes non-blocking synchronization fences to swap buffers once the `BACKGROUND` tier completes its current pass.

### 6. Risks & Mitigation
* **Ghosting:** Low-frequency `BACKGROUND` updates can lead to trails. **Mitigation:** Aggressive temporal denoising within the `DYNAMIC` tier.
* **Memory Bandwidth:** Staggering updates prevents memory bus saturation by spreading the GI load across multiple frames.

### 7. Future Scalability
This architecture allows for dynamic quality scaling. On lower-end hardware, the `BACKGROUND` update interval can be dynamically increased (e.g., from 4 frames to 16 frames), allowing the system to maintain "LIVE" performance while gracefully degrading indirect lighting fidelity.

---
*Status: Proposal / Draft*
*Contributor: [Your Name/Handle]*
