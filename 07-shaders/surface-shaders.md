# Surface Shaders — Triplanar, POM, Toon, Dissolve, Hologram

## Triplanar Mapping (Seamless UV-free Texturing)

Samples a texture 3x along world X, Y, Z axes and blends by surface normal.
Eliminates UV seams on complex meshes.

```glsl
// vertex shader
varying vec3 vWorldPos;
varying vec3 vWorldNormal;

void main() {
  vWorldPos    = (modelMatrix * vec4(position, 1.0)).xyz;
  vWorldNormal = normalize(mat3(modelMatrix) * normal);
  gl_Position  = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}

// fragment shader
uniform sampler2D uTexture;
uniform float     uScale;
varying vec3 vWorldPos;
varying vec3 vWorldNormal;

vec4 triplanar(sampler2D tex, vec3 worldPos, vec3 worldNormal, float scale) {
  vec3 blendWeights = abs(worldNormal);
  blendWeights = pow(blendWeights, vec3(4.0));         // sharpen blending
  blendWeights /= dot(blendWeights, vec3(1.0));        // normalize

  vec4 xSample = texture2D(tex, worldPos.yz * scale);
  vec4 ySample = texture2D(tex, worldPos.xz * scale);
  vec4 zSample = texture2D(tex, worldPos.xy * scale);

  return xSample * blendWeights.x
       + ySample * blendWeights.y
       + zSample * blendWeights.z;
}

void main() {
  gl_FragColor = triplanar(uTexture, vWorldPos, vWorldNormal, uScale);
}
```

## Parallax Occlusion Mapping (POM)

Raymarches the height field in tangent space to fake deep surface displacement.

```glsl
// Fragment shader POM
uniform sampler2D uHeightMap;
uniform float     uHeightScale;   // e.g. 0.1
uniform int       uMinLayers;     // 8
uniform int       uMaxLayers;     // 32

vec2 parallaxOcclusionMapping(vec2 uv, vec3 viewDir) {
  float numLayers = mix(
    float(uMaxLayers),
    float(uMinLayers),
    abs(dot(vec3(0.0, 0.0, 1.0), viewDir))
  );

  float layerDepth   = 1.0 / numLayers;
  float currentDepth = 0.0;
  vec2  P            = viewDir.xy / viewDir.z * uHeightScale;
  vec2  deltaUV      = P / numLayers;

  vec2  currentUV  = uv;
  float currentMap = texture2D(uHeightMap, currentUV).r;

  while (currentDepth < currentMap) {
    currentUV    -= deltaUV;
    currentMap    = texture2D(uHeightMap, currentUV).r;
    currentDepth += layerDepth;
  }

  // Linear interpolation refinement
  vec2 prevUV       = currentUV + deltaUV;
  float afterDepth  = currentMap - currentDepth;
  float beforeDepth = texture2D(uHeightMap, prevUV).r - currentDepth + layerDepth;
  float weight      = afterDepth / (afterDepth - beforeDepth);
  return mix(currentUV, prevUV, weight);
}
```

## Toon / Cel Shading

```javascript
const toonMaterial = new THREE.ShaderMaterial({
  uniforms: {
    uColor:    { value: new THREE.Color(0x4488ff) },
    uSteps:    { value: 4.0 },
    uLightDir: { value: new THREE.Vector3(1, 1, 1).normalize() },
  },
  vertexShader: `
    varying vec3 vNormal;
    void main() {
      vNormal = normalize(normalMatrix * normal);
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform vec3  uColor;
    uniform float uSteps;
    uniform vec3  uLightDir;
    varying vec3  vNormal;
    void main() {
      float NdotL  = dot(vNormal, normalize(uLightDir));
      float stepped = ceil(NdotL * uSteps) / uSteps;
      stepped       = clamp(stepped, 0.1, 1.0);
      gl_FragColor  = vec4(uColor * stepped, 1.0);
    }
  `
});

// Outline: render backfaces slightly enlarged
const outlineMaterial = new THREE.ShaderMaterial({
  side: THREE.BackSide,
  uniforms: { uOutlineWidth: { value: 0.03 } },
  vertexShader: `
    uniform float uOutlineWidth;
    void main() {
      vec3 expanded = position + normal * uOutlineWidth;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(expanded, 1.0);
    }
  `,
  fragmentShader: `void main() { gl_FragColor = vec4(0.0, 0.0, 0.0, 1.0); }`
});
```

## Dissolve / Burn Shader

```glsl
// Fragment shader
uniform float uProgress;   // 0 = visible, 1 = dissolved
uniform vec3  uBurnColor;  // e.g. vec3(1.0, 0.3, 0.0)
uniform float uBurnWidth;  // e.g. 0.05
varying vec2  vUv;

void main() {
  float noiseVal = fbm(vUv * 3.0); // 0..1

  if (noiseVal < uProgress) discard;

  vec4 texColor = texture2D(uTexture, vUv);

  float edgeDist = noiseVal - uProgress;
  float burnMask = 1.0 - smoothstep(0.0, uBurnWidth, edgeDist);
  vec3  finalColor = mix(texColor.rgb, uBurnColor, burnMask);

  gl_FragColor = vec4(finalColor, texColor.a);
}
```

Three.js setup: `material.transparent = true; material.side = THREE.DoubleSide;`

## Hologram Shader

```glsl
// Vertex shader
uniform float uTime;
varying vec3 vWorldNormal;
varying vec3 vViewDir;
varying vec3 vWorldPos;

void main() {
  // Vertex jitter for flickering
  vec3 displaced = position + normal * sin(position.y * 20.0 + uTime * 5.0) * 0.003;
  vec4 worldPos = modelMatrix * vec4(displaced, 1.0);
  vWorldPos    = worldPos.xyz;
  vWorldNormal = normalize(mat3(modelMatrix) * normal);
  vViewDir     = normalize(cameraPosition - worldPos.xyz);
  gl_Position = projectionMatrix * viewMatrix * worldPos;
}

// Fragment shader
uniform float uTime;
uniform vec3  uHoloColor;    // e.g. vec3(0.0, 1.0, 0.8)
uniform float uScanDensity;  // e.g. 80.0

varying vec3 vWorldNormal;
varying vec3 vViewDir;
varying vec3 vWorldPos;

void main() {
  float NdotV   = dot(normalize(vWorldNormal), normalize(vViewDir));
  float fresnel = pow(1.0 - NdotV, 3.0);

  float scanLine = step(0.5, fract(vWorldPos.y * uScanDensity + uTime * 2.0));
  float flicker  = 1.0 - 0.05 * fract(sin(uTime * 17.3) * 43758.5);

  float alpha = (fresnel * 0.8 + 0.2) * (scanLine * 0.4 + 0.6) * flicker;

  gl_FragColor = vec4(uHoloColor * (fresnel + 0.3), alpha);
}
```

Three.js setup:
```javascript
material.transparent = true;
material.depthWrite = false;
material.blending = THREE.AdditiveBlending;
```
