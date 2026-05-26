# Spatial Audio and Panning

## The Source–Listener Model

The Web Audio API models 3D audio as a source–listener system:

- **Source** — a `PannerNode` that carries position and orientation of the sound emitter.
- **Listener** — `AudioContext.listener` (an `AudioListener` object), which carries the
  position, forward vector, and up vector of the observer.

The browser computes the gain and stereo pan for each source relative to the listener on
every audio frame. Three.js keeps both in sync with the scene's transform graph via
`updateMatrixWorld()`.

---

## PannerNode Constructor Defaults

All defaults are from the `PannerNode()` constructor specification.

Source: https://developer.mozilla.org/en-US/docs/Web/API/PannerNode/PannerNode
Accessed: 2026-05-26

| Option | Default | Allowed Values |
|---|---|---|
| `panningModel` | `"equalpower"` | `"equalpower"`, `"HRTF"` |
| `distanceModel` | `"inverse"` | `"linear"`, `"inverse"`, `"exponential"` |
| `positionX/Y/Z` | `0` | any number |
| `orientationX` | `1` | any number |
| `orientationY/Z` | `0` | any number |
| `refDistance` | `1` | positive number |
| `maxDistance` | `10000` | positive number |
| `rolloffFactor` | `1` | non-negative number |
| `coneInnerAngle` | `360` | 0–360 degrees |
| `coneOuterAngle` | `360` | 0–360 degrees |
| `coneOuterGain` | `0` | 0–1 |

Note: `THREE.PositionalAudio` overrides `panningModel` to `"HRTF"` in its constructor.

Source: https://github.com/mrdoob/three.js/blob/dev/src/audio/PositionalAudio.js
Accessed: 2026-05-26

---

## panningModel: HRTF vs equalpower

### `"equalpower"`

- Simpler panning algorithm; uses equal-power crossfading between left and right channels.
- CPU-cheap; appropriate for 2D games, UI audio, and cases where positional realism is
  not required.
- Ignores elevation — a source above or below the listener sounds identical to one on
  the same horizontal plane.

### `"HRTF"` (Head-Related Transfer Function)

- Convolves the audio with a measured head-related transfer function, simulating the
  acoustic effects of pinnae, head shadow, and torso reflections.
- Produces convincing elevation and front/back differentiation over headphones.
- Higher CPU cost; uses the browser's built-in HRTF dataset (varies by browser).
- Default for `THREE.PositionalAudio`.

**Rule of thumb:** Use `"HRTF"` for headphone-targeted experiences where elevation
matters. Use `"equalpower"` when a lightweight left/right pan is sufficient or when
rendering many simultaneous sources.

Source: https://developer.mozilla.org/en-US/docs/Web/API/PannerNode
Accessed: 2026-05-26

---

## distanceModel: Attenuation Algorithms

The distance model controls how gain drops as the listener moves away from the source.
`d` = distance from listener to source, `ref` = `refDistance`, `max` = `maxDistance`,
`r` = `rolloffFactor`.

### `"linear"` — simple linear ramp

```
gain = 1 - rolloffFactor × ((d - refDistance) / (maxDistance - refDistance))
```

Gain is clamped to `[0, 1]`. Reaches zero at `maxDistance` when `rolloffFactor = 1`.
Predictable for game-design; can feel unnatural because real sound does not fall off
linearly.

### `"inverse"` (default) — inverse-distance law

```
gain = refDistance / (refDistance + rolloffFactor × (max(d, refDistance) - refDistance))
```

Approximates the physics of sound propagation in free space. `refDistance` is the
distance at which gain equals 1.0 (full volume). `rolloffFactor = 1` gives the
physically correct inverse-distance relationship. Good general default.

### `"exponential"` — power-law rolloff

```
gain = (max(d, refDistance) / refDistance) ^ (-rolloffFactor)
```

Most flexible. With `rolloffFactor = 1` it matches the inverse law exactly. Increase
`rolloffFactor` for a more aggressive falloff (underwater, small room), decrease it for
a large open space.

---

## refDistance, maxDistance, rolloffFactor

### refDistance (default `1`)

The distance from the source at which the panner produces unity gain (gain = 1.0, no
attenuation). Objects closer than `refDistance` are not amplified beyond 1.0 in the
`"inverse"` and `"exponential"` models. Must be positive.

```js
posSound.setRefDistance(5);   // full volume within 5 scene units
```

### maxDistance (default `10000`)

