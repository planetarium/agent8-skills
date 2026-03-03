---
name: vibe-starter-3d-environment-stage
description: Core document on creating procedurally generated structures (mazes, dungeons, arenas) using `vibe-starter-3d-environment` in a 3D project.

This document covers:
- Generating wall-based structures like mazes and dungeons.
- Controlling maze complexity, path density, and wall height.
- Customizing wall appearance: shape (cube, cylinder) and material (stone, metal).
- [Critically Important] This component only creates 'walls' and MUST be used with a Terrain component to provide a floor.
- Aligning generated walls to terrain contours.

Reference this document when a user wants to design or customize level structures like mazes or dungeons.

Important: You must install the package using `bun add vibe-starter-3d-environment` before you can use it.
---

## Instructions

1. Ensure the package is installed. If not, run: `bun add vibe-starter-3d-environment`
2. Read the documentation file at `node_modules/vibe-starter-3d-environment/docs/read_environment_stage.md` and follow its instructions.
