# GPU Tier Detection

## @pmndrs/detect-gpu

Industry standard library for detecting device GPU capability.
Used by R3F PerformanceMonitor internally.

```javascript
import { getGPUTier } from '@pmndrs/detect-gpu'

async function initWithTierDetection() {
  const { tier, isMobile, type, fps } = await getGPUTier({
    mobileTiers: [15, 30, 60]  // FPS thresholds for tier 1/2/3
  })

  // type values: 'WEBGL_UNSUPPORTED' | 'BLACKLISTED' | 'BENCHMARK' | 'FALLBACK'

  if (tier === 0 || type === 'WEBGL_UNSUPPORTED') {
    showStaticFallback()  // No WebGL: serve static image, no canvas
    return
  }

  const settings = getSettingsForTier(tier, isMobile)
  initRenderer(settings)
}

function getSettingsForTier(tier, isMobile) {
  return {
    pixelRatio:     isMobile ? Math.min(devicePixelRatio, 1.5) : Math.min(devicePixelRatio, 2),
    antialias:      tier >= 2,
    shadows:        tier >= 2,
    shadowMapSize:  tier >= 3 ? 1024 : 512,
    bloom:          tier >= 3 && !isMobile,
    ssao:           tier >= 3 && !isMobile,
    textureRes:     isMobile ? 1024 : (tier >= 3 ? 2048 : 1024),
    particles:      tier === 1 ? 1000 : tier === 2 ? 5000 : 20000,
    postProcessing: !isMobile && tier >= 2,
  }
}
```

## Tier Definitions

| Tier | FPS Benchmark | Examples | Treatment |
|------|--------------|----------|-----------|
| 0 | No WebGL | Ancient browsers, blocked | Static image fallback |
| 1 | < 15fps | Old iPhone, budget Android | Minimal: no shadows, no post-FX, low poly |
| 2 | 15-30fps | Mid-range mobile, old desktop | Reduced quality, no expensive passes |
| 3 | > 30fps | Modern mobile, desktop | Full experience |

## R3F PerformanceMonitor - Adaptive Real-Time

Adjusts DPR dynamically based on actual measured FPS:

```jsx
import { PerformanceMonitor } from '@react-three/drei'

function App() {
  const [dpr, setDpr] = useState(2)
  const [quality, setQuality] = useState('high')

  return (
    <PerformanceMonitor
      ms={250}          // measure over 250ms windows
      iterations={5}    // 5 iterations to confirm change
      threshold={0.1}   // 10% variance threshold
      bounds={(refreshrate) => [refreshrate/2, refreshrate]}
      onDecline={() => {
        setDpr(d => Math.max(1, d - 0.2))
        if (dpr <= 1.2) setQuality('low')
      }}
      onIncline={() => {
        setDpr(d => Math.min(2, d + 0.1))
      }}
      flipflops={3}     // after 3 flip-flops, use fallback quality permanently
      onFallback={() => setQuality('minimal')}
    >
      <Canvas dpr={dpr}>
        <Scene quality={quality} />
      </Canvas>
    </PerformanceMonitor>
  )
}
```

## Combined Detection Pattern (Production)

```javascript
import { getGPUTier } from '@pmndrs/detect-gpu'

async function detectAndInit() {
  // GPU capability
  const gpu = await getGPUTier()

  // Network quality
  const conn = navigator.connection
  const networkTier = !conn ? 'high'
    : conn.saveData ? 'low'
    : ({ 'slow-2g':'low', '2g':'low', '3g':'medium', '4g':'high' }[conn.effectiveType] ?? 'high')

  // User preferences
  const reducedMotion = matchMedia('(prefers-reduced-motion: reduce)').matches

  // Final quality decision
  const quality = (
    gpu.tier === 0         ? 'none'   :
    reducedMotion          ? 'low'    :
    networkTier === 'low'  ? 'low'    :
    gpu.tier === 1         ? 'low'    :
    networkTier === 'medium' || gpu.tier === 2 ? 'medium' :
    'high'
  )

  return { quality, isMobile: gpu.isMobile, gpu, networkTier }
}
```

## Note on detect-gpu Data Currency

gfxbench.com (detect-gpu's benchmark data source) stopped updating December 2025.
GPUs released after that date (A19, new Snapdragon variants) won't have benchmark entries.
These fall back to tier 2 by default.
The @pmndrs team is evaluating alternative data sources.
