# Asymmetrical-rendering

Theres 2 Pipelines:


Background: Which isnt as updated with heavy lighting and whatever else are calculated once then cached in VRAM and skipped for multiple frames, while a transition like dithering or something is used to merge it to a Live pipeline (or Live can be drawn ontop).

Live Pipline: Physics and inputs react like normal and you can move interactive objects and things such as signs, NPCs and the sky into the live pipeline if you want them to move (Or add another pipeline for them at a lower than live rate but higher than Background). By stopping the GPU and CPU from recalculating the universe every millisecond, you can get from 20 FPS to hundreds. And the multiple pipelines let you experiment aton.

### Pipeline Intersect & Compositing

The transition and overlap between pipelines are handled at the hardware layer with two methods:

1. Overlap: 
   The Live Pipeline evaluates immediate player geometry and proximity assets at native engine speeds. The GPU uses standard Z-buffering/depth testing to instantly overlay the Live pipeline's Color Buffer onto the cached Background buffer stored in VRAM.

2. Dither Merge:
   To stop pop in from the Background pipeline transitioning into the Live pipeline, a screen space dither shader can be used. By changing pixel sampling between the two pipeline layers using a temporal noise mask the transition becomes smoother.
