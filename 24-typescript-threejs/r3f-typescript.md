# React Three Fiber TypeScript Guide

> **Package versions used in this document**
> - `@react-three/fiber`: `^9.x` (pairs with React 19)
> - `three`: `0.184.0`
> - `@types/three`: `0.184.0`
> - `@types/react`: `^19.x`
> - TypeScript: `>=5.0`
> - Accessed: 2026-05-26

---

## 1. Installation

```bash
npm install three @react-three/fiber react react-dom
npm install --save-dev @types/three @types/react @types/react-dom
```

> **status: verified** — `@react-three/fiber` v9 pairs with React 19 and `three@0.184.x`.
> R3F does not ship its own Three.js types; `@types/three` must be installed separately.
> Source: [R3F TypeScript docs](https://r3f.docs.pmnd.rs/api/typescript),
> [pmndrs/react-three-fiber GitHub](https://github.com/pmndrs/react-three-fiber),
> accessed 2026-05-26.

### Minimal `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "esModuleInterop": true,
    "resolveJsonModule": true
  }
}
```

---

## 2. JSX Intrinsic Elements

React Three Fiber maps every Three.js class export into JSX via a dynamic reconciler.
All built-in Three.js elements are declared in the `ThreeElements` interface, which extends
`JSX.IntrinsicElements`. You get full autocomplete for props like `position`, `rotation`,
`args`, `attach`, and material/geometry properties.

```tsx
// All of these are valid typed JSX without any declaration file required:
<mesh position={[0, 1, 0]} castShadow>
  <boxGeometry args={[1, 1, 1]} />
  <meshStandardMaterial color="hotpink" roughness={0.4} metalness={0.1} />
</mesh>

<directionalLight intensity={2} position={[10, 10, 5]} castShadow />
<perspectiveCamera fov={75} near={0.1} far={1000} />
<group rotation={[0, Math.PI / 4, 0]} />
```

### Global JSX declaration for custom namespace extension

If you need to re-export all of `three/webgpu` as JSX elements (e.g. for WebGPU renderer):

```ts
// src/types/r3f.d.ts
import { ThreeToJSXElements } from '@react-three/fiber';
import * as THREE from 'three/webgpu';

declare module '@react-three/fiber' {
  interface ThreeElements extends ThreeToJSXElements<typeof THREE> {}
}
```

> **status: verified** — `ThreeToJSXElements` pattern documented for WebGPU usage.
> Source: [blog.pragmattic.dev](https://blog.pragmattic.dev/react-three-fiber-webgpu-typescript),
> accessed 2026-05-26.

---

## 3. Typing Component Props with `ThreeElements`

R3F exports the `ThreeElements` interface that maps element names to their full prop types
(including `position`, `rotation`, `scale`, `ref`, `args`, `attach`, event handlers, etc.).

```tsx
import { ThreeElements } from '@react-three/fiber';
import { useRef } from 'react';
import * as THREE from 'three';

// Extract mesh props from the R3F element catalog
type BoxProps = ThreeElements['mesh'] & {
  wireframe?: boolean;
};

function Box({ wireframe = false, ...props }: BoxProps) {
  const ref = useRef<THREE.Mesh>(null);

  return (
    <mesh ref={ref} {...props}>
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial wireframe={wireframe} />
    </mesh>
  );
}
```

> **status: verified** — `ThreeElements['mesh']` pattern shown in official R3F TypeScript docs.
> Source: [r3f.docs.pmnd.rs/api/typescript](https://r3f.docs.pmnd.rs/api/typescript),
> accessed 2026-05-26.

### Alternative: `JSX.IntrinsicElements`

```tsx
// Equivalent to ThreeElements['mesh'] — both resolve identically
type MeshNodeProps = JSX.IntrinsicElements['mesh'];

function Scene(props: JSX.IntrinsicElements['group']) {
  return <group {...props} />;
}
```

---

## 4. Typed `useRef`

Pass the Three.js class as a generic to `useRef`. The non-null assertion `null!` tells
TypeScript the ref will be populated by the time it is accessed in effects and event handlers.

```tsx
import { useRef } from 'react';
import * as THREE from 'three';

// General mesh ref
const meshRef = useRef<THREE.Mesh>(null);

// Narrowed ref — locks in geometry and material types
const boxRef = useRef<THREE.Mesh<THREE.BoxGeometry, THREE.MeshStandardMaterial>>(null);

// Group ref
const groupRef = useRef<THREE.Group>(null);

// Light ref
const lightRef = useRef<THREE.DirectionalLight>(null);

// Camera ref
const cameraRef = useRef<THREE.PerspectiveCamera>(null);
```

### Accessing ref values safely

```tsx
import { useFrame } from '@react-three/fiber';
import { useRef } from 'react';
import * as THREE from 'three';

