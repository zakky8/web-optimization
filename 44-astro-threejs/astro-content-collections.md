# Astro Content Collections for 3D Scene Configuration

> Verified against: https://docs.astro.build/en/guides/content-collections/  
> Astro version: 6.x (current as of 2026-05-26)  
> Sources accessed: 2026-05-26

---

## Why Use Content Collections for 3D?

Three.js scenes often need configuration data: which model file to load, what environment map to use, initial camera position, material overrides, hotspot coordinates. Hardcoding this in components makes scenes brittle. Content Collections provide a typed, validated, statically-analyzed alternative to a database or a scattered set of JSON imports.

Astro 6 uses the **Content Layer API** exclusively. The legacy v2 collections API (the one with `src/content/` magic auto-detection and no `loader`) was removed in Astro 6.

---

## 1. File Structure

```
src/
  content.config.ts        ← collection definitions (replaces old src/content/config.ts)
  content/
    scenes/
      hero.yaml
      product-viewer.yaml
      gallery.json          ← JSON works too
    models/
      manifest.json         ← single-file collection
```

---

## 2. Defining Collections in `src/content.config.ts`

```ts
// src/content.config.ts
import { defineCollection, z } from 'astro:content';
import { glob, file } from 'astro/loaders';

// --- Scene configuration collection ---
const scenes = defineCollection({
  loader: glob({ pattern: '**/*.{yaml,yml,json}', base: './src/content/scenes' }),
  schema: z.object({
    title: z.string(),
    modelPath: z.string(),           // relative to public/
    envMapPath: z.string().optional(),
    camera: z.object({
      position: z.tuple([z.number(), z.number(), z.number()]).default([0, 0, 5]),
      fov: z.number().min(10).max(120).default(75),
      near: z.number().default(0.1),
      far: z.number().default(1000),
    }),
    background: z.object({
      color: z.string().regex(/^#[0-9a-fA-F]{6}$/).optional(),
      skybox: z.string().optional(),
    }).optional(),
    lights: z.array(z.object({
      type: z.enum(['ambient', 'directional', 'point', 'spot']),
      color: z.string().default('#ffffff'),
      intensity: z.number().default(1),
      position: z.tuple([z.number(), z.number(), z.number()]).optional(),
    })).default([]),
    hotspots: z.array(z.object({
      id: z.string(),
      label: z.string(),
      position: z.tuple([z.number(), z.number(), z.number()]),
      content: z.string(),
    })).default([]),
    visible: z.boolean().default(true),
  }),
});

// --- Model manifest collection (single JSON file) ---
const models = defineCollection({
  loader: file('./src/content/models/manifest.json'),
  schema: z.object({
    name: z.string(),
    file: z.string(),
    format: z.enum(['glb', 'gltf', 'obj', 'fbx']),
    draco: z.boolean().default(false),
    ktx2: z.boolean().default(false),
    sizeBytes: z.number().optional(),
    tags: z.array(z.string()).default([]),
  }),
});

export const collections = { scenes, models };
```

---

## 3. Writing Scene Config Files

```yaml
# src/content/scenes/hero.yaml
title: "Hero Background Scene"
modelPath: "/assets/models/environment.glb"
envMapPath: "/assets/env/studio.hdr"
camera:
  position: [0, 1.5, 8]
  fov: 60
background:
  color: "#0a0a0f"
lights:
  - type: ambient
    intensity: 0.3
  - type: directional
    color: "#e8d4ff"
    intensity: 1.5
    position: [5, 10, 5]
hotspots: []
```

```yaml
# src/content/scenes/product-viewer.yaml
title: "Product Viewer"
modelPath: "/assets/models/product.glb"
envMapPath: "/assets/env/warehouse.hdr"
camera:
  position: [0, 0, 3]
  fov: 45
lights:
  - type: ambient
    intensity: 0.8
  - type: directional
    intensity: 1.2
    position: [2, 4, 2]
hotspots:
  - id: "feature-1"
    label: "USB-C Port"
    position: [0.42, -0.15, 0.3]
    content: "Supports 100W charging"
  - id: "feature-2"
    label: "Display"
    position: [0, 0.2, 0.45]
    content: "4K OLED, 120Hz"
```

---

## 4. Model Manifest (Single JSON File)

```json
[
  {
    "id": "environment",
    "name": "Environment",
    "file": "/assets/models/environment.glb",
    "format": "glb",
    "draco": true,
    "ktx2": false,
    "sizeBytes": 4200000,
    "tags": ["background", "static"]
  },
  {
    "id": "product",
    "name": "Product",
    "file": "/assets/models/product.glb",
    "format": "glb",
    "draco": true,
    "ktx2": true,
    "sizeBytes": 8100000,
    "tags": ["interactive", "product"]
  }
]
```

When using `file()` loader with a JSON array, each array element becomes a collection entry. The `id` field is automatically used as the entry identifier; if not present, Astro generates one.

---

## 5. Querying Collections in Pages

