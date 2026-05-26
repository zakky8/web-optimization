# gltfjsx CLI Reference

gltfjsx converts GLTF/GLB files into declarative JSX/TSX components for React Three Fiber.
Repo: https://github.com/pmndrs/gltfjsx
Last push: November 2024 (marginal activity â€” verify before using in new projects)

## Installation

```bash
npm install -g gltfjsx
# or via npx (no install needed)
npx gltfjsx model.glb
```

## Full Flag Reference

```bash
npx gltfjsx model.glb [flags]

# Output format
-o, --output <file>      Output file (default: Model.jsx)
-t, --types              Emit TypeScript (.tsx) â€” adds typed ref
-r, --root <path>        Root path for asset imports
-p, --printwidth <n>     Prettier print width (default: 120)

# Optimization
--transform              Run gltf-transform optimize pipeline
--resolution <n>         Texture resolution for --transform (default: 1024)
--keepnames              Preserve original mesh/material names
--keepgroups             Preserve <group> hierarchy (default: flatten)
--shadows                Add castShadow/receiveShadow to meshes
--instanceall            Instance ALL repeated geometry (see note)
--wireframe              Add wireframe prop
--simplify               Apply mesh simplification
--ratio <f>              Simplify ratio 0-1 (used with --simplify)
--error <f>              Simplify error threshold (used with --simplify)

# Meta
-v, --version            Print version
```

## Official Size Reduction Claim

> "gltfjsx reduces 3D scenes by **70-90%**"

Source: pmndrs/gltfjsx README â€” accessed 2026-05-26
âš ï¸ Earlier docs said "90%" â€” corrected to "70-90%" by A23 validation agent.

## Typical Workflow

```bash
# 1. Compress model first (meshopt + ktx2)
npx gltf-transform optimize input.glb optimized.glb --texture-compress ktx2

# 2. Generate JSX with types + shadows + transform
npx gltfjsx optimized.glb -o src/components/Model.tsx --types --shadows --transform

# 3. Result: a typed R3F component
# function Model({ ...props }: JSX.IntrinsicElements['group']) {
#   const { nodes, materials } = useGLTF('/optimized-transformed.glb')
#   return (...)
# }
```

## --instanceall Warning

`--instanceall` creates `<Instances>` for every mesh that appears more than once.
Not all mesh configurations are instance-compatible â€” verify visual output.
Use `useInstances` from drei instead for fine-grained control.

## Sources
- pmndrs/gltfjsx: https://github.com/pmndrs/gltfjsx
- drei useGLTF: https://github.com/pmndrs/drei#useGLTF
