# Spring-Based Camera Follow

## lerp vs damp — why lerp breaks at variable frame rates

### The lerp trap

A naive follow camera uses a fixed lerp factor per frame:

```js
// BAD — frame-rate dependent
camera.position.lerp(targetPosition, 0.1);
```

At 60 fps the camera closes ~65% of the gap per second. At 30 fps it closes ~46%. At 120 fps it overshoots visually snapping. The feel of the camera changes with hardware.

### THREE.MathUtils.damp — frame-rate independent lerp

`THREE.MathUtils.damp` implements the Zeno's paradox formula correctly with a delta time argument:

```js
// MathUtils.damp(x, y, lambda, dt)
// x       = current value
// y       = target value
// lambda  = decay rate (higher = faster / snappier)
// dt      = seconds elapsed since last frame
camera.position.x = THREE.MathUtils.damp(
  camera.position.x,
  target.x,
  4.0,   // lambda: ~4 = medium lag, ~10 = tight follow, ~1 = very floaty
  delta  // clock.getDelta()
);
```

The formula inside is `x + (y - x) * (1 - Math.exp(-lambda * dt))`. For any constant `lambda`, the approach curve is identical regardless of framerate.

**Use `damp` everywhere you previously used `lerp(factor)` in a render loop.**

---

## Velocity-Based Spring (second-order spring)

A true spring adds a velocity term, producing overshoot and oscillation that reads as physical weight.

### Spring state

```js
const springState = {
  position: new THREE.Vector3(),
  velocity: new THREE.Vector3(),
};
```

### Spring update function

```js
/**
 * Second-order spring update (per-axis, call once per frame).
 * @param {THREE.Vector3} current   - current value (modified in place)
 * @param {THREE.Vector3} velocity  - current velocity (modified in place)
 * @param {THREE.Vector3} target    - desired value
 * @param {number} stiffness        - spring constant k (e.g. 100–400)
 * @param {number} damping          - damping coefficient c (e.g. 10–30)
 * @param {number} dt               - delta time in seconds
 */
function springUpdate(current, velocity, target, stiffness, damping, dt) {
  // F = -k * displacement - c * velocity
  const displacement = current.clone().sub(target);
  const force = displacement.multiplyScalar(-stiffness)
                            .addScaledVector(velocity, -damping);

  // Semi-implicit Euler integration
  velocity.addScaledVector(force, dt);
  current.addScaledVector(velocity, dt);
}
```

### Critical damping — no oscillation, fastest settle

Critical damping coefficient: `c = 2 * sqrt(k * mass)`. With mass = 1:

```js
const stiffness = 200;
const damping   = 2 * Math.sqrt(stiffness); // ≈ 28.3 — critically damped
```

Below this value: underdamped (bounces). Above: overdamped (slow creep). For game cameras, slightly underdamped (`damping * 0.85`) adds life without annoying jitter.

### Usage in render loop

```js
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  const dt = Math.min(clock.getDelta(), 0.05); // clamp to avoid explosion on tab refocus

  springUpdate(
    springState.position,
    springState.velocity,
    player.position,
    200,   // stiffness
    24,    // damping (slightly underdamped for feel)
    dt
  );

  camera.position.copy(springState.position).add(cameraOffset);
  camera.lookAt(player.position);
  renderer.render(scene, camera);
}
```

---

## Look-Ahead Offset

A static follow camera feels dead — it always lags behind. Look-ahead shifts the camera target in the direction of travel so the player sees where they're going.

```js
const playerVelocity = new THREE.Vector3(); // updated by your movement system
const lookAheadFactor = 2.5;               // world units per unit of speed
const lookAheadTarget = new THREE.Vector3();

// In your update:
lookAheadTarget.copy(player.position)
               .addScaledVector(playerVelocity.clone().normalize(), 
                                playerVelocity.length() * lookAheadFactor);

// Feed lookAheadTarget to the spring instead of player.position
springUpdate(camTarget, camTargetVel, lookAheadTarget, 80, 18, dt);
camera.lookAt(camTarget);
```

