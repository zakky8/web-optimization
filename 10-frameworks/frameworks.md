# Frameworks

Three.js is the foundation, but several frameworks build on top of it to integrate 3D scenes cleanly into component-based UI architectures. The right choice depends on your frontend stack.

## The Landscape

| Framework | Renderer | Ecosystem | Stars (2026) | Best for |
|---|---|---|---|---|
| Three.js (vanilla) | WebGL / WebGPU | Largest | 105K | Full control, no framework dependency |
| React Three Fiber (R3F) | Three.js | Richest | 30K | React apps, ecosystem depth |
| TresJS | Three.js | Growing | 5K | Vue 3 apps |
| Threlte | Three.js | Active | 4K | Svelte / SvelteKit apps |
| Angular Three | Three.js | Small | 1K | Angular apps |
| Babylon.js | Custom | Large | 24K | Games, more built-in tools |

## React Three Fiber (R3F)

R3F is a React renderer for Three.js. Every Three.js object maps to a JSX element. Props map to properties. React state/refs drive animation.

### Core Concepts

```jsx
import { Canvas, useFrame, useThree } from "@react-three/fiber";
import { useRef } from "react";

// Canvas sets up renderer, scene, camera, animation loop
function App() {
  return (
    <Canvas
      camera={{ position: [0, 0, 5], fov: 45 }}
      gl={{ antialias: true, toneMapping: THREE.ACESFilmicToneMapping }}
      shadows
    >
      <ambientLight intensity={0.5} />
      <directionalLight position={[5, 5, 5]} castShadow />
      <Sphere />
    </Canvas>
  );
}

// Object component — ref, useFrame, constructor args
function Sphere() {
  const meshRef = useRef(null);

  useFrame((state, delta) => {
    // state.clock.elapsedTime, state.camera, state.gl, state.scene
    meshRef.current.rotation.y += delta;
  });

  return (
    <mesh ref={meshRef} castShadow>
      {/* Constructor args passed as array */}
      <sphereGeometry args={[1, 32, 32]} />
      <meshStandardMaterial color="hotpink" />
    </mesh>
  );
}
```

### Property Mapping

```jsx
// Three.js object                    JSX equivalent
mesh.position.set(1, 2, 3)        -> <mesh position={[1, 2, 3]}>
mesh.rotation.x = Math.PI / 2     -> <mesh rotation-x={Math.PI / 2}>
material.color.set("red")         -> <meshStandardMaterial color="red">
geometry = new BoxGeometry(2,2,2) -> <boxGeometry args={[2, 2, 2]} />

// Nested properties with dash notation
<mesh scale-x={2} rotation-y={Math.PI} />
```

### State Management

```jsx
import { useFrame, useThree } from "@react-three/fiber";

function SceneInfo() {
  const { camera, scene, gl, size, viewport } = useThree();
  // size: { width, height } in pixels
  // viewport: { width, height } in world units
  // gl: WebGLRenderer instance
  return null;
}

// Shared state via Zustand (recommended for R3F)
import { create } from "zustand";

const useStore = create((set) => ({
  hovered: false,
  setHovered: (v) => set({ hovered: v }),
}));

function HoverMesh() {
  const hovered    = useStore(s => s.hovered);
  const setHovered = useStore(s => s.setHovered);

  return (
    <mesh
      onPointerEnter={() => setHovered(true)}
      onPointerLeave={() => setHovered(false)}
    >
      <boxGeometry />
      <meshStandardMaterial color={hovered ? "orange" : "white"} />
    </mesh>
  );
}
```

### Drei — The Utility Library

```jsx
import {
  OrbitControls, Environment, useGLTF, Text,
  MeshTransmissionMaterial, Float, Stats, Perf
} from "@react-three/drei";

function Scene() {
  const { scene } = useGLTF("/model.glb");

  return (
    <>
      <OrbitControls makeDefault />
      <Environment preset="city" />                   {/* IBL lighting */}

      <primitive object={scene} />                    {/* raw GLTF scene */}

      <Float speed={2} rotationIntensity={0.5}>       {/* gentle floating */}
        <mesh>
          <torusGeometry args={[1, 0.3, 16, 100]} />
          <MeshTransmissionMaterial transmission={1} roughness={0} />
        </mesh>
      </Float>

      <Text
        position={[0, 2, 0]}
        fontSize={0.5}
        color="white"
        font="/fonts/Inter-Bold.woff"
      >
        Hello World
      </Text>

      <Perf position="top-left" />                    {/* dev-only stats */}
    </>
  );
}
```

### Performance Patterns

```jsx
// 1. Instance identical meshes
import { Instances, Instance } from "@react-three/drei";

function Particles({ count = 1000 }) {
  const positions = useMemo(() =>
    Array.from({ length: count }, () => [
      (Math.random() - 0.5) * 10,
      (Math.random() - 0.5) * 10,
      (Math.random() - 0.5) * 10,
    ]), [count]);

  return (
    <Instances limit={count}>
      <sphereGeometry args={[0.05, 8, 8]} />
      <meshBasicMaterial color="white" />
      {positions.map((pos, i) => (
        <Instance key={i} position={pos} />
      ))}
    </Instances>
  );
}

// 2. Suspend heavy assets, show fallback
import { Suspense } from "react";

<Suspense fallback={<Loader />}>
  <HeavyModel />
</Suspense>

// 3. Disable render when tab hidden
<Canvas frameloop="demand">          {/* only render when invalidated */}
  ...
</Canvas>

// Trigger render explicitly
import { useThree } from "@react-three/fiber";
const invalidate = useThree(s => s.invalidate);
invalidate(); // call when state changes
```

