# Extending Three.js Types: Module Augmentation and Custom Properties

> **Package versions used in this document**
> - `three`: `0.184.0`
> - `@types/three`: `0.184.0`
> - `@react-three/fiber`: `^9.x`
> - TypeScript: `>=5.0`
> - Accessed: 2026-05-26

---

## 1. The Core Limitation: Why Module Augmentation Is Constrained

Standard TypeScript module augmentation (`declare module 'three' { ... }`) has a known
limitation with Three.js: because `three/src/Three.ts` uses re-exports through a barrel
file, TypeScript cannot always merge declarations for class prototypes imported via named
imports such as `import { Object3D } from 'three'`.

> **status: verified** — confirmed as a TypeScript compiler design limitation (microsoft/TypeScript#9532).
> The root cause is that re-exported names from a barrel file don't satisfy the identity
> requirements for declaration merging on classes.
> Source: [GitHub issue #19468 — mrdoob/three.js](https://github.com/mrdoob/three.js/issues/19468),
> accessed 2026-05-26.

**What you can augment** in `declare module 'three'`:
- Interfaces (e.g., `EventMap` interfaces, `Object3DEventMap`)
- New interface declarations within the module namespace

**What you cannot reliably augment**:
- Adding new methods or properties to existing Three.js *classes* via prototype augmentation
- Extending `Object3D.prototype.myMethod` with type safety in a way the compiler honors across the project

The practical workarounds documented below work around this constraint without patching `node_modules`.

---

## 2. Typed `userData` — The Wrapper Pattern

Three.js types `userData` as `Record<string, unknown>` (loosely typed by design, since it is
a general-purpose scratchpad). The safest typed approach uses a typed accessor function rather
than relying on module augmentation:

```ts
// src/utils/userData.ts
import type { Object3D } from 'three';

// Define whatever application-level shapes your objects carry
export interface EntityData {
  entityId: string;
  health: number;
  maxHealth: number;
  faction: 'player' | 'enemy' | 'neutral';
  spawnTime: number;
}

export interface PickupData {
  pickupType: 'health' | 'ammo' | 'weapon';
  value: number;
  collected: boolean;
}

// Typed write
export function setEntityData(obj: Object3D, data: EntityData): void {
  obj.userData['entity'] = data;
}

// Typed read with runtime validation
export function getEntityData(obj: Object3D): EntityData | null {
  const d = obj.userData['entity'] as Partial<EntityData> | undefined;
  if (
    d &&
    typeof d.entityId === 'string' &&
    typeof d.health === 'number' &&
    d.faction !== undefined
  ) {
    return d as EntityData;
  }
  return null;
}

// Typed predicate (for filtering traversal results)
export function hasEntityData(obj: Object3D): boolean {
  return getEntityData(obj) !== null;
}
```

### Usage in a scene traversal

```ts
import { Object3D } from 'three';
import { setEntityData, getEntityData } from './utils/userData';

const hero = scene.getObjectByName('Hero')!;
setEntityData(hero, {
  entityId: 'hero-001',
  health: 100,
  maxHealth: 100,
  faction: 'player',
  spawnTime: performance.now(),
});

scene.traverse((child: Object3D) => {
  const data = getEntityData(child);
  if (data && data.health <= 0) {
    child.visible = false;
  }
});
```

---

## 3. Module Augmentation for `EventMap` Interfaces

Adding custom events to Three.js objects via typed EventDispatcher works by augmenting the
`*EventMap` interfaces — this pattern **does** work with current `@types/three`:

> **status: verified** — the `EventDispatcher<TEventMap>` generic and interface augmentation
> for `Object3DEventMap` was introduced via PR #369 in three-types/three-ts-types.
> Source: [three-types/three-ts-types PR #369](https://github.com/three-types/three-ts-types/pull/369),
> accessed 2026-05-26.

### Typed `EventDispatcher` base pattern

```ts
import { EventDispatcher } from 'three';

// Define a custom event map
interface GameObjectEventMap {
  damaged: { amount: number; source: string };
  healed: { amount: number };
  destroyed: {};
  stateChanged: { from: string; to: string };
}

class GameObject extends EventDispatcher<GameObjectEventMap> {
  private _health = 100;

  damage(amount: number, source: string): void {
    this._health -= amount;
    // Type-checked dispatch — `damaged` must match GameObjectEventMap['damaged']
    this.dispatchEvent({ type: 'damaged', amount, source });

    if (this._health <= 0) {
      this.dispatchEvent({ type: 'destroyed' });
    }
  }

  heal(amount: number): void {
    this._health = Math.min(100, this._health + amount);
    this.dispatchEvent({ type: 'healed', amount });
  }
}

// Type-checked listener — TypeScript knows the event shape
const obj = new GameObject();

obj.addEventListener('damaged', (e) => {
  // e.amount and e.source are fully typed
  console.log(`Took ${e.amount} damage from ${e.source}`);
});

obj.addEventListener('destroyed', (_e) => {
  console.log('Destroyed');
});

// This would be a compile error:
// obj.dispatchEvent({ type: 'damaged', badProp: true }); // Error
```

### Extending `Object3DEventMap` for scene objects

```ts
// src/types/three-events.d.ts

declare module 'three' {
  interface Object3DEventMap {
    // Add custom events to all Object3D descendants
    selected: { previouslySelected: boolean };
    deselected: {};
    healthChanged: { newHealth: number; delta: number };
  }
}
```

```ts
// Usage
import { Mesh, BoxGeometry, MeshStandardMaterial } from 'three';

const mesh = new Mesh(new BoxGeometry(), new MeshStandardMaterial());

// Now type-safe on any Object3D
mesh.addEventListener('selected', (e) => {
  console.log(e.previouslySelected); // boolean — typed
});

mesh.dispatchEvent({ type: 'healthChanged', newHealth: 80, delta: -20 });
```

### Inline generic extension (no global side-effects)

```ts
import { Object3D, Object3DEventMap } from 'three';

type SelectableEventMap = Object3DEventMap & {
  selected: { previouslySelected: boolean };
  deselected: {};
};

// Only this instance carries the extended event map
class SelectableObject3D extends Object3D<SelectableEventMap> {
  private _selected = false;

  select(): void {
    const was = this._selected;
    this._selected = true;
    this.dispatchEvent({ type: 'selected', previouslySelected: was });
  }

  deselect(): void {
    this._selected = false;
    this.dispatchEvent({ type: 'deselected' });
  }
}
```

---

## 4. Adding Custom Properties to Three.js Classes via Subclassing

Since prototype augmentation is unreliable, subclassing is the idiomatic approach for adding
typed properties and methods:

```ts
import { Mesh, BufferGeometry, Material, Object3D, Object3DEventMap } from 'three';

// Define custom event map
interface InteractableMeshEventMap extends Object3DEventMap {
  focus: {};
  blur: {};
  interact: { interactorId: string };
}

// Define custom properties interface for the extended class
interface InteractableState {
  focusable: boolean;
  focused: boolean;
  interactionRadius: number;
  lastInteractionAt: number | null;
}

class InteractableMesh<
  TGeometry extends BufferGeometry = BufferGeometry,
  TMaterial extends Material | Material[] = Material,
> extends Mesh<TGeometry, TMaterial, InteractableMeshEventMap> {
  // Custom typed properties
  readonly isInteractableMesh = true;
  focusable: boolean;
  focused = false;
  interactionRadius: number;
  lastInteractionAt: number | null = null;

  constructor(
    geometry?: TGeometry,
    material?: TMaterial,
    options: Partial<InteractableState> = {}
  ) {
    super(geometry, material);
    this.focusable = options.focusable ?? true;
    this.interactionRadius = options.interactionRadius ?? 2.0;
  }

  focus(): void {
    if (!this.focusable || this.focused) return;
    this.focused = true;
    this.dispatchEvent({ type: 'focus' });
  }

  blur(): void {
    if (!this.focused) return;
    this.focused = false;
    this.dispatchEvent({ type: 'blur' });
  }

  interact(interactorId: string): void {
    this.lastInteractionAt = performance.now();
    this.dispatchEvent({ type: 'interact', interactorId });
  }

  // Type guard for external use
  static is(obj: Object3D): obj is InteractableMesh {
    return (obj as InteractableMesh).isInteractableMesh === true;
  }
}

export { InteractableMesh };
```

### Using the subclass with React Three Fiber

```ts
// src/types/r3f-extensions.d.ts
import { ThreeElement } from '@react-three/fiber';
import { InteractableMesh } from '../InteractableMesh';

declare module '@react-three/fiber' {
  interface ThreeElements {
    interactableMesh: ThreeElement<typeof InteractableMesh>;
  }
}
```

```tsx
// In a component file
import { extend, ThreeEvent } from '@react-three/fiber';
import { InteractableMesh } from './InteractableMesh';
import { useRef } from 'react';

extend({ InteractableMesh });

function InteractableBox() {
  const ref = useRef<InteractableMesh>(null);

  const handleClick = (e: ThreeEvent<MouseEvent>) => {
    e.stopPropagation();
    ref.current?.interact('player-001');
  };

  const handlePointerOver = (e: ThreeEvent<PointerEvent>) => {
    e.stopPropagation();
    ref.current?.focus();
  };

  const handlePointerOut = (_e: ThreeEvent<PointerEvent>) => {
    ref.current?.blur();
  };

  return (
    <interactableMesh
      ref={ref}
      args={[undefined, undefined, { interactionRadius: 3.0 }]}
      onClick={handleClick}
      onPointerOver={handlePointerOver}
      onPointerOut={handlePointerOut}
    >
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color="royalblue" />
    </interactableMesh>
  );
}
```

---

## 5. Extending R3F's `ThreeElements` for Custom Classes

> **status: verified** — `ThreeElement<typeof CustomClass>` and module augmentation of
> `ThreeElements` is the official pattern for registering custom elements in R3F v9.
> Source: [r3f.docs.pmnd.rs/api/typescript](https://r3f.docs.pmnd.rs/api/typescript),
> accessed 2026-05-26.

```ts
// src/types/r3f-catalog.d.ts
import { ThreeElement } from '@react-three/fiber';
import { CustomGrid } from '../scene/CustomGrid';
import { HologramMaterial } from '../materials/HologramMaterial';
import { InteractableMesh } from '../objects/InteractableMesh';
import { ProbeVolume } from '../lighting/ProbeVolume';

declare module '@react-three/fiber' {
  interface ThreeElements {
    customGrid:        ThreeElement<typeof CustomGrid>;
    hologramMaterial:  ThreeElement<typeof HologramMaterial>;
    interactableMesh:  ThreeElement<typeof InteractableMesh>;
    probeVolume:       ThreeElement<typeof ProbeVolume>;
  }
}
```

---

## 6. Typed Utility Wrappers — Alternative to Augmentation

When module augmentation is impractical, a typed wrapper class achieves the same DX:

```ts
import { Object3D } from 'three';

// Typed tag/metadata store that wraps any Object3D
class TypedObject3D<TData extends Record<string, unknown> = Record<string, unknown>> {
  readonly object: Object3D;
  data: TData;

  constructor(object: Object3D, initialData: TData) {
    this.object = object;
    this.data = initialData;
  }

  get name(): string {
    return this.object.name;
  }

  updateData(partial: Partial<TData>): void {
    this.data = { ...this.data, ...partial };
  }
}

// Usage with a specific data shape
interface EnemyData {
  enemyType: 'grunt' | 'elite' | 'boss';
  tier: 1 | 2 | 3;
  patrolRadius: number;
}

const enemy = new TypedObject3D(
  scene.getObjectByName('EnemyA')!,
  {
    enemyType: 'grunt' as const,
    tier: 1 as const,
    patrolRadius: 5.0,
  }
);

enemy.updateData({ patrolRadius: 10.0 });
console.log(enemy.data.enemyType); // 'grunt' — fully typed
```

---

## 7. Typed Scene Graph Registry

A registry pattern gives you a project-wide typed map of named scene objects:

```ts
// src/SceneRegistry.ts
import * as THREE from 'three';

interface RegistryMap {
  [name: string]: THREE.Object3D;
}

type ExtractType<TMap extends RegistryMap, K extends keyof TMap> = TMap[K];

class SceneRegistry<TMap extends RegistryMap = RegistryMap> {
  private map = new Map<keyof TMap, TMap[keyof TMap]>();

  register<K extends keyof TMap>(name: K, object: TMap[K]): void {
    this.map.set(name, object);
  }

  get<K extends keyof TMap>(name: K): TMap[K] | undefined {
    return this.map.get(name) as TMap[K] | undefined;
  }

  require<K extends keyof TMap>(name: K): TMap[K] {
    const obj = this.map.get(name) as TMap[K] | undefined;
    if (!obj) throw new Error(`SceneRegistry: object '${String(name)}' not registered`);
    return obj;
  }
}

// Define the registry shape for your scene
interface AppSceneObjects {
  heroMesh:    THREE.SkinnedMesh;
  mainCamera:  THREE.PerspectiveCamera;
  sunLight:    THREE.DirectionalLight;
  environment: THREE.Group;
}

const registry = new SceneRegistry<AppSceneObjects>();

// After loading:
registry.register('heroMesh',    gltf.scene.getObjectByName('Hero') as THREE.SkinnedMesh);
registry.register('sunLight',    scene.getObjectByName('Sun') as THREE.DirectionalLight);

// Usage — fully typed, no cast needed at call sites
const hero = registry.require('heroMesh');   // THREE.SkinnedMesh
const sun  = registry.require('sunLight');   // THREE.DirectionalLight
sun.shadow.mapSize.set(2048, 2048);          // type-safe property access
```

---

## 8. Summary: Pattern Selection Guide

| Goal | Recommended Pattern |
|------|-------------------|
| Type-safe `userData` access | Typed accessor functions (`getEntityData`, `setEntityData`) |
| Add custom events to all `Object3D` | Augment `Object3DEventMap` in `declare module 'three'` |
| Custom events on a specific class | `class Foo extends Object3D<MyEventMap>` |
| Add typed properties/methods to a class | Subclass (`class InteractableMesh extends Mesh`) |
| Register custom class in R3F JSX | Augment `ThreeElements` in `declare module '@react-three/fiber'` |
| Scene-wide typed object catalog | `SceneRegistry<TMap>` pattern |
| One-off typed wrapper | `TypedObject3D<TData>` wrapper class |

---

## Sources

- [GitHub issue #19468 — module augmentation impossible](https://github.com/mrdoob/three.js/issues/19468) (accessed 2026-05-26)
- [three-types/three-ts-types PR #369 — typed EventDispatcher](https://github.com/three-types/three-ts-types/pull/369) (accessed 2026-05-26)
- [Three.js forum — module augmentation Object3D](https://discourse.threejs.org/t/typescript-module-augmentation-add-methods-and-properties-to-object3d/1934) (accessed 2026-05-26)
- [Three.js GitHub issue #13425](https://github.com/mrdoob/three.js/issues/13425) (accessed 2026-05-26)
- [Three.js Object3D userData docs](https://threejs.org/docs/#api/en/core/Object3D) (accessed 2026-05-26)
- [R3F TypeScript docs](https://r3f.docs.pmnd.rs/api/typescript) (accessed 2026-05-26)
- [three-types/three-ts-types GitHub](https://github.com/three-types/three-ts-types) (accessed 2026-05-26)
