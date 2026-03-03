---
name: projectile-systems
description: Mandatory reference for all projectile, launch, or throw system tasks in React Three Fiber (R3F). Applies to any object (rocks, spells, bullets, missiles, markers) being thrown, fired, or launched.

This document covers:
- Input handling for keyboard and mouse-based firing.
- Direction and trajectory calculation for FPS, TPS, and quarter view cameras.
- Physics integration with Rapier (RigidBody, collision groups, raycasting).
- Collision detection and lifecycle management of projectiles.
- RigidBody reference readiness patterns to prevent common failures.

This document must always be consulted first. The defined patterns must be strictly followed without exception.
---

# Projectile Input Systems Reference Guide

## 문서 개요 및 구성

이 가이드는 React Three Fiber를 사용한 투사체 시스템 구현에 관한 종합적인 참조 문서입니다. 이 문서는 다음과 같은 구조로 구성되어 있습니다:

0. **🚨 RigidBody 참조 문제 해결** (필수 읽기): 투사체 발사 실패의 가장 흔한 원인과 해결법
1. **키보드 입력 처리** (1번 섹션): 키보드를 통한 투사체 발사 구현의 기본 원리
2. **마우스 입력 처리** (2번 섹션): 마우스를 통한 투사체 발사 구현 방법
3. **방향 계산 유틸리티** (3번 섹션): 투사체 발사 방향 계산을 위한 접근법
4. **투사체 이동 및 충돌** (4번 섹션): **투사체의 이동과 충돌 감지의 필수 구현 사양**
5. **통합 솔루션** (5번 섹션): 위 모든 요소를 통합한 고수준 API (`useProjectile` 훅)
6. **🎯 프로젝트 구조 분석 및 적응형 통합** (프로젝트 최적화 가이드)

**중요 참고사항 :** 
1. 이 문서의 모든 내용은 투사체 구현에 모두 필요한 중요한 내용입니다. 문서의 모든 내용을 면밀히 살펴보고 상황에 알맞게 섹션별로 조합하여 투사체 기능을 구현하세요.
2. TPS와 FPS의 발사 로직은 근본적으로 다릅니다. 
3인칭 슈팅(TPS) 게임을 개발할 때, 1인칭 슈팅(FPS) 게임의 발사 로직을 그대로 적용하는 것은 흔한 실수이며, 이는 심각한 오류를 유발합니다. 코드를 작성하기 전, 아래의 차이점을 반드시 숙지하십시오.

TPS (3인칭): 투사체는 **캐릭터(Character)**를 기준으로, 캐릭터가 바라보는 방향으로 발사되어야 합니다.
FPS (1인칭): 투사체는 **카메라(Camera)**를 기준으로, 카메라가 바라보는 방향으로 발사됩니다.

## 필수 Import 목록

```typescript
// 기본 React 및 R3F
import React, { useRef, useState, useEffect, useCallback } from 'react';
import { useFrame, Canvas } from '@react-three/fiber';
import { useKeyboardControls, KeyboardControls } from '@react-three/drei';
import * as THREE from 'three';

// Rapier 물리 엔진
import { RigidBody, RapierRigidBody, Physics, Collider, useRapier, interactionGroups } from '@react-three/rapier';
import { ActiveCollisionTypes, InteractionGroups } from '@dimforge/rapier3d-compat';

// vibe-starter-3d (사용하는 경우)
import { useControllerState } from 'vibe-starter-3d';
```

## 📋 섹션별 조합 가이드

### 기본 키보드 투사체 시스템
- **0번 섹션** (RigidBody 준비) + **1번 섹션** (키보드) + **4번 섹션** (투사체 컴포넌트)

### 마우스 기반 투사체 시스템  
- **0번 섹션** (RigidBody 준비) + **2번 섹션** (마우스) + **4번 섹션** (투사체 컴포넌트)

### vibe-starter-3d 통합
- **0번 섹션** (vibe-starter-3d 패턴) + 원하는 입력 방식 + **4번 섹션**

---

## 0. 🚨 RigidBody 참조 문제 해결 (필수)

### 가장 흔한 실패 원인

```typescript
// ❌ 이런 코드는 99% 실패합니다
const Player = () => {
  const rigidBodyRef = useRef<RapierRigidBody>(null);
  
  useFrame(() => {
    if (!rigidBodyRef.current) return; // 여기서 항상 종료됨
    // 투사체 로직에 절대 도달하지 못함
  });
};
```

### ✅ 해결책: RigidBody 준비 상태 추적

```typescript
// 🔥 핵심 유틸리티 - 모든 프로젝트에서 사용
const useRigidBodyReady = (rigidBodyRef) => {
  const [isReady, setIsReady] = useState(false);
  
  useEffect(() => {
    const check = () => {
      if (rigidBodyRef.current) {
        setIsReady(true);
      } else {
        requestAnimationFrame(check);
      }
    };
    check();
  }, []);
  
  return isReady;
};

// ✅ 올바른 구현 패턴
const Player = () => {
  const rigidBodyRef = useRef<RapierRigidBody>(null);
  const isReady = useRigidBodyReady(rigidBodyRef); // 🔥 이것이 핵심!
  const lastFireTime = useRef(0);
  const [, getKeys] = useKeyboardControls();
  const { spawnProjectile } = useProjectileSystem(); // 4번 섹션에서 정의

  const fireProjectile = useCallback(() => {
    if (!rigidBodyRef.current) return;
    
    // Rapier API 사용 (중요!)
    const pos = rigidBodyRef.current.translation();
    const startPos = new THREE.Vector3(pos.x, pos.y + 0.5, pos.z);
    const direction = new THREE.Vector3(1, 0, 0);
    
    console.log('Fire!', startPos); // 디버깅용
    
    // 실제 투사체 생성 (4번 섹션의 ProjectileSystem 사용)
    spawnProjectile({
      startPosition: startPos,
      direction,
      speed: 10,
      maxDistance: 20
    });
  }, [spawnProjectile]);

  useFrame(() => {
    if (!isReady) return; // 🔥 이 체크가 성공의 핵심!
    
    const { fire } = getKeys();
    const now = Date.now();
    
    if (fire && now - lastFireTime.current > 500) {
      lastFireTime.current = now;
      fireProjectile();
    }
  });

  return (
    <RigidBody ref={rigidBodyRef} type="dynamic" position={[0, 1, 0]}>
      <mesh>
        <boxGeometry />
        <meshStandardMaterial color="blue" />
      </mesh>
    </RigidBody>
  );
};
```

