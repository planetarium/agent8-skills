---
name: how-to-use-3d-tree
description: |
  Core document on procedurally generating realistic 3D trees and forests in a React Three Fiber (R3F) environment using the `@dgreenheck/ez-tree` library.
  
  This document covers:
  - Placing single or multiple 3D trees in a scene with position, scale, and rotation control.
  - Pre-defined presets for various tree types (Oak, Pine, Ash, Aspen, Bush) and sizes (small, medium, large).
  - Environment configurations (dense-forest, sparse-forest, pine-forest, mixed-forest, bushland, park).
  - Customizing bark, branch, and leaf properties (texture, color, shape, count, etc.).
  - Performance optimization: LOD management, culling, branch level limits, and tree count tuning.
  
  Reference this document when a user wants to add trees or forests to a 3D scene.
---

```tsx
import { useEffect, useRef, useMemo } from 'react';
import { useThree } from '@react-three/fiber';
import { Tree, TreePreset } from '@dgreenheck/ez-tree';
import * as THREE from 'three';

/**
 * 🌲 Trees 컴포넌트 - 다양한 나무들을 렌더링합니다
 * 
 * 📌 사용 가능한 프리셋 목록:
 * - 'Ash Small', 'Ash Medium', 'Ash Large' (물푸레나무 - 넓은 잎, 밝은 녹색)
 * - 'Aspen Small', 'Aspen Medium', 'Aspen Large' (사시나무 - 하얀 수피, 둥근 잎)
 * - 'Bush 1', 'Bush 2', 'Bush 3' (관목/덤불 - 낮고 풍성한 형태)
 * - 'Oak Small', 'Oak Medium', 'Oak Large' (참나무 - 갈색 수피, 특징적인 잎)
 * - 'Pine Small', 'Pine Medium', 'Pine Large' (소나무 - 침엽수, 수직적 성장)
 */

// ========================================
// 🎨 식생 분위기 설정 섹션
// ========================================

// 🌍 환경 타입 선택 (원하는 분위기에 맞게 하나 선택)
type EnvironmentType = 
  | 'dense-forest'      // 울창한 숲
  | 'sparse-forest'     // 듬성듬성한 숲
  | 'pine-forest'       // 침엽수림
  | 'mixed-forest'      // 혼합림
  | 'bushland'          // 관목지대
  | 'park'              // 공원
  | 'custom';           // 커스텀

// ⚠️ 주의: 'dense-forest'는 나무가 많아 성능에 영향을 줄 수 있습니다.
// 성능 이슈가 있다면 다음을 시도해보세요:
// 1. TREE_CONFIG.count를 줄이기 (예: 30-40)
// 2. 'mixed-forest'나 'sparse-forest'로 변경
// 3. GENERATION_WEIGHTS.preset 비율을 높이기 (커스텀보다 가벼움)
const ENVIRONMENT: EnvironmentType = 'park'; // 👈 원하는 환경 선택

// 🌳 나무 배치 설정
const TREE_CONFIG = {
  // 나무 개수 (성능 고려: 50-100 권장, 최대 300)
  // ⚠️ dense-forest의 경우 30-50 권장
  count: 50,
  
  // 배치 영역 크기 (x, z 좌표 범위: -size/2 ~ +size/2)
  areaSize: 100,
  
  // 나무 간 최소 거리
  minDistance: 8,
  
  // 중앙 안전 지대 크기 (캐릭터 스폰 위치)
  safezoneSize: 10,
};

// 🎲 나무 생성 방법 비율 (합계가 1.0이 되도록 설정)
const GENERATION_WEIGHTS = {
  preset: 0.6,        // 프리셋 그대로 사용
  variation: 0.3,    // 프리셋 + 변이
  custom: 0.1,       // 완전 커스텀
};

// 🌿 환경별 프리셋 설정
const ENVIRONMENT_PRESETS: Record<EnvironmentType, {
  preferredPresets: string[];
  treeScale: { min: number; max: number };
  density: number;
  maxBranchLevels: number;
  performanceNote?: string;
}> = {
  'dense-forest': {
    preferredPresets: ['Oak Large', 'Oak Medium', 'Ash Large', 'Ash Medium'],
    treeScale: { min: 0.8, max: 1.4 },
    density: 1.2,
    // ⚠️ 성능 설정: dense-forest는 나무가 많아 branch.levels를 제한합니다
    maxBranchLevels: 1, // 성능을 위해 1로 제한
    performanceNote: '⚠️ 울창한 숲은 나무가 많아 성능에 영향을 줄 수 있습니다. branch.levels를 1로 제한합니다.'
  },
  'sparse-forest': {
    preferredPresets: ['Oak Small', 'Ash Small', 'Aspen Medium'],
    treeScale: { min: 0.6, max: 1.2 },
    density: 0.6,
    maxBranchLevels: 2, // 나무가 적어서 2까지 허용
  },
  'pine-forest': {
    preferredPresets: ['Pine Large', 'Pine Medium', 'Pine Small'],
    treeScale: { min: 0.9, max: 1.5 },
    density: 1.0,
    maxBranchLevels: 1, // 침엽수는 원래 1레벨이 일반적
  },
  'mixed-forest': {
    preferredPresets: ['Oak Medium', 'Pine Medium', 'Ash Medium', 'Aspen Medium'],
    treeScale: { min: 0.7, max: 1.3 },
    density: 0.9,
    maxBranchLevels: 2,
  },
  'bushland': {
    preferredPresets: ['Bush 1', 'Bush 2', 'Bush 3'],
    treeScale: { min: 0.5, max: 1.1 },
    density: 1.5,
    maxBranchLevels: 2, // 관목은 복잡해도 크기가 작아 괜찮음
  },
  'park': {
    preferredPresets: ['Oak Large', 'Ash Large', 'Bush 1'],
    treeScale: { min: 0.5, max: 0.8 },
    density: 0.4,
    maxBranchLevels: 2, // 나무가 적어서 디테일 허용
  },
  'custom': {
    preferredPresets: Object.keys(TreePreset), // 모든 프리셋 사용
    treeScale: { min: 0.5, max: 1.5 },
    density: 1.0,
    maxBranchLevels: 2,
  },
};

// ========================================
// 🎯 커스텀 나무 설정 (ENVIRONMENT가 'custom'일 때 주로 사용)
// ========================================

/**
 * 🌲 커스텀 나무 옵션 생성 함수
 * 각 파라미터의 의미를 이해하고 원하는 나무를 만들어보세요!
 */
function createCustomTreeOptions() {
  return {
    // 🎲 시드: 랜덤 생성을 위한 값 (같은 시드 = 같은 모양)
    seed: Math.floor(Math.random() * 100000),
    
    // 🌳 나무 타입
    type: Math.random() > 0.5 ? 'deciduous' : 'evergreen', // 활엽수 vs 침엽수
    
    // 🎨 나무껍질 설정
    bark: {
      // 텍스처 타입: 'oak'(참나무), 'pine'(소나무), 'birch'(자작나무), 'willow'(버드나무)
      type: 'birch', // 또는 'willow' 사용 가능
      
      // 색상 (16진수): 0xffffff = 흰색(원본), 0xcea07e = 갈색
      tint: 0xcea07e,
      
      // 플랫 셰이딩: true = 각진 느낌, false = 부드러운 느낌
      flatShading: false,
      
      // 텍스처 사용 여부
      textured: true,
      
      // 텍스처 스케일: x는 둘레방향, y는 높이방향 반복
      textureScale: { x: 0.5, y: 5 }
    },
    
    // 🌿 가지 설정 (가장 중요한 부분!)
    branch: {
      // ⚠️ 레벨 수: 0=줄기만, 1=1차가지, 2=2차가지 (성능상 1-2 권장!)
      levels: 1,
      
      // 각 레벨의 가지 각도 (도 단위)
      // 0도=수직위, 90도=수평, 180도=수직아래
      angle: {
        1: 40,   // 1차 가지: 40도 (위쪽 대각선)
        2: 35,   // 2차 가지: 35도
        3: 30    // 3차 가지: 30도
      },
      
      // 각 레벨의 자식 가지 개수
      children: {
        0: 8,    // 줄기에서 나오는 1차 가지 개수
        1: 4,    // 1차 가지에서 나오는 2차 가지 개수
        2: 2     // 2차 가지에서 나오는 3차 가지 개수
      },
      
      // 외부 힘 (바람, 중력 등)
      force: {
        direction: { x: 0, y: 1, z: 0 }, // 방향 (y=1: 위쪽)
        strength: -0.01  // 음수=아래로 처짐, 양수=위로 솟음
      },
      
      // 꺾임 정도 (0=직선, 1=매우 구불구불)
      gnarliness: {
        0: 0.05,  // 줄기: 약간 구부러짐
        1: 0.15,  // 1차 가지: 중간 구부러짐
        2: 0.10,  // 2차 가지
        3: 0.05   // 3차 가지
      },
      
      // 각 레벨의 길이
      length: {
        0: 30,    // 줄기 높이
        1: 20,    // 1차 가지 길이
        2: 10,    // 2차 가지 길이
        3: 5      // 3차 가지 길이
      },
      
      // 각 레벨의 두께 (반지름)
      radius: {
        0: 2.0,   // 줄기 두께
        1: 0.8,   // 1차 가지 두께
        2: 0.4,   // 2차 가지 두께
        3: 0.2    // 3차 가지 두께
      },
      
      // 각 가지의 분할 수 (높을수록 부드러움, 낮을수록 각짐)
      sections: {
        0: 10,    // 줄기 분할
        1: 8,     // 1차 가지 분할
        2: 6,     // 2차 가지 분할
        3: 4      // 3차 가지 분할
      },
      
      // 둘레 방향 분할 수 (3=삼각형, 8=팔각형, 높을수록 원형)
      segments: {
        0: 6,     // 줄기
        1: 5,     // 1차 가지
        2: 4,     // 2차 가지
        3: 3      // 3차 가지
      },
      
      // 가지 시작 위치 (0=시작점, 1=끝점)
      start: {
        1: 0.3,   // 1차 가지는 줄기의 30% 지점부터 시작
        2: 0.3,   // 2차 가지는 1차 가지의 30% 지점부터
        3: 0.1    // 3차 가지는 2차 가지의 10% 지점부터
      },
      
      // 끝으로 갈수록 가늘어지는 정도 (0=원통형, 1=원뿔형)
      taper: {
        0: 0.7,   // 줄기
        1: 0.6,   // 1차 가지
        2: 0.7,   // 2차 가지
        3: 0      // 3차 가지 (끝이므로 0)
      },
      
      // 비틀림 정도 (라디안 단위, 양수=시계방향)
      twist: {
        0: 0.1,   // 줄기 약간 비틀림
        1: -0.05, // 1차 가지 반대로 비틀림
        2: 0,     // 2차 가지 비틀림 없음
        3: 0      // 3차 가지 비틀림 없음
      }
    },
    
    // 🍃 잎 설정
    leaves: {
      // 잎 텍스처 타입: 'ash', 'pine', 'aspen', 'oak' 등
      type: 'aspen', 
      
      // 빌보드 타입: 'single'=한면, 'double'=양면(X자 형태)
      billboard: 'double',
      
      // 가지에 대한 잎의 각도 (도 단위)
      angle: 30,
      
      // 잎 개수 (성능에 영향)
      count: 200,
      
      // 잎이 시작되는 위치 (0=가지 시작, 1=가지 끝)
      start: 0.1,
      
      // 잎 크기
      size: 4,
      
      // 크기 변화량 (0=모두 같은 크기, 1=크기 다양)
      sizeVariance: 0.5,
      
      // 색상 틴트 (0xffffff=원본색상 유지)
      tint: 0xffffff,
      
      // 투명도 임계값 (0.1=거의 투명, 0.9=거의 불투명)
      alphaTest: 0.3
    }
  };
}

// ========================================
// 메인 컴포넌트
// ========================================

export function Trees() {
  const { scene } = useThree();
  const treesRef = useRef<THREE.Group[]>([]);

  // 현재 환경 설정 가져오기
  const envConfig = ENVIRONMENT_PRESETS[ENVIRONMENT];
  const adjustedTreeCount = Math.floor(TREE_CONFIG.count * envConfig.density);

  // 나무 위치 미리 계산 (렌더링 최적화)
  const treePositions = useMemo(() => {
    const positions = [];
    
    // 나무 위치 랜덤 생성 (겹치지 않게)
    for (let i = 0; i < adjustedTreeCount; i++) {
      let x, z;
      let overlapping = true;
      let attempts = 0;

      while (overlapping && attempts < 50) {
        x = Math.floor((Math.random() - 0.5) * TREE_CONFIG.areaSize);
        z = Math.floor((Math.random() - 0.5) * TREE_CONFIG.areaSize);
        
        // 중앙 안전지대 체크
        if (Math.abs(x) < TREE_CONFIG.safezoneSize / 2 && 
            Math.abs(z) < TREE_CONFIG.safezoneSize / 2) {
          attempts++;
          continue;
        }

        overlapping = false;
        // 기존 나무들과 거리 체크
        for (const pos of positions) {
          const distance = Math.sqrt(Math.pow(x - pos.x, 2) + Math.pow(z - pos.z, 2));
          if (distance < TREE_CONFIG.minDistance) {
            overlapping = true;
            break;
          }
        }
        attempts++;
      }

      if (!overlapping) {
        positions.push({ x, z });
      }
    }
    
    return positions;
  }, [adjustedTreeCount]);

  useEffect(() => {
    // 이전에 생성된 나무들을 제거
    treesRef.current.forEach(tree => {
      if (tree && scene.children.includes(tree)) {
        scene.remove(tree);
      }
    });
    treesRef.current = [];

    // 🚨 성능 경고 출력
    if (envConfig.performanceNote) {
      console.warn(envConfig.performanceNote);
    }

    // 나무 생성 및 배치
    treePositions.forEach((pos, index) => {
      try {
        const tree = new Tree();
        const options = tree.options;
        
        // 🎲 나무 생성 방법 결정
        const rand = Math.random();
        
        if (rand < GENERATION_WEIGHTS.preset) {
          // 📘 방법 1: 프리셋 직접 사용
          const preset = envConfig.preferredPresets[
            Math.floor(Math.random() * envConfig.preferredPresets.length)
          ];
          tree.loadPreset(preset);
          tree.options.seed = Math.floor(Math.random() * 100000);
          
        } else if (rand < GENERATION_WEIGHTS.preset + GENERATION_WEIGHTS.variation) {
          // 📗 방법 2: 프리셋 + 변이
          const preset = envConfig.preferredPresets[
            Math.floor(Math.random() * envConfig.preferredPresets.length)
          ];
          tree.loadPreset(preset);
          
          // 변이 적용
          options.seed = Math.floor(Math.random() * 100000);
          
          // 크기 변이
          if (options.branch.length[0]) {
            options.branch.length[0] *= (0.8 + Math.random() * 0.4);
          }
          
          // 가지 각도 변이
          if (options.branch.angle[1]) {
            options.branch.angle[1] += Math.floor((Math.random() - 0.5) * 20);
          }
          
          // 잎 변이
          if (options.leaves) {
            options.leaves.size *= (0.7 + Math.random() * 0.6);
            options.leaves.count = Math.floor(options.leaves.count * (0.8 + Math.random() * 0.4));
          }
          
          // 🌲 dense-forest 특별 변이
          if (ENVIRONMENT === 'dense-forest') {
            // 울창한 숲은 더 다양한 높이와 굵기 변화
            if (options.branch.radius[0]) {
              options.branch.radius[0] *= (0.7 + Math.random() * 0.6);
            }
            // 약간의 기울기 추가 (자연스러움)
            if (options.branch.force) {
              options.branch.force.strength *= (0.8 + Math.random() * 0.4);
            }
            // 잎 색상 약간 변화 (깊이감)
            if (options.leaves && options.leaves.tint) {
              const tintVar = 0.85 + Math.random() * 0.15;
              options.leaves.tint = Math.floor(0xffffff * tintVar);
            }
          }
          
        } else {
          // 📙 방법 3: 완전 커스텀
          const customOptions = createCustomTreeOptions();
          Object.assign(options, customOptions);
        }
        
        // ⚠️ 성능 최적화: branch.levels 제한
        if (options.branch && options.branch.levels > envConfig.maxBranchLevels) {
          options.branch.levels = envConfig.maxBranchLevels;
          
          // dense-forest의 경우 다른 방법으로 복잡성 추가
          if (ENVIRONMENT === 'dense-forest' && options.branch.levels === 1) {
            // 1레벨이지만 가지 수를 늘려 풍성하게
            if (options.branch.children[0]) {
              options.branch.children[0] = Math.min(options.branch.children[0] * 1.5, 15);
            }
            // 가지 길이 변화로 다양성 추가
            if (options.branch.length[1]) {
              options.branch.length[1] *= (0.7 + Math.random() * 0.6);
            }
          }
        }
        
        // 🎯 공통 설정
        tree.frustumCulled = true;
        tree.castShadow = false;
        tree.receiveShadow = false;
        
        // 나무 생성
        tree.generate();
        
        // 위치 설정
        tree.position.set(pos.x, 0, pos.z);
        
        // 환경별 크기 조절
        const scale = envConfig.treeScale.min + 
          Math.random() * (envConfig.treeScale.max - envConfig.treeScale.min);
        tree.scale.set(scale, scale, scale);
        
        // Y축 회전 (자연스러운 배치)
        tree.rotation.y = Math.random() * Math.PI * 2;
        
        // 씬에 추가
        scene.add(tree);
        treesRef.current.push(tree);
        
      } catch (error) {
        console.warn("나무 생성 중 오류 발생:", error);
      }
    });

    // 컴포넌트 언마운트 시 나무 제거
    return () => {
      treesRef.current.forEach(tree => {
        if (tree && scene.children.includes(tree)) {
          scene.remove(tree);
        }
      });
    };
  }, [scene, treePositions, envConfig]);

  return null;
}

/**
 * 📚 분위기별 식생 가이드
 * 
 * 🌲 울창한 숲 (dense-forest):
 * - 큰 나무 위주, 높은 밀도
 * - Oak Large, Ash Large 프리셋 사용
 * - 스케일: 0.8-1.4
 * - ⚠️ 성능 주의: branch.levels를 1로 자동 제한
 * - 💡 성능 팁: 나무 개수를 30-40개로 줄이거나 mixed-forest 사용 권장
 * 
 * 🌳 듬성듬성한 숲 (sparse-forest):
 * - 작은 나무, 낮은 밀도
 * - 나무 간격이 넓음
 * - 스케일: 0.6-1.2
 * 
 * 🌲 침엽수림 (pine-forest):
 * - Pine 프리셋만 사용
 * - 수직적이고 일정한 모습
 * - 스케일: 0.9-1.5
 * 
 * 🌿 관목지대 (bushland):
 * - Bush 프리셋만 사용
 * - 낮고 넓게 퍼진 형태
 * - 높은 밀도
 * 
 * 🏞️ 공원 (park):
 * - 잘 관리된 느낌
 * - 큰 나무와 관목 혼합
 * - 낮은 밀도
 * 
 * ✏️ 커스텀 팁:
 * 1. ENVIRONMENT를 'custom'으로 설정
 * 2. createCustomTreeOptions() 함수 수정
 * 3. TREE_CONFIG로 배치 조절
 * 4. 원하는 프리셋 조합 사용
 * 
 * 🚀 성능 최적화:
 * - branch.levels는 환경별로 자동 제한됨
 * - dense-forest: 1레벨 (가지 수로 복잡도 보완)
 * - 나무가 많을수록 낮은 레벨 권장
 * - preset 비율을 높이면 성능 향상
 */
```
