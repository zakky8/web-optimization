# drei — useGLTF, useTexture, useKTX2

**Sources:**  
- https://github.com/pmndrs/drei (v10.7.7, accessed 2026-05-26)  
- https://drei.docs.pmnd.rs/loaders/gltf-use-gltf (accessed 2026-05-26)  
- drei source: `src/core/Gltf.tsx`, `src/core/Texture.tsx`, `src/core/Ktx2.tsx` (read via GitHub API, 2026-05-26)

---

## useGLTF

### Function Signature (from source)

```ts
useGLTF<T extends string | string[]>(
  path: T,
  useDraco?: boolean | string,  // default: true
  useMeshopt?: boolean,          // default: true
  extendLoader?: (loader: GLTFLoader) => void
): T extends any[] ? (GLTF & ObjectMap)[] : GLTF & ObjectMap
```

### Return Type

```ts
// GLTF (from three-stdlib, mirrors THREE.GLTF):
interface GLTF {
  scene:      THREE.Group
  scenes:     THREE.Group[]
  animations: THREE.AnimationClip[]
  cameras:    THREE.Camera[]
  asset:      {
    copyright?:  string
    generator?:  string
    version?:    string
    minVersion?: string
    extensions?: any
    extras?:     any
  }
  parser:     GLTFParser
  userData:   any
}

// ObjectMap (added by @react-three/fiber's useLoader):
interface ObjectMap {
  nodes:     { [name: string]: THREE.Object3D }
  materials: { [name: string]: THREE.Material }
}
```

The `nodes` and `materials` maps are keyed by the object/material **name as set in Blender/DCC**. This is the primary way to destructure specific parts of a scene without traversal.

### Basic Usage

```tsx
import { useGLTF } from '@react-three/drei'

function Model() {
  const { scene, animations, nodes, materials } = useGLTF('/models/robot.glb')

  return <primitive object={scene} />
}
```

### Destructuring Named Nodes

```tsx
function Character() {
  const { nodes, materials } = useGLTF('/models/character.glb')

  return (
    <group>
      <mesh geometry={nodes.Body.geometry} material={materials.Skin} />
      <mesh geometry={nodes.Hair.geometry} material={materials.Hair} />
    </group>
  )
}
```

### Loading Multiple GLBs

Pass an array — returns an array in the same order:

```tsx
const [envGltf, charGltf] = useGLTF(['/models/env.glb', '/models/char.glb'])
```

---

## Suspense Integration

`useGLTF` is built on `useLoader` from `@react-three/fiber`, which throws a Promise when the asset is not yet loaded. React's Suspense boundary catches the throw and renders a fallback until the promise resolves.

```tsx
import { Suspense } from 'react'

function App() {
  return (
    <Canvas>
      <Suspense fallback={<LoadingPlaceholder />}>
        <Model />
      </Suspense>
    </Canvas>
  )
}
```

Without a `<Suspense>` boundary the component will throw an unhandled promise and unmount. Always wrap lazy-loaded components.

---

## useGLTF.preload()

Signature (from source):

```ts
useGLTF.preload = (
  path: string | string[],
  useDraco?: boolean | string,
  useMeshopt?: boolean,
  extendLoader?: (loader: GLTFLoader) => void
) => void
```

Call `preload()` **outside** any component — typically at module level — to start fetching and parsing before the component mounts. When the component renders and calls `useGLTF()`, the cached result is returned synchronously (no Suspense throw).

```tsx
// At module scope — fires immediately when this module loads
useGLTF.preload('/models/hero.glb')
useGLTF.preload(['/models/env.glb', '/models/props.glb'])

// In a route loader or navigation event handler:
function onUserHoverNextScene() {
  useGLTF.preload('/models/scene2.glb')
}
```

---

## Cache Clearing

```ts
useGLTF.clear = (path: string | string[]) => void
```

Removes the cached result from `useLoader`'s internal map. The next `useGLTF(path)` call will re-fetch and re-parse.

```tsx
// Clear a single asset (e.g. after a hot-reload or user-triggered asset swap)
useGLTF.clear('/models/hero.glb')

// Clear multiple
useGLTF.clear(['/models/a.glb', '/models/b.glb'])
```

---

## Global Draco Decoder Path

By default, the internal `DRACOLoader` singleton points to:

```
https://www.gstatic.com/draco/versioned/decoders/1.5.5/
```

Override this once at app startup to use a self-hosted decoder:

```ts
useGLTF.setDecoderPath('/draco/')
```

This sets the module-level `decoderPath` variable used by all subsequent `useGLTF` calls that have `useDraco = true`.

---

## Draco and Meshopt Options

