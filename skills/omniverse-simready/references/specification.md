# SimReady Specification Reference

Sources:
- [SimReady Specification](https://docs.omniverse.nvidia.com/simready/latest/overview/simready-spec.html)
- [SimReady Overview](https://docs.omniverse.nvidia.com/simready/latest/overview.html)

## Overview

The SimReady specification defines standards for creating 3D assets optimized for machine learning and simulation applications built on OpenUSD.

## Core Definition

**SimReady Assets:**
> "Versatile digital models optimized for use across various OpenUSD-compatible simulation platforms."

**Key Principle:**
Assets are defined by their **capabilities** (metadata and semantic information) rather than rigid structural hierarchies.

## What Makes an Asset SimReady

### Essential Capabilities

**1. Semantic Labeling**
- Embedded metadata describing asset properties
- WikiData integration for universal taxonomy
- Hierarchical classification
- Custom identifiers (SKU, ProductID)

**2. Physics Properties**
- USDPhysics schema implementation
- Rigid-body dynamics
- Collision meshes
- Physical materials (friction, density, restitution)
- Mass properties

**3. Visual Fidelity**
- Photoreal appearance
- PBR materials
- Proper texturing
- Real-world scale

**4. Simulation Metadata**
- Non-visual sensor attributes
- Dense captions
- Annotation support
- Domain-specific properties

## Current Scope

### Focus: Static Props

**Static Props Defined:**
- Non-moving objects
- Non-articulating components
- Rigid structures
- Environmental elements

**Examples:**
- Pallets
- Boxes
- Crates
- Traffic cones
- Shelving units
- Containers

**Not Yet Covered:**
- Articulated robots
- Deformable objects
- Animated characters
- Dynamic machinery

## Design Philosophy

### Flexibility Over Rigidity

**Multiple Valid Approaches:**
The specification allows different implementation methods as long as core requirements are met.

**Capability-Driven:**
Assets prove SimReady through functionality, not structure.

**Extensible:**
Specification grows with community needs and technological advances.

### Standardization Goals

**Interoperability:**
- Work across multiple simulation platforms
- PhysX, MuJoCo, Gazebo, Drake compatibility
- OpenUSD foundation ensures portability

**Consistency:**
- Predictable behavior
- Reliable metadata
- Standard schemas

**Reusability:**
- Modular composition
- USD references and variants
- Mix-and-match capabilities

## Technical Requirements

### OpenUSD Foundation

**Base Platform:**
- Universal Scene Description (OpenUSD)
- USD composition arcs
- USD schemas (Physics, Semantics)
- Cross-platform compatibility

**File Format:**
```
.usd   - Binary USD
.usda  - ASCII USD (human-readable)
.usdc  - Crate USD (compressed binary)
.usdz  - Packaged USD archive
```

### Schema Requirements

**USDPhysics Schema:**
```python
# Rigid body
UsdPhysics.RigidBodyAPI
UsdPhysics.MassAPI

# Collision
UsdPhysics.CollisionAPI
UsdPhysics.MeshCollisionAPI

# Materials
UsdPhysics.MaterialAPI
```

**Semantics Schema:**
```python
# Semantic labeling
Semantics.SemanticsAPI
Semantics.SemanticsLabelAPI
```

**UsdShade Schema:**
```python
# Materials
UsdShade.Material
UsdShade.Shader
UsdShade.MaterialBindingAPI
```

## Metadata Standards

### Required Metadata

**Semantic Labels:**
- Class (human-readable name)
- Hierarchy (taxonomy path)
- QCode (WikiData identifier)

**Physics Properties:**
- Mass (kilograms)
- Collider definitions
- Material properties

**Visual Properties:**
- Material assignments
- Texture bindings
- Shader networks

### Optional Metadata

**Custom Identifiers:**
- SKU numbers
- Product IDs
- Asset codes
- Version tags

**Domain-Specific:**
- Sensor properties
- Behavioral annotations
- Performance hints
- LOD information

## Validation Criteria

### Must Have

1. **Real-world scale** (meters)
2. **Physics colliders** (USDPhysics)
3. **Mass properties** (kilograms)
4. **Semantic labels** (consistent taxonomy)
5. **PBR materials** (Metal-Rough workflow)
6. **Clean geometry** (optimized topology)
7. **Proper UVs** (non-overlapping, 0-1 space)

### Should Have

1. **WikiData QCodes** (universal taxonomy)
2. **Physical materials** (friction, density, restitution)
3. **Dense captions** (contextual descriptions)
4. **Sensor attributes** (radar, LIDAR properties)
5. **Multiple LODs** (performance optimization)

### Nice to Have

1. **Variants** (color, configuration options)
2. **Animation support** (non-static elements)
3. **Custom properties** (domain extensions)
4. **Documentation** (usage notes)

## Compatibility Requirements

### Simulation Platforms

**Supported:**
- NVIDIA Omniverse
- Isaac Sim
- PhysX
- MuJoCo
- Gazebo
- Drake

**Through OpenUSD:**
- Any USD-compatible simulator
- Custom simulation frameworks
- Research platforms

### Rendering Engines

**Primary:**
- NVIDIA RTX
- Omniverse RTX

**Secondary:**
- USD Hydra delegates
- Custom renderers

## Use Case Alignment

### Robotics Simulation

**Requirements:**
- Accurate physics
- Sensor properties
- Semantic labels
- Manipulation testing

**Assets:**
- Objects for grasping
- Obstacles
- Work surfaces
- Tools

### Digital Twins

**Requirements:**
- Visual accuracy
- Layout precision
- Real-world measurements
- Scalability

**Assets:**
- Infrastructure
- Equipment
- Containers
- Furniture

### Autonomous Vehicles

**Requirements:**
- Street-level detail
- Traffic elements
- Environmental props
- Sensor validation

**Assets:**
- Vehicles
- Signs
- Barriers
- Street furniture

### Synthetic Data Generation

**Requirements:**
- Semantic segmentation
- Annotation support
- Randomization capability
- Domain coverage

**Assets:**
- Diverse object types
- Variant support
- Clean geometry
- Proper labeling

## Evolution and Extensions

### Current Focus

**Phase 1: Static Props**
- Rigid, non-articulating objects
- Foundation for future expansion
- Proven workflows

### Future Directions

**Planned Extensions:**
- Articulated assets (robots, machinery)
- Deformable objects (cloth, soft bodies)
- Animated characters
- Dynamic environments

**Community Input:**
- Open specification process
- GitHub proposals
- Alliance for OpenUSD (AOUSD)

### Versioning

**Specification Versions:**
- Semantic versioning
- Backward compatibility
- Migration guides
- Deprecation notices

## Best Practices

### For Asset Creators

1. **Start with specification**: Read before creating
2. **Use templates**: Follow proven workflows
3. **Validate early**: Test in target simulators
4. **Document decisions**: Explain custom choices
5. **Maintain consistency**: Same standards across library

### For Asset Users

1. **Check compliance**: Validate before use
2. **Report issues**: Help improve specification
3. **Share learnings**: Contribute to community
4. **Test thoroughly**: Verify in your pipeline

### For Platform Developers

1. **Support standards**: Implement USDPhysics, Semantics
2. **Provide feedback**: Shape specification evolution
3. **Document integration**: Help asset creators
4. **Contribute schemas**: Extend OpenUSD

## Resources

### Documentation

- SimReady specification docs
- OpenUSD documentation
- Omniverse developer guides
- Physics schema references

### Tools

- Omniverse Create (asset authoring)
- Isaac Sim (robotics simulation)
- SimReady Explorer (asset browser)
- USD Composer (scene assembly)

### Community

- NVIDIA Developer Forums
- Alliance for OpenUSD (AOUSD)
- GitHub repositories
- Technical blogs

## Compliance Checklist

```yaml
Geometry:
  ☐ Real-world scale (meters)
  ☐ Z-up orientation
  ☐ Clean topology
  ☐ Named hierarchy
  ☐ Optimal poly count

UVs:
  ☐ Non-overlapping
  ☐ UV Channel 1 only
  ☐ 0-1 space
  ☐ 512+ pixels/meter

Materials:
  ☐ PBR Metal-Rough
  ☐ UsdPreviewSurface
  ☐ 4K/8K textures
  ☐ One material per object

Physics:
  ☐ USDPhysics schema
  ☐ Collision meshes
  ☐ Mass properties
  ☐ Physical materials

Semantics:
  ☐ Class label
  ☐ Hierarchy
  ☐ WikiData QCode
  ☐ Consistent taxonomy

USD:
  ☐ Proper file format
  ☐ Clean stage
  ☐ Schema compliance
  ☐ Composition arcs
