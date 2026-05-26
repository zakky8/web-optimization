# Physics and Interaction

Adding physics to a 3D scene enables collisions, gravity, constraints, and realistic object behavior. The two main choices for web are Rapier.js (Rust-compiled WASM, maintained) and Cannon-es (JS fork of the abandoned cannon.js). Both integrate with Three.js and React Three Fiber.

## Choosing an Engine

| Feature | Rapier | Cannon-es |
|---|---|---|
| Language | Rust → WASM | JavaScript |
| Performance | Excellent | Good |
| Bundle size | ~650 KB WASM | ~80 KB JS |
| Maintenance | Active (Dimforge) | Community fork |
| Determinism | Yes | No |
| Joints | Full constraint system | Basic |
| R3F integration | @react-three/rapier | use-cannon |
| Best for | Games, complex sims | Light interactions |

## Rapier.js

### Setup

```bash
npm install @dimforge/rapier3d-compat
```

```js
import RAPIER from "@dimforge/rapier3d-compat";
await RAPIER.init(); // loads WASM

const gravity = { x: 0.0, y: -9.81, z: 0.0 };
const world   = new RAPIER.World(gravity);
```

### Rigid Bodies

```js
// Dynamic body (falls under gravity)
const bodyDesc = RAPIER.RigidBodyDesc.dynamic()
  .setTranslation(0.0, 5.0, 0.0);
const body = world.createRigidBody(bodyDesc);

// Attach a collider
const colliderDesc = RAPIER.ColliderDesc.ball(0.5);   // radius
world.createCollider(colliderDesc, body);

// Fixed body (immovable)
const floorDesc = RAPIER.RigidBodyDesc.fixed()
  .setTranslation(0.0, -1.0, 0.0);
const floor = world.createRigidBody(floorDesc);
world.createCollider(RAPIER.ColliderDesc.cuboid(5.0, 0.1, 5.0), floor);

// Step the simulation (call in render loop)
world.step();

// Sync Three.js mesh with physics body
const pos = body.translation();
mesh.position.set(pos.x, pos.y, pos.z);
const rot = body.rotation();
mesh.quaternion.set(rot.x, rot.y, rot.z, rot.w);
```

### Complete Simulation Loop

```js
const bodies = [];
const meshes = [];

function addBall(x, y, z) {
  const desc = RAPIER.RigidBodyDesc.dynamic().setTranslation(x, y, z);
  const body = world.createRigidBody(desc);
  world.createCollider(RAPIER.ColliderDesc.ball(0.3), body);

  const mesh = new THREE.Mesh(
    new THREE.SphereGeometry(0.3),
    new THREE.MeshStandardMaterial({ color: 0xff6600 })
  );
  scene.add(mesh);
  bodies.push(body);
  meshes.push(mesh);
}

renderer.setAnimationLoop(() => {
  world.step();

  for (let i = 0; i < bodies.length; i++) {
    const t = bodies[i].translation();
    const r = bodies[i].rotation();
    meshes[i].position.set(t.x, t.y, t.z);
    meshes[i].quaternion.set(r.x, r.y, r.z, r.w);
  }

  renderer.render(scene, camera);
});
```

### React Three Fiber + @react-three/rapier

```jsx
import { Physics, RigidBody, CuboidCollider } from "@react-three/rapier";

function Scene() {
  return (
    <Physics gravity={[0, -9.81, 0]}>
      {/* Dynamic sphere */}
      <RigidBody position={[0, 5, 0]} restitution={0.8}>
        <mesh>
          <sphereGeometry args={[0.5]} />
          <meshStandardMaterial color="orange" />
        </mesh>
      </RigidBody>

      {/* Floor */}
      <RigidBody type="fixed">
        <CuboidCollider args={[5, 0.1, 5]} position={[0, -1, 0]} />
        <mesh position={[0, -1, 0]}>
          <boxGeometry args={[10, 0.2, 10]} />
          <meshStandardMaterial color="#444" />
        </mesh>
      </RigidBody>
    </Physics>
  );
}
```

## Cannon-es

### Setup

```bash
npm install cannon-es
```

```js
import * as CANNON from "cannon-es";

const world = new CANNON.World({ gravity: new CANNON.Vec3(0, -9.82, 0) });
world.broadphase = new CANNON.SAPBroadphase(world); // faster than default NaiveBroadphase

// Material and contact behavior
const groundMaterial = new CANNON.Material("ground");
const ballMaterial   = new CANNON.Material("ball");
const contact = new CANNON.ContactMaterial(groundMaterial, ballMaterial, {
  friction: 0.3,
  restitution: 0.6,
});
world.addContactMaterial(contact);
```

```js
// Sphere
const body = new CANNON.Body({
  mass: 1,
  material: ballMaterial,
  shape: new CANNON.Sphere(0.5),
  position: new CANNON.Vec3(0, 5, 0),
});
world.addBody(body);

// Fixed plane (ground)
const ground = new CANNON.Body({
  mass: 0,
  material: groundMaterial,
  shape: new CANNON.Plane(),
});
ground.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
world.addBody(ground);

const clock = new THREE.Clock();
renderer.setAnimationLoop(() => {
  const delta = Math.min(clock.getDelta(), 0.05);
  world.fixedStep(1 / 60, delta);

  mesh.position.copy(body.position);
  mesh.quaternion.copy(body.quaternion);
  renderer.render(scene, camera);
});
```

