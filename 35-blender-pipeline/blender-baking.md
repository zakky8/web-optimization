# Blender Texture Baking for Three.js

**Source:** Blender glTF-Blender-IO exporter documentation; Blender Cycles/EEVEE bake documentation; Three.js GLTFLoader  
**Accessed:** 2026-05-26

---

## Why Baking Is Required for Three.js

Three.js reads textures from glTF `image` objects. It has no access to Blender's procedural shader graph — noise textures, color ramps, math nodes, and multi-BSDF layer mixing all live in Blender's CPU/GPU memory and are never exported. Baking converts the output of any shader network into a flat image texture that can be referenced by glTF channel properties.

Baking is also the only way to export:
- Ambient Occlusion (AO) maps
- Shadow maps (baked light, lightmaps)
- Combined/diffuse illumination for unlit materials
- Procedural noise-based roughness or scratches
- Normal maps derived from high-poly sculpt to low-poly cage

---

## Cycles vs EEVEE Baking

| Feature | Cycles | EEVEE |
|---------|--------|-------|
| Ambient Occlusion (ray-traced) | Yes | Limited (screen-space only, less accurate) |
| Shadow baking | Yes | Approximate only |
| Normal from high-poly | Yes | Yes |
| Emission baking | Yes | Yes |
| Combined pass (full lighting) | Yes | Limited |
| Render time | Slow (path-traced) | Fast |
| GPU acceleration | CUDA/OptiX/Metal/HIP | GPU only |

**Recommendation:** Use Cycles for AO, shadow, and normal-from-highpoly bakes that need ray-traced accuracy. Use EEVEE for fast diffuse/emission passes during iteration.

Switch render engine: `Properties > Render Properties > Render Engine`.

---

## Bake Workflow: Step-by-Step

### 1. Prepare the Target Image

In the **UV Editor**, create a new image that will receive the baked data:

1. UV Editor header: **Image > New Image**
2. Set resolution (2048×2048 for mid-range, 4096×4096 for hero assets).
3. Set Color Space to **Non-Color** for all non-color maps (AO, roughness, metalness, normal). Use **sRGB** only for diffuse/albedo bakes.
4. Leave `Float Buffer` off unless baking HDR values.

### 2. Add an Image Texture Node to the Material

In the **Shader Editor** (not connected to anything — just present and selected):

1. Add > Texture > **Image Texture** node.
2. Assign the image you just created.
3. **Select this node** (highlighted border) — Blender bakes into whichever Image Texture node is currently selected.
4. Do not connect this node into the shader graph; it is a bake target only.

### 3. Unwrap UVs

The bake writes pixels based on UV coordinates. Every face that needs a baked value must have a valid, non-overlapping UV island on UV map `UVMap` (UV channel 0 by default).

For **lightmaps** (AO, shadow used as `aoMap` in Three.js), you need a **separate, non-overlapping UV map** on channel `UVMap.001` (UV channel 1). See UV2 Lightmap Baking section below.

UV unwrap: `Tab` (Edit Mode) > `U` > **Smart UV Project** (fast, for environment props) or **Unwrap** (for organic shapes with seams marked via `Ctrl+E > Mark Seam`).

### 4. Bake Settings

Navigate to **Properties > Render Properties > Bake** (visible only when Cycles or EEVEE is active).

| Setting | Value for Each Bake Type |
|---------|--------------------------|
| Bake Type | See table below |
| Selected to Active | Off (unless baking high-poly to low-poly cage) |
| Extrusion | 0.05–0.1 m (for high-to-low poly baking) |
| Max Ray Distance | 0.05–0.1 m (for high-to-low poly baking) |
| Output > Margin | 4–16 px (prevents seam bleeding at UV island edges) |
| Output > Margin Type | Adjacent Faces (Blender 3.4+) or Extend |
| Output > Target | Image Textures (uses the selected Image Texture node) |

### 5. Bake

Click **Bake** in the Bake panel. For high-resolution textures with Cycles + AO, this can take several minutes. Monitor progress in the bottom status bar.

---

## Bake Types and Their Three.js Channel Destinations

### Ambient Occlusion (AO)

| Setting | Value |
|---------|-------|
| Bake Type | **Ambient Occlusion** |
| Samples | 64–256 (more = less noise) |
| Color Space | Non-Color |

**glTF destination:** `occlusionTexture` (red channel of the ORM texture, or as a standalone texture).  
**Three.js property:** `material.aoMap`  
**Requires UV2 in Three.js:** Yes — see UV2 section below.

The AO factor multiplier is:  
`material.aoMapIntensity` (default 1.0, range 0–1 in practice but not clamped).

### Normal Map (Low-poly from High-poly Sculpt)

| Setting | Value |
|---------|-------|
| Bake Type | **Normal** |
| Space | **Tangent** (for Three.js `normalMap`, which is tangent-space) |
| Selected to Active | On |
| High-poly object | Source |
| Low-poly object | Active (selected last) |
| Color Space | Non-Color |

**glTF destination:** `normalTexture`  
**Three.js property:** `material.normalMap`

After baking, add a **Normal Map** node in the low-poly's shader graph:  
`Image Texture (baked normal)` → `Color` of Normal Map node → `Normal` of Principled BSDF.  
Set the Normal Map node Strength to 1.0 (the exporter reads this as `normalTexture.scale`).

### Roughness (from Procedural Nodes)

| Setting | Value |
|---------|-------|
| Bake Type | **Roughness** |
| Color Space | Non-Color |

**glTF destination:** Green channel of `metallicRoughnessTexture`  
**Three.js property:** `material.roughnessMap`