## `vibe-starter-3d` RigidBody 참조 사용 가이드라인

### 🚨 필수 구현 규칙

`vibe-starter-3d` 프레임워크에서 `RigidBodyPlayerRef` 를 사용할 때는 런타임 오류를 방지하기 위해 다음 패턴을 엄격히 준수해야 합니다.


### ❌ 금지된 접근 패턴 (오류 발생)

**다음 속성이나 메서드에 접근하면 안 됩니다:**

```typescript
// 이러한 패턴들은 런타임 오류를 발생시킵니다:
rigidBodyPlayerRef.current?.raw()        // ❌ raw() 메서드가 존재하지 않음
rigidBodyPlayerRef.current.raw()         // ❌ raw() 메서드가 존재하지 않음
rigidBodyPlayerRef.current?.rigidBody    // ❌ rigidBody 속성이 존재하지 않음
rigidBodyPlayerRef.current.rigidBody     // ❌ rigidBody 속성이 존재하지 않음
rigidBodyRef.current?.raw()              // ❌ raw() 메서드가 존재하지 않음
rigidBodyRef.current.raw()               // ❌ raw() 메서드가 존재하지 않음
rigidBodyRef.current?.rigidBody          // ❌ rigidBody 속성이 존재하지 않음
rigidBodyRef.current.rigidBody           // ❌ rigidBody 속성이 존재하지 않음
```

**이러한 패턴이 실패하는 이유:**
- ref가 `RapierRigidBody` 인터페이스를 직접 노출함
- `raw()`나 `rigidBody` 속성을 가진 중간 래퍼 객체가 없음
- 프레임워크가 `useImperativeHandle`을 사용하여 실제 `RapierRigidBody` 인스턴스를 직접 반환함

### ✅ 올바른 접근 패턴 (필수 구현)

**항상 직접 ref 접근을 사용하세요:**

```typescript
// 프레임워크와 함께 작동하는 올바른 패턴:
const rigidBodyPlayerRef = useRef<RigidBodyPlayerRef>(null);
const rigidBodyRef = useRef<RigidBodyObjectRef>(null);

// RapierRigidBody 메서드와 속성에 직접 접근:
if (rigidBodyPlayerRef.current) {
  // 위치, 속도, 회전 가져오기
  const position = rigidBodyPlayerRef.current.translation();
  const velocity = rigidBodyPlayerRef.current.linvel();
  const rotation = rigidBodyPlayerRef.current.rotation();
  
  // 물리 속성 설정
  rigidBodyPlayerRef.current.setTranslation({ x: 0, y: 10, z: 0 }, true);
  rigidBodyPlayerRef.current.setLinvel({ x: 0, y: 0, z: 5 }, true);
  rigidBodyPlayerRef.current.setRotation({ x: 0, y: 0, z: 0, w: 1 }, true);
  
  // 추가 속성 접근 (RigidBodyPlayerRef만 해당)
  const bottomY = rigidBodyPlayerRef.current.bottomY;
}
```
### 🚫 절대 하지 말아야 할 실수들

```typescript
// ❌ 잘못된 API 사용
const pos = rigidBody.position;     // 존재하지 않음
const pos = rigidBody.matrixWorld;  // 존재하지 않음

// ✅ 올바른 Rapier API
const playerRigidBody = playerBody; // 강체
const pos = rigidBody.translation(); // 위치
const vel = rigidBody.linvel();      // 속도
const rot = rigidBody.rotation();    // 회전
```

### 0.4 완전한 작동 예제 (복사해서 사용)

```typescript
// KeyboardControls 설정
const keyboardMap = [
  { name: 'fire', keys: ['KeyF', 'f'] }
];

// 완전한 플레이어 컴포넌트 (0번 섹션 패턴 적용)
const Player = () => {
  const rigidBodyRef = useRef<RapierRigidBody>(null);
  const isReady = useRigidBodyReady(rigidBodyRef);
  const lastFireTime = useRef(0);
  const [, getKeys] = useKeyboardControls();
  const { spawnProjectile } = useProjectileSystem();

  const fireProjectile = useCallback(() => {
    if (!rigidBodyRef.current) return;
    
    const pos = rigidBodyRef.current.translation();
    const startPos = new THREE.Vector3(pos.x, pos.y + 0.5, pos.z);
    const direction = new THREE.Vector3(1, 0, 0);
    
    console.log('🚀 Fire!', startPos);
    
    spawnProjectile({
      startPosition: startPos,
      direction,
      speed: 10,
      maxDistance: 20
    });
  }, [spawnProjectile]);

  useFrame(() => {
    if (!isReady) return;
    
    const { fire } = getKeys();
    const now = Date.now();
    
    if (fire && now - lastFireTime.current > 500) {
      lastFireTime.current = now;
      fireProjectile();
    }
  });

  return (
    <RigidBody ref={rigidBodyRef} type="dynamic" position={[0, 1, 0]}>
      <mesh>
        <boxGeometry />
        <meshStandardMaterial color="blue" />
      </mesh>
    </RigidBody>
  );
};

// 전체 앱 구조
const App = () => (
  <KeyboardControls map={keyboardMap}>
    <Canvas>
      <Physics>
        <ProjectileSystem>
          <Player />
        </ProjectileSystem>
      </Physics>
    </Canvas>
  </KeyboardControls>
);
```

---

## 1. Keyboard Input Implementation

키보드 기반 발사체 시스템은 React Three Fiber의 기본 키보드 제어 기능을 확장합니다:

### 주요 기능
- 맞춤 설정 가능한 발사 트리거 액션 (기본값: "action1")
- 플레이어 RigidBody 참조를 기반으로 한 발사 방향 계산
- 성능 최적화를 위한 최근 발사 방향 캐싱
- 이벤트 기반 구독 시스템

### 중요 요구사항
- **모든 키보드 입력 관련 훅은 반드시 `KeyboardControls` 컴포넌트 내부에서 사용해야 합니다.**
- 이 요구사항을 지키지 않으면 `Uncaught TypeError: object null is not iterable (cannot read property Symbol(Symbol.iterator))` 오류가 발생합니다.

