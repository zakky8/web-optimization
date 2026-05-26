# Typing ShaderMaterial in Three.js r184

> **Package versions used in this document**
> - `three`: `0.184.0`
> - `@types/three`: `0.184.0`
> - TypeScript: `>=5.0`
> - Accessed: 2026-05-26

---

## 1. The `Uniform` Class and `IUniform<T>` Type

Three.js models shader uniforms through the `Uniform` class. The TypeScript types in
`@types/three` expose `IUniform<T>` as a generic interface:

```ts
// From @types/three (simplified)
interface IUniform<T = any> {
  value: T;
}

class Uniform<T = any> implements IUniform<T> {
  value: T;
  constructor(value: T);
}
```

> **status: verified** — `IUniform` is defined in `@types/three`. The `Uniform` class is
> documented as a core Three.js type.
> Source: [threejs.org/docs Uniform](https://threejs.org/docs/#api/en/core/Uniform),
> [three-types/three-ts-types](https://github.com/three-types/three-ts-types),
> accessed 2026-05-26.

### GLSL type → JavaScript/TypeScript mapping

> **status: verified** — mappings taken directly from Three.js manual.
> Source: [threejs.org/manual uniform-types](https://threejs.org/manual/en/uniform-types.html),
> accessed 2026-05-26.

| GLSL type | TypeScript value type |
|-----------|----------------------|
| `float` | `number` |
| `int` / `uint` | `number` |
| `bool` | `boolean \| number` |
| `vec2` | `THREE.Vector2 \| Float32Array \| [number, number]` |
| `vec3` | `THREE.Vector3 \| THREE.Color \| Float32Array \| [number, number, number]` |
| `vec4` | `THREE.Vector4 \| THREE.Quaternion \| Float32Array \| [number, number, number, number]` |
| `mat3` | `THREE.Matrix3 \| Float32Array` |
| `mat4` | `THREE.Matrix4 \| Float32Array` |
| `sampler2D` | `THREE.Texture` |
| `samplerCube` | `THREE.CubeTexture` |

---

## 2. Typing `ShaderMaterial` Uniforms

### Typed uniform map interface pattern

```ts
import * as THREE from 'three';

// 1. Define an interface for your uniform values
interface WaveUniforms {
  uTime: IUniform<number>;
  uAmplitude: IUniform<number>;
  uFrequency: IUniform<number>;
  uColor: IUniform<THREE.Color>;
  uResolution: IUniform<THREE.Vector2>;
  uTexture: IUniform<THREE.Texture | null>;
}

// 2. Create the uniforms object, satisfying the interface
const waveUniforms: WaveUniforms = {
  uTime:       { value: 0.0 },
  uAmplitude:  { value: 0.5 },
  uFrequency:  { value: 2.0 },
  uColor:      { value: new THREE.Color(0x00aaff) },
  uResolution: { value: new THREE.Vector2(window.innerWidth, window.innerHeight) },
  uTexture:    { value: null },
};

// 3. Pass to ShaderMaterial
const material = new THREE.ShaderMaterial({
  uniforms: waveUniforms,
  vertexShader: /* glsl */`
    uniform float uTime;
    uniform float uAmplitude;
    uniform float uFrequency;
    varying vec2 vUv;

    void main() {
      vUv = uv;
      vec3 pos = position;
      pos.z += sin(pos.x * uFrequency + uTime) * uAmplitude;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: /* glsl */`
    uniform vec3 uColor;
    uniform sampler2D uTexture;
    varying vec2 vUv;

    void main() {
      vec4 texColor = texture2D(uTexture, vUv);
      gl_FragColor = vec4(uColor, 1.0) * texColor;
    }
  `,
});
```

### Updating typed uniforms at runtime

```ts
// Full type safety — TypeScript knows the value type for each key
function tick(delta: number): void {
  waveUniforms.uTime.value += delta;
  waveUniforms.uResolution.value.set(window.innerWidth, window.innerHeight);
  // waveUniforms.uTime.value = "string" // Error: Type 'string' is not assignable to type 'number'
}
```

---

## 3. Custom Material Classes with TypeScript

Subclassing `ShaderMaterial` is the cleanest way to encapsulate typed uniforms and keep
them co-located with the shaders.

### Basic typed subclass

```ts
import * as THREE from 'three';

// Typed uniform shape
interface HologramParameters {
  color?: THREE.ColorRepresentation;
  speed?: number;
  glitchStrength?: number;
}

interface HologramUniforms {
  uTime:          IUniform<number>;
  uColor:         IUniform<THREE.Color>;
  uSpeed:         IUniform<number>;
  uGlitchStrength: IUniform<number>;
}

class HologramMaterial extends THREE.ShaderMaterial {
  // Re-declare uniforms with the narrow type so callers get full intellisense
  declare uniforms: HologramUniforms;

  constructor(params: HologramParameters = {}) {
    const {
      color = 0x00ffcc,
      speed = 1.0,
      glitchStrength = 0.05,
    } = params;

    super({
      uniforms: {
        uTime:           { value: 0 },
        uColor:          { value: new THREE.Color(color) },
        uSpeed:          { value: speed },
        uGlitchStrength: { value: glitchStrength },
      } satisfies HologramUniforms,
      vertexShader: /* glsl */`
        uniform float uTime;
        uniform float uSpeed;
        varying vec2 vUv;
        varying float vNoise;

        // Simple pseudo-random
        float random(vec2 st) {
          return fract(sin(dot(st, vec2(12.9898, 78.233))) * 43758.5453);
        }

        void main() {
          vUv = uv;
          vec3 pos = position;
          float glitch = step(0.98, random(vec2(floor(pos.y * 10.0), uTime * uSpeed)));
          pos.x += glitch * 0.05;
          gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
        }
      `,
      fragmentShader: /* glsl */`
        uniform vec3 uColor;
        uniform float uGlitchStrength;
        uniform float uTime;
        varying vec2 vUv;

        void main() {
          float scanLine = sin(vUv.y * 100.0 + uTime * 5.0) * 0.05;
          vec3 col = uColor + vec3(scanLine);
          float alpha = 0.7 + scanLine;
          gl_FragColor = vec4(col, alpha);
        }
      `,
      transparent: true,
      side: THREE.DoubleSide,
    });
  }

  tick(delta: number): void {
    this.uniforms.uTime.value += delta;
  }
}

export { HologramMaterial, HologramParameters };
```

### Using the custom material class

```ts
const material = new HologramMaterial({ color: 0x00ff99, speed: 2.0 });

const mesh = new THREE.Mesh(
  new THREE.TorusKnotGeometry(1, 0.3, 128, 32),
  material,
);
scene.add(mesh);

// In animation loop — full type safety
renderer.setAnimationLoop((_, delta) => {
  material.tick(delta);
  renderer.render(scene, camera);
});
```

### Using in React Three Fiber

```tsx
import { useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import * as THREE from 'three';
import { HologramMaterial } from './HologramMaterial';

function HologramMesh() {
  const matRef = useRef<HologramMaterial>(null);

  useFrame((_, delta) => {
    matRef.current?.tick(delta);
  });

  return (
    <mesh>
      <torusKnotGeometry args={[1, 0.3, 128, 32]} />
      {/* primitiveObject bypasses R3F's JSX catalog for custom classes */}
      <primitive object={new HologramMaterial({ color: 0x00ff99 })} ref={matRef} attach="material" />
    </mesh>
  );
}
```

---

## 4. Typing `onBeforeCompile`

`onBeforeCompile` receives a `WebGLProgramParametersWithUniforms` object (the compiled
shader object). You can inject GLSL and add uniforms before the program is linked.

```ts
// Type signature (from @types/three)
type WebGLProgramParametersWithUniforms = {
  vertexShader: string;
  fragmentShader: string;
  uniforms: { [key: string]: IUniform };
  // ... renderer internals
};

// onBeforeCompile is a method on Material
onBeforeCompile?: (
  parameters: WebGLProgramParametersWithUniforms,
  renderer: THREE.WebGLRenderer
) => void;
```

> **status: verified** — `onBeforeCompile` signature from Three.js source and `@types/three`.
> The argument type is `WebGLProgramParametersWithUniforms` in r184 (previously `Shader`).
> Source: [three.js forum — onBeforeCompile PR #11475](https://github.com/mrdoob/three.js/issues/11475),
> accessed 2026-05-26.

### Typed `onBeforeCompile` usage

```ts
import * as THREE from 'three';

// Shared ref so we can update the uniform value externally
const uniformRef = { uTime: { value: 0.0 } };

const material = new THREE.MeshStandardMaterial({ color: 0x888888 });

material.onBeforeCompile = (shader) => {
  // Inject our custom uniforms into the compiled shader's uniform block
  shader.uniforms.uTime = uniformRef.uTime;

  // Prepend declarations to vertex shader
  shader.vertexShader = `
    uniform float uTime;
    varying float vWave;
    ${shader.vertexShader}
  `.replace(
    '#include <begin_vertex>',
    /* glsl */`
    #include <begin_vertex>
    float wave = sin(position.x * 3.0 + uTime * 2.0) * 0.1;
    transformed.y += wave;
    vWave = wave;
    `
  );

  shader.fragmentShader = `
    varying float vWave;
    ${shader.fragmentShader}
  `.replace(
    '#include <dithering_fragment>',
    /* glsl */`
    #include <dithering_fragment>
    gl_FragColor.rgb += vWave * 0.5;
    `
  );
};

// Update each frame
function tick(delta: number): void {
  uniformRef.uTime.value += delta;
}
```

### Encapsulated `onBeforeCompile` utility

```ts
import * as THREE from 'three';

interface ShaderPatch {
  uniforms?: Record<string, IUniform>;
  vertexPrepend?: string;
  fragmentPrepend?: string;
  replacements?: Array<{ token: string; replacement: string; target: 'vertex' | 'fragment' }>;
}

function patchMaterial(
  material: THREE.Material,
  patch: ShaderPatch
): void {
  material.onBeforeCompile = (shader) => {
    // Inject uniforms
    if (patch.uniforms) {
      Object.assign(shader.uniforms, patch.uniforms);
    }

    // Prepend declarations
    if (patch.vertexPrepend) {
      shader.vertexShader = patch.vertexPrepend + '\n' + shader.vertexShader;
    }
    if (patch.fragmentPrepend) {
      shader.fragmentShader = patch.fragmentPrepend + '\n' + shader.fragmentShader;
    }

    // Apply token replacements
    for (const r of patch.replacements ?? []) {
      if (r.target === 'vertex') {
        shader.vertexShader = shader.vertexShader.replace(r.token, r.replacement);
      } else {
        shader.fragmentShader = shader.fragmentShader.replace(r.token, r.replacement);
      }
    }
  };

  // Necessary for cache busting when patch changes
  material.needsUpdate = true;
}
```

---

## 5. Typed `RawShaderMaterial`

`RawShaderMaterial` skips Three.js's built-in shader chunks — you write the full shader.
TypeScript typing is identical to `ShaderMaterial`:

```ts
import * as THREE from 'three';

interface ParticleUniforms {
  uTime:       IUniform<number>;
  uPointSize:  IUniform<number>;
  uCameraFar:  IUniform<number>;
}

const particleUniforms: ParticleUniforms = {
  uTime:      { value: 0 },
  uPointSize: { value: 3.0 },
  uCameraFar: { value: 1000 },
};

const particleMaterial = new THREE.RawShaderMaterial({
  uniforms: particleUniforms,
  vertexShader: /* glsl */`
    precision highp float;
    attribute vec3 position;
    attribute vec2 uv;

    uniform mat4 modelViewMatrix;
    uniform mat4 projectionMatrix;
    uniform float uTime;
    uniform float uPointSize;

    void main() {
      vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
      gl_PointSize = uPointSize * (300.0 / -mvPosition.z);
      gl_Position = projectionMatrix * mvPosition;
    }
  `,
  fragmentShader: /* glsl */`
    precision highp float;

    void main() {
      vec2 uv = gl_PointCoord - vec2(0.5);
      float dist = length(uv);
      if (dist > 0.5) discard;
      float alpha = 1.0 - smoothstep(0.4, 0.5, dist);
      gl_FragColor = vec4(1.0, 1.0, 1.0, alpha);
    }
  `,
  transparent: true,
  depthWrite: false,
});
```

---

## 6. Struct Uniforms in TypeScript

GLSL structs map to plain JavaScript objects in uniform values:

```ts
// GLSL:
// struct PointLight {
//   vec3 position;
//   vec3 color;
//   float intensity;
// };
// uniform PointLight uLights[4];

interface PointLightStruct {
  position: THREE.Vector3;
  color: THREE.Vector3;
  intensity: number;
}

interface SceneUniforms {
  uLights: IUniform<PointLightStruct[]>;
  uLightCount: IUniform<number>;
}

const sceneUniforms: SceneUniforms = {
  uLights: {
    value: [
      { position: new THREE.Vector3(5, 5, 0), color: new THREE.Vector3(1, 0.8, 0.6), intensity: 2.0 },
      { position: new THREE.Vector3(-5, 3, 2), color: new THREE.Vector3(0.5, 0.7, 1.0), intensity: 1.5 },
    ],
  },
  uLightCount: { value: 2 },
};
```

> **status: verified** — struct uniform pattern from Three.js uniform types manual.
> Source: [threejs.org/manual/uniform-types.html](https://threejs.org/manual/en/uniform-types.html),
> accessed 2026-05-26.

---

## 7. TSL (Three Shading Language) — Type-Safe Alternative to GLSL Strings

r184 ships TSL (`three/tsl`) as a node-based, fully TypeScript-aware shading API.
It eliminates GLSL strings entirely and compiles to GLSL (WebGL) or WGSL (WebGPU).

> **status: verified** — TSL shipped as a first-class API in r184.
> Source: [TSL specification docs](https://threejs.org/docs/TSL.html),
> [Three.js r184 release](https://github.com/mrdoob/three.js/releases/tag/r184),
> accessed 2026-05-26.

```ts
import {
  color,
  mix,
  uv,
  positionLocal,
  uniform,
  vec3,
  sin,
  time,
  Fn,
} from 'three/tsl';
import * as THREE from 'three/webgpu';

// uniform() returns a typed UniformNode — fully type-checked
const uHovered = uniform(0.0); // UniformNode<number>
const uBaseColor = uniform(new THREE.Color(0x00aaff)); // UniformNode<Color>

// Shader logic is pure TypeScript — no string interpolation
const colorNode = Fn(() => {
  const baseCol = vec3(uBaseColor);
  const hoverCol = vec3(color(0xff6600));
  return mix(baseCol, hoverCol, uHovered);
});

const material = new THREE.MeshStandardNodeMaterial();
material.colorNode = colorNode();

// Update at runtime — type-safe
function onHoverEnter(): void {
  uHovered.value = 1.0;
}

function onHoverLeave(): void {
  uHovered.value = 0.0;
}
```

---

## Sources

- [Three.js Uniform docs](https://threejs.org/docs/#api/en/core/Uniform) (accessed 2026-05-26)
- [Three.js uniform types manual](https://threejs.org/manual/en/uniform-types.html) (accessed 2026-05-26)
- [three-types/three-ts-types](https://github.com/three-types/three-ts-types) (accessed 2026-05-26)
- [TSL specification](https://threejs.org/docs/TSL.html) (accessed 2026-05-26)
- [TSL: A Better Way to Write Shaders — Three.js Roadmap blog](https://threejsroadmap.com/blog/tsl-a-better-way-to-write-shaders-in-threejs) (accessed 2026-05-26)
- [onBeforeCompile GitHub issue #11475](https://github.com/mrdoob/three.js/issues/11475) (accessed 2026-05-26)
- [Three.js custom shader material — blog.cjgammon.com](https://blog.cjgammon.com/threejs-custom-shader-material/) (accessed 2026-05-26)
- [THREE-CustomShaderMaterial — FarazzShaikh](https://github.com/FarazzShaikh/THREE-CustomShaderMaterial) (accessed 2026-05-26)
- [blog.pragmattic.dev — R3F WebGPU TypeScript](https://blog.pragmattic.dev/react-three-fiber-webgpu-typescript) (accessed 2026-05-26)
