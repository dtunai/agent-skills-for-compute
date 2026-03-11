---
name: omniverse-simready
description: "NVIDIA Omniverse SimReady — OpenUSD standard for physically accurate 3D assets with metadata for robotics, digital twins, synthetic data generation"
license: MIT
metadata:
  author: Agent Cluster
  tags: [omniverse, simready, openusd, usd, isaac-sim, digital-twins, robotics, simulation, physics, semantic-labeling]
---

# NVIDIA Omniverse SimReady Skill

Standard and ecosystem for physically accurate 3D assets built on OpenUSD with semantic labeling, physics properties, and metadata for robotics simulation, digital twins, and synthetic data generation.

**Official Sources:**
- [SimReady Overview](https://docs.omniverse.nvidia.com/simready/latest/overview.html)
- [SimReady Specification](https://docs.omniverse.nvidia.com/simready/latest/overview/simready-spec.html)
- [Asset Creation Guide](https://docs.omniverse.nvidia.com/simready/latest/simready-asset-creation.html)
- [Semantic Labeling](https://docs.omniverse.nvidia.com/simready/latest/sim-needs/semantic-labeling.html)
- [Physics Best Practices](https://docs.omniverse.nvidia.com/simready/latest/simready-asset-creation/physics-best-practices.html)
- [Isaac Sim Integration](https://docs.isaacsim.omniverse.nvidia.com/latest/assets/usd_assets_third_party.html)

## What is SimReady?

**Definition:**
> "Simulation-ready assets are physically accurate 3D objects that incorporate real-world properties, behaviors, and data bindings."

**Built on OpenUSD:**
- Universal Scene Description platform
- Modular, flexible composition
- Cross-platform compatibility

**More Than Visual:**
- Semantic labeling (WikiData integration)
- Physics properties (USDPhysics schema)
- Non-visual sensor attributes
- Dense captions for context

## Core Requirements

### Modeling Standards

```yaml
Scale:
  - Real-world scale
  - Z-up orientation
  - Export in meters
  - Pivot at origin

Geometry:
  - Optimal poly count
  - Clean topology
  - Named hierarchies
  - Forward-facing in viewport

UVs:
  - Non-overlapping islands
  - UV Channel 1 only
  - 0-1 UV space
  - 512+ pixels/meter texel density
```

### Material Standards

```yaml
Workflow:
  - PBR Metal-Rough mandatory
  - UsdPreviewSurface for portability
  - OmniPBR/OmniGlass/SimPBR for Omniverse

Textures:
  - 4K or 8K maximum resolution
  - One material per object

Shaders:
  - UsdPreviewSurface (cross-platform)
  - Omniverse MDL materials (advanced)
```

### Physics Requirements

```yaml
USDPhysics Schema:
  - Rigid body colliders
  - Mass in kilograms
  - Physical materials (friction, density, restitution)
  - Collision meshes (no convex hulls)

Compatibility:
  - PhysX
  - MuJoCo
  - Gazebo
  - Drake
```

## Quick Start

### Download SimReady Assets

```bash
# NVIDIA Physical AI Dataset
# Download from: https://developer.nvidia.com/isaac/sim

# Isaac Sim Asset Packs
# 1000+ SimReady assets (conveyors, boxes, pallets)
# Available in Isaac Sim
```

### Load in Isaac Sim

```python
import omni.isaac.core.utils.stage as stage_utils
from omni.isaac.core.utils.prims import create_prim

# Load SimReady asset
asset_path = "/Isaac/Environments/Simple_Warehouse/Props/S_TrafficCone.usd"
prim_path = "/World/Cone"

create_prim(
    prim_path=prim_path,
    usd_path=asset_path,
    position=[0, 0, 0]
)
```

### Inspect SimReady Properties

```python
from pxr import Usd, UsdPhysics, Semantics

# Open stage
stage = Usd.Stage.Open("asset.usd")
prim = stage.GetPrimAtPath("/World/Asset")

# Check physics
if prim.HasAPI(UsdPhysics.RigidBodyAPI):
    rb_api = UsdPhysics.RigidBodyAPI(prim)
    print(f"Mass: {rb_api.GetMassAttr().Get()}")

# Check semantic labels
if prim.HasAPI(Semantics.SemanticsAPI):
    sem_api = Semantics.SemanticsAPI(prim)
    labels = sem_api.GetSemanticLabelsRel().GetTargets()
    print(f"Labels: {labels}")
```

## Semantic Labeling

### WikiData Integration

```python
# SimReady schema metadata
metadata = {
    "class": "cup",                    # Human-readable
    "hierarchy": "container/cup",      # Taxonomy
    "qcode": "Q81727",                 # WikiData unique ID
    "sku": "PROD-12345"               # Custom identifier
}

# WikiData: 100M+ objects, multi-language
# Access: https://wikidata.org
# QCode provides:
# - Synonyms
# - Taxonomic subtrees
# - Multi-language translations
```

### Apply Semantic Labels

```python
from pxr import Semantics, Sdf

stage = Usd.Stage.Open("asset.usd")
prim = stage.GetPrimAtPath("/World/Asset")

# Add SemanticsAPI
if not prim.HasAPI(Semantics.SemanticsAPI):
    Semantics.SemanticsAPI.Apply(prim)

sem_api = Semantics.SemanticsAPI(prim)

# Create semantic label prim
label_path = Sdf.Path("/World/Semantics/Labels/Cup")
label_prim = stage.DefinePrim(label_path, "SemanticsLabel")

# Set properties
label_prim.CreateAttribute("semantic:class", Sdf.ValueTypeNames.String).Set("cup")
label_prim.CreateAttribute("semantic:hierarchy", Sdf.ValueTypeNames.String).Set("container/cup")
label_prim.CreateAttribute("semantic:qcode", Sdf.ValueTypeNames.String).Set("Q81727")

# Link to asset
sem_api.GetSemanticLabelsRel().AddTarget(label_path)

stage.Save()
```

## Physics Configuration

### Add Rigid Body

```python
from pxr import UsdPhysics, PhysxSchema

stage = Usd.Stage.Open("asset.usd")
root_prim = stage.GetPrimAtPath("/World/Asset")

# Apply RigidBodyAPI to root
if not root_prim.HasAPI(UsdPhysics.RigidBodyAPI):
    rb_api = UsdPhysics.RigidBodyAPI.Apply(root_prim)

# Set mass
mass_api = UsdPhysics.MassAPI.Apply(root_prim)
mass_api.GetMassAttr().Set(2.5)  # kg

stage.Save()
```

### Add Collision Mesh

```python
# Apply CollisionAPI to geometry
mesh_prim = stage.GetPrimAtPath("/World/Asset/Mesh")

if not mesh_prim.HasAPI(UsdPhysics.CollisionAPI):
    collision_api = UsdPhysics.CollisionAPI.Apply(mesh_prim)

# PhysX extension (optional)
if not mesh_prim.HasAPI(PhysxSchema.PhysxCollisionAPI):
    physx_collision = PhysxSchema.PhysxCollisionAPI.Apply(mesh_prim)
    physx_collision.GetContactOffsetAttr().Set(0.02)
    physx_collision.GetRestOffsetAttr().Set(0.0)

stage.Save()
```

### Add Physical Material

```python
# Create physics material
material_path = Sdf.Path("/World/PhysicsMaterials/Metal")
material_prim = stage.DefinePrim(material_path, "PhysicsMaterial")

# Apply PhysicsMaterialAPI
physics_mat = UsdPhysics.MaterialAPI.Apply(material_prim)
physics_mat.CreateStaticFrictionAttr().Set(0.5)
physics_mat.CreateDynamicFrictionAttr().Set(0.4)
physics_mat.CreateRestitutionAttr().Set(0.3)
physics_mat.CreateDensityAttr().Set(7850.0)  # kg/m^3 for steel

# Bind to collision mesh
binding_api = UsdShade.MaterialBindingAPI.Apply(mesh_prim)
binding_api.Bind(UsdShade.Material(material_prim),
                 bindingStrength=UsdShade.Tokens.weakerThanDescendants,
                 materialPurpose="physics")

stage.Save()
```

## Asset Creation Workflow

### 1. Modeling

```yaml
Requirements:
  - Real-world scale (meters)
  - Z-up orientation
  - Clean topology
  - Optimal poly count
  - Named hierarchies
  - Pivot at origin
  - Faces forward in viewport

Tools:
  - Maya
  - Blender
  - 3ds Max
  - Houdini
```

### 2. UV Layout

```yaml
Standards:
  - Non-overlapping islands
  - UV Channel 1 only
  - 0-1 UV space
  - 512+ pixels/meter texel density
  - Align to material patterns
  - Re-UV CAD models

Tools:
  - RizomUV
  - Maya UV Toolkit
  - Blender UV Editor
```

### 3. Materials

```yaml
Workflow: PBR Metal-Rough
Required Maps:
  - Base Color (albedo)
  - Metallic
  - Roughness
  - Normal (optional)
  - Opacity (if needed)

Resolution: 4K or 8K
Format: PNG, TIFF, or EXR
```

### 4. Physics Setup

```yaml
Colliders:
  - Multiple components allowed
  - No convex hull requirement
  - Per-mesh configuration

Mass:
  - Set in kilograms
  - Real-world accurate

Materials:
  - Static friction: 0-1
  - Dynamic friction: 0-1
  - Restitution: 0-1
  - Density: kg/m^3
```

### 5. Semantic Labeling

```yaml
Required Metadata:
  - Class: Human-readable name
  - Hierarchy: Taxonomy path
  - QCode: WikiData identifier
  - Custom: SKU, ProductID

Consistency: Use same taxonomy across dataset
```

## Integration with Isaac Sim

### Load SimReady Assets

```python
from omni.isaac.core import World
from omni.isaac.core.utils.stage import add_reference_to_stage

world = World()

# Add SimReady asset
add_reference_to_stage(
    usd_path="/Isaac/Props/Pallets/pallet.usd",
    prim_path="/World/Pallet"
)

world.reset()
```

### Synthetic Data Generation

```python
import omni.replicator.core as rep

# Setup camera
camera = rep.create.camera(position=(5, 5, 5))

# Render with semantic labels
rp = rep.create.render_product(camera, (1024, 1024))

# Annotators
rep.AnnotatorRegistry.get_annotator("rgb")
rep.AnnotatorRegistry.get_annotator("semantic_segmentation")
rep.AnnotatorRegistry.get_annotator("bounding_box_2d_tight")
rep.AnnotatorRegistry.get_annotator("distance_to_camera")

# Randomize assets
with rep.trigger.on_frame():
    rep.randomizer.scatter_2d(
        surface_prims=["/World/Ground"],
        asset_prim="/World/Pallet"
    )

rep.orchestrator.run(num_frames=1000)
```

## Available Asset Categories

```yaml
Warehouse/Logistics:
  - Pallets (wood, plastic, metal)
  - Boxes (cardboard, plastic)
  - Conveyors
  - Shelving
  - Racks
  - Forklifts

Manufacturing:
  - Machinery
  - Tools
  - Parts bins
  - Work surfaces

Infrastructure:
  - Traffic cones
  - Barriers
  - Signs
  - Bollards

Generic Props:
  - Containers
  - Furniture
  - Equipment
```

## Validation

### Check SimReady Compliance

```python
from pxr import Usd, UsdPhysics, Semantics

def validate_simready(asset_path):
    stage = Usd.Stage.Open(asset_path)
    root = stage.GetDefaultPrim()

    checks = {
        "has_physics": root.HasAPI(UsdPhysics.RigidBodyAPI),
        "has_mass": root.HasAPI(UsdPhysics.MassAPI),
        "has_semantics": root.HasAPI(Semantics.SemanticsAPI),
        "scale_in_meters": True,  # Manual check
        "clean_hierarchy": True   # Manual check
    }

    # Check for colliders in children
    has_collider = False
    for prim in Usd.PrimRange(root):
        if prim.HasAPI(UsdPhysics.CollisionAPI):
            has_collider = True
            break
    checks["has_collider"] = has_collider

    return checks

# Validate
results = validate_simready("asset.usd")
print(results)
```

## Best Practices

### Modeling
1. **Real-world scale**: Export in meters
2. **Optimization**: Balance detail vs performance
3. **Naming**: Use clear hierarchies
4. **Pivot points**: Position for proper placement

### UVs
1. **Non-overlapping**: Required for baking
2. **Texel density**: Maintain consistency
3. **Channel 1**: Don't use additional channels
4. **Pattern alignment**: Before packing

### Physics
1. **Multi-component**: Use multiple colliders
2. **Mass accuracy**: Real-world values
3. **Material tuning**: Dial in friction/restitution
4. **Runtime compatibility**: USDPhysics schema

### Semantics
1. **Consistent taxonomy**: Across all assets
2. **WikiData QCodes**: For interoperability
3. **Hierarchical labels**: Enable discovery
4. **Custom IDs**: SKUs for tracking

## Common Use Cases

```yaml
Robotics:
  - Robot training (manipulation, navigation)
  - Sensor simulation (cameras, LIDAR)
  - Synthetic data generation
  - Sim-to-real transfer

Digital Twins:
  - Factory layouts
  - Warehouse optimization
  - Datacenter planning
  - Infrastructure design

Autonomous Vehicles:
  - Street scene simulation
  - Perception training
  - Scenario testing
  - Sensor validation

Healthcare:
  - Facility planning
  - Safety analysis
  - Equipment placement
  - Workflow optimization
```

## References

- **[Specification](references/specification.md)** - SimReady standard, requirements, scope
- **[Asset Creation](references/asset-creation.md)** - Modeling, UVs, materials, workflow
- **[Physics Setup](references/physics-setup.md)** - USDPhysics, colliders, mass, materials
- **[Semantic Labeling](references/semantic-labeling.md)** - WikiData, taxonomy, metadata
- **[USD Structure](references/usd-structure.md)** - OpenUSD, composition, schemas
- **[Isaac Sim Integration](references/isaac-sim-integration.md)** - Loading assets, synthetic data
