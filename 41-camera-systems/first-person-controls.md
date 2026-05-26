# First-Person Camera Controls

## Pointer Lock API

The Pointer Lock API captures the mouse cursor so that `mousemove` events report raw delta values (`movementX`, `movementY`) rather than clamped viewport coordinates. This is the standard mechanism for FPS-style look controls on desktop.

### Requesting pointer lock

```js
const canvas = renderer.domElement;

canvas.addEventListener('click', () => {
  canvas.requestPointerLock();
});

document.addEventListener('pointerlockchange', () => {
  if (document.pointerLockElement === canvas) {
    console.log('Pointer locked — FPS mode active');
  } else {
    console.log('Pointer released');
  }
});

document.addEventListener('pointerlockerror', () => {
  console.error('Pointer lock failed — check HTTPS or user gesture requirement');
});
```

**Requirements:**
- Must be triggered inside a transient user gesture (click, keydown, etc.)
- Page must be served over HTTPS (or localhost)
- Some browsers require a short delay between lock releases before re-locking

### mousemove delta for look

```js
const euler = new THREE.Euler(0, 0, 0, 'YXZ'); // YXZ order: yaw first, then pitch
const sensitivity = 0.002; // radians per pixel

document.addEventListener('mousemove', (e) => {
  if (document.pointerLockElement !== canvas) return;

  euler.y -= e.movementX * sensitivity; // yaw (horizontal)
  euler.x -= e.movementY * sensitivity; // pitch (vertical)

  // Clamp pitch to avoid flipping upside down
  euler.x = Math.max(-Math.PI / 2 + 0.01, Math.min(Math.PI / 2 - 0.01, euler.x));

  camera.quaternion.setFromEuler(euler);
});
```

**Why `'YXZ'` Euler order?** In FPS controls yaw is applied in world space, pitch in camera-local space. `'YXZ'` applies Y rotation first (world yaw), then X (local pitch). Using `'XYZ'` causes roll to appear when pitching while yawed.

---

## WASD Movement in requestAnimationFrame

Movement must happen inside the render loop (driven by RAF), not in key event handlers — event handlers fire at OS repeat rate, not frame rate.

```js
const keys = {
  KeyW: false,
  KeyA: false,
  KeyS: false,
  KeyD: false,
  Space: false,
  ShiftLeft: false,
};

document.addEventListener('keydown', (e) => { if (e.code in keys) keys[e.code] = true;  });
document.addEventListener('keyup',   (e) => { if (e.code in keys) keys[e.code] = false; });

// Movement constants
const SPEED      = 5;   // world units per second
const SPRINT_MUL = 2.5;

const _forward  = new THREE.Vector3();
const _right    = new THREE.Vector3();
const _move     = new THREE.Vector3();

function updateMovement(dt) {
  if (document.pointerLockElement !== canvas) return;

  const speed = (keys.ShiftLeft ? SPEED * SPRINT_MUL : SPEED) * dt;

  // Extract camera forward/right without the pitch component
  camera.getWorldDirection(_forward);
  _forward.y = 0;
  _forward.normalize();
  _right.crossVectors(_forward, camera.up).normalize(); // camera.up is (0,1,0)

  _move.set(0, 0, 0);
  if (keys.KeyW) _move.addScaledVector(_forward,  speed);
  if (keys.KeyS) _move.addScaledVector(_forward, -speed);
  if (keys.KeyD) _move.addScaledVector(_right,    speed);
  if (keys.KeyA) _move.addScaledVector(_right,   -speed);

  camera.position.add(_move);

  // Vertical movement (noclip / flying)
  if (keys.Space)     camera.position.y += speed;
  if (keys.ShiftLeft) camera.position.y -= speed; // if not used for sprint
}

// Render loop
function animate() {
  requestAnimationFrame(animate);
  const dt = clock.getDelta();
  updateMovement(dt);
  renderer.render(scene, camera);
}
```

**Flatten forward vector:** By zeroing `y` before normalizing, forward movement stays on the horizontal plane regardless of where the player is looking vertically.

---

## Mobile Look — DeviceOrientationEvent