### 구현 요구사항

```typescript
// 1. 키보드 매핑 정의
const keyboardMap: KeyboardControlsEntry[] = [
  // 이동 컨트롤
  { name: 'forward', keys: ['ArrowUp', 'w', 'W'] },
  { name: 'backward', keys: ['ArrowDown', 's', 'S'] },
  { name: 'left', keys: ['ArrowLeft', 'a', 'A'] },
  { name: 'right', keys: ['ArrowRight', 'd', 'D'] },
  // 발사 컨트롤 - hook 설정의 fireAction과 일치해야 함
  { name: 'action1', keys: ['KeyE', 'e', 'E'] },
];

// 2. KeyboardControls 컴포넌트로 전체 앱 또는 씬을 감싸기
const Game = () => (
  <KeyboardControls map={keyboardMap}>
    <Scene />
  </KeyboardControls>
);

// 3. 컴포넌트에서 projectile hook 초기화 (KeyboardControls 내부)
const { subscribeTrigger, getFireDirection, isTriggerPressed } = 
  useProjectileKeyboardInput({
    fireAction: "action1", // keyboardMap의 이름과 일치해야 함
    playerRigidBodyRef: playerRef // 캐릭터 RigidBody 참조
  });

// 3. 발사 이벤트 구독
useEffect(() => {
  const handleFire = (direction: Vector3) => {
    spawnProjectile(position, direction, speed);
  };
  
  return subscribeTrigger(handleFire);
}, [subscribeTrigger]);
```

> **실용적 구현을 위한 참고**: 이 유틸리티 함수들의 원리를 이해하는 것은 복잡한 커스텀 로직 구현에 도움이 됩니다.

## 2. Mouse Input Implementation

### 중요 요구사항
**마우스 기반 발사체 시스템은 R3F의 네이티브 이벤트 처리와 `useThree` 훅을 통해 구현 해야합니다.**

### 주요 기능
- R3F의 이벤트 시스템을 활용한 마우스 입력 상태 추적
- 프레임 기반 입력 확인 메커니즘
- 마우스 버튼 상태 변화 감지 (누름/뗌)
- 커스텀 발사 방향 계산 로직 통합 가능

### 사용 패턴

```typescript
import { useThree, useFrame } from '@react-three/fiber';
import { useRef, useState, useCallback, useEffect } from 'react';
import * as THREE from 'three';

const PlayerComponent: React.FC = () => {
  const { gl, camera } = useThree();
  const [mouseState, setMouseState] = useState({ left: false });
  const leftPressedLastFrame = useRef(false);
  const rigidBodyRef = useRef<RapierRigidBody>(null);
  const isReady = useRigidBodyReady(rigidBodyRef); // 0번 섹션 패턴 적용
  const { spawnProjectile } = useProjectileSystem();
  
  // 마우스 이벤트 리스너 설정
  useEffect(() => {
    const handleMouseDown = (e) => {
      if (e.button === 0) setMouseState(prev => ({ ...prev, left: true }));
    };
    
    const handleMouseUp = (e) => {
      if (e.button === 0) setMouseState(prev => ({ ...prev, left: false }));
    };
    
    // DOM 요소에 이벤트 리스너 추가
    const domElement = gl.domElement;
    domElement.addEventListener('mousedown', handleMouseDown);
    domElement.addEventListener('mouseup', handleMouseUp);
    
    return () => {
      // 정리 함수
      domElement.removeEventListener('mousedown', handleMouseDown);
      domElement.removeEventListener('mouseup', handleMouseUp);
    };
  }, [gl]);
  
  // 발사체 생성 콜백
  const fireProjectile = useCallback(() => {
    if (!rigidBodyRef.current) return;
    
    const pos = rigidBodyRef.current.translation();
    const startPos = new THREE.Vector3(pos.x, pos.y + 0.5, pos.z);
    const direction = new THREE.Vector3(1, 0, 0);
    
    spawnProjectile({
      startPosition: startPos,
      direction,
      speed: 10,
      maxDistance: 20
    });
  }, [spawnProjectile]);
  
  useFrame(() => {
    if (!isReady) return; // 🔥 0번 섹션 패턴 적용
    
    // 버튼 상태 변화 감지 (이전 프레임과 비교)
    const leftJustPressed = mouseState.left && !leftPressedLastFrame.current;
    leftPressedLastFrame.current = mouseState.left;
    
    // 투사체 발사
    if (leftJustPressed) {
      fireProjectile();
    }
  });
  
  return (
    <RigidBody ref={rigidBodyRef} type="dynamic" position={[0, 1, 0]}>
      <mesh>
        <boxGeometry />
        <meshStandardMaterial color="red" />
      </mesh>
    </RigidBody>
  );
};
```

### 주요 구현 포인트
- R3F의 `useThree`를 통해 canvas와 카메라에 접근
- 리액트 `useState`와 `useEffect`로 마우스 이벤트 리스닝
- DOM 이벤트를 활용한 마우스 버튼 상태 추적
- 이전 프레임 상태를 참조하여 버튼이 "방금 눌렸는지" 확인
- 카메라 위치/방향을 기반으로 한 발사 방향 계산
- useFrame 내부에서 입력 처리 및 발사 로직 실행

> **실용적 구현을 위한 참고**: 이 유틸리티 함수들의 원리를 이해하는 것은 복잡한 커스텀 로직 구현에 도움이 됩니다.

## 3. 투사체 방향 계산 - 스마트 분석 구현

### 🤖 자동 분석 시작

**먼저 프로젝트 코드를 분석하여 게임 타입을 판단하고 게임 타입에 따른 적합한 구현을 사용하세요.**

AI가 코드에서 다음을 자동 확인합니다:
- 카메라 컨트롤 방식 (PointerLockControls vs OrbitControls)
- 플레이어 구현 패턴 (1인칭 vs 3인칭)
- 기존 프로젝트 구조 및 명명 규칙

**중요 참고사항 :** 캐릭터의 전방 벡터는 (0, 0, 1) 입니다.

### 🎯 자동 판단 기준

