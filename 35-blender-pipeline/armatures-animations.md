# Armatures and Animations: Blender to Three.js

**Source:** KhronosGroup/glTF-Blender-IO (exporter `__init__.py`); Three.js `GLTFLoader.js`, Three.js Animation docs  
**Accessed:** 2026-05-26

---

## Armature Export from Blender

### Requirements Before Export

1. **The armature and its mesh must be in the same collection** (or both selected with `use_selection = true`).
2. **Parenting:** The mesh must be parented to the armature via `Ctrl+P > Armature Deform > With Automatic Weights` (or manual weights). The exporter follows the Object hierarchy.
3. **Apply scale:** Run `Ctrl+A > All Transforms` on both the armature and mesh objects before export. Non-uniform scale on an armature produces incorrect bone matrices in Three.js.
4. **`export_rest_position_armature = true`** (default): The exporter uses the armature's rest pose as the glTF skin bind pose (`inverseBindMatrices`). Only disable if your rig intentionally has a non-rest bind pose.

### Bone Names

Bone names in Blender map directly to bone names in the glTF `skin.joints` array. Three.js `GLTFLoader` then creates a `Bone` object per joint, accessible via `SkinnedMesh.skeleton.getBoneByName(name)`.

**Rules:**
- Names must be unique within a rig. Duplicate names cause silent bone-mapping failures.
- Blender L/R suffixes (`.L` / `.R` or `_L` / `_R`) are preserved verbatim in the export. Three.js does not rename them.
- Do not use names that are valid JavaScript property identifiers to avoid potential conflicts in traversal code.

### Weight Painting

Weight painting determines `JOINTS_0` and `WEIGHTS_0` vertex attributes in the glTF mesh. Each vertex can influence up to **4 bones** in the standard glTF 2.0 spec. The exporter automatically limits to 4 influences per vertex and normalizes weights to sum to 1.0.

**Common pitfalls:**
- Vertices with zero total weight are assigned to the root bone. In Three.js this creates unexpected rigid attachment to the skeleton origin for those vertices.
- Overlapping vertex groups from multiple armatures are not supported — each mesh should be deformed by exactly one armature.
- Very small weight values (< 0.001) are typically pruned by the exporter to stay within the 4-influence limit.

### Export Settings for Armatures

| Property | Recommended | Notes |
|----------|-------------|-------|
| `export_apply` | `false` | Applying modifiers strips armature deformation information. |
| `export_rest_position_armature` | `true` (default) | Correct for all standard rigs. |
| `export_hierarchy_flatten_bones` | `false` | Only enable if your bone hierarchy contains non-decomposable (skew/shear) transforms. |
| `export_yup` | `true` | Required. See `blender-gltf-export.md`. |

---

## Shape Keys (Morph Targets)

### Blender Setup

Shape keys in Blender become **morph targets** in glTF (`mesh.primitives[i].targets`). Each shape key becomes one target with position deltas (and optionally normal/tangent deltas) stored relative to the base mesh.

**Requirements:**
- `export_apply = false` is mandatory. Applying modifiers removes shape key data.
- All shape keys must be on the same mesh object.
- Shape key **names** carry over to glTF `mesh.extras.targetNames` and then to Three.js `geometry.morphAttributes` names.

### Shape Keys vs Armature: When to Use Each

| Technique | Best For | Limitations |
|-----------|----------|-------------|
| **Shape keys (morph targets)** | Facial expressions, blend shapes, LOD transitions, soft body approximations | No volume preservation; each target is a linear blend of vertex positions |
| **Armature (skeletal)** | Body/limb movement, procedural animation, IK | Requires weight painting; less suited for facial micro-detail |
| **Combined** | Facial rig: armature for jaw/eye bones + shape keys for phonemes and brow shapes | Both export simultaneously when `export_apply = false` |

### Three.js Morph Target Usage

After loading a GLTF with shape keys, Three.js provides:

```js
// Accessing morph targets on a loaded mesh
loader.load('character.glb', (gltf) => {
  const mesh = gltf.scene.getObjectByName('Head'); // SkinnedMesh or Mesh

  // geometry.morphAttributes contains position, normal arrays
  console.log(mesh.geometry.morphAttributes.position); // array of BufferAttribute

  // Target names (from Blender shape key names)
  console.log(mesh.morphTargetDictionary); // { 'smile': 0, 'brow_up': 1, ... }

  // Set influence (0.0 = no effect, 1.0 = full effect)
  mesh.morphTargetInfluences[0] = 0.5; // 50% of 'smile'
  // or by name:
  mesh.morphTargetInfluences[mesh.morphTargetDictionary['smile']] = 0.5;
});
```

`mesh.morphTargetInfluences` is a plain `Float32Array`. Three.js does not clamp values — you can exceed 0–1 for exaggerated expressions, though results depend on target geometry.

**`MeshStandardMaterial` and morph targets:** Three.js `MeshStandardMaterial` supports morph targets natively. Set `material.morphTargets = true` (or the mesh will not deform even with valid influence values set). `GLTFLoader` sets this flag automatically when it detects morph attributes.

---

