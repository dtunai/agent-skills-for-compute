# SimReady Asset Creation Reference

Sources:
- [Asset Creation Guide](https://docs.omniverse.nvidia.com/simready/latest/simready-asset-creation.html)

## Modeling Guidelines

### Scale and Orientation
- **Scale**: Real-world measurements in meters
- **Orientation**: Z-up coordinate system
- **Pivot**: Position at origin, asset sits on ground plane
- **Direction**: Faces forward in viewport

### Geometry
- **Topology**: Clean edge flow, optimal poly count
- **Curvature**: Balance detail vs performance
- **Naming**: Descriptive, avoid Box01/Cube02
- **Hierarchy**: Logical parent-child relationships

### Optimization
- **Poly count**: Only as much as needed
- **Instances**: Reuse geometry where possible
- **LODs**: Create multiple detail levels (future)
- **Cleanup**: Remove hidden faces, n-gons

## UV Layout Standards

### Requirements
- **Islands**: Non-overlapping only
- **Channel**: UV Channel 1 exclusively
- **Space**: 0-1 UV space
- **Density**: 512+ pixels/meter minimum

### Workflow
1. **Align**: Orient UVs to material patterns
2. **Pack**: Maximize UV space utilization
3. **Validate**: Check for overlaps
4. **Scale**: Match texel density across assets

### CAD Models
- **Re-UV**: CAD exports require complete re-mapping
- **Tools**: Use RizomUV, Maya UV Toolkit, Blender
- **Consistency**: Match hand-modeled asset standards

## Material Creation

### PBR Metal-Rough Workflow

**Required Maps:**
- Base Color (albedo, no lighting)
- Metallic (0=dielectric, 1=metal)
- Roughness (0=glossy, 1=rough)
- Normal (tangent space)
- Opacity (if needed)

**Resolution:**
- 4K (4096x4096) standard
- 8K (8192x8192) for hero assets
- Power-of-two dimensions

### UsdPreviewSurface

```python
from pxr import UsdShade, Sdf

# Create material
material_path = Sdf.Path("/Materials/Metal")
material = UsdShade.Material.Define(stage, material_path)

# Create shader
shader = UsdShade.Shader.Define(stage, material_path.AppendChild("Shader"))
shader.CreateIdAttr("UsdPreviewSurface")

# Set inputs
shader.CreateInput("diffuseColor", Sdf.ValueTypeNames.Color3f).Set((0.8, 0.8, 0.8))
shader.CreateInput("metallic", Sdf.ValueTypeNames.Float).Set(1.0)
shader.CreateInput("roughness", Sdf.ValueTypeNames.Float).Set(0.3)

# Connect output
shader.CreateOutput("surface", Sdf.ValueTypeNames.Token)
material.CreateSurfaceOutput().ConnectToSource(shader.ConnectableAPI(), "surface")
```

### Omniverse MDL Materials

**OmniPBR:**
- Extended PBR with additional features
- Clearcoat, sheen, transmission
- Omniverse-specific optimizations

**OmniGlass:**
- Physically accurate glass
- Refraction, absorption
- Thin-wall mode

**SimPBR:**
- Optimized for simulation
- Fast rendering
- Physics integration

## Export Settings

### From Maya

```python
# USD export settings
- File Format: .usd or .usda
- Up Axis: Z
- Unit: centimeter → meter conversion
- Materials: Export shading networks
- Normals: Export vertex normals
```

### From Blender

```python
# USD export addon
- Scale: 0.01 (cm to m)
- Up Axis: Z
- Forward Axis: -Y
- Apply Modifiers: Yes
- Materials: UsdPreviewSurface
```

### From 3ds Max

```python
# USD export plugin
- System Units: Meters
- Up Axis: Z
- Materials: Physical Material → UsdPreviewSurface
- Export: Selected or all
```

## Validation

### Geometry Checks
- ☐ Real-world scale
- ☐ Z-up orientation
- ☐ Clean topology
- ☐ No ngons
- ☐ Proper normals
- ☐ Named meshes

### UV Checks
- ☐ Non-overlapping
- ☐ UV Channel 1
- ☐ 0-1 space
- ☐ Consistent density
- ☐ No distortion

### Material Checks
- ☐ PBR workflow
- ☐ Correct maps
- ☐ Proper resolution
- ☐ One material per object
- ☐ USD shaders

## Best Practices

1. **Reference real objects**: Measure, photograph
2. **Optimize early**: Don't add unnecessary detail
3. **Test in simulator**: Validate before finalizing
4. **Document decisions**: Note any deviations
5. **Maintain library**: Consistent standards across assets
