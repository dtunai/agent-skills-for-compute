# SimReady Semantic Labeling Reference

Sources:
- [Semantic Labeling](https://docs.omniverse.nvidia.com/simready/latest/sim-needs/semantic-labeling.html)
- [USD Semantic Schema](https://github.com/NVIDIA-Omniverse/USD-proposals/blob/main/proposals/semantic_schema/README.md)

## Overview

Semantic labeling provides "embedded metadata that describes the properties of an asset," enabling simulations to understand what they're "seeing" within a stage.

## WikiData Integration

### Why WikiData?

**Challenge**: Different organizations use different names
- "car" vs "automobile" vs "sedan" vs "vehicle"
- All technically correct, but semantically distinct

**Solution**: WikiData provides universal taxonomy
- 100M+ objects, brands, locations
- Unique QCode identifiers (e.g., Q81727 for "cup")
- Language-independent
- Synonym lists
- Taxonomic subtrees
- Multi-language translations

**Access**: https://wikidata.org

### QCode Benefits

```python
# QCode example for "cup" (Q81727)
qcode_data = {
    "id": "Q81727",
    "label": {
        "en": "cup",
        "es": "taza",
        "fr": "tasse",
        "de": "Tasse"
    },
    "synonyms": ["drinking vessel", "mug", "tumbler"],
    "hierarchy": ["container", "tableware", "drinkware"]
}
```

## SimReady Label Schema

### Four Core Elements

**1. Class**
Human-readable object classification
```python
class_label = "cup"
```

**2. Hierarchy**
Ordered relational structure for asset discovery
```python
hierarchy = "container/tableware/drinkware/cup"
```

**3. QCode**
WikiData unique identifier
```python
qcode = "Q81727"
```

**4. Augmented Identifiers**
Custom additions (SKU, ProductID)
```python
custom_ids = {
    "sku": "PROD-12345",
    "barcode": "012345678901",
    "manufacturer_code": "MFG-CUP-001"
}
```

## Implementation

### Using USDSemanticLabels API

```python
from pxr import Usd, Semantics, Sdf

stage = Usd.Stage.CreateNew("asset.usd")
asset_prim = stage.DefinePrim("/World/Cup", "Xform")

# Apply SemanticsAPI
if not asset_prim.HasAPI(Semantics.SemanticsAPI):
    Semantics.SemanticsAPI.Apply(asset_prim)

sem_api = Semantics.SemanticsAPI(asset_prim)

# Create semantic label prim
label_path = Sdf.Path("/World/Semantics/Labels/Cup")
label_prim = stage.DefinePrim(label_path, "SemanticsLabel")

# Set label properties
label_prim.CreateAttribute(
    "semantic:class",
    Sdf.ValueTypeNames.String
).Set("cup")

label_prim.CreateAttribute(
    "semantic:hierarchy",
    Sdf.ValueTypeNames.String
).Set("container/tableware/drinkware/cup")

label_prim.CreateAttribute(
    "semantic:qcode",
    Sdf.ValueTypeNames.String
).Set("Q81727")

# Custom identifiers
label_prim.CreateAttribute(
    "semantic:sku",
    Sdf.ValueTypeNames.String
).Set("PROD-12345")

# Link label to asset
sem_api.GetSemanticLabelsRel().AddTarget(label_path)

stage.Save()
```

### Multiple Labels

```python
# Asset can have multiple semantic labels
# Example: Object is both "container" and "recyclable"

label_container = stage.DefinePrim("/Semantics/Labels/Container", "SemanticsLabel")
label_container.CreateAttribute("semantic:class", Sdf.ValueTypeNames.String).Set("container")

label_recyclable = stage.DefinePrim("/Semantics/Labels/Recyclable", "SemanticsLabel")
label_recyclable.CreateAttribute("semantic:class", Sdf.ValueTypeNames.String).Set("recyclable_item")

# Add both labels
sem_api.GetSemanticLabelsRel().AddTarget(label_container.GetPath())
sem_api.GetSemanticLabelsRel().AddTarget(label_recyclable.GetPath())
```

## Annotation Types

Semantic labels enable generation of annotated imagery:

### 2D Bounding Boxes

```python
# Tight bounding box around object instances
# Format: [x_min, y_min, x_max, y_max]
# Use: Object detection training
```

### Instance Segmentation

```python
# Per-pixel instance masks
# Each object instance gets unique ID
# Use: Instance-aware perception
```

### Semantic Segmentation

```python
# Per-pixel class masks
# All cups get same class ID
# Use: Scene understanding
```

### Depth Maps

```python
# Distance to camera per pixel
# Use: 3D reconstruction, SLAM
```

## Taxonomy Design

### Consistent Taxonomy is Critical

**Before Starting:**
1. Define taxonomy for entire dataset
2. Document naming conventions
3. Create taxonomy tree
4. Get team agreement

**Example Taxonomy:**
```
vehicle/
  ├─ car/
  │  ├─ sedan
  │  ├─ coupe
  │  └─ hatchback
  ├─ truck/
  │  ├─ pickup
  │  └─ semi
  └─ emergency_vehicle/
     ├─ fire_engine
     ├─ ambulance
     └─ police_car
```

### Hierarchical Labels

**Benefits:**
- Enable broad searches ("all vehicles")
- Support fine-grained queries ("fire engines only")
- Facilitate asset discovery
- Support multi-level annotations

**Implementation:**
```python
# Hierarchical labeling
labels = [
    "/Semantics/Labels/Vehicle",           # Top level
    "/Semantics/Labels/EmergencyVehicle",  # Mid level
    "/Semantics/Labels/FireEngine"         # Specific
]

for label_path in labels:
    sem_api.GetSemanticLabelsRel().AddTarget(Sdf.Path(label_path))
```

## Integration with Replicator

### Synthetic Data Generation

```python
import omni.replicator.core as rep

# Setup randomization with semantic labels
with rep.trigger.on_frame():
    # Randomize assets
    rep.randomizer.scatter_2d(
        surface_prims=["/World/Ground"],
        asset_prim="/World/Objects"
    )
    
    # Randomize materials
    with rep.get.prims(semantics=[("class", "cup")]):
        rep.randomizer.color(
            colors=rep.distribution.uniform((0, 0, 0), (1, 1, 1))
        )

# Capture annotated data
rp = rep.create.render_product(camera, (1024, 1024))

# Get semantic segmentation
seg_annotator = rep.AnnotatorRegistry.get_annotator("semantic_segmentation")
seg_annotator.attach(rp)

# Get bounding boxes
bbox_annotator = rep.AnnotatorRegistry.get_annotator("bounding_box_2d_tight")
bbox_annotator.attach(rp)

# Run
rep.orchestrator.run(num_frames=1000)
```

### Filtering by Semantics

```python
# Get all "vehicle" prims
vehicles = rep.get.prims(semantics=[("class", "vehicle")])

# Get specific hierarchy level
fire_engines = rep.get.prims(
    semantics=[("hierarchy", "vehicle/emergency_vehicle/fire_engine")]
)

# Get by QCode
cups = rep.get.prims(semantics=[("qcode", "Q81727")])
```

## Best Practices

### Taxonomy

1. **Plan before creating**: Define taxonomy first
2. **Use WikiData**: Leverage existing QCodes
3. **Hierarchical structure**: Enable multi-level queries
4. **Consistent naming**: Same terms across dataset
5. **Document decisions**: Record taxonomy choices

### Implementation

1. **Apply to root**: Usually top-level prim
2. **Multiple labels**: Use when appropriate
3. **Test queries**: Verify labels work in simulator
4. **Version control**: Track taxonomy changes

### Integration

1. **Early labeling**: Add semantics during creation
2. **Validate**: Check labels load correctly
3. **Test SDG**: Verify annotation generation
4. **Iterate**: Refine taxonomy based on use

## Common Patterns

### Product Library

```python
# Label products with SKU and category
products = {
    "PROD-001": {"class": "box", "category": "packaging"},
    "PROD-002": {"class": "pallet", "category": "logistics"},
    "PROD-003": {"class": "crate", "category": "storage"}
}

for sku, metadata in products.items():
    label = stage.DefinePrim(f"/Semantics/Labels/{sku}", "SemanticsLabel")
    label.CreateAttribute("semantic:class", Sdf.ValueTypeNames.String).Set(metadata["class"])
    label.CreateAttribute("semantic:sku", Sdf.ValueTypeNames.String).Set(sku)
    label.CreateAttribute("semantic:category", Sdf.ValueTypeNames.String).Set(metadata["category"])
```

### Warehouse Simulation

```python
# Label warehouse assets with location
warehouse_labels = {
    "shelf_A1": {"class": "shelf", "aisle": "A", "row": 1},
    "pallet_B3": {"class": "pallet", "aisle": "B", "row": 3}
}
```

### Robotics Dataset

```python
# Label manipulation objects
manipulation_labels = {
    "graspable": True,
    "weight_kg": 2.5,
    "fragile": False,
    "stackable": True
}
```