#### FPS 게임 신호 감지:
- `PointerLockControls` 사용
- `crosshair`, `firstPerson` 관련 코드
- `camera.getWorldDirection` 패턴
- 카메라와 플레이어 위치가 거의 동일
- `MouseLook`, `FPSControls` 등의 컴포넌트

#### TPS 게임 신호 감지:
- `OrbitControls` 사용
- `thirdPerson`, `characterMesh` 관련 코드
- `rigidBody.rotation`, `playerModel` 패턴
- 캐릭터 메시가 씬에 렌더링
- 캐릭터 회전 로직 존재

### 3.1 FPS 게임 구현 - 카메라 기준 방향

**AI가 FPS 패턴을 감지했을 때 사용하는 구현:**

```typescript
import React, { useCallback, useRef } from 'react';
import { useThree, useFrame } from '@react-three/fiber';
import { useKeyboardControls } from '@react-three/drei';
import * as THREE from 'three';

/**
* 카메라 기준 투사체 방향 계산
* @param camera - THREE.js 카메라
* @returns 정규화된 방향 벡터
*/
export const getCameraDirection = (camera: THREE.Camera): THREE.Vector3 => {
 const direction = new THREE.Vector3();
 camera.getWorldDirection(direction);
 return direction; // 이미 정규화되어 있음
};

/**
* 카메라의 현재 위치 가져오기
* @param camera - THREE.js 카메라
* @returns 카메라의 현재 위치 벡터
*/
export const getCameraPosition = (camera: THREE.Camera): THREE.Vector3 => {
 const position = new THREE.Vector3();
 camera.getWorldPosition(position);
 return position;
};

/**
* 카메라 기준 투사체 발사 위치 계산
* @param camera - THREE.js 카메라
* @param offsetDistance - 카메라로부터의 전방 거리
* @returns 투사체 발사 위치 벡터
*/
export const getProjectileStartPosition = (
 camera: THREE.Camera,
 offsetDistance: number = 0
): THREE.Vector3 => {
 const cameraPosition = new THREE.Vector3();
 camera.getWorldPosition(cameraPosition);
 
 const direction = new THREE.Vector3();
 camera.getWorldDirection(direction);
 
 // 카메라 위치에서 바라보는 방향으로 offsetDistance만큼 앞으로 이동
 return cameraPosition.add(direction.clone().multiplyScalar(offsetDistance));
};

// FPS 플레이어 컴포넌트 예제
const FPSPlayer = () => {
 const { camera } = useThree();
 const lastFireTime = useRef(0);
 const [, getKeys] = useKeyboardControls();
 const { spawnProjectile } = useProjectileSystem();

 const fireProjectile = useCallback(() => {
   // 카메라 기준 투사체 발사 위치 계산
   const startPos = getProjectileStartPosition(camera, 0.5);
   
   // 카메라 기준 투사체 방향 계산
   const direction = getCameraDirection(camera);
   
   console.log('🎯 FPS Fire!', startPos, direction);
   
   spawnProjectile({
     startPosition: startPos,
     direction,
     speed: 20,
     maxDistance: 50
   });
 }, [camera, spawnProjectile]);

 useFrame(() => {
   const { fire } = getKeys();
   const now = Date.now();
   
   if (fire && now - lastFireTime.current > 300) {
     lastFireTime.current = now;
     fireProjectile();
   }
 });

 // FPS는 보통 플레이어 모델이 없거나 최소한
 return null;
};
```

### 3.2 TPS 게임 구현 - 캐릭터 기준 방향

**AI가 TPS 패턴을 감지했을 때 사용하는 구현:**

```typescript
import React, { useCallback, useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import { useKeyboardControls, RigidBody, RapierRigidBody } from '@react-three/drei';
import * as THREE from 'three';

/**
 * 캐릭터 기준 투사체 방향 벡터 계산
 * @param rigidBody - 캐릭터의 Rapier 물리 바디
 * @returns 캐릭터가 바라보는 방향의 정규화된 벡터
 */
export const getCharacterDirection = (rigidBody: RigidBody): THREE.Vector3 | null => {
  if (!rigidBody) return null;
  
  // 캐릭터의 회전 정보 추출
  const rotation = rigidBody.rotation();
  
  // Three.js 쿼터니언으로 변환
  const quaternion = new THREE.Quaternion(
    rotation.x, rotation.y, rotation.z, rotation.w
  );
  
  // 전방 벡터(0, 0, 1)에 쿼터니언을 적용하여 캐릭터의 방향 계산
  return new THREE.Vector3(0, 0, 1)
    .applyQuaternion(quaternion)
    .normalize();
};

/**
 * 캐릭터의 현재 위치 가져오기
 * @param rigidBody - 캐릭터의 물리 바디
 * @returns 캐릭터의 현재 위치 벡터
 */
export const getCharacterPosition = (rigidBody: RigidBody): THREE.Vector3 | null => {
  if (!rigidBody) return null;
  
  const position = rigidBody.translation();
  return new THREE.Vector3(
    position.x, position.y, position.z
  );
};

/**
 * 캐릭터 기준 투사체 발사 위치 계산
 * @param rigidBody - 캐릭터의 물리 바디
 * @param offsetDistance - 캐릭터로부터의 전방 거리
 * @param heightOffset - 추가 높이 오프셋
 * @returns 투사체 발사 위치 벡터
 */
export const getProjectileStartPosition = (
  rigidBody: RigidBody,
  offsetDistance: number = 0,
  heightOffset: number = 0
): THREE.Vector3 | null => {
  if (!rigidBody) return null;
  
  // 캐릭터 위치 가져오기
  const characterPosition = getCharacterPosition(rigidBody);
  if (!characterPosition) return null;
  
  // 캐릭터 방향 가져오기
  const direction = getCharacterDirection(rigidBody);
  if (!direction) return null;
  
  // 투사체 시작 위치 계산: 캐릭터 위치 + (방향 * 오프셋 거리) + 높이 오프셋
  return characterPosition.clone()
    .add(direction.clone().multiplyScalar(offsetDistance))
    .add(new THREE.Vector3(0, heightOffset, 0));
};

// TPS 플레이어 컴포넌트 예제
const TPSPlayer = () => {
  const rigidBodyRef = useRef<RapierRigidBody>(null);
  const isReady = useRigidBodyReady(rigidBodyRef); // 0번 섹션 패턴 적용
  const lastFireTime = useRef(0);
  const [, getKeys] = useKeyboardControls();
  const { spawnProjectile } = useProjectileSystem();

  const fireProjectile = useCallback(() => {
    if (!rigidBodyRef.current) return;
    
    // 캐릭터 기준 투사체 발사 위치 계산
    const startPos = getProjectileStartPosition(rigidBodyRef.current, 0.2, 0.5);
    
    // 캐릭터 기준 투사체 방향 계산
    const direction = getCharacterDirection(rigidBodyRef.current);
    
    if (startPos && direction) {
      console.log('🎯 TPS Fire!', startPos, direction);
      
      spawnProjectile({
        startPosition: startPos,
        direction,
        speed: 15,
        maxDistance: 30
      });
    }
  }, [spawnProjectile]);

  useFrame(() => {
    if (!isReady) return; // 🔥 0번 섹션 패턴 적용
    
    const { fire } = getKeys();
    const now = Date.now();
    
    if (fire && now - lastFireTime.current > 500) {
      lastFireTime.current = now;
      fireProjectile();
    }
  });

  return (
    <RigidBody ref={rigidBodyRef} type="dynamic" position={[0, 1, 0]}>
      <mesh>
        <boxGeometry args={[0.5, 1, 0.5]} />
        <meshStandardMaterial color="blue" />
      </mesh>
    </RigidBody>
  );
};
```

