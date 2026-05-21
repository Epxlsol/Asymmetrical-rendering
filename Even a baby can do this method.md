# Easy method: Asynchronous GI Rendering

A "Lighting Traffic Controller" for real-time engines that makes high-quality Global Illumination accessible on low-end hardware by abandoning the "do everything every frame" mindset.

## The Problem
Modern real-time engines are addicted to "Brute Force"—trying to calculate every bounce of light, every single frame. This kills performance and wastes GPU cycles on lighting that the human eye doesn't even need to be frame-accurate.

## The "Even A Baby Can Do It" Method
Instead of recomputing the entire world's lighting at once, we use a **Hybrid Pipeline** that separates lighting into two tiers:

### 1. The LIVE Pipeline (Direct & Immediate)
- Handles dynamic lights, shadows, and primary light interactions near the player.
- **Priority:** High-frequency, frame-accurate.
- **Result:** Immediate responsiveness to player actions.

### 2. The BACKGROUND Tier (Indirect & Staggered)
- Handles global illumination, indirect bounces, and distant environmental lighting.
- **Priority:** Low-frequency, asynchronous.
- **Method:** We update "sleepy" light probes only when necessary. By staggering these updates and using temporal blending (hysteresis), we distribute the heavy math over many frames.



## Why it works
By treating lighting as a **Traffic Controller** rather than a **Static Bake**, we achieve:
* **Massive Performance Gains:** Dramatically lower GPU overhead.
* **Cinematic Softness:** Temporal blending eliminates "flicker" and creates naturally soft, unified lighting.
* **Hardware Inclusivity:** Bring high-end visual fidelity to modest hardware without sacrificing the "live" feel of your game.

## Concept vs. Implementation
This project is an architectural framework. It is not about writing thousands of lines of code; it is about choosing **where** and **when** the math happens. 

---

*This project will become a Proof of Concept for asynchronous rendering architecture. Contributions, critique, and architectural discussions are welcome.*
