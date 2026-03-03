---
name: input-system-guide
description: Core document on implementing unified input management for mobile and desktop in React Three Fiber (R3F) using InputController and Zustand stores.

This document covers:
- Centralized input handling for keyboard, joystick, and touch via InputController.
- Movement and player action store architecture using Zustand.
- Custom input processing instead of R3F's `<KeyboardControls>`.
- Consistent state management across input methods with analog support.
- Custom key mapping and cross-platform performance optimization.

Reference this document when a user needs input handling, player controls, or mobile touch controls.

Important: You must install the required dependencies using `bun add nipplejs` before you can use this system.
---

# Mobile Input System Guide

## Overview

This guide explains how to implement a unified input management system for both mobile and desktop devices in React Three Fiber (R3F) projects.

## Key Features

- **Unified Input Processing**: Centralized input management through InputController component
- **Multi-Platform Support**: Keyboard, joystick, and touch inputs managed via Zustand stores
- **Custom Input Processing**: Custom input handling instead of React Three Fiber's `<KeyboardControls>`
- **Separated Architecture**: Movement controls and player actions separated with dedicated store architecture
- **Consistent State Management**: Consistent state management across different input methods
- **Custom Key Mapping**: Enhanced control flexibility with custom key mapping and analog input support

## Project Structure

```
input-system/
├── package.json                     # Dependencies
├── src/
│   ├── stores/
│   │   ├── inputStore.ts           # Input state management store
│   │   └── playerActionStore.ts    # Player action state management store
│   └── components/
│       └── ui/
│           ├── InputController.tsx  # Main input controller
│           └── GameSceneUI.tsx     # Game scene UI component
```

## Dependencies

```json
{
  "dependencies": {
    "nipplejs": "0.10.2"
  }
}
```

**Installation**: If these dependencies are not already included in your project's package.json, install them using:

```bash
bun add nipplejs@0.10.2
```

## 1. Input State Management Store (inputStore.ts)

> **Note**: You have two options for implementing the input store:
>
> 1. **Custom Implementation**: Use the complete implementation provided below
> 2. **vibe-3d-starter Integration**: If you're using vibe-3d-starter, utilize the input store provided by the framework instead of creating your own

### Option 1: Custom Implementation

Use this implementation if you're building your own React Three Fiber project from scratch.

```typescript
import { create } from "zustand";

/* Interface for movement input with analog support */
export interface MovementInput {
  // Analog input: normalized direction vector (-1 to 1) with magnitude representing intensity
  direction: { x: number; y: number };
  intensity: number; // 0.0 to 1.0 (0: no input, 1: maximum input)
}

/* Simplified input interface - only essential inputs */
export interface InputStates {
  // Movement (replaces forward, backward, leftward, rightward)
  movement: MovementInput;

  // Action inputs
  jump: boolean;
  run: boolean;

  // Meta information
  isAnalogInput: boolean; // true if using analog input (joystick), false for digital (keyboard)
}

/* Input source types */
export type InputSource = "keyboard" | "touch" | "joystick" | "gamepad";

export interface InputState extends InputStates {
  // Current active input source
  activeInputSource: InputSource | null;

  // Actions for movement input
  setMovementInput: (
    direction: { x: number; y: number },
    intensity: number,
    source?: InputSource
  ) => void;

  // Actions for non-movement inputs
  setActionInput: (
    action: "jump" | "run",
    pressed: boolean,
    source?: InputSource
  ) => void;

  // Meta actions
  setAnalogInputMode: (isAnalog: boolean) => void;
  setActiveInputSource: (source: InputSource | null) => void;
  resetAllInputs: () => void;
}

export const initialInputStates: InputStates = {
  // Movement input
  movement: {
    direction: { x: 0, y: 0 },
    intensity: 0,
  },

  // Action inputs
  jump: false,
  run: false,

  // Meta
  isAnalogInput: false,
};

export const useInputStore = create<InputState>((set, get) => ({
  ...initialInputStates,
  activeInputSource: null,

  setMovementInput: (
    direction: { x: number; y: number },
    intensity: number,
    source?: InputSource
  ) => {
    set((state) => {
      const updates: Partial<InputState> = {
        movement: {
          direction: { ...direction },
          intensity: Math.max(0, Math.min(1, intensity)), // Clamp to 0-1 range
        },
        isAnalogInput: source === "joystick" || source === "gamepad",
      };

      // Update active input source
      if (source && source !== state.activeInputSource) {
        updates.activeInputSource = source;
      }

      return updates;
    });
  },

  setActionInput: (
    action: "jump" | "run",
    pressed: boolean,
    source?: InputSource
  ) => {
    set((state) => {
      const updates: Partial<InputState> = {
        [action]: pressed,
      };

      // Update active input source
      if (source && source !== state.activeInputSource) {
        updates.activeInputSource = source;
      }

      return updates;
    });
  },

  setAnalogInputMode: (isAnalog: boolean) =>
    set(() => ({ isAnalogInput: isAnalog })),

  setActiveInputSource: (source: InputSource | null) =>
    set(() => ({ activeInputSource: source })),

  resetAllInputs: () =>
    set(() => ({
      ...initialInputStates,
      activeInputSource: null,
    })),
}));
```