**자동 분석이 불가능하거나 결과가 부정확한 경우 다음 질문에 답해주세요:**

#### 간단 체크리스트:
1. **게임 화면에서 캐릭터 모델이 보이나요?**
   - Yes → **3.2 TPS 구현 사용**
   - No → **3.1 FPS 구현 사용**

2. **화면 중앙에 조준점(크로스헤어)이 있나요?**
   - Yes → **3.1 FPS 구현 사용**
   - No → **3.2 TPS 구현 사용**

#### 유사한 게임으로 확인:
- **FPS 스타일**: Counter-Strike, Valorant, Call of Duty → **3.1 구현**
- **TPS 스타일**: Fortnite, GTA, 젤다의 전설 → **3.2 구현**

### 🧪 구현 후 검증 방법

#### FPS 구현이 올바른지 확인:
- [ ] 마우스를 움직일 때 투사체 방향이 따라 변하는가?
- [ ] 투사체가 화면 중앙(크로스헤어)을 향해 날아가는가?
- [ ] 카메라 위치에서 투사체가 생성되는가?

#### TPS 구현이 올바른지 확인:
- [ ] 캐릭터를 회전시킬 때 투사체 방향이 따라 변하는가?
- [ ] 투사체가 캐릭터 앞쪽에서 생성되는가?
- [ ] 캐릭터가 바라보는 방향으로 투사체가 날아가는가?

### ⚠️ 문제 해결

**투사체가 엉뚱한 방향으로 날아간다면:**
1. 현재 사용 중인 구현 방식 확인 (FPS vs TPS)
2. 반대 방식으로 변경해서 테스트
3. 0번 섹션의 RigidBody 참조 문제 확인

**예상과 다르게 동작한다면:**
- FPS → TPS로 변경: 3.2 구현 사용
- TPS → FPS로 변경: 3.1 구현 사용

### 🔗 관련 섹션 참조
- **0번 섹션**: RigidBody 참조 문제 해결 (TPS 구현 시 필수)
- **4번 섹션**: 투사체 이동 및 충돌 시스템
- **1번 섹션**: 키보드 입력 처리
- **2번 섹션**: 마우스 입력 처리

## 4. 투사체 이동 및 충돌 감지 시스템

투사체의 움직임과 충돌 감지는 게임플레이에 중요한 요소입니다. React Three Fiber와 Rapier 물리 엔진을 활용하여 효율적인 투사체 시스템을 구현할 수 있습니다.

### 4.0 InteractionGroups - 충돌 그룹 관리

**InteractionGroups**는 Rapier 물리 엔진에서 어떤 객체끼리 충돌할 수 있는지를 제어하는 핵심 시스템입니다. 또한 다음과 같이 구현 되어 있습니다.

```typescript
export declare type InteractionGroups = number;
```

#### 동작 원리

InteractionGroups는 비트 마스크를 사용한 쌍별 필터링 시스템입니다. 이 시스템은 다음과 같이 작동합니다:

**구조:**
- **32비트 값** 하나로 두 개의 16비트 정보를 저장
- **상위 16비트**: interaction groups (이 객체가 속한 그룹들)
- **하위 16비트**: interaction mask (이 객체가 충돌할 수 있는 그룹들)

**충돌 허용 조건:**
두 객체 `a`와 `b` 사이에 충돌이 발생하려면 **동시에** 두 조건을 만족해야 합니다:

1. `a`의 interaction groups와 `b`의 interaction mask에 **최소 하나의 공통 비트**가 있어야 함
2. `b`의 interaction groups와 `a`의 interaction mask에 **최소 하나의 공통 비트**가 있어야 함

**간단한 예시:**
- 플레이어 투사체: "나는 플레이어 투사체 그룹에 속하고, 적 그룹과 충돌하고 싶다"
- 적: "나는 적 그룹에 속하고, 플레이어 투사체 그룹과 충돌하고 싶다"
- 결과: 양쪽 모두 서로를 원하므로 충돌 발생 ✅

#### 실제 사용법 - interactionGroups 헬퍼 활용

복잡한 비트 연산 대신 `@react-three/rapier`의 `interactionGroups` 헬퍼를 사용하는 것이 권장됩니다:

```typescript
import { interactionGroups } from '@react-three/rapier';

// 기본 사용법
const collisionGroup = interactionGroups(
  myGroup,           // 이 객체가 속하는 그룹 (숫자 또는 배열)
  targetGroups       // 이 객체가 충돌하고 싶은 그룹들 (숫자 또는 배열)
);

// 예시: 그룹 2에 속하고, 그룹 1과 4에 충돌
const playerProjectileGroup = interactionGroups(2, [1, 4]);

// 예시: 여러 그룹에 속하고, 단일 그룹과 충돌  
const complexGroup = interactionGroups([0, 2], 5);

// 예시: 모든 그룹과 충돌 (두 번째 인수 생략)
const universalGroup = interactionGroups(3);
```