```astro
---
// src/pages/viewer/[scene].astro
import { getCollection, getEntry, render } from 'astro:content';
import SceneViewer from '../../components/SceneViewer';
import BaseLayout from '../../layouts/BaseLayout.astro';

export async function getStaticPaths() {
  const scenes = await getCollection('scenes', ({ data }) => data.visible !== false);
  return scenes.map(scene => ({
    params: { scene: scene.id },
    props: { scene },
  }));
}

const { scene } = Astro.props;
---
<BaseLayout>
  <slot slot="title">{scene.data.title}</slot>

  <!-- SSR: page title and meta are available to crawlers -->
  <h1>{scene.data.title}</h1>

  <!-- Island: 3D viewer receives typed, validated config as props -->
  <SceneViewer
    client:only="react"
    config={scene.data}
  />
</BaseLayout>
```

### Typed Props in the React Component

```tsx
// src/components/SceneViewer.tsx
import { Canvas } from '@react-three/fiber';
import { useGLTF, Environment } from '@react-three/drei';

// This type can be imported from a shared types file or inferred from the collection schema
interface SceneConfig {
  title: string;
  modelPath: string;
  envMapPath?: string;
  camera: {
    position: [number, number, number];
    fov: number;
    near: number;
    far: number;
  };
  lights: Array<{
    type: 'ambient' | 'directional' | 'point' | 'spot';
    color: string;
    intensity: number;
    position?: [number, number, number];
  }>;
  hotspots: Array<{
    id: string;
    label: string;
    position: [number, number, number];
    content: string;
  }>;
}

export default function SceneViewer({ config }: { config: SceneConfig }) {
  return (
    <Canvas
      camera={{
        position: config.camera.position,
        fov: config.camera.fov,
        near: config.camera.near,
        far: config.camera.far,
      }}
    >
      {config.envMapPath && <Environment files={config.envMapPath} />}
      {config.lights.map((light, i) => {
        if (light.type === 'ambient') {
          return <ambientLight key={i} color={light.color} intensity={light.intensity} />;
        }
        if (light.type === 'directional') {
          return (
            <directionalLight
              key={i}
              color={light.color}
              intensity={light.intensity}
              position={light.position}
            />
          );
        }
        return null;
      })}
      <Model path={config.modelPath} />
    </Canvas>
  );
}

function Model({ path }: { path: string }) {
  const { scene } = useGLTF(path);
  return <primitive object={scene} />;
}
```

---

## 6. Fetching a Single Entry

```astro
---
import { getEntry } from 'astro:content';

// Fetch by collection name + entry ID
const heroScene = await getEntry('scenes', 'hero');

if (!heroScene) {
  return Astro.redirect('/404');
}
---
<SceneViewer client:only="react" config={heroScene.data} />
```

---

## 7. Filtering and Sorting Collections

```ts
// All visible scenes
const allScenes = await getCollection('scenes', ({ data }) => data.visible !== false);

// Models that use Draco compression
const dracoModels = await getCollection('models', ({ data }) => data.draco === true);

// Sort scenes alphabetically
const sorted = allScenes.sort((a, b) => a.data.title.localeCompare(b.data.title));
```

---

## 8. Zod Schema Features Useful for 3D Config

| Zod method | Use case |
|---|---|
| `z.tuple([z.number(), z.number(), z.number()])` | XYZ position vectors |
| `z.enum(['glb', 'gltf', 'obj'])` | Limit to known formats |
| `z.string().url()` | Validate external texture URLs |
| `z.string().regex(/^#[0-9a-fA-F]{6}$/)` | Validate hex color codes |
| `z.number().min(0).max(1)` | Clamp opacity / blend values |
| `.default(value)` | Sensible defaults so configs stay minimal |
| `.optional()` | Fields that are only needed for some scenes |

Astro uses Zod 4 in Astro 6 (upgraded from Zod 3). The Zod 4 API is mostly compatible but some validation methods changed. If upgrading from Astro 5, check your schema definitions against [Zod 4 migration notes](https://zod.dev).

---

## 9. Static vs. Runtime Data

Content Collections are evaluated **at build time** for static output (`output: 'static'`). All YAML/JSON is parsed and validated during `astro build`, not at request time. This means:

- Invalid config (wrong type, missing required field) throws a build error, not a runtime error
- The config data is embedded in the generated HTML as JSON props on the island — no server needed
- Very large collections (thousands of entries) may increase build times; profile with `--verbose`

For data that changes frequently (e.g., real-time model availability), use a live collection (Astro 6 Content Layer API "live" mode) or a regular `fetch()` call inside the client-side component.

---

## Key Points

- Astro 6 requires the Content Layer API — the legacy auto-magic `src/content/` system without `loader` no longer exists.
- Use `glob()` for folders of YAML/JSON files; use `file()` for a single JSON array file.
- Zod schemas provide build-time type safety for 3D config (positions, colors, paths). A bad config value produces a build error, not a runtime WebGL crash.
- Pass `entry.data` (the validated, typed object) directly as props to a `client:only` island. The object is JSON-serialized by Astro.
- For routes driven by collections, use `getStaticPaths()` with `getCollection()` — this generates one page per scene entry.
