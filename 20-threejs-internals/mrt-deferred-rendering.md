# MRT & Deferred Rendering in Three.js

## WebGL2 Multiple Render Targets

Write to multiple textures in a single pass:

```js
// Three.js r155+ — WebGLRenderTarget with count
const gBuffer = new THREE.WebGLRenderTarget(width, height, {
  count: 4,  // 4 output textures
  minFilter: THREE.NearestFilter,
  magFilter: THREE.NearestFilter,
  type: THREE.HalfFloatType,
  format: THREE.RGBAFormat,
  depthBuffer: true,
  depthTexture: new THREE.DepthTexture(width, height, THREE.UnsignedInt248Type),
});

// Access individual textures
const albedoTexture  = gBuffer.textures[0];
const normalTexture  = gBuffer.textures[1];
const ormTexture     = gBuffer.textures[2];  // Occlusion/Roughness/Metalness
const emissiveTexture = gBuffer.textures[3];

// Render G-buffer pass
renderer.setRenderTarget(gBuffer);
renderer.render(gBufferScene, camera);
renderer.setRenderTarget(null);
```

## G-Buffer Shader (MRT Output)

```js
// Geometry pass material
const gBufferMat = new THREE.ShaderMaterial({
  glslVersion: THREE.GLSL3,
  vertexShader: `
    out vec3 vWorldNormal;
    out vec2 vUv;
    out vec3 vWorldPos;

    void main() {
      vec4 worldPos = modelMatrix * vec4(position, 1.0);
      vWorldPos     = worldPos.xyz;
      vWorldNormal  = normalize(mat3(modelMatrix) * normal);
      vUv           = uv;
      gl_Position   = projectionMatrix * viewMatrix * worldPos;
    }
  `,
  fragmentShader: `
    layout(location = 0) out vec4 gAlbedo;
    layout(location = 1) out vec4 gNormal;
    layout(location = 2) out vec4 gORM;
    layout(location = 3) out vec4 gEmissive;

    uniform sampler2D uAlbedoMap;
    uniform sampler2D uNormalMap;
    uniform sampler2D uOrmMap;

    in vec3 vWorldNormal;
    in vec2 vUv;
    in vec3 vWorldPos;

    void main() {
      gAlbedo  = texture(uAlbedoMap, vUv);
      gNormal  = vec4(normalize(vWorldNormal) * 0.5 + 0.5, 1.0);
      gORM     = texture(uOrmMap, vUv);
      gEmissive = vec4(0.0);
    }
  `,
});
```

## Deferred Lighting Pass

```js
// Full-screen quad reads G-buffer, applies all lights
const lightingMat = new THREE.ShaderMaterial({
  glslVersion: THREE.GLSL3,
  uniforms: {
    gAlbedo:   { value: gBuffer.textures[0] },
    gNormal:   { value: gBuffer.textures[1] },
    gORM:      { value: gBuffer.textures[2] },
    gDepth:    { value: gBuffer.depthTexture },
    uCameraPos: { value: camera.position },
    uLights:   { value: lightDataArray },
  },
  vertexShader: FULLSCREEN_VERT,
  fragmentShader: LIGHTING_FRAG,
});

const fsQuad = new THREE.Mesh(
  new THREE.PlaneGeometry(2, 2),
  lightingMat
);
// Render to screen
renderer.setRenderTarget(null);
renderer.render(lightingScene, orthoCamera);
```

## Post-Processing with EffectComposer

```js
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { SSAOPass } from 'three/addons/postprocessing/SSAOPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

const composer = new EffectComposer(renderer);

composer.addPass(new RenderPass(scene, camera));

const ssao = new SSAOPass(scene, camera, width, height);
ssao.kernelRadius = 16;
ssao.minDistance = 0.005;
ssao.maxDistance = 0.1;
composer.addPass(ssao);

const bloom = new UnrealBloomPass(
  new THREE.Vector2(width, height),
  0.8,   // strength
  0.4,   // radius
  0.85   // threshold
);
composer.addPass(bloom);

composer.addPass(new OutputPass());  // tone mapping + gamma

// Render
renderer.setAnimationLoop(() => {
  composer.render();
});

// IMPORTANT: dispose ALL passes manually (EffectComposer.dispose() misses some)
function disposeComposer(composer) {
  composer.passes.forEach(pass => {
    pass.dispose?.();
    if (pass.renderTargetBright) pass.renderTargetBright.dispose();
    if (pass.renderTargetsHorizontal) {
      pass.renderTargetsHorizontal.forEach(rt => rt.dispose());
      pass.renderTargetsVertical.forEach(rt => rt.dispose());
    }
  });
  composer.dispose();
}
```

## Sources
- Three.js WebGLRenderTarget: https://threejs.org/docs/#api/en/renderers/WebGLRenderTarget
- Three.js MRT example: https://threejs.org/examples/#webgl2_multiple_rendertargets
- Three.js postprocessing: https://threejs.org/docs/#examples/en/postprocessing/EffectComposer