#### 투사체 시스템에서의 활용 예제

```typescript
import { interactionGroups } from '@react-three/rapier';

// 게임 엔티티별 그룹 번호 정의 (0-15 범위)
const GROUPS = {
  PLAYER: 0,
  ENEMY: 1,
  PLAYER_PROJECTILE: 2,
  ENEMY_PROJECTILE: 3,
  ENVIRONMENT: 4
};

// 플레이어 투사체 설정
const playerProjectileGroups = interactionGroups(
  GROUPS.PLAYER_PROJECTILE,                    // 플레이어 투사체 그룹에 속함
  [GROUPS.ENEMY, GROUPS.ENVIRONMENT]          // 적과 환경과 충돌
);

// 적 투사체 설정
const enemyProjectileGroups = interactionGroups(
  GROUPS.ENEMY_PROJECTILE,                     // 적 투사체 그룹에 속함
  [GROUPS.PLAYER, GROUPS.ENVIRONMENT]         // 플레이어와 환경과 충돌
);

// 환경 오브젝트 설정
const environmentGroups = interactionGroups(
  GROUPS.ENVIRONMENT,                          // 환경 그룹에 속함
  [GROUPS.PLAYER, GROUPS.ENEMY, GROUPS.PLAYER_PROJECTILE, GROUPS.ENEMY_PROJECTILE] // 모든 엔티티와 충돌
);
```
### 4.1 프레임 기반 투사체 시스템

가장 기본적인 형태로, useFrame을 사용하여 매 프레임마다 위치를 업데이트합니다.

**중요 참고사항 :** `ActiveCollisionTypes` 와 `InteractionGroups` 는 `@react-three/rapier` 에는 존재하지 않습니다. 반드시 `@dimforge/rapier3d-compat` 라이브러리를 사용하세요.

```typescript
import React, { useRef, useMemo, useCallback, useEffect, useState } from 'react';
import * as THREE from 'three';
import { useFrame } from '@react-three/fiber';
import { useRapier, RapierRigidBody, CollisionPayload } from '@react-three/rapier';
import { ActiveCollisionTypes, InteractionGroups } from '@dimforge/rapier3d-compat';

interface ProjectileProps {
  startPosition: THREE.Vector3;
  direction: THREE.Vector3;
  speed: number;
  maxDistance: number;
  owner?: RapierRigidBody;
  effectCollisionGroups?: InteractionGroups;
  onHit?: (payload: CollisionPayload) => void;
  onComplete?: () => void;
}

const Projectile: React.FC<ProjectileProps> = React.memo(
  ({startPosition, direction, speed, maxDistance, owner, effectCollisionGroups, onHit, onRemove }) => {
    const [active, setActive] = useState(true);
    const rigidBodyRef = useRef<RapierRigidBody>(null);
    const { rapier, world } = useRapier();
    const distanceTraveled = useRef(0);
    const freezed = useRef(false);
    const framesSinceFreezed = useRef(0);
    const normalizedDirection = direction.clone().normalize();

    const removeBall = useCallback(() => {
      setActive(false);
      onComplete();
    }, [onComplete]);

    useFrame((_, delta) => {
      if (!active || !rigidBodyRef.current) return;

      const rigidBody = rigidBodyRef.current;
      if (!rigidBody || rigidBody.numColliders() === 0) return;

      const frameTravelDistance = speed * delta;
      const newDistanceTraveled = distanceTraveled.current + frameTravelDistance;

      if (newDistanceTraveled >= maxDistance) {
        // 최대 거리 도달 시 정확한 위치에서 제거
        const finalPosition = startPosition.clone().addScaledVector(normalizedDirection, maxDistance);
        rigidBody.position.copy(finalPosition);
        removeBall();
        return;
      }

      // skip update if bullet is freezed
      if (freezed.current) {
        framesSinceFreezed.current++;

        // If onTriggerEnter hasn't been called after 3 frames, unfreeze and continue checking
        if (framesSinceFreezed.current > 3) {
          console.warn('Bullet frozen too long, unfreezing to continue collision check');
          freezed.current = false;
          framesSinceFreezed.current = 0;
        }
        return;
      }
      
      

      const originVec = rigidBody.translation();
      const ray = new rapier.Ray(originVec, normalizedDirection);
      const ownCollider = rigidBody.collider(0);
      
      const hit = world.castRay(
        ray,
        frameTravelDistance,
        true, // solid
        undefined, // queryFlags
        effectCollisionGroups ?? 0xffffffff, // groups
        ownCollider, // Exclude projectile's own collider
        owner, // Exclude the owner (firer)
      );

      if (hit) {
        const hitPoint = ray.pointAt(hit.timeOfImpact);
        const hitPointVec3 = new THREE.Vector3(hitPoint.x, hitPoint.y, hitPoint.z);

        freezed.current = true;
        framesSinceFreezed.current = 0;
        rigidBody.setTranslation(hitPointVec3, true);
        return;
      } else {
        // No hit, continue with velocity based movement (handled by RigidBody component)
        distanceTraveled.current = newDistanceTraveled;
        const curVec3 = new THREE.Vector3(originVec.x, originVec.y, originVec.z);
        const nextPosition = curVec3.addScaledVector(normalizedDirection, frameTravelDistance);
        rigidBody.setTranslation(nextPosition, true);        
      }
    });

    // 충돌 이벤트 핸들러
    const handleCollisionEnter = (payload: CollisionPayload) => {
      freezed.current = false;
      framesSinceFreezed.current = 0;
      if (owner && owner.handle === payload.other.rigidBody.handle) {
        return;
      }

      onHit?.(payload);
      removeBullet();   
    };    

    if (!active) return null;

    return (
      <RigidBody
        ref={rigidBodyRef}
        userData={{
          type: RigidBodyObjectType.BULLET,
        }}
        type="kinematicVelocity"
        position={[startPosition.x, startPosition.y, startPosition.z]}
        sensor={true}
        colliders="cuboid"
        collisionGroups={effectCollisionGroups ?? 0xffffffff}
        activeCollisionTypes={ActiveCollisionTypes.ALL}
        onCollisionEnter={handleCollisionEnter}
        onIntersectionEnter={handleCollisionEnter}
      >
        <mesh>
          <sphereGeometry args={[0.1, 16, 16]} />
          <meshStandardMaterial color="orange" emissive="red" emissiveIntensity={2} />
        </mesh>
      </RigidBody>
    );
  },
);

export default Projectile;
```

