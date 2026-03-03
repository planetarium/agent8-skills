---
name: vibe-starter-3d-environment-terrain
description: |
  Core document on procedurally generating terrain (ground, land) and controlling its appearance and material using `vibe-starter-3d-environment` in a 3D project.
  
  This document covers:
  - Creating the basic ground/terrain for the world.
  - Adjusting terrain shape (rugged, mountainous, flat).
  - Blending textures (grass, dirt, stone) based on height or slope (Texture Splatting).
  - Setting up and utilizing TerrainData for all terrain information.
  - Obtaining terrain height to accurately place objects or the player.
  - [Critically Important] The essential initialization method for placing the player safely on terrain at game start, preventing fall-through.
  
  Reference this document when a user wants to generate terrain, customize its shape/texture, work with TerrainData, or set the player's starting position.
  
  [Important] You must install the package using `bun add vibe-starter-3d-environment` before you can use it.
---

## Instructions

1. Ensure the package is installed. If not, run: `bun add vibe-starter-3d-environment`
2. Read the documentation file at `node_modules/vibe-starter-3d-environment/docs/read_environment_terrain.md` and follow its instructions.