Tune `lookAheadFactor` relative to your world scale. Too high and the camera swings wildly on turns; too low and it feels like the original static follow.

---

## Camera Lag for Game Feel

Camera lag is the deliberate delay between character movement and camera response. It is a design tool, not a bug.

| Lag amount | Feel | Typical use |
|---|---|---|
| Near zero (lambda ~15) | Tight, responsive | Fast-paced FPS, racing |
| Medium (lambda ~5–8) | Weighted, cinematic | Third-person action |
| Heavy (lambda ~2–3) | Floaty, dreamy | Exploration, puzzle |

### Separate lag for position vs rotation

Position lag and rotation lag should be tuned independently:

```js
// Position: follows with lag
camera.position.x = THREE.MathUtils.damp(camera.position.x, target.x, 5, dt);
camera.position.y = THREE.MathUtils.damp(camera.position.y, target.y + 2, 8, dt);
camera.position.z = THREE.MathUtils.damp(camera.position.z, target.z + 6, 5, dt);

// Rotation: snap the look-at with its own lag
lerpedLookAt.lerp(player.position, 1 - Math.exp(-6 * dt));
camera.lookAt(lerpedLookAt);
```

Y-axis lag is often set tighter than XZ because vertical camera bounce (from jumping) looks worse than horizontal drift.

---

## Full Implementation Example — Spring Follow Camera

```js
import * as THREE from 'three';

class SpringFollowCamera {
  constructor(camera, config = {}) {
    this.camera = camera;
    this.offset    = config.offset    ?? new THREE.Vector3(0, 3, 7);
    this.stiffness = config.stiffness ?? 180;
    this.damping   = config.damping   ?? 22;
    this.lookLambda = config.lookLambda ?? 7;
    this.lookAheadScale = config.lookAheadScale ?? 2;

    this._pos = new THREE.Vector3();
    this._vel = new THREE.Vector3();
    this._lookAt = new THREE.Vector3();
    this._initialized = false;
  }

  update(target, velocity, dt) {
    const clampedDt = Math.min(dt, 0.05);

    const desiredPos = target.clone()
      .add(this.offset)
      .addScaledVector(velocity.clone().normalize(),
                       velocity.length() * this.lookAheadScale);

    if (!this._initialized) {
      this._pos.copy(desiredPos);
      this._lookAt.copy(target);
      this._initialized = true;
    }

    // Spring integrate position
    const displacement = this._pos.clone().sub(desiredPos);
    const force = displacement.multiplyScalar(-this.stiffness)
                              .addScaledVector(this._vel, -this.damping);
    this._vel.addScaledVector(force, clampedDt);
    this._pos.addScaledVector(this._vel, clampedDt);

    // Damp the look-at point
    this._lookAt.x = THREE.MathUtils.damp(this._lookAt.x, target.x, this.lookLambda, clampedDt);
    this._lookAt.y = THREE.MathUtils.damp(this._lookAt.y, target.y, this.lookLambda, clampedDt);
    this._lookAt.z = THREE.MathUtils.damp(this._lookAt.z, target.z, this.lookLambda, clampedDt);

    this.camera.position.copy(this._pos);
    this.camera.lookAt(this._lookAt);
  }
}

// Usage
const followCam = new SpringFollowCamera(camera, {
  offset: new THREE.Vector3(0, 4, 8),
  stiffness: 200,
  damping: 25,
  lookLambda: 8,
});

// In render loop:
followCam.update(player.position, player.velocity, delta);
```

---

## Notes

- Always clamp `dt` before feeding into spring integration. Tab-switch or debugger pause produces huge dt values that will explode the spring.
- `THREE.MathUtils.damp` is available in Three.js r128+. Earlier versions require a manual implementation.
- For camera shake, add a separate spring layer on top with a high-stiffness, low-damping spring driven by an impulse, then sum it into the final camera position.
- Spring cameras interact badly with collision systems — run collision resolution after spring integration, not before, otherwise the spring fights the constraint.