### Option 2: vibe-3d-starter Integration

If you're **already using vibe-3d-starter** in your project, the framework provides its own input store that you should use instead of creating your own. The vibe-3d-starter input store offers:

- **Built-in Input Management**: Pre-configured input handling for common game scenarios
- **Framework Integration**: Seamless integration with other vibe-3d-starter components
- **Consistent API**: Same interface as the custom implementation above
- **Optimized Performance**: Framework-level optimizations for input handling

When your project already uses vibe-3d-starter, the input store should already be available:

```typescript
// Use the existing input store from vibe-3d-starter
import { useInputStore } from "vibe-3d-starter";

// The API remains the same
function MyComponent() {
  const { movement, jump, run } = useInputStore();
  // ... rest of your component logic
}
```

**Important**: Do not create your own inputStore.ts file when already using vibe-3d-starter. Always use the framework's provided store to ensure compatibility and receive framework updates.

## 2. Player Action State Management Store (playerActionStore.ts)

```typescript
import { create } from "zustand";
import { subscribeWithSelector } from "zustand/middleware";

interface PlayerActionState {
  punch: boolean;
  kick: boolean;
  meleeAttack: boolean;
  cast: boolean;
}

interface PlayerActionStore extends PlayerActionState {
  setPlayerAction: (action: string, pressed: boolean) => void;
  getPlayerAction: (action: string) => boolean;
  resetAllPlayerActions: () => void;
}

const initialPlayerActionState: PlayerActionState = {
  punch: false,
  kick: false,
  meleeAttack: false,
  cast: false,
};

export const usePlayerActionStore = create<PlayerActionStore>()(
  subscribeWithSelector((set, get) => ({
    ...initialPlayerActionState,

    setPlayerAction: (action: string, pressed: boolean) => {
      set((state) => ({
        ...state,
        [action]: pressed,
      }));
    },

    getPlayerAction: (action: string) => {
      return get()[action as keyof PlayerActionState];
    },

    resetAllPlayerActions: () => {
      set(() => ({ ...initialPlayerActionState }));
    },
  }))
);
```

## 3. Main Input Controller (InputController.tsx)