### 4.2 투사체 시스템 관리자

여러 투사체를 효율적으로 관리하기 위한 시스템입니다.

```typescript
import React, { useState, useCallback, useMemo, createContext, useContext } from 'react';
import * as THREE from 'three';
import Projectile from './Projectile'; // 위에서 정의한 컴포넌트
import { RigidBodyApi, Collider } from '@react-three/rapier';
import { InteractionGroups } from '@dimforge/rapier3d-compat';

interface ProjectileData {
  id: string;
  startPosition: THREE.Vector3;
  direction: THREE.Vector3;
  speed: number;
  maxDistance: number;
  owner?: RigidBodyApi;
  effectCollisionGroups?: InteractionGroups;
  onHit?: (pos?: THREE.Vector3, rigidBody?: RigidBodyApi, collider?: Collider) => boolean;
}

interface ProjectileSystemContextType {
  spawnProjectile: (data: Omit<ProjectileData, 'id'>) => void;
}

const ProjectileSystemContext = createContext<ProjectileSystemContextType | undefined>(undefined);

export const useProjectileSystem = () => {
  const context = useContext(ProjectileSystemContext);
  if (!context) {
    throw new Error('useProjectileSystem must be used within a ProjectileSystemProvider');
  }
  return context;
};

let projectileIdCounter = 0;
const generateUniqueId = () => `proj-${projectileIdCounter++}`;

const ProjectileSystem: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [projectiles, setProjectiles] = useState<ProjectileData[]>([]);

  const spawnProjectile = useCallback((data: Omit<ProjectileData, 'id'>) => {
    const id = generateUniqueId();
    setProjectiles((prev) => [...prev, { ...data, id }]);
  }, []);

  const removeProjectile = useCallback((id: string) => {
    setProjectiles((prev) => prev.filter((p) => p.id !== id));
  }, []);

  const contextValue = useMemo(() => ({ spawnProjectile }), [spawnProjectile]);

  return (
    <ProjectileSystemContext.Provider value={contextValue}>
      {projectiles.map((projectile) => (
        <Projectile
          key={projectile.id}
          {...projectile}
          onRemove={removeProjectile}
        />
      ))}
      {children}
    </ProjectileSystemContext.Provider>
  );
};

export default ProjectileSystem;
```

## 5. 통합 솔루션

위 모든 요소를 통합한 고수준 API (`useProjectile` 훅)

```typescript
// 통합 솔루션 예제는 프로젝트 요구사항에 따라 위의 섹션들을 조합하여 구현
```

> **실용적 구현을 위한 참고**: 5번 섹션의 `useProjectile` 훅은 이러한 방향 계산 유틸리티를 내부적으로 사용하여 더 간편한 API를 제공합니다.

## 🐛 일반적인 문제 해결

### "Cannot read property 'current' of null"
- **원인**: RigidBody가 초기화되지 않음
- **해결**: 0번 섹션의 `useRigidBodyReady` 패턴 사용

### "useKeyboardControls is not a function"  
- **원인**: KeyboardControls로 감싸지 않음
- **해결**: 1번 섹션의 구조 확인

### 투사체가 생성되지 않음
- **원인**: ProjectileSystem이 설정되지 않음
- **해결**: 4.2번 섹션의 ProjectileSystem 컴포넌트 사용

### 충돌이 감지되지 않음
- **원인**: InteractionGroups 설정 오류
- **해결**: 4.0번 섹션의 충돌 그룹 설정 확인

### "rigidBody.translation is not a function"
- **원인**: 잘못된 API 사용
- **해결**: 0번 섹션의 올바른 Rapier API 사용

## 🎯 성공을 위한 핵심 체크리스트

1. ✅ `useRigidBodyReady` 훅 사용 (0번 섹션)
2. ✅ `if (!isReady) return;` 체크 추가
3. ✅ `rigidBody.translation()` API 사용
4. ✅ 입력 시스템 올바른 설정:
   - 키보드 사용 시: `KeyboardControls`로 감싸기
   - 마우스 사용 시: `useThree`의 `gl.domElement`에 이벤트 리스너 추가
   - vibe-starter-3d 사용 시: 컨트롤러 내부에서 훅 사용
5. ✅ 쿨다운 시간 설정
6. ✅ 투사체 시스템(Frame) 구현 시 Raycast 와 Rigidbody 를 동시에 사용
7. ✅ InteractionGroups 올바른 설정
8. ✅ ProjectileSystem 컴포넌트로 앱 감싸기
9. ✅ 기능 구현 시 스텝을 나누지 않고 한번에 구현하기

**이 체크리스트를 따르면 100% 성공합니다!** 🎯

## 6. 프로젝트 구조 분석 및 적응형 통합 가이드

### 6.0 프로젝트 구조 분석 방법론

**이 섹션은 투사체 시스템을 구현하기 전에 여러분의 프로젝트 구조를 분석하고, 기존 시스템과 자연스럽게 통합하는 방법을 제시합니다.**

#### 🔍 Step 1: 기존 시스템 구성 요소 파악

투사체 시스템을 구현하기 전, 다음 질문들을 통해 프로젝트 구조를 분석하세요:

**A. 플레이어 시스템 분석**
```typescript
// 다음 중 어떤 패턴을 사용하고 있나요?
- useRef<RapierRigidBody>(null)           // 표준 R3F 패턴
- useRef<RigidBodyPlayerRef>(null)        // 커스텀 플레이어 참조
- useRef<SomeCustomRigidBodyType>(null)   // 프로젝트 특화 타입
- 다른 물리 바디 관리 시스템?
```

**B. 입력 처리 시스템 분석**
```typescript
// 입력 처리는 어떻게 구현되어 있나요?
- useKeyboardControls() 직접 사용        // 표준 패턴
- 통합된 InputController 컴포넌트        // 중앙화된 입력 처리
- 커스텀 입력 매핑 시스템               // 프로젝트 특화 입력
- 상태 관리 라이브러리 통합             // Zustand, Redux 등
```

