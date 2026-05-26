# Rapier Physics Engine

Rapier is a Rust-compiled WASM physics engine. Industry standard for Three.js/R3F projects.
Repo: https://github.com/dimforge/rapier.js — Active as of 2026-05

## Installation

```bash
# R3F users
npm install @react-three/rapier

# Vanilla Three.js
npm install @dimforge/rapier3d
```

## Vite Config (Required)

```js
// vite.config.js
import wasm from 'vite-plugin-wasm';
import topLevelAwait from 'vite-plugin-top-level-await';

export default {
  plugins: [wasm(), topLevelAwait()],
  optimizeDeps: {
    exclude: ['@dimforge/rapier3d', '@dimforge/rapier2d'],
  },
};
```

## Vanilla Three.js + Rapier

```js
import RAPIER from '@dimforge/rapier3d';  // top-level await via plugin

async function initPhysics() {
  // 1. Init WASM
  await RAPIER.init();

  // 2. Create world
  const gravity = new RAPIER.Vector3(0, -9.81, 0);
  const world   = new RAPIER.World(gravity);

  // 3. Static ground (no movement)
  const groundDesc   = RAPIER.RigidBodyDesc.fixed().setTranslation(0, -0.5, 0);
  const groundBody   = world.createRigidBody(groundDesc);
  const groundCollider = world.createCollider(
    RAPIER.ColliderDesc.cuboid(10, 0.5, 10),
    groundBody
  );

  // 4. Dynamic box
  const boxDesc   = RAPIER.RigidBodyDesc.dynamic().setTranslation(0, 5, 0);
  const boxBody   = world.createRigidBody(boxDesc);
  const boxCollider = world.createCollider(
    RAPIER.ColliderDesc.cuboid(0.5, 0.5, 0.5),
    boxBody
  );

  // 5. Step physics each frame
  const boxMesh = new THREE.Mesh(
    new THREE.BoxGeometry(1, 1, 1),
    new THREE.MeshStandardMaterial({ color: 0x0080ff })
  );

  function animate() {
    world.step();  // Advance physics simulation

    const pos = boxBody.translation();
    const rot = boxBody.rotation();

    boxMesh.position.set(pos.x, pos.y, pos.z);
    boxMesh.quaternion.set(rot.x, rot.y, rot.z, rot.w);

    renderer.render(scene, camera);
    requestAnimationFrame(animate);
  }

  animate();
}
```

## React Three Fiber + @react-three/rapier

```jsx
import { Physics, RigidBody, CuboidCollider } from '@react-three/rapier';

function PhysicsScene() {
  return (
    <Physics gravity={[0, -9.81, 0]} debug={false}>
      {/* Static floor */}
      <RigidBody type="fixed">
        <mesh position={[0, -0.5, 0]}>
          <boxGeometry args={[20, 1, 20]} />
          <meshStandardMaterial color="#333" />
        </mesh>
      </RigidBody>

      {/* Dynamic ball */}
      <RigidBody
        position={[0, 5, 0]}
        restitution={0.7}    /* bounciness 0-1 */
        friction={0.5}       /* friction coefficient */
      >
        <mesh>
          <sphereGeometry args={[0.5]} />
          <meshStandardMaterial color="#ff6600" />
        </mesh>
      </RigidBody>

      {/* Custom collider shape */}
      <RigidBody>
        <CuboidCollider args={[1, 0.5, 1]} />
      </RigidBody>
    </Physics>
  );
}
```

## Collider Types

| Collider | Description | Cost |
|----------|-------------|------|
| `cuboid(hx, hy, hz)` | Box (half-extents) | Low |
| `ball(radius)` | Sphere | Low |
| `capsule(hy, radius)` | Capsule | Low |
| `cylinder(hy, radius)` | Cylinder | Low |
| `cone(hy, radius)` | Cone | Low |
| `convexHull(vertices)` | Convex mesh | Medium |
| `trimesh(verts, indices)` | Exact mesh shape | High — static only |
| `heightfield(...)` | Terrain | Medium |

## Performance Tips

```js
// Fixed timestep — deterministic, stable
const world = new RAPIER.World(gravity);
world.timestep = 1/60;  // 60 Hz physics (default)
world.timestep = 1/120; // 120 Hz for competitive physics

// Broad-phase: SAP (sweep and prune) for many objects
// Default in Rapier — no config needed

// Sleep inactive bodies automatically
// Rapier does this by default when velocity < threshold
```

## Sources
- Rapier docs: https://rapier.rs/docs/
- react-three/rapier: https://github.com/pmndrs/react-three-rapier
- Rapier JS: https://github.com/dimforge/rapier.js