function SpinningMesh() {
  const ref = useRef<THREE.Mesh>(null);

  useFrame((state, delta) => {
    // Optional chaining handles the one frame where ref.current may be null
    ref.current?.rotateY(delta);
  });

  return (
    <mesh ref={ref}>
      <sphereGeometry args={[1, 32, 32]} />
      <meshStandardMaterial color="orange" />
    </mesh>
  );
}
```

### Typing a ref to a custom material subclass

```tsx
interface GlassMesh extends THREE.Mesh {
  material: THREE.ShaderMaterial;
}

const glassRef = useRef<GlassMesh>(null);

useFrame((_, delta) => {
  if (glassRef.current?.material.uniforms.time) {
    glassRef.current.material.uniforms.time.value += delta;
  }
});
```

> **status: verified** — pattern from production R3F TypeScript usage.
> Source: [GitHub Gist by whoisryosuke](https://gist.github.com/whoisryosuke/d91d5101d1c290aca784b2ae31eb7a24),
> accessed 2026-05-26.

---

## 5. Typed `useFrame` State (`RootState`)

`useFrame` calls its callback before every rendered frame with two arguments:
`state: RootState` and `delta: number` (seconds since last frame, floating point).

```ts
useFrame((state: RootState, delta: number, xrFrame?: XRFrame) => void, renderPriority?: number): void
```

> **status: verified** — exact signature from R3F hooks documentation.
> Source: [r3f.docs.pmnd.rs/api/hooks](https://r3f.docs.pmnd.rs/api/hooks),
> accessed 2026-05-26.

### `RootState` properties

```ts
import { RootState } from '@react-three/fiber';

// Full RootState shape (key properties):
interface RootState {
  gl: THREE.WebGLRenderer;      // the renderer instance
  scene: THREE.Scene;           // the root scene
  camera: THREE.Camera;         // the active camera
  raycaster: THREE.Raycaster;   // shared raycaster
  clock: THREE.Clock;           // elapsed time, delta
  size: { width: number; height: number; top: number; left: number };
  viewport: {
    width: number;
    height: number;
    factor: number;
    distance: number;
    aspect: number;
  };
  performance: {
    current: number;
    min: number;
    max: number;
    debounce: number;
    regress: () => void;
  };
  mouse: THREE.Vector2;         // normalized device coords
  pointer: THREE.Vector2;       // same as mouse (alias)
  set: (state: Partial<RootState>) => void;
  get: () => RootState;
  invalidate: () => void;
  advance: (timestamp: number, runGlobalEffects?: boolean) => void;
}
```

### `useFrame` examples

```tsx
import { useFrame, RootState } from '@react-three/fiber';
import { useRef } from 'react';
import * as THREE from 'three';

function AnimatedMesh() {
  const ref = useRef<THREE.Mesh>(null);

  // Basic rotation
  useFrame((_state: RootState, delta: number) => {
    if (!ref.current) return;
    ref.current.rotation.x += delta * 0.5;
    ref.current.rotation.y += delta * 0.3;
  });

  // Access clock for elapsed time
  useFrame(({ clock }: RootState) => {
    const t = clock.getElapsedTime();
    ref.current?.position.setY(Math.sin(t) * 0.5);
  });

  // Camera-relative animation
  useFrame(({ camera }: RootState) => {
    ref.current?.lookAt(camera.position);
  });

  return (
    <mesh ref={ref}>
      <torusGeometry args={[1, 0.3, 16, 100]} />
      <meshStandardMaterial color="cyan" />
    </mesh>
  );
}
```

### `useFrame` with render priority

Higher priority runs first (lower number = earlier). Use when you need post-processing or
explicit render control:

```tsx
useFrame(({ gl, scene, camera }) => {
  gl.render(scene, camera);
}, 1); // priority 1 — disable auto-render by setting frameloop="never" on <Canvas>
```

### `useThree` — access state outside the loop

```tsx
import { useThree } from '@react-three/fiber';

function CameraController() {
  const { camera, gl, size } = useThree();
  // camera, gl, size are typed from RootState

  useEffect(() => {
    (camera as THREE.PerspectiveCamera).aspect = size.width / size.height;
    (camera as THREE.PerspectiveCamera).updateProjectionMatrix();
  }, [size]);

  return null;
}
```

---

## 6. Typed Event Handlers (`ThreeEvent`)

R3F exports `ThreeEvent<T>` where `T` is the underlying DOM event type. Event handlers on
mesh elements receive this union type combining DOM event data with Three.js raycaster
intersection data.

> **status: verified** — `ThreeEvent` exported from `@react-three/fiber`.
> Source: [r3f.docs.pmnd.rs/api/typescript](https://r3f.docs.pmnd.rs/api/typescript),
> [r3f.docs.pmnd.rs/api/events](https://r3f.docs.pmnd.rs/api/events),
> accessed 2026-05-26.

### `ThreeEvent` shape

```ts
import { ThreeEvent } from '@react-three/fiber';
import * as THREE from 'three';