```tsx
// Draco on (default) — uses internal singleton DRACOLoader
useGLTF('/model.glb')

// Custom draco decoder path for this call only
useGLTF('/model.glb', '/custom-draco/')

// Disable draco
useGLTF('/model.glb', false)

// Disable meshopt
useGLTF('/model.glb', true, false)

// Extend the loader (e.g. attach KTX2Loader manually)
useGLTF('/model.glb', true, true, (loader) => {
  loader.setKTX2Loader(myKtx2Loader)
})
```

---

## TypeScript — Typed Return Pattern

Use gltfjsx (CLI) to generate typed component stubs. Without it, manually extend the return:

```ts
import { GLTF } from 'three-stdlib'

type MyGLTF = GLTF & {
  nodes: {
    Body: THREE.Mesh
    Head: THREE.Mesh
  }
  materials: {
    Skin: THREE.MeshStandardMaterial
    Eyes: THREE.MeshStandardMaterial
  }
}

function Model() {
  const { nodes, materials } = useGLTF('/models/character.glb') as MyGLTF
  // nodes.Body.geometry is now typed as THREE.BufferGeometry
}
```

---

## useTexture

### Source: `src/core/Texture.tsx`

```ts
useTexture<Url extends string[] | string | Record<string, string>>(
  input: Url,
  onLoad?: (texture: MappedTextureType<Url>) => void
): MappedTextureType<Url>
```

Returns a `THREE.Texture`, `THREE.Texture[]`, or `{ [key]: THREE.Texture }` depending on input shape. Integrates with Suspense via `useLoader`.

**GPU upload on first render is triggered automatically** — the source calls `gl.initTexture()` to upload to GPU immediately rather than waiting for the first draw call. This avoids a frame hitch on first render.

```tsx
// Single texture
const map = useTexture('/textures/wood.jpg')

// Multiple textures
const [map, normalMap] = useTexture(['/textures/wood.jpg', '/textures/wood-normal.jpg'])

// Object form — keys become material props
const props = useTexture({
  map:          '/textures/diffuse.jpg',
  normalMap:    '/textures/normal.jpg',
  roughnessMap: '/textures/roughness.jpg',
})
return <meshStandardMaterial {...props} />
```

Preload:

```tsx
useTexture.preload('/textures/wood.jpg')
```

---

## useKTX2

### Source: `src/core/Ktx2.tsx`

```ts
useKTX2<Url extends string[] | string | Record<string, string>>(
  input: Url,
  basisPath?: string  // default: 'https://cdn.jsdelivr.net/gh/pmndrs/drei-assets@master/basis/'
): Url extends any[] ? Texture[] : Url extends object ? { [key in keyof Url]: Texture } : Texture
```

Wraps `useLoader(KTX2Loader, ...)`. Calls `loader.detectSupport(gl)` and `loader.setTranscoderPath(basisPath)` internally. Also calls `gl.initTexture()` to pre-upload to the GPU.

```tsx
// Single KTX2 texture (uses CDN transcoder by default)
const texture = useKTX2('/textures/diffuse.ktx2')

// With self-hosted transcoder
const texture = useKTX2('/textures/diffuse.ktx2', '/basis/')

// Multiple
const [diffuse, normal] = useKTX2(['/textures/diffuse.ktx2', '/textures/normal.ktx2'])

// Object form
const maps = useKTX2({ map: '/textures/diffuse.ktx2', normalMap: '/textures/normal.ktx2' })
return <meshStandardMaterial {...maps} />
```

Preload and clear:

```ts
useKTX2.preload('/textures/diffuse.ktx2', '/basis/')
useKTX2.clear('/textures/diffuse.ktx2')
```

---

## useLoader — The Underlying Primitive

`useGLTF`, `useTexture`, and `useKTX2` are all thin wrappers around `useLoader` from `@react-three/fiber`:

```ts
// Simplified internal signature
useLoader<T, U extends string | string[]>(
  Proto: { new(): Loader },
  input: U,
  extensions?: (loader: Loader) => void,
  onProgress?: (event: ProgressEvent) => void
): U extends string[] ? T[] : T
```

- Throws a Promise on first call (Suspense integration).
- Caches results by `[Proto, input]` key in a WeakMap.
- `useLoader.preload(Proto, input, extensions)` warms the cache.
- `useLoader.clear(Proto, input)` invalidates the cache entry.

---

## Comparison: useGLTF vs GLTFLoader Directly

| Concern | `useGLTF` (drei) | `GLTFLoader` directly |
|---|---|---|
| React integration | Suspense, automatic | Manual state management |
| Caching | Parsed scene cached | Raw bytes via `THREE.Cache` |
| Preloading | `useGLTF.preload()` | `THREE.Cache` + prefetch |
| Draco setup | Automatic singleton | Manual `setDRACOLoader` |
| Multi-scene apps | Natural with Suspense boundaries | Requires manual lifecycle |
| Non-React code | Cannot use | Preferred |
