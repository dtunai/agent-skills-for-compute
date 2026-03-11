# SimReady Physics Setup Reference

Sources:
- [Physics Best Practices](https://docs.omniverse.nvidia.com/simready/latest/simready-asset-creation/physics-best-practices.html)

## USDPhysics Schema

### Foundation
SimReady uses Pixar's USDPhysics schema for cross-platform compatibility with PhysX, MuJoCo, Gazebo, and Drake.

### Three Core Components

**1. Collider**
```python
from pxr import UsdPhysics

# Apply collision to mesh
mesh = stage.GetPrimAtPath("/World/Asset/Mesh")
UsdPhysics.CollisionAPI.Apply(mesh)
```

**2. Mass**
```python
# Set mass in kilograms
root = stage.GetPrimAtPath("/World/Asset")
mass_api = UsdPhysics.MassAPI.Apply(root)
mass_api.GetMassAttr().Set(5.0)  # 5 kg
```

**3. Physical Material**
```python
# Create physics material
mat_path = Sdf.Path("/PhysicsMaterials/Wood")
mat_prim = stage.DefinePrim(mat_path, "PhysicsMaterial")

physics_mat = UsdPhysics.MaterialAPI.Apply(mat_prim)
physics_mat.CreateStaticFrictionAttr().Set(0.6)
physics_mat.CreateDynamicFrictionAttr().Set(0.5)
physics_mat.CreateRestitutionAttr().Set(0.2)
physics_mat.CreateDensityAttr().Set(700.0)  # kg/m^3

# Bind to collision
UsdShade.MaterialBindingAPI.Apply(mesh).Bind(
    UsdShade.Material(mat_prim),
    materialPurpose="physics"
)
```

## Rigid Body Setup

### Root-Level Control

```python
# Apply RigidBodyAPI to root prim
root = stage.GetPrimAtPath("/World/Asset")

if not root.HasAPI(UsdPhysics.RigidBodyAPI):
    rb_api = UsdPhysics.RigidBodyAPI.Apply(root)

# Configure rigid body
rb_api.CreateRigidBodyEnabledAttr().Set(True)
rb_api.CreateKinematicEnabledAttr().Set(False)  # Dynamic
rb_api.CreateVelocityAttr().Set((0, 0, 0))
rb_api.CreateAngularVelocityAttr().Set((0, 0, 0))
```

### Per-Mesh Colliders

```python
# Child meshes automatically contribute colliders
for prim in Usd.PrimRange(root):
    if prim.IsA(UsdGeom.Mesh):
        # Apply collision
        if not prim.HasAPI(UsdPhysics.CollisionAPI):
            UsdPhysics.CollisionAPI.Apply(prim)
```

## Collision Mesh Best Practices

### Multiple Components

**SimReady allows multiple collision components** (no convex hull requirement):

```python
# Example: Chair with separate collision for each part
- /Chair/Seat      → CollisionAPI
- /Chair/Back      → CollisionAPI
- /Chair/Leg1      → CollisionAPI
- /Chair/Leg2      → CollisionAPI
- /Chair/Leg3      → CollisionAPI
- /Chair/Leg4      → CollisionAPI
```

### Simplified Geometry

```python
# Use simplified meshes for collision
- Visual mesh: 50K triangles
- Collision mesh: 500 triangles

# Create collision mesh in modeling software
- Fit bounding volume
- Simplify topology
- Test in physics sim
```

### Convex Decomposition

```python
from pxr import PhysxSchema

# PhysX convex decomposition (optional)
collision = stage.GetPrimAtPath("/World/Asset/Mesh")

physx_collision = PhysxSchema.PhysxCollisionAPI.Apply(collision)
physx_collision.CreateConvexDecompositionEnabledAttr().Set(True)
```

## Material Tuning

### Friction

**Static Friction (0-1):**
- 0.0: Ice
- 0.3: Smooth metal
- 0.6: Wood
- 0.9: Rubber

**Dynamic Friction (0-1):**
- Slightly less than static
- Affects sliding behavior

### Restitution (Bounciness)

**Values (0-1):**
- 0.0: Clay (no bounce)
- 0.2: Wood
- 0.5: Hard plastic
- 0.9: Rubber ball

### Density

**Common Materials (kg/m^3):**
```python
densities = {
    "styrofoam": 50,
    "wood": 700,
    "water": 1000,
    "plastic": 1200,
    "concrete": 2400,
    "aluminum": 2700,
    "steel": 7850,
    "lead": 11340
}
```

## Mass Configuration

### Automatic from Density

```python
# Set density, mass auto-calculated
mass_api = UsdPhysics.MassAPI.Apply(root)
mass_api.CreateDensityAttr().Set(7850.0)  # Steel
# Mass = volume × density
```

### Manual Mass

```python
# Set explicit mass
mass_api.CreateMassAttr().Set(15.5)  # kg
```

### Center of Mass

```python
# Override center of mass
mass_api.CreateCenterOfMassAttr().Set((0, 0, 0.5))
```

## PhysX Extensions (Optional)

### Advanced Contact

```python
physx_collision = PhysxSchema.PhysxCollisionAPI.Apply(collision)

# Contact offset (meters)
physx_collision.CreateContactOffsetAttr().Set(0.02)

# Rest offset (meters)
physx_collision.CreateRestOffsetAttr().Set(0.0)
```

### Simulation Control

```python
physx_rb = PhysxSchema.PhysxRigidBodyAPI.Apply(root)

# Solver iterations
physx_rb.CreateSolverPositionIterationCountAttr().Set(4)
physx_rb.CreateSolverVelocityIterationCountAttr().Set(1)

# Sleep threshold
physx_rb.CreateSleepThresholdAttr().Set(0.005)
```

## Testing and Validation

### In Isaac Sim

```python
from omni.isaac.core import World

world = World()
world.scene.add_default_ground_plane()

# Load SimReady asset
world.scene.add(
    DynamicCuboid(
        prim_path="/World/Asset",
        usd_path="asset.usd",
        position=(0, 0, 5)
    )
)

# Drop test
world.reset()
for i in range(100):
    world.step(render=True)
```

### Common Issues

**Asset falls through floor:**
- Check collision mesh exists
- Verify mass > 0
- Ensure ground has collision

**Unstable simulation:**
- Reduce solver iterations
- Simplify collision geometry
- Adjust contact offsets

**Unrealistic behavior:**
- Tune friction values
- Adjust restitution
- Check mass/density

## Best Practices

1. **Use USDPhysics**: Cross-platform compatibility
2. **Multiple components**: Don't force convex hulls
3. **Simplified collision**: Lower poly than visual
4. **Real-world values**: Measure actual objects
5. **Material tuning**: Most important step
6. **Test early**: Validate in target simulator
