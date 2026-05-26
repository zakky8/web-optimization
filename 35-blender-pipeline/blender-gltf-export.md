# Blender GLTF 2.0 Export Settings

**Source:** Blender glTF-Blender-IO exporter (`io_scene_gltf2/__init__.py`)  
**Accessed:** 2026-05-26  
**Canonical URL:** https://github.com/KhronosGroup/glTF-Blender-IO  
**Blender path:** File > Export > glTF 2.0 (.glb/.gltf)

---

## Output Format

Two output formats are available at the top of the export dialog:

| Option | Extension | Notes |
|--------|-----------|-------|
| glTF Binary | `.glb` | Single self-contained file; geometry, textures, and JSON packed together. Preferred for production web delivery. |
| glTF Separate | `.gltf` + `.bin` + textures | Human-readable JSON; useful for debugging and CI inspection. |
| glTF Embedded | `.gltf` | All data base64-encoded inside the JSON. Larger file, no separate assets. Avoid for web. |

---

## Include — What Gets Exported

### Object Selection

| Property | Default | Description |
|----------|---------|-------------|
| `use_selection` | `false` | Export **selected objects only**. Fastest way to isolate a single prop. |
| `use_visible` | `false` | Export only objects not hidden in the viewport (eye icon in Outliner). |
| `use_renderable` | `false` | Export only objects enabled for render (camera icon). |
| `use_active_collection` | `false` | Restrict to the currently active collection. |
| `use_active_collection_with_nested` | `true` | When active-collection mode is on, also include nested child collections. |
| `collection` | `""` | Export only objects from a named collection and its children. |
| `at_collection_center` | `false` | Translate root objects so the collection center of mass is at world origin. |

**Workflow tip:** For multi-mesh characters, use a dedicated collection per LOD. Enable `use_active_collection` and toggle the target before each export. This avoids accidental export of scene-wide proxy objects.

---

## Mesh Processing

| Property | Default | Description |
|----------|---------|-------------|
| `export_apply` | `false` | **Apply modifiers** (excluding Armature) before export. |
| `export_normals` | `true` | Include per-vertex normals in the mesh data. |
| `export_texcoords` | `true` | Include UV maps. Required for any textured material. |
| `export_tangents` | `false` | Include tangent vectors. Required when using normal maps in custom shaders that do not auto-derive tangents. |
| `use_mesh_edges` | `false` | Export loose edges as `GL_LINES`. |
| `use_mesh_vertices` | `false` | Export loose points as `GL_POINTS`. |

### Apply Modifiers Warning

`export_apply = true` bakes the modifier stack into the exported geometry at export time. **Shape keys (morph targets) cannot be exported when this is enabled**, because shape keys operate on the base mesh before modifiers. If your workflow needs both a subdivision modifier and shape keys, you must apply the modifier manually in a duplicate before export, or use the Geometry Nodes route.

---

## Draco Mesh Compression

Draco is a lossy/lossless geometry compression library by Google. Blender exposes it under **Data > Geometry** in the export dialog. Three.js requires `DRACOLoader` to be supplied to `GLTFLoader` before Draco-compressed files will load.

| Property | Default | Range | Description |
|----------|---------|-------|-------------|
| `export_draco_mesh_compression_enable` | `false` | — | Master toggle. Writes `KHR_draco_mesh_compression` extension into the file. |
| `export_draco_mesh_compression_level` | `6` | 0–10 | Compression effort. 0 = fastest encode/decode, minimal size saving. 6 = balanced. 10 = maximum compression, slowest decode. |
| `export_draco_position_quantization` | `14` | 0–30 | Precision bits for vertex positions. 0 = off. 14 bits ≈ 1/16384 precision per unit. |
| `export_draco_normal_quantization` | `10` | 0–30 | Precision bits for normals. |
| `export_draco_texcoord_quantization` | `12` | 0–30 | Precision bits for UV coordinates. |
| `export_draco_color_quantization` | `10` | 0–30 | Precision bits for vertex colors. |
| `export_draco_generic_quantization` | `12` | 0–30 | Precision bits for generic attributes (bone weights, etc.). |

**Meshopt compression** (`export_meshopt_compression_enable`) is an alternative to Draco using the `EXT_meshopt_compression` or `KHR_meshopt_compression` extension. Three.js supports both via `MeshoptDecoder`.

**When to use Draco vs raw:** For models under ~500 KB (uncompressed), Draco overhead may not be worth the Three.js wasm decoder cost at runtime. For large architectural scenes or multi-mesh characters, Draco typically delivers 70–90 % size reduction on geometry data.

---

## Materials

| Property | Default | Options | Description |
|----------|---------|---------|-------------|
| `export_materials` | `'EXPORT'` | `EXPORT`, `PLACEHOLDER`, `VIEWPORT`, `NONE` | `EXPORT` = full PBR export. `PLACEHOLDER` = keeps slots but exports no texture data. `VIEWPORT` = uses viewport shading colors only. `NONE` = strips all materials. |

Only **Principled BSDF** nodes are fully mapped to glTF PBR. Complex node trees outside the Principled BSDF are either baked or dropped — see `materials-for-threejs.md` for the full mapping table.

### Vertex Colors

| Property | Default | Description |
|----------|---------|-------------|
| `export_vertex_color` | `'MATERIAL'` | `MATERIAL` = export only vertex colors referenced by the material. `ACTIVE` = export the currently active vertex color. `NAME` = export by name. `NONE` = strip all. |
| `export_all_vertex_colors` | `true` | When on, exports all vertex color layers, even unused ones. |

