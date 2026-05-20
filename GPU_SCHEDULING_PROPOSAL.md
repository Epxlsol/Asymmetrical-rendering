# GPU-Driven Asynchronous GI Pipeline (v2.0)
## Technical Design: AI-Accelerated Three-Tier Architecture

### 1. Abstract
This architecture defines a hardware-accelerated Global Illumination (GI) pipeline. By leveraging GPU Compute Queues for asynchronous voxelization and dedicated Tensor/AI Cores for temporal denoising, we achieve high-fidelity dynamic lighting with minimal frame-time impact.

### 2. The Three-Tier GPU Scheduling Model
We map our task tiers directly to GPU hardware queues and specialized compute units.

| Tier | Function | Hardware Unit | Priority |
| :--- | :--- | :--- | :--- |
| **LIVE** | Direct Raster/Cluster | Graphics Queue (SMs) | Immediate |
| **DYNAMIC** | AI Denoising/Reprojection | Tensor/AI Cores | High (Async) |
| **BACKGROUND** | Voxelization/Ray Tracing | Compute Queue (Async) | Low (Staggered) |



### 3. Deep Dive: AI-Accelerated Denoising
The core innovation is moving the **denoiser** from a standard compute shader to a **Tensor-accelerated AI pass**.

* **Why AI?** Traditional denoisers use static kernels that blur edges. AI-based denoising (e.g., temporal-spatial reconstruction) understands scene features. It can "fill in the blanks" from our low-frequency `BACKGROUND` updates, effectively predicting what the light should look like between frames.
* **The Workflow:**
    1.  `BACKGROUND` tier computes raw, noisy VoxelGI radiance (Low frequency).
    2.  `DYNAMIC` tier takes the noisy buffer and historical frames.
    3.  **Tensor Inference Pass:** AI cores analyze the temporal movement and reconstruct a high-quality, stable GI texture.
    4.  The final result is injected as a uniform into the `LIVE` Forward+ pipeline.

### 4. Hardware-Level Synchronization
To ensure this doesn't bottleneck the GPU, we implement **Non-Blocking Compute Barriers**:

* **Async Compute Queue:** By offloading `BACKGROUND` and `DYNAMIC` to the Async Compute Queue, these tasks can execute *concurrently* with `LIVE` graphics commands.
* **Resource Aliasing:** We utilize memory aliasing for our 3D Voxel Textures, allowing the hardware to overwrite stale GI data buffers in memory without requiring expensive `memcpy` operations.



### 5. Throughput Management
* **Memory Bandwidth:** By staggering `BACKGROUND` updates (e.g., every 8th frame), we keep memory pressure low. 
* **AI Core Occupancy:** Since `DYNAMIC` denoising is "bursty" (it happens as soon as the `BACKGROUND` pass finishes), it doesn't need to run every frame if the motion vectors indicate low scene complexity. This dynamically scales AI core usage based on the scene's movement.

### 6. Summary of Innovation
* **Decoupled Logic:** The simulation of light (Background) is physically separated from the rendering of pixels (Live).
* **AI Reconstruction:** We stop trying to "perfectly" calculate every ray, and instead use AI to reconstruct a perfect image from sparse, asynchronous samples.
* **Future Proof:** This architecture is essentially an implementation of modern Path Tracing logic, but adapted for standard VoxelGI/real-time constraints.