`DeviceOrientationEvent` gives absolute orientation of the device in world space (alpha/beta/gamma). On iOS 13+ you must request permission first.

```js
// iOS 13+ permission request (must be triggered by a user gesture)
async function requestOrientationPermission() {
  if (typeof DeviceOrientationEvent?.requestPermission === 'function') {
    const state = await DeviceOrientationEvent.requestPermission();
    if (state !== 'granted') throw new Error('Orientation permission denied');
  }
}

document.getElementById('startBtn').addEventListener('click', () => {
  requestOrientationPermission().then(() => {
    window.addEventListener('deviceorientation', handleOrientation, true);
  });
});
```

### Converting orientation to camera quaternion

Three.js `DeviceOrientationControls` (in `three/addons`) handles this, but understanding the manual conversion is useful for custom implementations:

```js
import { DeviceOrientationControls } from 'three/addons/controls/DeviceOrientationControls.js';

const controls = new DeviceOrientationControls(camera);

// In render loop:
controls.update();
renderer.render(scene, camera);
```

### Manual implementation

```js
const deviceQuat = new THREE.Quaternion();
const zee        = new THREE.Vector3(0, 0, 1);
const euler      = new THREE.Euler();
const q0         = new THREE.Quaternion();
const q1         = new THREE.Quaternion(-Math.sqrt(0.5), 0, 0, Math.sqrt(0.5)); // -90° X

window.addEventListener('deviceorientation', (e) => {
  const alpha = THREE.MathUtils.degToRad(e.alpha || 0); // compass heading
  const beta  = THREE.MathUtils.degToRad(e.beta  || 0); // front-back tilt
  const gamma = THREE.MathUtils.degToRad(e.gamma || 0); // left-right tilt

  euler.set(beta, alpha, -gamma, 'YXZ');
  deviceQuat.setFromEuler(euler);
  deviceQuat.multiply(q1);

  // Account for screen orientation
  const orient = (screen.orientation?.angle ?? window.orientation ?? 0);
  q0.setFromAxisAngle(zee, -THREE.MathUtils.degToRad(orient));
  deviceQuat.multiply(q0);

  camera.quaternion.copy(deviceQuat);
});
```

**Caveat:** `DeviceOrientationEvent` gives absolute orientation relative to Earth's magnetic north. Combine with a manual yaw offset if you want the player to start facing a specific direction.

---

## Gamepad API

The Gamepad API provides access to controller axes and buttons via polling — it does not emit events for axis changes, only for button connect/disconnect.

```js
// Connection events
window.addEventListener('gamepadconnected',    (e) => console.log('Gamepad:', e.gamepad.id));
window.addEventListener('gamepaddisconnected', (e) => console.log('Disconnected:', e.gamepad.id));

// Standard gamepad layout indices (most controllers follow this)
const AXES = {
  LEFT_X:  0,  // left stick horizontal
  LEFT_Y:  1,  // left stick vertical
  RIGHT_X: 2,  // right stick horizontal (look)
  RIGHT_Y: 3,  // right stick vertical (look)
};

const BUTTONS = {
  A:      0,
  B:      1,
  X:      2,
  Y:      3,
  LB:     4,
  RB:     5,
  LT:     6,  // value 0..1
  RT:     7,  // value 0..1
  SELECT: 8,
  START:  9,
};

// Deadzone helper
function applyDeadzone(value, threshold = 0.1) {
  return Math.abs(value) < threshold ? 0 : value;
}

function updateGamepad(dt) {
  const gamepads = navigator.getGamepads();
  const gp = gamepads[0];
  if (!gp) return;

  const moveX = applyDeadzone(gp.axes[AXES.LEFT_X]);
  const moveZ = applyDeadzone(gp.axes[AXES.LEFT_Y]);
  const lookX = applyDeadzone(gp.axes[AXES.RIGHT_X]);
  const lookY = applyDeadzone(gp.axes[AXES.RIGHT_Y]);

  const LOOK_SPEED = 2.0; // radians per second (at full stick deflection)
  const MOVE_SPEED = 5.0;

  // Look (apply to Euler, same as mouse)
  euler.y -= lookX * LOOK_SPEED * dt;
  euler.x -= lookY * LOOK_SPEED * dt;
  euler.x  = Math.max(-Math.PI / 2 + 0.01, Math.min(Math.PI / 2 - 0.01, euler.x));
  camera.quaternion.setFromEuler(euler);

  // Move (same forward/right extraction as WASD)
  camera.getWorldDirection(_forward);
  _forward.y = 0;
  _forward.normalize();
  _right.crossVectors(_forward, camera.up).normalize();

  camera.position.addScaledVector(_forward, -moveZ * MOVE_SPEED * dt);
  camera.position.addScaledVector(_right,    moveX * MOVE_SPEED * dt);
}
```

