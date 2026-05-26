# Typed Scene Objects in Three.js r184

> **Package versions used in this document**
> - `three`: `0.184.0`
> - `@types/three`: `0.184.0`
> - TypeScript: `>=5.0` (TypeScript 6 supported; see WebGPU note below)
> - Accessed: 2026-05-26

---

## 1. Package Setup

Three.js ships **without bundled type declarations**. Types are maintained separately at
[three-types/three-ts-types](https://github.com/three-types/three-ts-types) and published to
DefinitelyTyped. Install both packages:

```bash
npm install three
npm install --save-dev @types/three
```

> **status: verified** — Three.js `package.json` contains no `types` or `typings` field;
> `@types/three` is required as a separate dev dependency.
> Source: [three-types/three-ts-types](https://github.com/three-types/three-ts-types),
> [sbcode.net install guide](https://sbcode.net/threejs/install-threejs-and-types/),
> accessed 2026-05-26.

### r184 WebGPU / TypeScript 6 note

> **status: verified** — `@types/three@0.184.0` removed `@webgpu/types` as a peer dependency
> because WebGPU types are now included with TypeScript 6. If you are still on TypeScript 5 and
> need GPU types, install `@webgpu/types` separately.
> Source: [three-types/three-ts-types releases](https://github.com/three-types/three-ts-types/releases),
> accessed 2026-05-26.

---

## 2. Object3D Type Hierarchy

Every renderable object in Three.js extends `Object3D`. Understanding the hierarchy is the
prerequisite for writing correct type guards and narrowing traversal callbacks.

```
Object3D (base)
├── Mesh / InstancedMesh / BatchedMesh / SkinnedMesh
├── Line / LineLoop / LineSegments
├── Points
├── Sprite
├── Bone
├── Group
├── LOD
├── Camera (base)
│   ├── PerspectiveCamera
│   ├── OrthographicCamera
│   ├── ArrayCamera
│   └── CubeCamera / StereoCamera
└── Light (base)
    ├── AmbientLight
    ├── DirectionalLight
    ├── PointLight
    ├── SpotLight
    ├── HemisphereLight
    ├── RectAreaLight
    ├── IESSpotLight   (r184)
    ├── ProjectorLight (r184)
    └── LightProbe
```

> **status: verified** — hierarchy derived from Three.js r184 documentation index.
> Source: [threejs.org/docs](https://threejs.org/docs/), accessed 2026-05-26.

### Typed import strategy

Always import from the `three` namespace rather than sub-paths for r184:

```ts
import * as THREE from 'three';
// or named imports
import {
  Object3D,
  Mesh,
  InstancedMesh,
  Light,
  DirectionalLight,
  PerspectiveCamera,
  Group,
  BufferGeometry,
  Material,
  MeshStandardMaterial,
} from 'three';
```

---

## 3. Typed Refs

### Basic mesh ref

```ts
import { useRef, useEffect } from 'react';
import * as THREE from 'three';

const meshRef = useRef<THREE.Mesh>(null);

useEffect(() => {
  if (!meshRef.current) return; // strict null guard
  meshRef.current.rotation.y = Math.PI / 4;
  meshRef.current.castShadow = true;
}, []);
```

### Ref to a specific geometry/material combo

```ts
// Typing the generic parameters narrows geometry and material access
const meshRef = useRef<THREE.Mesh<THREE.BoxGeometry, THREE.MeshStandardMaterial>>(null);

useEffect(() => {
  if (!meshRef.current) return;
  // No cast needed — TypeScript knows the material type
  meshRef.current.material.color.set(0xff0000);
  meshRef.current.geometry.computeBoundingBox();
}, []);
```

### Ref to a group

```ts
const groupRef = useRef<THREE.Group>(null);

useEffect(() => {
  groupRef.current?.traverse((child) => {
    if (child instanceof THREE.Mesh) {
      child.castShadow = true;
    }
  });
}, []);
```

### Vanilla Three.js refs (non-React)

```ts
let renderer: THREE.WebGLRenderer;
let camera: THREE.PerspectiveCamera;
let scene: THREE.Scene;

function init(canvas: HTMLCanvasElement): void {
  renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
  camera = new THREE.PerspectiveCamera(75, canvas.width / canvas.height, 0.1, 1000);
  scene = new THREE.Scene();
}
```

---

## 4. Type Guards: `instanceof` vs `isMesh` / `isLight`

Three.js attaches boolean flags like `isMesh`, `isLight`, and `isCamera` to class prototypes.
These flags **do not narrow TypeScript types** on their own — accessing them on `Object3D`
produces a compiler error (`Property 'isMesh' does not exist on type 'Object3D'`).

> **status: verified** — confirmed as a known issue with a recommended workaround (`instanceof`).
> Source: [three.js forum discussion](https://discourse.threejs.org/t/gltf-scene-traverse-property-ismesh-does-not-exist-on-type-object3d/27212),
> [GitHub issue #15231](https://github.com/mrdoob/three.js/issues/15231), accessed 2026-05-26.
> Quote from donmccurdy: *"three.js' `isFoo` properties are difficult to use with TypeScript,
> just due to the nature of the type checking."*

### Recommended approach: `instanceof`

```ts
import { Object3D, Mesh, Light, DirectionalLight, Camera, PerspectiveCamera } from 'three';

function processNode(node: Object3D): void {
  if (node instanceof Mesh) {
    // TypeScript narrows to Mesh — geometry and material are accessible
    node.castShadow = true;
    node.geometry.computeBoundingBox();
  }

  if (node instanceof Light) {
    // Narrows to Light base — intensity, color available
    node.intensity = 2.0;
  }

  if (node instanceof DirectionalLight) {
    // Further narrowed — shadow, target, etc.
    node.shadow.mapSize.set(2048, 2048);
  }

  if (node instanceof PerspectiveCamera) {
    node.fov = 60;
    node.updateProjectionMatrix();
  }
}
```

### Fallback: type assertion with `isMesh`

Use only when `instanceof` is unreliable (e.g., objects crossing iframe boundaries or
`InterleavedBufferAttribute` patterns):

```ts
import { Mesh, Object3D } from 'three';

function isMeshGuard(obj: Object3D): obj is Mesh {
  return (obj as Mesh).isMesh === true;
}

// Usage
scene.traverse((child) => {
  if (isMeshGuard(child)) {
    child.material; // typed as Material | Material[]
  }
});
```

### Custom user-defined type predicates

For objects with known geometry/material types from a loaded GLTF:

```ts
import { Mesh, MeshStandardMaterial, BufferGeometry, Object3D } from 'three';

type StandardMesh = Mesh<BufferGeometry, MeshStandardMaterial>;

function isStandardMesh(obj: Object3D): obj is StandardMesh {
  return obj instanceof Mesh && obj.material instanceof MeshStandardMaterial;
}

function traverseGLTF(root: Object3D): void {
  root.traverse((child) => {
    if (isStandardMesh(child)) {
      // Full type safety: geometry, material, roughness, metalness all known
      child.material.roughness = 0.4;
      child.material.metalness = 0.8;
    }
  });
}
```

---

## 5. TypeScript Strict Null Checks with Three.js

### Enabling strict mode in `tsconfig.json`

```json
{
  "compilerOptions": {
    "strict": true,
    "strictNullChecks": true,
    "noImplicitAny": true,
    "target": "ES2022",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"]
  }
}
```

### Null-safe Object3D patterns

```ts
import * as THREE from 'three';

const scene = new THREE.Scene();

// getObjectByName returns Object3D | undefined — must guard
const target = scene.getObjectByName('hero');
if (target instanceof THREE.Mesh) {
  target.visible = false;
}

// getObjectById also returns Object3D | undefined
const light = scene.getObjectById(42);
if (light instanceof THREE.DirectionalLight) {
  light.shadow.camera.far = 500;
}
```

### Optional chaining in animation loops

```ts
let meshRef: THREE.Mesh | null = null;

function animate(delta: number): void {
  meshRef?.rotateY(delta);
  meshRef?.position.addScalar(0.001);
}
```

### Non-null assertion — use sparingly

The non-null assertion `!` tells TypeScript a value cannot be null/undefined at runtime.
Reserve it for cases where you have reliable initialization guarantees:

```ts
// Acceptable: canvas is always present if the app loaded
const canvas = document.getElementById('canvas') as HTMLCanvasElement;
const renderer = new THREE.WebGLRenderer({ canvas });

// Risky pattern — avoid unless you fully control initialization order
const meshRef = useRef<THREE.Mesh>(null!);
```

---

## 6. Traversal Patterns with Full Type Safety

### Collecting meshes from a loaded scene

```ts
import { Object3D, Mesh, BufferGeometry, Material } from 'three';

function collectMeshes(root: Object3D): Mesh[] {
  const meshes: Mesh[] = [];
  root.traverse((child) => {
    if (child instanceof Mesh) {
      meshes.push(child);
    }
  });
  return meshes;
}
```

### Building a typed node map from GLTF

```ts
import { Object3D, Mesh, Light, Camera } from 'three';

interface SceneIndex {
  meshes: Map<string, Mesh>;
  lights: Map<string, Light>;
  cameras: Map<string, Camera>;
}

function indexScene(root: Object3D): SceneIndex {
  const index: SceneIndex = {
    meshes: new Map(),
    lights: new Map(),
    cameras: new Map(),
  };

  root.traverse((child) => {
    if (child.name === '') return;

    if (child instanceof Mesh) {
      index.meshes.set(child.name, child);
    } else if (child instanceof Light) {
      index.lights.set(child.name, child);
    } else if (child instanceof Camera) {
      index.cameras.set(child.name, child);
    }
  });

  return index;
}
```

### Filtering children by type with generics

```ts
import { Object3D } from 'three';

function getChildrenOfType<T extends Object3D>(
  parent: Object3D,
  ctor: new (...args: never[]) => T
): T[] {
  return parent.children.filter((c): c is T => c instanceof ctor);
}

// Usage
import { Mesh, DirectionalLight } from 'three';

const meshes = getChildrenOfType(scene, Mesh);
const lights = getChildrenOfType(scene, DirectionalLight);
```

---

## 7. Typed `userData` Storage

Three.js types `userData` as `{ [key: string]: unknown }` (or a plain record) in r184.
To attach typed data, use a wrapper or casting:

```ts
import { Mesh, BufferGeometry, Material } from 'three';

// Define your application-specific payload
interface GameEntityData {
  entityId: string;
  health: number;
  faction: 'player' | 'enemy' | 'neutral';
}

function tagMesh(mesh: Mesh, data: GameEntityData): void {
  // Cast to your interface when writing
  (mesh.userData as GameEntityData) = { ...data };
}

function readMeshData(mesh: Mesh): GameEntityData | null {
  const d = mesh.userData as Partial<GameEntityData>;
  if (d.entityId !== undefined && d.health !== undefined && d.faction !== undefined) {
    return d as GameEntityData;
  }
  return null;
}
```

For typed module augmentation alternatives, see `type-augmentation.md`.

---

## Sources

- [three-types/three-ts-types](https://github.com/three-types/three-ts-types) — official Three.js TypeScript definitions repository (accessed 2026-05-26)
- [Three.js r184 release notes](https://github.com/mrdoob/three.js/releases/tag/r184) (accessed 2026-05-26)
- [Three.js Object3D docs](https://threejs.org/docs/#api/en/core/Object3D) (accessed 2026-05-26)
- [isMesh type guard forum thread](https://discourse.threejs.org/t/gltf-scene-traverse-property-ismesh-does-not-exist-on-type-object3d/27212) (accessed 2026-05-26)
- [GitHub issue #15231 — isMesh TypeScript](https://github.com/mrdoob/three.js/issues/15231) (accessed 2026-05-26)
- [@types/three npm](https://www.npmjs.com/package/@types/three) (accessed 2026-05-26)
- [sbcode.net — Install three.js and types](https://sbcode.net/threejs/install-threejs-and-types/) (accessed 2026-05-26)
