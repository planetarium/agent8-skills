---
name: environment-terrain
description: This is a core technical document on how to procedurally generate the fundamental terrain (ground, land) of a game world and control its appearance and material, when using the vibe-starter-3d-environment package in a 3D project.

This document provides guidance on the following user requirements and critically important rules:
- How to create the basic ground/terrain for the world.
- How to adjust the overall shape of the terrain, such as making it more rugged and mountainous or flat.
- How to automatically blend various textures like grass, dirt, and stone based on the terrain's height or slope to create a realistic surface (Texture Splatting).
- How to set up and utilize TerrainData, which is the core data for all terrain information.
- How to obtain the necessary terrain height information to accurately place other objects or the player on it.
- [Critically Important] The essential initialization method for placing the player safely on the terrain at game start, preventing them from falling through.

When a user intends to generate the world's terrain, customize its shape and texture, work directly with TerrainData, or set the player's starting position, this document must be referenced to confirm the correct features and the mandatory initialization procedures.

Important: You must install the package using `bun add vibe-starter-3d-environment` before you can use it.
---

## Instructions

1. Ensure the package is installed. If not, run: `bun add vibe-starter-3d-environment`
2. Read the documentation file at `node_modules/vibe-starter-3d-environment/docs/read_environment_terrain.md` and follow its instructions.