**Important:** `navigator.getGamepads()` must be called on every frame — it returns a snapshot, not a live object. Cache the index, not the gamepad object.

---

## Combined Input Manager

In production, unify keyboard, mouse, and gamepad into one movement state:

```js
class FirstPersonInputManager {
  constructor(canvas, camera) {
    this.canvas   = canvas;
    this.camera   = camera;
    this.euler    = new THREE.Euler(0, 0, 0, 'YXZ');
    this.keys     = {};
    this.sensitivity  = { mouse: 0.002, gamepad: 2.0 };
    this._fwd = new THREE.Vector3();
    this._right = new THREE.Vector3();

    canvas.addEventListener('click', () => canvas.requestPointerLock());
    document.addEventListener('mousemove',  this._onMouseMove.bind(this));
    document.addEventListener('keydown', (e) => { this.keys[e.code] = true; });
    document.addEventListener('keyup',   (e) => { this.keys[e.code] = false; });
  }

  _onMouseMove(e) {
    if (document.pointerLockElement !== this.canvas) return;
    this.euler.y -= e.movementX * this.sensitivity.mouse;
    this.euler.x -= e.movementY * this.sensitivity.mouse;
    this.euler.x = THREE.MathUtils.clamp(this.euler.x, -Math.PI/2+0.01, Math.PI/2-0.01);
    this.camera.quaternion.setFromEuler(this.euler);
  }

  update(dt) {
    this._applyGamepadLook(dt);
    this._applyMovement(dt);
  }

  _applyGamepadLook(dt) {
    const gp = navigator.getGamepads()[0];
    if (!gp) return;
    const lx = Math.abs(gp.axes[2]) > 0.1 ? gp.axes[2] : 0;
    const ly = Math.abs(gp.axes[3]) > 0.1 ? gp.axes[3] : 0;
    this.euler.y -= lx * this.sensitivity.gamepad * dt;
    this.euler.x -= ly * this.sensitivity.gamepad * dt;
    this.euler.x = THREE.MathUtils.clamp(this.euler.x, -Math.PI/2+0.01, Math.PI/2-0.01);
    this.camera.quaternion.setFromEuler(this.euler);
  }

  _applyMovement(dt) {
    const speed = 5 * dt;
    this.camera.getWorldDirection(this._fwd);
    this._fwd.y = 0;
    this._fwd.normalize();
    this._right.crossVectors(this._fwd, this.camera.up).normalize();

    if (this.keys['KeyW']) this.camera.position.addScaledVector(this._fwd,   speed);
    if (this.keys['KeyS']) this.camera.position.addScaledVector(this._fwd,  -speed);
    if (this.keys['KeyD']) this.camera.position.addScaledVector(this._right,  speed);
    if (this.keys['KeyA']) this.camera.position.addScaledVector(this._right, -speed);
  }
}
```

---

## Notes

- Pointer lock is blocked in cross-origin iframes. Ensure your canvas is same-origin.
- `movementX`/`movementY` are not scaled by `devicePixelRatio` — they are raw hardware delta values. Do not divide by DPR.
- `DeviceOrientationEvent` is deprecated in the W3C spec but remains the only practical API for absolute device orientation in browsers as of 2026.
- Xbox / PlayStation controllers map consistently to the Standard Gamepad Layout in Chrome and Firefox; older controllers may not. Always test with `gp.mapping === 'standard'`.
- For a full production FPS controller, pair this input system with a physics body (rapier.js, cannon-es) rather than moving the camera directly.