Only meaningful for `"linear"` model — defines the distance at which gain clamps to 0.
For `"inverse"` and `"exponential"` models the source is still audible (though
extremely quiet) beyond `maxDistance`. Must be positive.

```js
posSound.setMaxDistance(100);
```

### rolloffFactor (default `1`)

Scales the rate of attenuation. Higher values make the source drop off faster. Must be
non-negative. A value of `0` means no distance attenuation regardless of model.

```js
posSound.setRolloffFactor(2);   // twice as much falloff
```

---

## Directional Cone (coneInnerAngle, coneOuterAngle, coneOuterGain)

A cone models a directional speaker — think a loudspeaker horn or a directional sound
prop. The cone is relative to the source's orientation vector
(`orientationX/Y/Z`).

```
                    coneInnerAngle (full volume arc)
                    ┌──────────────────────┐
orientationVector → │                      │
                    └──────────────────────┘
              coneOuterAngle (attenuated arc boundary)
```

| Property | Default | Description |
|---|---|---|
| `coneInnerAngle` | `360` | Full-cone angle in degrees. Gain = 1.0 inside. |
| `coneOuterAngle` | `360` | Outer-cone angle. Between inner and outer, gain interpolates toward `coneOuterGain`. |
| `coneOuterGain` | `0` | Gain beyond the outer cone. `0` = silent; `1` = no attenuation. |

**Default `360/360`** means omnidirectional — the cone wraps the full sphere.

### Narrow speaker example

```js
posSound.setDirectionalCone(
  30,    // coneInnerAngle — full volume within ±15° of orientation
  60,    // coneOuterAngle — attenuates from 15° to 30°, then...
  0.1    // coneOuterGain — 10% volume outside 30°
);
// Aim the source
posSound.panner.orientationX.value = 0;
posSound.panner.orientationY.value = 0;
posSound.panner.orientationZ.value = -1;   // pointing forward in Three.js
```

Source: https://developer.mozilla.org/en-US/docs/Web/API/PannerNode/PannerNode
Accessed: 2026-05-26

---

## AudioContext.listener vs PannerNode

`AudioContext.listener` represents the observer (the ears). It exposes `AudioParam`
properties for position and orientation that Three.js writes every frame from the
camera's world matrix.

`PannerNode` represents one sound source. Each `THREE.PositionalAudio` owns one
`PannerNode`. There is only one listener per context.

Do not set `AudioContext.listener` manually when using Three.js — `AudioListener`'s
`updateMatrixWorld()` handles it.

---

## Three.js PositionalAudio: API Reference

```js
const pos = new THREE.PositionalAudio(listener);
mesh.add(pos);

// Choose model
pos.setDistanceModel('inverse');       // 'linear' | 'inverse' | 'exponential'
pos.setPanningModel('HRTF');           // 'equalpower' | 'HRTF'

// Distance
pos.setRefDistance(3);
pos.setMaxDistance(50);
pos.setRolloffFactor(1.5);

// Cone
pos.setDirectionalCone(45, 90, 0.05);

// Inspect
console.log(pos.getRefDistance());     // 3
console.log(pos.getDistanceModel());   // 'inverse'
```

Direct panner access for properties not wrapped:

```js
pos.panner.panningModel  = 'HRTF';
pos.panner.coneInnerAngle = 45;
pos.panner.coneOuterAngle = 90;
pos.panner.coneOuterGain  = 0.05;
```

Source: https://github.com/mrdoob/three.js/blob/dev/src/audio/PositionalAudio.js
Accessed: 2026-05-26

---

## Checklist for Spatial Audio Setup

- [ ] `AudioListener` added as a child of the camera (not the scene root).
- [ ] `AudioContext` resumed after user gesture before first play call.
- [ ] `PositionalAudio` added as a child of the mesh it belongs to.
- [ ] `refDistance` tuned to scene scale (Three.js default units are meters).
- [ ] `distanceModel` chosen and `rolloffFactor` tested by ear.
- [ ] If using HRTF: test on headphones; result will sound flat on speakers.
- [ ] Cone angles set for directional sources; leave at `360/360` for omnidirectional.

---

## Sources

| URL | Tier | Accessed |
|---|---|---|
| https://developer.mozilla.org/en-US/docs/Web/API/PannerNode | 1 (MDN) | 2026-05-26 |
| https://developer.mozilla.org/en-US/docs/Web/API/PannerNode/PannerNode | 1 (MDN) | 2026-05-26 |
| https://github.com/mrdoob/three.js/blob/dev/src/audio/PositionalAudio.js | 1 (source) | 2026-05-26 |