```typescript
import React, { useCallback, useEffect, useRef, useState } from "react";
import * as THREE from "three";
import { useInputStore } from "../../stores/inputStore";
import { usePlayerActionStore } from "../../stores/playerActionStore";
import nipplejs from "nipplejs";

export const IS_MOBILE =
  /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
    navigator.userAgent
  ) ||
  "ontouchstart" in window ||
  navigator.maxTouchPoints > 0;

/**
 * Key mapping configuration - simple Record<string, string[]>
 */
type KeyMapping = Record<string, string[]>; // action: [key1, key2, ...]

/**
 * Controller key mapping
 */
const CONTROL_KEY_MAPPING: KeyMapping = {
  forward: ["KeyW", "ArrowUp"],
  backward: ["KeyS", "ArrowDown"],
  leftward: ["KeyA", "ArrowLeft"],
  rightward: ["KeyD", "ArrowRight"],
  jump: ["Space"],
  run: ["ShiftLeft", "ShiftRight"],
};

/**
 * Player action key mapping
 */
const ACTION_KEY_MAPPING: KeyMapping = {
  punch: ["KeyF", "Mouse0"],
  kick: ["KeyG", "Mouse2"],
  meleeAttack: ["KeyQ", "KeyC"],
  cast: ["KeyE", "Mouse1"],
};

// Movement speed constants
const MOVEMENT_SPEED_WALK = 0.6;
const MOVEMENT_SPEED_RUN_BOOST = 0.4;
const MOVEMENT_SPEED_MAX = 1.0;
const JOYSTICK_RANGE_MULTIPLIER = 2.0; // Converts joystick range (0~0.5) to full range (0~1.0)

interface InputControllerProps {
  disabled?: boolean;
  disableKeyboard?: boolean;
  disableJoystick?: boolean;
}

export const InputController: React.FC<InputControllerProps> = ({
  disabled = false,
  disableKeyboard = false,
  disableJoystick = false,
}) => {
  // Store actions to controller
  const {
    setMovementInput,
    setActionInput,
    resetAllInputs,
    setActiveInputSource,
  } = useInputStore();
  const { setPlayerAction, resetAllPlayerActions } = usePlayerActionStore();

  // Button states
  const [isJumpPressed, setIsJumpPressed] = useState(false);
  const [isAttackPressed, setIsAttackPressed] = useState(false);

  // Keyboard state tracking
  const keyboardStateRef = useRef({
    forward: false,
    backward: false,
    leftward: false,
    rightward: false,
    run: false,
  });

  // Helper function to calculate movement from keyboard state
  const calculateKeyboardMovement = useCallback(() => {
    const state = keyboardStateRef.current;

    // Calculate direction
    const x = (state.leftward ? 1 : 0) + (state.rightward ? -1 : 0);
    const y = (state.forward ? 1 : 0) + (state.backward ? -1 : 0);

    // Use Three.js Vector2 for efficient normalization
    const direction = new THREE.Vector2(x, y);
    const magnitude = direction.length();

    // Normalize diagonal movement
    if (magnitude > 0) {
      direction.normalize();
    }

    // Calculate intensity: base speed + run boost
    const baseIntensity = magnitude > 0 ? MOVEMENT_SPEED_WALK : 0; // Base walking speed
    const runBoost = state.run ? MOVEMENT_SPEED_RUN_BOOST : 0; // Additional speed when running
    const intensity = Math.min(baseIntensity + runBoost, MOVEMENT_SPEED_MAX);

    return {
      direction: { x: direction.x, y: direction.y },
      intensity,
    };
  }, []);

  // Joystick input handling with analog support
  useEffect(() => {
    if (disabled || disableJoystick || !IS_MOBILE) return;

    // Create div element for left side area of screen
    const joystickZone = document.createElement("div");
    joystickZone.style.position = "fixed";
    joystickZone.style.left = "0";
    joystickZone.style.top = "0";
    joystickZone.style.width = "50%"; // Left 50% of screen
    joystickZone.style.height = "100%";
    joystickZone.style.zIndex = "1000";
    joystickZone.style.pointerEvents = "auto";
    joystickZone.style.backgroundColor = "transparent";
    // Disable long touch events
    joystickZone.style.touchAction = "none";
    joystickZone.style.userSelect = "none";
    joystickZone.style.setProperty("-webkit-user-select", "none");
    joystickZone.style.setProperty("-webkit-touch-callout", "none");
    document.body.appendChild(joystickZone);

    const options: nipplejs.JoystickManagerOptions = {
      zone: joystickZone,
      color: "white",
      mode: "dynamic",
      shape: "circle",
    };

    const manager = nipplejs.create(options);

    manager.on("move", (evt, data) => {
      if (disabled || disableJoystick) return; // Check disabled state
      setActiveInputSource("joystick");

      // Extract analog data from nipplejs
      const angle = data.angle?.radian || 0; // Angle in radians
      const distance = data.distance || 0; // Distance from center
      const maxDistance = data.instance.options.size || 100; // Maximum distance

      // Calculate normalized direction vector
      // nipplejs uses mathematical coordinate system (0° = right, 90° = up)
      // We need to convert to game coordinate system (0° = up, 90° = right)
      const gameAngle = angle - Math.PI / 2; // Rotate by -90 degrees
      const directionX = Math.sin(gameAngle); // Right/Left
      const directionY = Math.cos(gameAngle); // Forward/Backward

      // Calculate intensity (0.0 to 1.0) based on distance from center
      const intensity = Math.min(
        (distance / maxDistance) * JOYSTICK_RANGE_MULTIPLIER,
        MOVEMENT_SPEED_MAX
      );

      // Set analog movement input
      setMovementInput({ x: directionX, y: directionY }, intensity, "joystick");
    });

    manager.on("end", () => {
      if (disabled || disableJoystick) return; // Check disabled state
      // Reset all inputs when joystick ends
      setMovementInput({ x: 0, y: 0 }, 0, "joystick");
    });

    return () => {
      manager.destroy();
      if (joystickZone.parentNode) {
        joystickZone.parentNode.removeChild(joystickZone);
      }
    };
  }, [disabled, disableJoystick, setMovementInput, setActiveInputSource]);

  // Keyboard input handling - convert to movement immediately
  useEffect(() => {
    if (disabled || disableKeyboard) return;

    const handleKeyDown = (event: KeyboardEvent) => {
      if (disabled || disableKeyboard) return; // Additional check in handler
      let stateChanged = false;

      // Handle movement keys
      if (
        CONTROL_KEY_MAPPING["forward"]?.includes(event.code) &&
        !keyboardStateRef.current.forward
      ) {
        keyboardStateRef.current.forward = true;
        stateChanged = true;
      }
      if (
        CONTROL_KEY_MAPPING["backward"]?.includes(event.code) &&
        !keyboardStateRef.current.backward
      ) {
        keyboardStateRef.current.backward = true;
        stateChanged = true;
      }
      if (
        CONTROL_KEY_MAPPING["leftward"]?.includes(event.code) &&
        !keyboardStateRef.current.leftward
      ) {
        keyboardStateRef.current.leftward = true;
        stateChanged = true;
      }
      if (
        CONTROL_KEY_MAPPING["rightward"]?.includes(event.code) &&
        !keyboardStateRef.current.rightward
      ) {
        keyboardStateRef.current.rightward = true;
        stateChanged = true;
      }
      if (
        CONTROL_KEY_MAPPING["run"]?.includes(event.code) &&
        !keyboardStateRef.current.run
      ) {
        keyboardStateRef.current.run = true;
        stateChanged = true;
      }

      // Handle action keys
      if (CONTROL_KEY_MAPPING["jump"]?.includes(event.code)) {
        setActionInput("jump", true, "keyboard");
      }

      // Handle player action keys
      Object.keys(ACTION_KEY_MAPPING).forEach((action) => {
        if (ACTION_KEY_MAPPING[action]?.includes(event.code)) {
          setPlayerAction(action, true);
        }
      });

      // Update movement if any movement-related key changed
      if (stateChanged) {
        const movement = calculateKeyboardMovement();
        setMovementInput(movement.direction, movement.intensity, "keyboard");
      }
    };

    const handleKeyUp = (event: KeyboardEvent) => {
      if (disabled || disableKeyboard) return; // Additional check in handler
      let stateChanged = false;

      // Handle movement keys
      if (
        CONTROL_KEY_MAPPING["forward"]?.includes(event.code) &&
        keyboardStateRef.current.forward
      ) {
        keyboardStateRef.current.forward = false;
        stateChanged = true;
      }
      if (
        CONTROL_KEY_MAPPING["backward"]?.includes(event.code) &&
        keyboardStateRef.current.backward
      ) {
        keyboardStateRef.current.backward = false;
        stateChanged = true;
      }
      if (
        CONTROL_KEY_MAPPING["leftward"]?.includes(event.code) &&
        keyboardStateRef.current.leftward
      ) {
        keyboardStateRef.current.leftward = false;
        stateChanged = true;
      }
      if (
        CONTROL_KEY_MAPPING["rightward"]?.includes(event.code) &&
        keyboardStateRef.current.rightward
      ) {
        keyboardStateRef.current.rightward = false;
        stateChanged = true;
      }
      if (
        CONTROL_KEY_MAPPING["run"]?.includes(event.code) &&
        keyboardStateRef.current.run
      ) {
        keyboardStateRef.current.run = false;
        stateChanged = true;
      }

      // Handle action keys
      if (CONTROL_KEY_MAPPING["jump"]?.includes(event.code)) {
        setActionInput("jump", false, "keyboard");
      }

      // Handle player action keys
      Object.keys(ACTION_KEY_MAPPING).forEach((action) => {
        if (ACTION_KEY_MAPPING[action]?.includes(event.code)) {
          setPlayerAction(action, false);
        }
      });

      // Update movement if any movement-related key changed
      if (stateChanged) {
        const movement = calculateKeyboardMovement();
        setMovementInput(movement.direction, movement.intensity, "keyboard");
      }
    };

    const handleMouseDown = (event: MouseEvent) => {
      if (disabled || disableKeyboard) return; // Additional check in handler

      // Handle player action mouse keys
      const mouseButton = `Mouse${event.button}`;
      Object.keys(ACTION_KEY_MAPPING).forEach((action) => {
        if (ACTION_KEY_MAPPING[action]?.includes(mouseButton)) {
          setPlayerAction(action, true);
        }
      });
    };

    const handleMouseUp = (event: MouseEvent) => {
      if (disabled || disableKeyboard) return; // Additional check in handler

      // Handle player action mouse keys
      const mouseButton = `Mouse${event.button}`;
      Object.keys(ACTION_KEY_MAPPING).forEach((action) => {
        if (ACTION_KEY_MAPPING[action]?.includes(mouseButton)) {
          setPlayerAction(action, false);
        }
      });
    };

    document.addEventListener("keydown", handleKeyDown);
    document.addEventListener("keyup", handleKeyUp);
    document.addEventListener("mousedown", handleMouseDown);
    document.addEventListener("mouseup", handleMouseUp);

    return () => {
      document.removeEventListener("keydown", handleKeyDown);
      document.removeEventListener("keyup", handleKeyUp);
      document.removeEventListener("mousedown", handleMouseDown);
      document.removeEventListener("mouseup", handleMouseUp);
    };
  }, [
    disabled,
    disableKeyboard,
    setMovementInput,
    setActionInput,
    setPlayerAction,
    calculateKeyboardMovement,
  ]);

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      resetAllInputs();
      resetAllPlayerActions();
    };
  }, [resetAllInputs, resetAllPlayerActions]);

  // Reset inputs when disabled
  useEffect(() => {
    if (disabled || disableKeyboard || disableJoystick) {
      // Reset keyboard state if keyboard is disabled
      if (disabled || disableKeyboard) {
        keyboardStateRef.current = {
          forward: false,
          backward: false,
          leftward: false,
          rightward: false,
          run: false,
        };
      }
      // Reset button states
      setIsJumpPressed(false);
      setIsAttackPressed(false);
      resetAllInputs();
      resetAllPlayerActions();
    }
  }, [
    disabled,
    disableKeyboard,
    disableJoystick,
    resetAllInputs,
    resetAllPlayerActions,
  ]);

  // Button handlers
  const handleJumpStart = useCallback(
    (event: React.MouseEvent | React.TouchEvent) => {
      if (disabled) return;
      event.preventDefault();
      event.stopPropagation(); // Prevent event propagation
      setIsJumpPressed(true);
      setActionInput("jump", true, "touch");
    },
    [disabled, setActionInput]
  );

  const handleJumpEnd = useCallback(
    (event: React.MouseEvent | React.TouchEvent) => {
      if (disabled) return;
      event.preventDefault();
      event.stopPropagation(); // Prevent event propagation
      setIsJumpPressed(false);
      setActionInput("jump", false, "touch");
    },
    [disabled, setActionInput]
  );

  const handleAttackStart = useCallback(
    (event: React.MouseEvent | React.TouchEvent) => {
      if (disabled) return;
      event.preventDefault();
      event.stopPropagation();
      setIsAttackPressed(true);
      // NOTE: change to other action if needed
      setPlayerAction("punch", true);
    },
    [disabled, setPlayerAction]
  );

  const handleAttackEnd = useCallback(
    (event: React.MouseEvent | React.TouchEvent) => {
      if (disabled) return;
      event.preventDefault();
      event.stopPropagation();
      setIsAttackPressed(false);
      // NOTE: change to other action if needed
      setPlayerAction("punch", false);
    },
    [disabled, setPlayerAction]
  );

  // Don't render action buttons if joystick is disabled
  if (disableJoystick) {
    return null;
  }

  // Render action buttons
  return (
    <div className="fixed bottom-8 right-8 z-[1001]">
      {/* Attack Button */}
      <div
        className={`w-20 h-20 rounded-full bg-white/30 
                   flex items-center justify-center cursor-pointer select-none touch-none
                   ${
                     isAttackPressed ? "scale-90" : "scale-100"
                   } transition-transform`}
        onMouseDown={handleAttackStart}
        onMouseUp={handleAttackEnd}
        onMouseLeave={handleAttackEnd}
        onTouchStart={handleAttackStart}
        onTouchEnd={handleAttackEnd}
      >
        <span className="text-white text-xs font-bold">ATTACK</span>
      </div>

      {/* Jump Button */}
      <div
        className={`absolute bottom-0 -left-12 -top-12 w-14 h-14 rounded-full bg-white/30 
                   flex items-center justify-center cursor-pointer select-none touch-none
                   ${
                     isJumpPressed ? "scale-90" : "scale-100"
                   } transition-transform`}
        onMouseDown={handleJumpStart}
        onMouseUp={handleJumpEnd}
        onMouseLeave={handleJumpEnd}
        onTouchStart={handleJumpStart}
        onTouchEnd={handleJumpEnd}
      >
        <span className="text-white text-xs font-bold">JUMP</span>
      </div>
    </div>
  );
};
```