// ThreeEvent<T> extends T (the DOM event) and adds:
interface ThreeEventExtras {
  object: THREE.Object3D;          // the mesh with the handler
  eventObject: THREE.Object3D;     // the propagation target
  intersections: THREE.Intersection[];
  point: THREE.Vector3;            // 3D intersection point
  pointOnLine: THREE.Vector3;
  distance: number;                // distance from camera
  unprojectedPoint: THREE.Vector3;
  ray: THREE.Ray;
  camera: THREE.Camera;
  delta: number;                   // px between pointerdown and pointerup
  nativeEvent: PointerEvent | MouseEvent | WheelEvent;
  stopped: boolean;
  stopPropagation: () => void;
}
```

### Typed event handler examples

```tsx
import { ThreeEvent } from '@react-three/fiber';

function InteractiveMesh() {
  const handleClick = (e: ThreeEvent<MouseEvent>) => {
    e.stopPropagation();
    console.log('Clicked at:', e.point);           // THREE.Vector3
    console.log('Distance:', e.distance);           // number
    console.log('Object name:', e.object.name);     // string
  };

  const handlePointerOver = (e: ThreeEvent<PointerEvent>) => {
    e.stopPropagation();
    document.body.style.cursor = 'pointer';
  };

  const handlePointerOut = (_e: ThreeEvent<PointerEvent>) => {
    document.body.style.cursor = 'default';
  };

  const handlePointerMove = (e: ThreeEvent<PointerEvent>) => {
    // Access all intersections in the ray
    e.intersections.forEach((hit) => {
      console.log(hit.point, hit.distance);
    });
  };

  const handleWheel = (e: ThreeEvent<WheelEvent>) => {
    console.log('Scroll delta Y:', e.nativeEvent.deltaY);
  };

  return (
    <mesh
      onClick={handleClick}
      onPointerOver={handlePointerOver}
      onPointerOut={handlePointerOut}
      onPointerMove={handlePointerMove}
      onWheel={handleWheel}
    >
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color="royalblue" />
    </mesh>
  );
}
```

### Pointer capture

```tsx
const handlePointerDown = (e: ThreeEvent<PointerEvent>) => {
  e.stopPropagation();
  // Capture pointer to this mesh for drag tracking
  (e.target as HTMLElement).setPointerCapture(e.pointerId);
};

const handlePointerUp = (e: ThreeEvent<PointerEvent>) => {
  (e.target as HTMLElement).releasePointerCapture(e.pointerId);
};
```

### Available event handler props

| Prop | DOM Event Type |
|------|---------------|
| `onClick` | `MouseEvent` |
| `onContextMenu` | `MouseEvent` |
| `onDoubleClick` | `MouseEvent` |
| `onWheel` | `WheelEvent` |
| `onPointerUp` | `PointerEvent` |
| `onPointerDown` | `PointerEvent` |
| `onPointerOver` | `PointerEvent` |
| `onPointerOut` | `PointerEvent` |
| `onPointerEnter` | `PointerEvent` |
| `onPointerLeave` | `PointerEvent` |
| `onPointerMove` | `PointerEvent` |
| `onPointerMissed` | `MouseEvent` |
| `onUpdate` | — (called on each update) |

---

## 7. Extending R3F with Custom Three.js Classes

### Method A: Module augmentation (global extension)

```ts
// src/types/r3f-extensions.d.ts
import { ThreeElement } from '@react-three/fiber';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

declare module '@react-three/fiber' {
  interface ThreeElements {
    orbitControls: ThreeElement<typeof OrbitControls>;
  }
}
```

Then register it:

```tsx
import { extend } from '@react-three/fiber';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

extend({ OrbitControls });

// Now typed in JSX:
function Controls() {
  return <orbitControls />;
}
```

### Method B: Factory `extend()` — local typing, no namespace bleed

```tsx
import { extend } from '@react-three/fiber';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

// Returns a typed component directly — no global declaration needed
const OrbitControlsElement = extend(OrbitControls);

function Controls() {
  return <OrbitControlsElement makeDefault />;
}
```

> **status: verified** — both patterns documented in official R3F TypeScript reference.
> Source: [r3f.docs.pmnd.rs/api/typescript](https://r3f.docs.pmnd.rs/api/typescript),
> accessed 2026-05-26.

### Custom class extension example

```ts
// src/CustomGrid.ts
import { GridHelper } from 'three';

export class CustomGrid extends GridHelper {
  readonly isCustomGrid = true;

