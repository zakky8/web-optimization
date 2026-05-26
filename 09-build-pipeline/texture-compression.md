# Texture Compression Pipeline

## Compression Formats

| Format | Tool | Use Case | VRAM Reduction |
|--------|------|----------|----------------|
| KTX2/ETC1S | toktx | Color maps, non-critical textures | 4-8Ã— |
| KTX2/UASTC | toktx | Normal maps, ORM, high-quality | 2-4Ã— |
| KTX2/ZSTD | gltf-transform | ETC1S with supercompression | Smaller file |
| Draco | gltf-transform | Mesh geometry | 70-98% mesh data |
| Meshopt | gltfpack | Geometry + quantization | 70-90% geometry |
| WebP | Sharp/Squoosh | Fallback for unsupported KTX2 | 25-35% vs PNG |

âš ï¸ "4-8Ã— VRAM" for KTX2 is the correct range. Claims of "8Ã—" alone are marketing.
âš ï¸ Draco reduction is "70-98%", not "50-70%". (Verified: A23 validation)

## toktx (KTX2 Creator)

```bash
# Install: https://github.com/KhronosGroup/KTX-Software/releases

# ETC1S â€” color maps (lossy, tiny file, universal GPU support via transcoding)
toktx --encode etc1s --clevel 4 --qlevel 192 output.ktx2 input.png

# UASTC â€” normal/ORM maps (higher quality, larger file)
toktx --encode uastc --uastc_quality 3 output.ktx2 input.png

# UASTC + ZSTD supercompression (best file size for normals)
toktx --encode uastc --uastc_quality 2 --zcmp 18 output.ktx2 input.png

# Flags
--clevel 0-5      ETC1S compression level (4 recommended)
--qlevel 0-255    ETC1S quality (192 = good default)
--uastc_quality 0-4  UASTC quality (2-3 recommended)
--zcmp 0-22       Zstandard level (18 recommended)
--mipmap          Generate mipmaps
--genmipmap       Generate mipmaps (alias)
```

## gltf-transform (Node.js CLI)

```bash
npm install -g @gltf-transform/cli

# Full optimization pipeline
gltf-transform optimize input.glb output.glb   --texture-compress ktx2   --vertex-compress draco

# Individual operations
gltf-transform draco input.glb output.glb           # Draco geometry compression
gltf-transform meshopt input.glb output.glb          # Meshopt compression
gltf-transform webp input.glb output.glb             # WebP textures
gltf-transform etc1s input.glb output.glb            # ETC1S for all textures
gltf-transform uastc input.glb output.glb            # UASTC for all textures
gltf-transform resize input.glb output.glb --width 1024 --height 1024

# Prune unused data
gltf-transform prune input.glb output.glb

# Inspect
gltf-transform inspect input.glb
```

## gltfpack

```bash
npm install -g gltfpack

# Flags
gltfpack -i input.glb -o output.glb
  -si 0.5       Simplify to 50% triangle count
  -cc           Enable compressed chunks (meshopt)
  -tc           Enable KTX2 texture compression (requires toktx)
  -tu           UASTC mode (instead of ETC1S)
  -kn           Keep named nodes
  -km           Keep named materials
  -ke           Keep extras/extensions

# Recommended for production
gltfpack -i model.glb -o model-compressed.glb -cc -tc -kn
```

## GitHub Actions Pipeline

```yaml
# .github/workflows/optimize-assets.yml
name: Optimize 3D Assets

on:
  push:
    paths:
      - 'public/models/**/*.glb'
      - 'public/models/**/*.gltf'

jobs:
  optimize:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install KTX-Software
        run: |
          wget https://github.com/KhronosGroup/KTX-Software/releases/latest/download/KTX-Software-*.deb
          sudo dpkg -i KTX-Software-*.deb

      - name: Install gltf-transform
        run: npm install -g @gltf-transform/cli

      - name: Optimize GLB files
        run: |
          for f in public/models/**/*.glb; do
            gltf-transform optimize "$f" "${f%.glb}-opt.glb" --texture-compress ktx2
          done

      - name: Commit optimized assets
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "chore: optimize 3D assets"
```

## Sources
- KTX-Software: https://github.com/KhronosGroup/KTX-Software
- gltf-transform CLI: https://gltf-transform.dev/cli
- gltfpack: https://meshoptimizer.org/gltf/
- Three.js KTX2 loader: https://threejs.org/docs/#examples/en/loaders/KTX2Loader
