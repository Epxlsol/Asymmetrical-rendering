# Asymmetrical-rendering

Theres 2 Pipelines:


Background: Which isnt as updated with heavy lighting and whatever else are calculated once then cached in VRAM and skipped for multiple frames, while a transition like dithering or something is used to merge it to a Live pipeline (or Live can be drawn ontop).

Live Pipline: Physics and inputs react like normal and you can move interactive objects and things such as signs, NPCs and the sky into the live pipeline if you want them to move (Or add another pipeline for them at a lower than live rate but higher than Background). By stopping the GPU and CPU from recalculating the universe every millisecond, you can get from 20 FPS to hundreds. And the multiple pipelines let you experiment aton.