## 4. Game Scene UI Component (GameSceneUI.tsx)

```typescript
import { InputController } from "./InputController";

/**
 * Game Scene UI Component
 *
 * This component manages UI overlays for the game scene.
 */
const GameSceneUI = () => {
  // TODO: set true when the game is ready
  const isReady = true;
  return (
    <>
      {/* Input Controller - Global input management (keyboard, touch) */}
      <InputController disableKeyboard={false} disabled={isReady} />
    </>
  );
};

export default GameSceneUI;
```

## Usage

### 1. Basic Setup

```typescript
import { GameSceneUI } from "./components/ui/GameSceneUI";

// Add to your game scene
function App() {
  return (
    <div>
      {/* 3D Scene Component */}
      <Canvas>{/* 3D Elements */}</Canvas>

      {/* Input System UI */}
      <GameSceneUI />
    </div>
  );
}
```

### 2. Using Input States

```typescript
import { useInputStore } from "./stores/inputStore";
import { usePlayerActionStore } from "./stores/playerActionStore";

function Player() {
  const { movement, jump, run } = useInputStore();
  const { punch, kick } = usePlayerActionStore();

  // Handle movement
  useEffect(() => {
    if (movement.intensity > 0) {
      // Player movement logic
      const speed = movement.intensity * (run ? 1.5 : 1.0);
      // Use movement.direction.x, movement.direction.y
    }
  }, [movement, run]);

  // Handle actions
  useEffect(() => {
    if (punch) {
      // Execute punch action
    }
  }, [punch]);
}
```