---

## Images (Texture Format)

| Property | Default | Options | Description |
|----------|---------|---------|-------------|
| `export_image_format` | `'AUTO'` | `AUTO`, `JPEG`, `WEBP`, `NONE` | `AUTO` = PNG for images with alpha, JPEG for opaque images. `JPEG` = force JPEG for all. `WEBP` = force WebP for all. `NONE` = no texture files exported (embedded data only). |
| `export_image_add_webp` | `false` | — | Creates an additional WebP version alongside every texture. Use with `export_image_webp_fallback` for maximum browser compatibility. |
| `export_image_webp_fallback` | `false` | — | For every WebP texture, also export a PNG fallback. The GLTF file then lists both; Three.js and compliant loaders pick WebP when supported. |
| `export_keep_originals` | `false` | — | Reference the original texture files at their current path instead of copying them. **Warning:** Can cause broken references if paths differ between artist machine and server. |
| `export_texture_dir` | `""` | — | Subdirectory for texture files, relative to the exported `.gltf`. Example: `"textures"` puts all images in `./textures/`. |

**Format decision guide for Three.js:**
- **PNG:** Use for normal maps (lossless is critical for blue-channel precision), opacity/alpha channels, and small UI textures.
- **JPEG:** Use for diffuse/albedo, roughness, and metalness when file size matters and micro-artefacts are acceptable.
- **WebP:** Best size/quality ratio for web. Three.js supports WebP natively. Use `AUTO` + `export_image_add_webp = true` for a safe fallback-aware bundle.

---

## Animation

| Property | Default | Options | Description |
|----------|---------|---------|-------------|
| `export_animations` | `true` | — | Master toggle. Exports active Actions and NLA tracks. |
| `export_animation_mode` | `'ACTIONS'` | `ACTIONS`, `ACTIVE_ACTIONS`, `BROADCAST`, `NLA_TRACKS`, `SCENE` | See table below. |
| `export_frame_range` | `false` | — | Clip animation to the scene's playback in/out range. |
| `export_frame_step` | `1` | 1–120 | Bake every N-th frame. 1 = full resolution. Increase for long idle animations to reduce file size. |
| `export_force_sampling` | `true` | — | Sample all curves rather than exporting raw bezier keyframes. Required for drivers and constraints. |
| `export_anim_slide_to_zero` | `false` | — | Shift all animations so they start at 0.0 s. |
| `export_negative_frame` | `'SLIDE'` | `SLIDE`, `CROP` | `SLIDE` = shift animation to start at frame 0. `CROP` = discard frames below 0. |
| `export_bake_animation` | `false` | — | Force bake animation for every object, even those without keyframes (captures constraint-driven transforms). |
| `export_pointer_animation` | `false` | — | **Experimental.** Export material, light, and camera property animation as Animation Pointer. Only available in `NLA_TRACKS` and `SCENE` modes. |

### Animation Mode Reference

| Mode | Behavior |
|------|----------|
| `ACTIONS` | Each Action on every object becomes a separate `AnimationClip`. Most common for character rigs. |
| `ACTIVE_ACTIONS` | Only the currently active Action per object. |
| `BROADCAST` | All objects share one combined animation. |
| `NLA_TRACKS` | Each NLA track becomes a separate `AnimationClip`. Best for non-linear mixed animations. |
| `SCENE` | Entire scene timeline baked into one clip. |

---

## Transform — Y-up vs Z-up

Blender uses a **Z-up right-hand coordinate system** (Z = up, Y = forward). glTF 2.0 and Three.js use a **Y-up right-hand coordinate system** (Y = up, −Z = forward, X = right).

| Property | Default | Description |
|----------|---------|-------------|
| `export_yup` | `true` | Convert to glTF convention (+Y up). **Always leave this enabled** when targeting Three.js. |

When `export_yup = true`, the exporter applies a −90° rotation around X to convert from Blender's Z-up to glTF's Y-up. This rotation is baked into the root node transform (or the mesh data directly for static meshes), so it is invisible in Three.js — objects arrive right-side-up without needing a manual `rotation.x = -Math.PI / 2` correction.

**If you see models arriving sideways in Three.js**, the most common cause is `export_yup = false`, or a manually rotated parent armature that wasn't applied before export. Run `Ctrl+A > All Transforms` in Blender before exporting to bake any lingering rotation into the mesh.

---

## Armature-Specific Settings

| Property | Default | Description |
|----------|---------|-------------|
| `export_rest_position_armature` | `true` | Use the armature's rest pose as the skeletal bind pose. Disable only if your rig intentionally has a non-rest bind pose. |
| `export_hierarchy_flatten_objs` | `false` | Flatten object hierarchy for non-decomposable (non-TRS) transformations. |
| `export_hierarchy_flatten_bones` | `false` | Flatten bone hierarchy for non-decomposable transformations. Rarely needed. |

---

## Export Workflow Checklist

1. Apply all transforms (`Ctrl+A > All Transforms`) unless you need a specific pivot offset.
2. Ensure `export_yup = true`.
3. Set `export_animation_mode` to `NLA_TRACKS` if using the NLA editor; `ACTIONS` otherwise.
4. Do not enable `export_apply` if you need shape keys.
5. Choose image format: `AUTO` for mixed scenes, `WEBP` for max compression with `export_image_webp_fallback = true`.
6. Enable Draco only for geometry-heavy files (>200 K triangles); remember to register `DRACOLoader` in Three.js.
7. Export as `.glb` for production; `.gltf` for debugging.
