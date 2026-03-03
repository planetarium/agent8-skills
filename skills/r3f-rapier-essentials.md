---
name: r3f-rapier-essentials
description: Core document on essential principles, patterns, and performance optimization for using React Three Fiber (R3F) with the Rapier physics engine in a 3D project.

This document covers:
- Physics-based movement (character control, jumps, continuous forces like thrust).
- Sensor zones for non-physical contact detection (item collection, zone entry).
- Kinematic movement for scripted objects (moving platforms, elevators, rotating obstacles).
- Essential R3F architecture rules (e.g., all 3D components must be inside Canvas).
- Performance optimization and collider type selection guidance.

Reference this document when a user asks about physics-based movement, collision detection, interaction patterns with Rapier, or R3F core architecture.
---

# React Three Fiber & Rapier Essentials

A comprehensive guide to building 3D games and interactive applications with React Three Fiber and Rapier physics engine. Includes practical patterns and examples for physics-based movement, collision detection, and performance optimization.

## Table of Contents

1. [Essential R3F Guidelines](#essential-r3f-guidelines)
2. [Physics Usage](#physics-usage) - 3 Core Patterns with Technical Constraints

---

## Essential R3F Guidelines

### Core Rules

1. **Canvas Boundaries**

   - All R3F-related components MUST be inside `<Canvas>`:

     - All 3D components (`<mesh>`, `<boxGeometry>`, `<meshStandardMaterial>`)
     - All R3F components (`<RigidBody>`, `<Physics>`)
     - All Three.js related elements

   - NEVER place R3F components outside `<Canvas>`

   - NEVER place regular HTML elements inside `<Canvas>`:
     - No `<div>`, `<button>`, `<p>`, etc.
     - No regular React components
     - No elements with regular CSS styling

2. **UI Implementation**

   - Avoid using `drei/Html` component when possible
   - Place UI elements outside `<Canvas>` using React portals or separate components
   - Use Three.js materials and geometries for in-scene UI elements

3. **Performance Best Practices**

   - Use `useFrame` for animations and continuous updates
   - Implement object pooling for frequently created/destroyed objects
   - Use `useMemo` for complex calculations
   - Avoid unnecessary re-renders with proper component structure

4. **State Management**

   - Use refs for frequently updated values
   - Keep state updates outside animation loops
   - Use `useCallback` for event handlers

5. **Resource Management**
   - Load assets using `useLoader`
   - Implement proper cleanup in `useEffect`
   - Dispose of geometries and materials when no longer needed

## Physics Usage

### Basic Setup

```tsx
<Canvas>
  <Physics gravity={[0, -9.81, 0]}>{/* Your physics objects here */}</Physics>
</Canvas>
```

### Usage 1: Movement Control

#### Basics

**Concept:** Complete movement control using three primary methods

**Core APIs:**

- `setLinvel(velocity, wakeUp)` - Set velocity instantly (general movement)
- `applyImpulse(impulse, wakeUp)` - Apply instant force (jump, explosion)
- `addForce(force, wakeUp)` - Apply continuous force (thrusters, wind)

**Movement Methods Comparison:**

| Method           | When to Use          | Characteristics                   | Examples                  |
| ---------------- | -------------------- | --------------------------------- | ------------------------- |
| `setLinvel()`    | **General Movement** | Instant velocity, ignores physics | Character movement        |
| `applyImpulse()` | **Instant Force**    | Physically natural acceleration   | Jump, explosion, shooting |
| `addForce()`     | **Continuous Force** | Applied every frame               | Thrusters, wind, magnets  |

**Key Principles:**

- Use `setLinvel()` for general character movement
- Use `applyImpulse()` for physics-based effects
- Use `addForce()` in useFrame for continuous forces
- Control rotation with `lockRotations` or `enabledRotations`

#### Examples

**Example 1: Basic Character Movement (setLinvel)**

```tsx
function CharacterMovement() {
  const playerRef = useRef<RapierRigidBody>(null);
  const speed = 5;

  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (!playerRef.current) return;

      const velocity = { x: 0, y: 0, z: 0 };

      // WASD movement - instant velocity change
      if (e.key === 'w') velocity.z = -speed;
      if (e.key === 's') velocity.z = speed;
      if (e.key === 'a') velocity.x = -speed;
      if (e.key === 'd') velocity.x = speed;

      // Preserve current Y velocity (for gravity)
      const currentVel = playerRef.current.linvel();
      velocity.y = currentVel.y;

      playerRef.current.setLinvel(velocity, true);
    };

    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [speed]);

  return (
    <RigidBody
      ref={playerRef}
      colliders="ball"
      position={[0, 1, 0]}
      lockRotations // Prevent rolling for stable movement
    >
      <mesh>
        <sphereGeometry args={[0.5]} />
        <meshStandardMaterial color="blue" />
      </mesh>
    </RigidBody>
  );
}
```

**Example 2: Physics-based Jump (applyImpulse)**

```tsx
function JumpingCharacter() {
  const playerRef = useRef<RapierRigidBody>(null);
  const canJump = useRef(true);

  const jump = useCallback(() => {
    if (playerRef.current && canJump.current) {
      // Physically natural jump
      playerRef.current.applyImpulse({ x: 0, y: 8, z: 0 }, true);
      canJump.current = false;
    }
  }, []);

  return (
    <>
      <RigidBody
        ref={playerRef}
        colliders="ball"
        position={[0, 1, 0]}
        enabledRotations={[false, true, false]}
        onCollisionEnter={() => (canJump.current = true)}
      >
        <mesh>
          <sphereGeometry args={[0.5]} />
          <meshStandardMaterial color="green" />
        </mesh>
      </RigidBody>

      <button onClick={jump}>Jump</button>
    </>
  );
}
```

**Example 3: Continuous Force (addForce)**

```tsx
function ThrusterSystem() {
  const vehicleRef = useRef<RapierRigidBody>(null);
  const [thrusterActive, setThrusterActive] = useState(false);
  const [windStrength, setWindStrength] = useState(5);

  useFrame(() => {
    if (vehicleRef.current) {
      // Thruster force
      if (thrusterActive) {
        vehicleRef.current.addForce({ x: 0, y: 15, z: 0 }, true);
      }

      // Environmental wind force
      vehicleRef.current.addForce({ x: windStrength, y: 0, z: 0 }, true);
    }
  });

  return (
    <>
      <RigidBody ref={vehicleRef} colliders="cuboid" position={[0, 1, 0]}>
        <mesh>
          <boxGeometry args={[1, 0.5, 2]} />
          <meshStandardMaterial color={thrusterActive ? 'orange' : 'gray'} />
        </mesh>
      </RigidBody>

      <Html position={[0, 4, 0]}>
        <div style={{ color: 'white' }}>
          <button onMouseDown={() => setThrusterActive(true)} onMouseUp={() => setThrusterActive(false)}>
            🚀 Thrust
          </button>
          <br />
          <label>Wind: {windStrength}</label>
          <input type="range" min="0" max="20" value={windStrength} onChange={(e) => setWindStrength(Number(e.target.value))} />
        </div>
      </Html>
    </>
  );
}
```

#### Technical Constraints

**RigidBody Performance Order:**

- `fixed` > `kinematicPosition` > `dynamic` (fastest to slowest)

**Collider Types and Performance:**

| Type      | Description               | Performance | Recommended Use Cases                |
| --------- | ------------------------- | ----------- | ------------------------------------ |
| `ball`    | Sphere collider           | Fastest     | Characters, rolling objects, coins   |
| `cuboid`  | Box collider              | Fast        | Platforms, walls, boxes, cylinders\* |
| `hull`    | Convex hull (auto-fit)    | Medium      | Irregular but convex shapes          |
| `trimesh` | Triangle mesh (exact fit) | Slowest     | Static, complex, non-convex objects  |

\*Note: `cylinder` collider is not supported. For cylinder-shaped objects, use `cuboid` or a custom collider.

**Critical Rules:**

- Never use `trimesh` colliders for moving objects
- Never read and write physics state in same frame
- Use object pooling for >10 similar objects
- Use InstancedRigidBodies for >50 similar objects
- Use refs instead of state for physics objects

### Usage 2: Sensor Detection

#### Basics

**Concept:** Create trigger zones that detect without physical collision

**Core APIs:**

- `sensor={true}` - Creates non-physical detection area
- `onIntersectionEnter={callback}` - When object enters
- `onIntersectionExit={callback}` - When object exits

**When to Use:**

- Item collection systems
- Trigger zones (doors, switches, checkpoints)
- Game event locations (finish lines, spawn points)
- Area-based game mechanics

**Pattern:**

```tsx
<RigidBody sensor onIntersectionEnter={handleDetection}>
  {/* Detection area */}
</RigidBody>
```

#### Examples

**Example 1: Coin Collection System**

```tsx
function CoinCollectionSystem() {
  const [score, setScore] = useState(0);
  const [coins, setCoins] = useState([
    { id: 1, position: [2, 1, 0], collected: false },
    { id: 2, position: [-2, 1, 0], collected: false },
    { id: 3, position: [0, 1, -3], collected: false },
  ]);

  const collectCoin = useCallback((coinId: number) => {
    setCoins((prev) => prev.map((coin) => (coin.id === coinId ? { ...coin, collected: true } : coin)));
    setScore((prev) => prev + 10);
  }, []);

  return (
    <>
      {/* Player */}
      <RigidBody colliders="ball" position={[0, 1, 0]}>
        <mesh>
          <sphereGeometry args={[0.5]} />
          <meshStandardMaterial color="blue" />
        </mesh>
      </RigidBody>

      {/* Coins */}
      {coins.map(
        (coin) =>
          !coin.collected && (
            <RigidBody key={coin.id} position={coin.position} sensor colliders="cuboid" onIntersectionEnter={() => collectCoin(coin.id)}>
              <mesh>
                <cylinderGeometry args={[0.3, 0.3, 0.1]} />
                <meshStandardMaterial color="gold" />
              </mesh>
            </RigidBody>
          ),
      )}

      {/* UI */}
      <Html position={[0, 3, 0]}>
        <div style={{ color: 'white', fontSize: '24px' }}>Score: {score}</div>
      </Html>
    </>
  );
}
```

**Example 2: Finish Line with Timer**

```tsx
function RaceFinishLine() {
  const [gameState, setGameState] = useState<'playing' | 'finished'>('playing');
  const [completionTime, setCompletionTime] = useState<number>(0);
  const startTime = useRef(Date.now());

  const handleFinish = useCallback(() => {
    if (gameState === 'finished') return;

    const endTime = Date.now();
    const duration = (endTime - startTime.current) / 1000;
    setCompletionTime(duration);
    setGameState('finished');
  }, [gameState]);

  const resetGame = useCallback(() => {
    setGameState('playing');
    setCompletionTime(0);
    startTime.current = Date.now();
  }, []);

  return (
    <>
      {/* Player */}
      <RigidBody colliders="ball" position={[0, 1, 0]}>
        <mesh>
          <sphereGeometry args={[0.5]} />
          <meshStandardMaterial color="blue" />
        </mesh>
      </RigidBody>

      {/* Finish Line Trigger */}
      <RigidBody type="fixed" sensor position={[0, 1, -10]} onIntersectionEnter={handleFinish}>
        <mesh>
          <boxGeometry args={[5, 3, 0.5]} />
          <meshStandardMaterial color="gold" transparent opacity={0.5} />
        </mesh>
      </RigidBody>

      {/* UI */}
      {gameState === 'finished' && (
        <Html position={[0, 3, 0]}>
          <div style={{ color: 'white', textAlign: 'center' }}>
            <h2>🏁 Finished!</h2>
            <p>Time: {completionTime.toFixed(2)}s</p>
            <button onClick={resetGame}>Play Again</button>
          </div>
        </Html>
      )}
    </>
  );
}
```

#### Technical Constraints

**Sensor Requirements:**

- Must use `sensor={true}` for trigger zones
- Sensors don't physically collide but detect intersections
- Use `onIntersectionEnter/Exit` for detection events

### Usage 3: Kinematic Movement

#### Basics

**Concept:** Script-controlled movement that ignores physics laws

**Core APIs:**

- `type="kinematicPosition"` - Position-based control
- `setNextKinematicTranslation()` - Set next position
- `setNextKinematicRotation()` - Set next rotation

**When to Use:**

- Rotating obstacles and platforms
- Elevators, moving platforms
- Precise AI character movement
- Animated environment objects

**Performance Tips:**

- Reuse Quaternion/Euler objects to prevent GC
- Use useRef for accumulating values

#### Examples

**Example 1: Rotating Obstacle**

```tsx
function RotatingObstacle() {
  const obstacleRef = useRef<RapierRigidBody>(null);

  // Performance: Reuse objects to avoid garbage collection
  const quaternionRef = useRef(new THREE.Quaternion());
  const eulerRef = useRef(new THREE.Euler(0, 0, 0));
  const angleRef = useRef(0);
  const rotationSpeed = 1.5;

  useFrame((_, delta) => {
    if (obstacleRef.current) {
      angleRef.current += delta * rotationSpeed;

      eulerRef.current.set(0, angleRef.current, 0);
      quaternionRef.current.setFromEuler(eulerRef.current);

      obstacleRef.current.setNextKinematicRotation(quaternionRef.current);
    }
  });

  return (
    <>
      {/* Kinematic Rotating Obstacle */}
      <RigidBody ref={obstacleRef} type="kinematicPosition" colliders="cuboid" position={[0, 1.5, 0]}>
        <mesh>
          <boxGeometry args={[8, 0.4, 0.4]} />
          <meshStandardMaterial color="red" />
        </mesh>
      </RigidBody>

      {/* Ground */}
      <RigidBody type="fixed" colliders="cuboid">
        <mesh scale={[10, 0.2, 10]}>
          <boxGeometry args={[1, 1, 1]} />
          <meshStandardMaterial color="gray" />
        </mesh>
      </RigidBody>

      {/* Dynamic Test Objects */}
      {Array.from({ length: 3 }, (_, i) => (
        <RigidBody key={i} colliders="ball" position={[i * 2 - 2, 3, 0]}>
          <mesh>
            <sphereGeometry args={[0.3]} />
            <meshStandardMaterial color="blue" />
          </mesh>
        </RigidBody>
      ))}
    </>
  );
}
```

**Example 2: Elevator System**

```tsx
function ElevatorSystem() {
  const elevatorRef = useRef<RapierRigidBody>(null);
  const time = useRef(0);

  useFrame((state, delta) => {
    if (!elevatorRef.current) return;

    // Update time
    time.current += delta;

    // Use sine function to generate value between -1 and 1
    // Multiply by 2 to get range -2 to 2, then add 3 to get range 1 to 5
    const y = Math.sin(time.current) * 2 + 3;

    // Keep current x, z position while updating y position
    const currentPos = elevatorRef.current.translation();
    elevatorRef.current.setNextKinematicTranslation({
      x: currentPos.x,
      y: y,
      z: currentPos.z,
    });
  });

  return (
    <>
      {/* Elevator Platform */}
      <RigidBody ref={elevatorRef} type="kinematicPosition" colliders="cuboid" position={[0, 1, 0]}>
        <mesh>
          <boxGeometry args={[3, 0.2, 3]} />
          <meshStandardMaterial color="brown" />
        </mesh>
      </RigidBody>
    </>
  );
}
```

#### Technical Constraints

**Kinematic Movement Rules:**

- Use `type="kinematicPosition"` for precise control
- Reuse Quaternion/Euler objects to prevent garbage collection
- Use `setNextKinematicTranslation/Rotation()` for movement
- Kinematic objects ignore physics forces but can push dynamic objects