### 3. Custom Key Mapping

```typescript
// Modify CONTROL_KEY_MAPPING
const CONTROL_KEY_MAPPING: KeyMapping = {
  forward: ["KeyW", "ArrowUp"],
  backward: ["KeyS", "ArrowDown"],
  leftward: ["KeyA", "ArrowLeft"],
  rightward: ["KeyD", "ArrowRight"],
  jump: ["Space"],
  run: ["ShiftLeft", "ShiftRight"],
};

// Modify ACTION_KEY_MAPPING
const ACTION_KEY_MAPPING: KeyMapping = {
  punch: ["KeyF", "Mouse0"],
  kick: ["KeyG", "Mouse2"],
  meleeAttack: ["KeyQ", "KeyC"],
  cast: ["KeyE", "Mouse1"],
};
```

## Key Features

### 1. Analog Input Support

- Provides both direction and intensity information from joystick
- Converts digital input to the same interface for keyboard

### 2. Mobile Optimization

- Optimized touch event handling
- Utilizes the left 50% of screen as joystick zone
- Action buttons placed in bottom-right corner

### 3. Input Source Tracking

- Detects currently active input source
- Automatic switching between keyboard, touch, and joystick

### 4. Performance Optimization

- Efficient vector calculations using Three.js Vector2
- Minimized re-renders with useCallback
- Proper cleanup of unnecessary event listeners

## Extensibility

### 1. Adding New Actions

```typescript
// Add new action to playerActionStore.ts
interface PlayerActionState {
  punch: boolean;
  kick: boolean;
  meleeAttack: boolean;
  cast: boolean;
  // Add new action
  specialAttack: boolean;
}
```

### 2. Supporting New Input Sources

```typescript
// Add new input source to inputStore.ts
export type InputSource =
  | "keyboard"
  | "touch"
  | "joystick"
  | "gamepad"
  | "controller";
```

### 3. Custom Settings Support

```typescript
// Custom settings interface
interface InputSettings {
  sensitivity: number;
  invertY: boolean;
  customKeyMapping: KeyMapping;
}
```

This system enables consistent input experience across mobile and desktop environments.
