# SimReady USD Structure Reference

Sources:
- [OpenUSD Documentation](https://openusd.org/release/index.html)
- [Omniverse USD](https://docs.omniverse.nvidia.com/usd/latest/)

## OpenUSD Foundation

### Universal Scene Description

**USD is:**
- Scene description format
- Composition framework
- Schema system
- Interchange standard

**SimReady built on USD provides:**
- Cross-platform compatibility
- Modular asset composition
- Schema extensibility
- Future-proof workflows

## File Formats

### USD File Types

```python
# Binary (fastest)
.usd   # Generic binary
.usdc  # Crate format (recommended)

# Text (human-readable)
.usda  # ASCII USD

# Package (with dependencies)
.usdz  # ZIP archive containing USD + textures
```

### When to Use

```python
# Development: .usda (readable, version control)
# Production: .usdc (fast, compact)
# Distribution: .usdz (self-contained)
```

## Stage Hierarchy

### Typical SimReady Asset Structure

```
/World
  /Asset                    # Root prim (RigidBodyAPI, MassAPI)
    /Mesh_01                # Visual mesh (CollisionAPI)
    /Mesh_02                # Visual mesh (CollisionAPI)
    /Mesh_03                # Visual mesh (CollisionAPI)
/Materials
  /Metal                    # Material definition
  /Plastic                  # Material definition
/PhysicsMaterials
  /MetalPhysics             # Physics material
  /PlasticPhysics           # Physics material
/Semantics
  /Labels
    /Container              # Semantic label
    /Recyclable             # Semantic label
```

## Composition Arcs

### References

```python
from pxr import Usd, Sdf

# Create reference to external USD
stage = Usd.Stage.CreateNew("scene.usd")
asset_prim = stage.DefinePrim("/World/Asset", "Xform")

# Add reference
asset_prim.GetReferences().AddReference(
    assetPath="./assets/pallet.usd",
    primPath="/Pallet"
)
```

### Payloads

```python
# Lazy-loading for large assets
asset_prim.GetPayloads().AddPayload(
    assetPath="./assets/warehouse.usd",
    primPath="/Warehouse"
)

# Load/unload dynamically
stage.Load("/World/Asset")
stage.Unload("/World/Asset")
```

### Variants

```python
# Create variant set
vset = asset_prim.GetVariantSets().AddVariantSet("color")

# Add variants
vset.AddVariant("red")
vset.SetVariantSelection("red")
with vset.GetVariantEditContext():
    # Define red variant
    material_binding.Bind(red_material)

vset.AddVariant("blue")
vset.SetVariantSelection("blue")
with vset.GetVariantEditContext():
    # Define blue variant
    material_binding.Bind(blue_material)

# Select variant
vset.SetVariantSelection("red")
```

## USD Schemas

### Core Schemas

**UsdGeom:** Geometry primitives
```python
from pxr import UsdGeom

mesh = UsdGeom.Mesh.Define(stage, "/World/Asset/Mesh")
mesh.CreatePointsAttr(points)
mesh.CreateFaceVertexIndicesAttr(indices)
mesh.CreateFaceVertexCountsAttr(counts)
```

**UsdShade:** Materials and shaders
```python
from pxr import UsdShade

material = UsdShade.Material.Define(stage, "/Materials/Metal")
shader = UsdShade.Shader.Define(stage, "/Materials/Metal/Shader")
```

**UsdPhysics:** Physics properties
```python
from pxr import UsdPhysics

UsdPhysics.RigidBodyAPI.Apply(prim)
UsdPhysics.CollisionAPI.Apply(prim)
UsdPhysics.MassAPI.Apply(prim)
```

**Semantics:** Semantic labeling
```python
from pxr import Semantics

Semantics.SemanticsAPI.Apply(prim)
```

### Schema Application

```python
# Check if schema applied
if prim.HasAPI(UsdPhysics.RigidBodyAPI):
    print("Has RigidBodyAPI")

# Apply schema
UsdPhysics.RigidBodyAPI.Apply(prim)

# Remove schema
prim.RemoveAPI(UsdPhysics.RigidBodyAPI)
```

## Attributes and Relationships

### Attributes

```python
# Create attribute
mass_attr = prim.CreateAttribute(
    "physics:mass",
    Sdf.ValueTypeNames.Float
)
mass_attr.Set(5.0)

# Get attribute
mass = prim.GetAttribute("physics:mass").Get()

# Time-sampled attributes
mass_attr.Set(5.0, time=0)
mass_attr.Set(10.0, time=100)
```

### Relationships

```python
# Create relationship
material_rel = prim.CreateRelationship("material:binding")
material_rel.AddTarget("/Materials/Metal")

# Get targets
targets = material_rel.GetTargets()
```

## Metadata

### Prim Metadata

```python
# Set metadata
prim.SetMetadata("kind", "component")
prim.SetMetadata("assetInfo", {"version": "1.0"})

# Custom metadata
prim.SetCustomDataByKey("simready:validated", True)
prim.SetCustomDataByKey("simready:version", "2.0")
```

### Stage Metadata

```python
# Up axis
stage.SetMetadata("upAxis", "Z")

# Meters per unit
stage.SetMetadata("metersPerUnit", 1.0)

# Default prim
stage.SetDefaultPrim(root_prim)
```

## Layer Stack

### Layer Composition

```python
# Root layer
root_layer = stage.GetRootLayer()

# Sublayers (strongest to weakest)
root_layer.subLayerPaths = [
    "./overrides.usd",
    "./base.usd"
]

# Session layer (temporary edits)
session_layer = stage.GetSessionLayer()
```

### Layer Editing

```python
# Edit in specific layer
with Usd.EditContext(stage, root_layer):
    prim.GetAttribute("physics:mass").Set(10.0)

# Edit in session layer (temporary)
with Usd.EditContext(stage, session_layer):
    prim.GetAttribute("xformOp:translate").Set((5, 0, 0))
```

## Best Practices

### File Organization

```
assets/
  props/
    pallet/
      pallet.usd          # Main asset
      textures/           # Texture files
      collision/          # Collision meshes (optional)
  materials/
    materials.usd         # Shared materials
  scenes/
    warehouse.usd         # Scene composition
```

### Naming Conventions

```python
# Prim paths: CamelCase
/World/MyAsset/MeshComponent

# Attributes: namespace:attributeName
physics:mass
semantic:class

# Files: lowercase_with_underscores
my_asset.usd
wood_material.usd
```

### Performance

1. **Use .usdc**: Faster than .usda
2. **Payloads for large assets**: Lazy-loading
3. **Instancing**: Reuse geometry
4. **Variants**: Color/config options
5. **Clean hierarchy**: Remove unnecessary prims