After baking, the resulting image must be used as the texture plugged into the Roughness socket of Principled BSDF (replaces the procedural node chain).

### Combined / Diffuse (Lightmap Baking)

| Setting | Value |
|---------|-------|
| Bake Type | **Combined** or **Diffuse** |
| Contributions (Diffuse) | Enable Direct + Indirect; disable Color to bake pure light without albedo |
| Color Space | sRGB |

**Use case:** Pre-baked lighting for static scenes rendered with `MeshBasicMaterial` or `MeshStandardMaterial.lightMap`.  
**glTF destination:** Not a standard glTF channel; attach via custom `lightMap` property or `aoMap` hack.  
**Three.js property:** `material.lightMap` + `material.lightMapIntensity`.

### Shadow Map

| Setting | Value |
|---------|-------|
| Bake Type | **Shadow** |
| Color Space | Non-Color |

Produces a grayscale texture of shadow coverage. Multiply against diffuse in a Multiply node in Blender, then bake the **Combined** pass after compositing. Alternatively, use as a separate `lightMap` in Three.js.

### Emission

| Setting | Value |
|---------|-------|
| Bake Type | **Emit** |
| Color Space | sRGB |

**glTF destination:** `emissiveTexture`  
**Three.js property:** `material.emissiveMap`  
Useful when emission comes from a complex node chain (masked emissive, gradient emissive) that cannot export procedurally.

---

## UV2 Lightmap Baking for Three.js aoMap

Three.js requires the `aoMap` (and `lightMap`) texture to be read from **UV channel 1** (the second UV map), not UV channel 0. This is the `uv2` vertex attribute in the buffer geometry.

### Creating UV2 in Blender

1. Select the mesh, enter **Edit Mode**.
2. Open **Properties > Object Data > UV Maps**.
3. Click `+` to add a second UV map. Name it `UVMap.001` or `UVChannel1` (name is irrelevant; index matters).
4. With `UVMap.001` active, unwrap using **Smart UV Project** with no overlapping islands — lightmap UVs must be non-overlapping.
5. Return to the Shader Editor, add a `UV Map` node set to `UVMap.001`, and route it into the Image Texture node of your bake target.

### Exporting UV2

The Blender glTF exporter exports all UV maps when `export_texcoords = true`. `UVMap` → `TEXCOORD_0`, `UVMap.001` → `TEXCOORD_1`. Three.js `GLTFLoader` maps:

- `TEXCOORD_0` → `geometry.attributes.uv`
- `TEXCOORD_1` → `geometry.attributes.uv1`

`MeshStandardMaterial.aoMap` reads from `uv1` automatically.

### Setting aoMap in Three.js Code

```js
loader.load('scene.glb', (gltf) => {
  const mesh = gltf.scene.getObjectByName('Wall');

  const aoTexture = new THREE.TextureLoader().load('ao.png');
  aoTexture.colorSpace = THREE.NoColorSpace; // Non-color data

  mesh.material.aoMap = aoTexture;
  mesh.material.aoMapIntensity = 1.0;
  mesh.material.needsUpdate = true;
});
```

If the AO texture is packed into the ORM texture (red channel), extract it or reference the full ORM texture:

```js
// ORM texture: R=AO, G=Roughness, B=Metalness
// GLTFLoader automatically handles this when the texture is in occlusionTexture slot
// If loading manually:
mesh.material.aoMap = ormTexture;          // reads R channel
mesh.material.roughnessMap = ormTexture;   // reads G channel
mesh.material.metalnessMap = ormTexture;   // reads B channel
```

---

## Packing ORM: Compositing in Blender

Instead of three separate textures, pack AO (R), Roughness (G), Metalness (B) into one texture to reduce draw calls and GPU memory:

1. Bake AO, Roughness, and Metalness as separate images.
2. In the **Compositor** (`Ctrl+Space`, switch to Compositor view):
   - Add three **Image** input nodes.
   - Add a **Combine RGBA** node: R = AO image, G = Roughness image, B = Metalness image, A = white.
   - Output to **File Output** node → save as PNG.
3. In the Principled BSDF shader, use a **Separate Color** node on the ORM image:
   - R output → AO (not directly into BSDF; leave for Three.js only)
   - G output → Roughness socket
   - B output → Metallic socket

The glTF exporter will detect the G→Roughness and B→Metallic connections and pack them into `metallicRoughnessTexture` automatically, producing the standard ORM layout Three.js expects.

---

## Baking to WebP for Delivery

After baking to PNG in Blender, convert to WebP for production:

```js
// In a build pipeline (Node.js, using sharp or squoosh):
// sharp('ao_roughness_metalness.png').webp({ quality: 90 }).toFile('orm.webp');
```

Or enable `export_image_format = WEBP` in the Blender glTF exporter — it converts existing image nodes to WebP at export time, but does not re-bake procedural content.

---

## Baking Checklist

- [ ] Image Texture node present and selected in Shader Editor before baking
- [ ] Image Color Space set to Non-Color for all non-albedo maps
- [ ] UV islands non-overlapping for any map that will be used as aoMap/lightMap (UV2)
- [ ] Margin set to 4–16 px to prevent seam bleeding
- [ ] Cycles selected for ray-traced AO and normal-from-highpoly bakes
- [ ] After baking: replace procedural nodes with the baked Image Texture node
- [ ] Save baked images (they are unsaved in Blender's memory until explicitly saved with Image > Save or Image > Save All Images)
- [ ] ORM packed into single texture (R=AO, G=Roughness, B=Metalness) for efficient Three.js loading
- [ ] UV2 created and non-overlapping if aoMap will be used in Three.js