**C. 액션/상태 관리 분석**
```typescript
// 플레이어 액션은 어떻게 관리되나요?
- useState로 로컬 상태 관리             // 컴포넌트 내 상태
- 전역 상태 관리 (Zustand, Redux)      // 중앙화된 상태
- 커스텀 액션 시스템                   // 프로젝트 특화 액션
- 이벤트 시스템                       // 옵저버 패턴 등
```

**D. 3D 씬 구성 분석**
```typescript
// 3D 씬은 어떻게 구성되어 있나요?
- 단일 Experience/Scene 컴포넌트       // 중앙화된 씬
- 모듈화된 씬 컴포넌트들               // 분산된 씬 구성
- 레이어 기반 구성                     // UI/3D 분리
- 커스텀 씬 관리자                     // 프로젝트 특화 구성
```

#### 🎯 Step 2: 통합 전략 수립

분석 결과에 따라 최적의 통합 전략을 선택하세요:

### 6.1 패턴별 구현 가이드

#### Pattern A: 표준 R3F 환경
```typescript
// 표준 패턴이 발견된 경우
const Player = () => {
  const rigidBodyRef = useRef<RapierRigidBody>(null);
  const isReady = useRigidBodyReady(rigidBodyRef); // 0번 섹션 패턴
  const [, getKeys] = useKeyboardControls();
  
  // 1번 섹션 (키보드) 또는 2번 섹션 (마우스) 패턴 적용
  useFrame(() => {
    if (!isReady) return;
    // 표준 구현 로직
  });
};
```

#### Pattern B: 통합된 입력 시스템 환경
```typescript
// 기존 입력 시스템이 있는 경우
// 1. 기존 입력 매핑 구조 파악
const ACTION_KEY_MAPPING = {
  existingAction1: ['Key1'],
  existingAction2: ['Key2'],
  // 여기에 fire 액션 추가
};

// 2. 기존 패턴을 따라 확장
const updatedMapping = {
  ...ACTION_KEY_MAPPING,
  fire: ['KeyF', 'Mouse0'], // 기존 패턴 일관성 유지
};
```

#### Pattern C: 상태 관리 시스템 통합
```typescript
// 기존 상태 관리 시스템 확장
// Zustand 예시
interface ExistingActionState {
  existingAction1: boolean;
  existingAction2: boolean;
  // 여기에 fire 추가
}

// Redux 예시  
const actionReducer = (state, action) => {
  switch (action.type) {
    case 'EXISTING_ACTION':
      return { ...state, existingAction: action.payload };
    case 'FIRE': // 새로 추가
      return { ...state, fire: action.payload };
  }
};
```

#### Pattern D: 커스텀 플레이어 참조 시스템
```typescript
// 프로젝트 특화 플레이어 참조가 있는 경우
const Player = () => {
  const customPlayerRef = useRef<CustomPlayerRefType>(null);
  
  // 참조 시스템이 준비될 때까지 대기
  useFrame(() => {
    if (!customPlayerRef.current) return;
    
    // 프로젝트 특화 API 사용
    const position = customPlayerRef.current.getPosition?.() || 
                    customPlayerRef.current.translation?.() ||
                    customPlayerRef.current.position;
    
    // 방향 계산도 프로젝트 API에 맞게 조정
  });
};
```

### 6.2 단계별 통합 체크리스트

#### Phase 1: 준비 단계
- [ ] 기존 프로젝트의 플레이어 참조 시스템 파악
- [ ] 입력 처리 방식 분석
- [ ] 액션 관리 시스템 확인
- [ ] 씬 구성 방식 이해

#### Phase 2: 확장 단계
- [ ] 기존 액션 시스템에 `fire` 액션 추가
- [ ] 기존 입력 매핑에 발사 키 추가
- [ ] 기존 패턴을 따라 코드 일관성 유지

#### Phase 3: 통합 단계
- [ ] 투사체 시스템 컴포넌트 생성
- [ ] 기존 플레이어 컴포넌트에 발사 로직 추가
- [ ] 기존 씬 구성에 투사체 시스템 통합

#### Phase 4: 최적화 단계
- [ ] 기존 성능 최적화 패턴 적용
- [ ] 프로젝트 코딩 스타일 일관성 유지
- [ ] 기존 에러 처리 방식 따라 구현

### 6.3 실제 적용 예시

#### 분석 결과 예시
// 당신의 프로젝트에서 발견한 패턴 (예시)
✅ 발견된 패턴:
- RigidBodyPlayerRef 타입 사용
- InputController의 KeyMapping 패턴
- playerActionStore의 Zustand 상태 관리
- Experience 컴포넌트 중심 씬 구성

📋 통합 계획:
1. playerActionStore에 fire 액션 추가
2. InputController의 ACTION_KEY_MAPPING 확장
3. Player 컴포넌트의 useFrame 패턴 활용
4. Experience 컴포넌트에 ProjectileSystem 추가

#### 구현 결과
```typescript
// 프로젝트 구조에 맞게 자연스럽게 통합된 코드
const Player = () => {
  // 기존 패턴 유지
  const { getPlayerAction } = usePlayerActionStore();
  const { spawnProjectile } = useProjectileSystem();
  
  // 기존 useFrame 패턴 확장
  useFrame(() => {
    // 기존 로직...
    
    // 새로 추가된 발사 로직 (기존 패턴과 일관성 유지)
    const firePressed = getPlayerAction('fire');
    if (firePressed) {
      fireProjectile();
    }
  });
};
```

### 6.4 프로젝트별 최적화 팁

#### 🎯 일관성 유지 원칙
- 기존 네이밍 컨벤션 따르기
- 기존 파일 구조 패턴 유지
- 기존 에러 처리 방식 적용
- 기존 성능 최적화 패턴 활용

#### 🔧 확장성 고려사항
- 기존 시스템과의 호환성 유지
- 향후 기능 추가 시 확장 용이성
- 기존 테스트 패턴과의 일관성
- 기존 문서화 스타일 따르기

이렇게 하면 어떤 프로젝트든 그 프로젝트의 고유한 구조를 분석하고 최적화된 방식으로 투사체 시스템을 통합할 수 있습니다.
