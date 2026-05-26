# pmndrs Ecosystem

The Poimandres collective — primary maintainers of the R3F ecosystem.
Org: https://github.com/pmndrs

## Core Libraries — Version Status

| Library | Version | Last Release | Status |
|---------|---------|-------------|--------|
| react-three-fiber | v9.6.1 | 2026-04-28 | ✅ Active |
| drei | v10.7.7 | Nov 2025 (formal) / code pushes continue | ✅ Active |
| react-spring | v9.7.3 | 2023 | ⚠️ Marginal |
| zustand | v5.x | Active | ✅ Active |
| jotai | v2.x | Active | ✅ Active |
| react-three-rapier | v2.0 | Nov 2025 | ⚠️ Marginal |
| leva | v0.10.x | 2023 | ⚠️ Marginal |
| gltfjsx | CLI tool | Nov 2024 | ⚠️ Marginal |
| r3f-perf | v7.x | Dec 2024 | ⚠️ Marginal |
| detect-gpu | v5.x | 2026-05-24 | ✅ Active |

All dates verified against GitHub — 2026-05-26

## react-three-fiber (R3F) — Core Patterns

```jsx
import { Canvas, useFrame, useThree, useLoader } from '@react-three/fiber';
import { Suspense } from 'react';

// Canvas — creates WebGL context
<Canvas
  camera={{ position: [0, 2, 5], fov: 60 }}
  dpr={[1, 2]}        // Pixel ratio range
  frameloop="demand"  // 'always' | 'demand' | 'never'
  shadows             // Enable shadow maps
  gl={{ antialias: true }}
>
  <Suspense fallback={null}>
    <Scene />
  </Suspense>
</Canvas>

// useFrame — runs on every animation frame (RAF)
function RotatingBox() {
  const meshRef = useRef();
  useFrame((state, delta) => {
    meshRef.current.rotation.y += delta;
    // state.clock, state.camera, state.gl, state.scene available
  });
  return <mesh ref={meshRef}><boxGeometry /><meshStandardMaterial /></mesh>;
}

// useThree — access the Three.js context
function CameraController() {
  const { camera, gl, size } = useThree();
  // camera = THREE.PerspectiveCamera
  // gl = THREE.WebGLRenderer
  // size = { width, height }
}

// frameloop: "demand" — only renders when state invalidated
// Reduces GPU usage on static scenes
import { useThree } from '@react-three/fiber';
const invalidate = useThree(s => s.invalidate);
// Call invalidate() when something changes
```

## drei — Key Components

```jsx
import {
  OrbitControls, Environment, ContactShadows,
  Text, Html, Instances, Instance,
  useGLTF, useTexture, useFBO,
  Float, MeshTransmissionMaterial, Sparkles,
  PerspectiveCamera, Stage, Bounds, Center,
  MeshReflectorMaterial, Sky, Stars,
  AdaptiveDpr, PerformanceMonitor,
  BVHInstancedMesh, Outlines,
} from '@react-three/drei';

// AdaptiveDpr — reduce pixel ratio when FPS drops
<AdaptiveDpr pixelated />

// PerformanceMonitor — callback-based
<PerformanceMonitor onDecline={() => setDpr(1)} onIncline={() => setDpr(2)} />

// Float — gentle floating animation
<Float speed={2} rotationIntensity={0.5} floatIntensity={1}>
  <mesh />
</Float>

// Instances — GPU instancing via JSX
<Instances>
  <sphereGeometry />
  <meshStandardMaterial />
  {positions.map((pos, i) => (
    <Instance key={i} position={pos} />
  ))}
</Instances>
```

## detect-gpu

Classifies device GPU tier for adaptive quality.

```js
import { getGPUTier } from 'detect-gpu';

const gpuTier = await getGPUTier();
/*
  gpuTier.tier: 0 = blacklisted, 1 = low-end, 2 = mid, 3 = high-end
  gpuTier.type: 'ANGLE' | 'WEBGL' | 'WEBGL2' | 'WEBGPU'
  gpuTier.fps: benchmark result
  gpuTier.gpu: GPU name string
*/

// Adaptive quality
const quality = gpuTier.tier >= 3 ? 'ultra' : gpuTier.tier >= 2 ? 'medium' : 'low';
```

## zustand — Global State for R3F

```js
import { create } from 'zustand';

// State store — works in and out of Canvas
const useStore = create((set) => ({
  quality: 'high',
  paused: false,
  setQuality: (q) => set({ quality: q }),
  togglePause: () => set(s => ({ paused: !s.paused })),
}));

// Inside Canvas (runs in React context)
function Scene() {
  const quality = useStore(s => s.quality);
  // ...
}

// Outside Canvas (UI components)
function Controls() {
  const { quality, setQuality } = useStore();
  return <button onClick={() => setQuality('low')}>Low quality</button>;
}
```

## Sources
- pmndrs GitHub: https://github.com/pmndrs
- R3F docs: https://docs.pmnd.rs/react-three-fiber/
- Drei: https://github.com/pmndrs/drei
- detect-gpu: https://github.com/pmndrs/detect-gpu