  constructor(size = 10, divisions = 10) {
    super(size, divisions);
    this.name = 'CustomGrid';
  }

  tick(delta: number): void {
    this.rotation.y += delta * 0.1;
  }
}

// src/types/r3f-extensions.d.ts
import { ThreeElement } from '@react-three/fiber';
import { CustomGrid } from '../CustomGrid';

declare module '@react-three/fiber' {
  interface ThreeElements {
    customGrid: ThreeElement<typeof CustomGrid>;
  }
}

// In your component file:
import { extend } from '@react-three/fiber';
extend({ CustomGrid });

function Scene() {
  return <customGrid args={[20, 20]} position={[0, -1, 0]} />;
}
```

---

## 8. `useLoader` and `useGraph` Typing

```tsx
import { useLoader, useGraph } from '@react-three/fiber';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { GLTF } from 'three/examples/jsm/loaders/GLTFLoader.js';
import * as THREE from 'three';
import { Suspense } from 'react';

// useLoader infers the return type from the loader class
function Model() {
  const gltf = useLoader(GLTFLoader, '/models/hero.glb') as GLTF;

  // useGraph returns { nodes, materials } typed as records
  const { nodes, materials } = useGraph(gltf.scene);
  // nodes: Record<string, THREE.Object3D>
  // materials: Record<string, THREE.Material>

  return (
    <primitive object={gltf.scene} dispose={null} />
  );
}

function App() {
  return (
    <Suspense fallback={null}>
      <Model />
    </Suspense>
  );
}
```

---

## 9. Complete Typed Component Example

```tsx
import { useRef, useState, useCallback } from 'react';
import { useFrame, ThreeElements, ThreeEvent } from '@react-three/fiber';
import * as THREE from 'three';

type TorusKnotProps = ThreeElements['mesh'] & {
  speed?: number;
  hoverColor?: string;
};

export function TorusKnotMesh({
  speed = 1.0,
  hoverColor = 'hotpink',
  ...meshProps
}: TorusKnotProps) {
  const ref = useRef<THREE.Mesh<THREE.TorusKnotGeometry, THREE.MeshStandardMaterial>>(null);
  const [hovered, setHovered] = useState(false);
  const [active, setActive] = useState(false);

  useFrame(({ clock }: import('@react-three/fiber').RootState, delta: number) => {
    if (!ref.current) return;
    ref.current.rotation.x = clock.getElapsedTime() * speed * 0.3;
    ref.current.rotation.y += delta * speed * 0.5;
  });

  const handleClick = useCallback((e: ThreeEvent<MouseEvent>) => {
    e.stopPropagation();
    setActive((v) => !v);
  }, []);

  const handlePointerOver = useCallback((e: ThreeEvent<PointerEvent>) => {
    e.stopPropagation();
    setHovered(true);
    document.body.style.cursor = 'pointer';
  }, []);

  const handlePointerOut = useCallback((_e: ThreeEvent<PointerEvent>) => {
    setHovered(false);
    document.body.style.cursor = 'default';
  }, []);

  return (
    <mesh
      ref={ref}
      scale={active ? 1.5 : 1}
      onClick={handleClick}
      onPointerOver={handlePointerOver}
      onPointerOut={handlePointerOut}
      {...meshProps}
    >
      <torusKnotGeometry args={[1, 0.3, 128, 32]} />
      <meshStandardMaterial
        color={hovered ? hoverColor : 'orange'}
        roughness={0.3}
        metalness={0.7}
      />
    </mesh>
  );
}
```

---

## Sources

- [React Three Fiber TypeScript docs](https://r3f.docs.pmnd.rs/api/typescript) (accessed 2026-05-26)
- [React Three Fiber events docs](https://r3f.docs.pmnd.rs/api/events) (accessed 2026-05-26)
- [React Three Fiber hooks docs](https://r3f.docs.pmnd.rs/api/hooks) (accessed 2026-05-26)
- [pmndrs/react-three-fiber GitHub](https://github.com/pmndrs/react-three-fiber) (accessed 2026-05-26)
- [@react-three/fiber npm](https://www.npmjs.com/package/@react-three/fiber) (accessed 2026-05-26)
- [sbcode.net — R3F TypeScript](https://sbcode.net/react-three-fiber/typescript/) (accessed 2026-05-26)
- [sbcode.net — R3F extend](https://sbcode.net/react-three-fiber/extend/) (accessed 2026-05-26)
- [blog.pragmattic.dev — R3F WebGPU TypeScript](https://blog.pragmattic.dev/react-three-fiber-webgpu-typescript) (accessed 2026-05-26)
- [GitHub Gist — typed ShaderMaterial in R3F](https://gist.github.com/whoisryosuke/d91d5101d1c290aca784b2ae31eb7a24) (accessed 2026-05-26)