## TresJS (Vue 3)

TresJS is the Vue 3 equivalent of R3F. It uses Vue's reactivity system to drive Three.js.

```bash
npm install @tresjs/core
```

```vue
<template>
  <TresCanvas :camera="{ position: [0, 0, 5] }" window-size>
    <TresPerspectiveCamera :fov="45" />
    <OrbitControls />

    <TresMesh ref="meshRef" :rotation-y="rotation">
      <TresSphereGeometry :args="[1, 32, 32]" />
      <TresMeshStandardMaterial color="hotpink" />
    </TresMesh>

    <TresAmbientLight :intensity="0.5" />
    <TresDirectionalLight :position="[5, 5, 5]" />
  </TresCanvas>
</template>

<script setup lang="ts">
import { TresCanvas } from "@tresjs/core";
import { OrbitControls } from "@tresjs/cientos";
import { ref } from "vue";
import { useRenderLoop } from "@tresjs/core";

const meshRef = ref(null);
const rotation = ref(0);

const { onLoop } = useRenderLoop();
onLoop(({ delta }) => {
  rotation.value += delta;
});
</script>
```

## Threlte (Svelte)

Threlte integrates Three.js with Svelte's reactivity and lifecycle.

```bash
npm install @threlte/core @threlte/extras three
```

```svelte
<script>
  import { Canvas } from "@threlte/core";
  import { T, useTask } from "@threlte/core";
  import { OrbitControls } from "@threlte/extras";

  let rotation = 0;

  useTask((delta) => {
    rotation += delta;
  });
</script>

<Canvas>
  <T.PerspectiveCamera position={[0, 0, 5]} makeDefault>
    <OrbitControls />
  </T.PerspectiveCamera>

  <T.AmbientLight intensity={0.5} />
  <T.DirectionalLight position={[5, 5, 5]} />

  <T.Mesh rotation.y={rotation}>
    <T.SphereGeometry args={[1, 32, 32]} />
    <T.MeshStandardMaterial color="hotpink" />
  </T.Mesh>
</Canvas>
```

## Framework Comparison

### When to Use What

**Vanilla Three.js**
- No React/Vue/Svelte in the project
- Need maximum control over the render loop
- Tight performance budgets (no virtual DOM overhead)
- Migrating an existing non-framework project

**React Three Fiber**
- Already using React
- Need deep integration with React state, context, routing
- Want access to the Drei/Poimandres ecosystem
- Building interactive product configurators or data visualizations

**TresJS**
- Already using Vue 3 / Nuxt
- Want Vue reactivity to drive animations declaratively

**Threlte**
- Already using Svelte / SvelteKit
- Want Svelte's compile-time reactivity for scene updates

### Performance Notes

| Aspect | Vanilla | R3F | TresJS | Threlte |
|---|---|---|---|---|
| Startup overhead | None | React reconciler | Vue reactivity | Svelte compiler overhead |
| Per-frame cost | Minimal | ~0.1ms reconciler | ~0.1ms | ~0.05ms |
| Memory | Minimal | React fiber nodes | Vue proxies | Minimal |
| Bundle addition | 0 | ~7 KB (react-fiber) | ~10 KB | ~5 KB |

The overhead differences are negligible for most scenes. The bigger performance wins come from geometry instancing, texture compression, and draw call batching — not from which framework you use.

### R3F-Specific Performance Tips

```jsx
// Avoid creating objects in render
// BAD: creates new Vector3 every frame
useFrame(() => {
  mesh.current.position.set(Math.sin(clock.elapsedTime), 0, 0);
});

// GOOD: mutate ref directly — R3F does not re-render on ref changes
const pos = useRef(new THREE.Vector3());
useFrame(({ clock }) => {
  pos.current.x = Math.sin(clock.elapsedTime);
  mesh.current.position.copy(pos.current);
});

// Use memo for geometry/material to avoid re-creation
const geometry = useMemo(() => new THREE.SphereGeometry(1, 32, 32), []);
const material = useMemo(() => new THREE.MeshStandardMaterial({ color: "red" }), []);
```

## Tools and Libraries

| Tool | Purpose |
|---|---|
| [React Three Fiber](https://docs.pmnd.rs/react-three-fiber) | R3F docs |
| [Drei](https://github.com/pmndrs/drei) | R3F helper components |
| [TresJS](https://tresjs.org) | Vue 3 Three.js integration |
| [Threlte](https://threlte.xyz) | Svelte Three.js integration |
| [R3F Performance Monitor](https://github.com/pmndrs/r3f-perf) | In-scene profiling (Perf component) |
| [Zustand](https://github.com/pmndrs/zustand) | Recommended state manager for R3F |