## Animations in Three.js

### AnimationClip

`THREE.AnimationClip` is the data container for a named animation sequence. Each clip holds an array of `KeyframeTrack` objects, one per animated property channel. `GLTFLoader` creates one `AnimationClip` per glTF animation object (which corresponds to one Blender Action or NLA track, depending on export mode).

```js
loader.load('character.glb', (gltf) => {
  const clips = gltf.animations; // Array of THREE.AnimationClip
  clips.forEach(clip => console.log(clip.name, clip.duration));
});
```

### AnimationMixer

`THREE.AnimationMixer` drives playback of clips on a specific object (or its subtree). One mixer per animated object/scene is the standard pattern.

```js
const mixer = new THREE.AnimationMixer(gltf.scene);

// Get and play a named clip
const walkClip = THREE.AnimationClip.findByName(gltf.animations, 'Walk');
const action = mixer.clipAction(walkClip);
action.play();

// In the render loop — delta is seconds since last frame
const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  mixer.update(clock.getDelta());
  renderer.render(scene, camera);
}
animate();
```

### AnimationAction

`mixer.clipAction(clip)` returns a `THREE.AnimationAction`, which controls:

| Property / Method | Description |
|-------------------|-------------|
| `action.play()` | Start playback. |
| `action.stop()` | Stop and reset. |
| `action.paused` | Boolean; freeze at current time. |
| `action.timeScale` | Playback speed multiplier. Negative = reverse. |
| `action.weight` | Blend weight (0–1) for additive/cross-fade mixing. |
| `action.setLoop(mode, count)` | `THREE.LoopRepeat`, `THREE.LoopOnce`, `THREE.LoopPingPong`. |
| `action.clampWhenFinished` | When `LoopOnce`, hold last frame instead of resetting. |
| `action.crossFadeTo(other, duration, warp)` | Smooth transition between two actions. |

### Cross-Fading Between Animations

```js
// Smooth blend from 'Idle' to 'Walk' over 0.3 seconds
const idleAction = mixer.clipAction(
  THREE.AnimationClip.findByName(gltf.animations, 'Idle')
);
const walkAction = mixer.clipAction(
  THREE.AnimationClip.findByName(gltf.animations, 'Walk')
);

idleAction.play();
// Trigger transition:
idleAction.crossFadeTo(walkAction, 0.3, true);
walkAction.play();
```

### Animating Morph Targets via AnimationClip

Blender shape key animation (keyframed shape key values in the Action) exports as `NumberKeyframeTrack` targeting `mesh.morphTargetInfluences[i]`. Three.js plays these back through the mixer:

```js
// The track name format for morph targets:
// "Head.morphTargetInfluences[0]"
// GLTFLoader creates these automatically from shape key animation data
```

No manual track construction is needed — `GLTFLoader` generates the correct track binding strings from `mesh.extras.targetNames` index ordering.

---

## Skeletal Animation Keyframe Types

Three.js reads glTF animation channel samplers and creates the appropriate `KeyframeTrack` subclass:

| glTF Target Path | Three.js Track | Interpolation Modes |
|-----------------|----------------|---------------------|
| `translation` | `VectorKeyframeTrack` | LINEAR, STEP, CUBICSPLINE |
| `rotation` | `QuaternionKeyframeTrack` | LINEAR, STEP, CUBICSPLINE |
| `scale` | `VectorKeyframeTrack` | LINEAR, STEP, CUBICSPLINE |
| `weights` (morph) | `NumberKeyframeTrack` | LINEAR, STEP, CUBICSPLINE |

`CUBICSPLINE` in glTF maps to Blender's Bezier interpolation and is handled by `GLTFCubicSplineInterpolant` in Three.js.

---

## NLA Track Workflow

For complex characters with many animations, use Blender's Non-Linear Animation editor:

1. Create individual Actions (Walk, Run, Jump, Idle) in the Action Editor.
2. Push each Action to the NLA stack via the NLA editor.
3. Set export mode to `NLA_TRACKS` in the glTF export dialog.
4. Each NLA track exports as one `AnimationClip` named after the track.

This is the recommended workflow for game-ready characters: each animation stays isolated and can be blended freely with `AnimationMixer`.

---

## Common Armature Export Problems and Fixes

| Symptom in Three.js | Likely Cause in Blender | Fix |
|---------------------|------------------------|-----|
| Model appears sideways or upside-down | `export_yup = false` | Enable Y-up in export settings |
| Bones in wrong position at rest | Scale not applied to armature | `Ctrl+A > All Transforms` before export |
| Mesh deforms to wrong bones | Duplicate bone names in rig | Rename bones to be unique |
| Vertices snap to origin during animation | Zero-weight vertices | Select all, `Weights > Clean` with threshold 0.001 |
| Shape keys not in exported file | `export_apply = true` was checked | Uncheck Apply Modifiers |
| Animation plays but mesh doesn't deform | Armature and mesh not in same hierarchy | Re-parent mesh to armature |
| `morphTargetInfluences` has no effect | `material.morphTargets` not true | Usually auto-set by GLTFLoader; check if material was replaced post-load |