## Raycasting

Raycasting maps screen coordinates to 3D world positions. Use it for hover detection, click picking, and drag interactions.

### Three.js Raycaster

```js
const raycaster = new THREE.Raycaster();
const pointer   = new THREE.Vector2();

window.addEventListener("pointermove", (e) => {
  // Normalize to -1..1
  pointer.x = (e.clientX / window.innerWidth)  *  2 - 1;
  pointer.y = (e.clientY / window.innerHeight) * -2 + 1;
});

renderer.setAnimationLoop(() => {
  raycaster.setFromCamera(pointer, camera);
  const hits = raycaster.intersectObjects(scene.children, true);

  if (hits.length > 0) {
    const hit = hits[0];
    // hit.point    -> THREE.Vector3 world position
    // hit.object   -> THREE.Object3D
    // hit.distance -> distance from camera
    hit.object.material.color.set(0xff0000);
  }

  renderer.render(scene, camera);
});
```

### Raycasting with Rapier (Physics Queries)

```js
// Cast a ray in physics world (ignores visual geometry)
const origin    = { x: 0, y: 10, z: 0 };
const direction = { x: 0, y: -1, z: 0 };
const maxToi    = 50.0;
const solid     = true;

const hit = world.castRay(
  new RAPIER.Ray(origin, direction),
  maxToi,
  solid
);

if (hit) {
  const point = {
    x: origin.x + direction.x * hit.timeOfImpact,
    y: origin.y + direction.y * hit.timeOfImpact,
    z: origin.z + direction.z * hit.timeOfImpact,
  };
  console.log("Physics hit at", point);
}
```

## Drag Controls

### Three.js DragControls

```js
import { DragControls } from "three/addons/controls/DragControls.js";

const draggables = [mesh1, mesh2, mesh3];
const controls  = new DragControls(draggables, camera, renderer.domElement);

controls.addEventListener("dragstart", (e) => {
  orbitControls.enabled = false; // disable orbit while dragging
  e.object.material.opacity = 0.5;
});
controls.addEventListener("drag", (e) => {
  // e.object.position is already updated
  // Constrain to horizontal plane:
  e.object.position.y = 0;
});
controls.addEventListener("dragend", (e) => {
  orbitControls.enabled = true;
  e.object.material.opacity = 1.0;
});
```

### Physics-Aware Drag (Rapier)

```js
// Technique: disable body gravity while dragging, move via setNextKinematicTranslation
let dragging = null;

controls.addEventListener("dragstart", (e) => {
  const body = bodyMap.get(e.object);
  if (body) {
    body.setBodyType(RAPIER.RigidBodyType.KinematicPositionBased);
    dragging = body;
  }
});

controls.addEventListener("drag", (e) => {
  if (dragging) {
    const p = e.object.position;
    dragging.setNextKinematicTranslation({ x: p.x, y: p.y, z: p.z });
  }
});

controls.addEventListener("dragend", () => {
  if (dragging) {
    dragging.setBodyType(RAPIER.RigidBodyType.Dynamic);
    dragging = null;
  }
});
```

## Performance Implications

- **Fixed time step** (`world.fixedStep` in cannon-es, `world.step()` with Rapier): decouple physics from render rate. Cap delta at ~50ms to prevent spiral of death.
- **Broadphase selection**: SAP (Sweep and Prune) is O(n log n) vs Naive O(n^2). Use SAP for scenes with more than ~20 bodies.
- **Sleep thresholds**: bodies below a velocity threshold are marked "sleeping" and excluded from collision checks. Tune `sleepSpeedThreshold`.
- **Collider geometry**: convex hulls and primitives (box, sphere, capsule) are orders of magnitude cheaper than trimesh colliders. Reserve trimeshes for static scenery.
- **Off-thread physics (Rapier)**: Rapier's WASM module can run on a Web Worker with `SharedArrayBuffer` + Atomics. Saves 5–15ms/frame on complex scenes.
- **Raycaster optimization**: pass only the candidate objects to `intersectObjects`, not the entire scene. Use layers to filter.

## Tools and Libraries

| Tool | Purpose |
|---|---|
| [Rapier](https://rapier.rs) | Primary physics engine |
| [@react-three/rapier](https://github.com/pmndrs/react-three-rapier) | R3F bindings for Rapier |
| [cannon-es](https://github.com/pmndrs/cannon-es) | Lighter JS physics |
| [use-cannon](https://github.com/pmndrs/use-cannon) | R3F bindings for cannon-es |
| [Three.js DragControls](https://threejs.org/docs/#examples/en/controls/DragControls) | Built-in drag interaction |
| [Three.js Raycaster](https://threejs.org/docs/#api/en/core/Raycaster) | Built-in ray picking |
