# cannon-es Physics

cannon-es is a maintained fork of cannon.js — pure JavaScript physics (no WASM).
Repo: https://github.com/pmndrs/cannon-es

## When to Choose cannon-es vs Rapier

| Factor | cannon-es | Rapier |
|--------|-----------|--------|
| Startup time | Instant (no WASM) | ~100ms WASM init |
| Performance | Moderate | Excellent (Rust/WASM) |
| Bundle size | ~100KB | ~2.5MB WASM |
| Constraint types | Many | Many |
| Best for | Small scenes, quick prototypes | Production, large scenes |

## Installation

```bash
npm install cannon-es
```

## Basic Setup

```js
import * as CANNON from 'cannon-es';
import * as THREE from 'three';

// 1. Create world
const world = new CANNON.World({
  gravity: new CANNON.Vec3(0, -9.82, 0),
  broadphase: new CANNON.SAPBroadphase(),  // Sweep and Prune — faster
  allowSleep: true,  // Inactive bodies stop computing
});

world.solver.iterations = 10;  // Constraint solver quality (5-20)

// 2. Materials and contact behavior
const groundMaterial = new CANNON.Material('ground');
const ballMaterial   = new CANNON.Material('ball');

const contactMaterial = new CANNON.ContactMaterial(
  groundMaterial, ballMaterial,
  { friction: 0.4, restitution: 0.6 }
);
world.addContactMaterial(contactMaterial);

// 3. Ground
const groundBody = new CANNON.Body({
  type: CANNON.Body.STATIC,
  shape: new CANNON.Plane(),
  material: groundMaterial,
});
groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
world.addBody(groundBody);

// 4. Dynamic sphere
const sphereBody = new CANNON.Body({
  mass: 1,
  shape: new CANNON.Sphere(0.5),
  material: ballMaterial,
  position: new CANNON.Vec3(0, 5, 0),
});
world.addBody(sphereBody);

// 5. Sync loop
const sphereMesh = new THREE.Mesh(
  new THREE.SphereGeometry(0.5),
  new THREE.MeshStandardMaterial({ color: 0x0080ff })
);

const FIXED_STEP = 1 / 60;

function animate() {
  requestAnimationFrame(animate);

  world.fixedStep(FIXED_STEP);

  // Copy physics position to Three.js mesh
  sphereMesh.position.copy(sphereBody.position);
  sphereMesh.quaternion.copy(sphereBody.quaternion);

  renderer.render(scene, camera);
}
```

## Constraints

```js
// Point-to-point (ball joint)
const constraint = new CANNON.PointToPointConstraint(
  bodyA, new CANNON.Vec3(0, 1, 0),  // pivot on A
  bodyB, new CANNON.Vec3(0, -1, 0)  // pivot on B
);
world.addConstraint(constraint);

// Hinge
const hinge = new CANNON.HingeConstraint(
  bodyA, bodyB,
  {
    pivotA: new CANNON.Vec3(1, 0, 0),
    pivotB: new CANNON.Vec3(-1, 0, 0),
    axisA: new CANNON.Vec3(0, 1, 0),   // rotation axis
    axisB: new CANNON.Vec3(0, 1, 0),
  }
);
world.addConstraint(hinge);

// Distance
const dist = new CANNON.DistanceConstraint(bodyA, bodyB, 2.0);
world.addConstraint(dist);
```

## Sources
- cannon-es: https://github.com/pmndrs/cannon-es
- cannon-es docs: https://pmndrs.github.io/cannon-es/docs/
