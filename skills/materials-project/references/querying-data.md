# Materials Project Data Querying Reference

Sources:
- [Querying Data](https://docs.materialsproject.org/downloading-data/using-the-api/querying-data)
- [API Examples](https://docs.materialsproject.org/downloading-data/using-the-api/examples)

## Search Methods

### Primary Search Method

```python
# All searches use the search() method
with MPRester() as mpr:
    docs = mpr.materials.summary.search(
        material_ids=["mp-149"],
        elements=["Si", "O"],
        band_gap=(0.5, 1.0)
    )
```

### Search by Material ID

```python
# Single ID
docs = mpr.materials.summary.search(material_ids=["mp-149"])

# Multiple IDs
docs = mpr.materials.summary.search(
    material_ids=["mp-149", "mp-13", "mp-22526"]
)

# Get all IDs for formula
ids = mpr.get_material_ids("TiO2")
docs = mpr.materials.summary.search(material_ids=ids)
```

### Search by Composition

```python
# Chemical system (exact elements)
docs = mpr.materials.summary.search(chemsys="Li-Fe-O")

# Elements (at least these)
docs = mpr.materials.summary.search(elements=["Si", "O"])

# Exclude elements
docs = mpr.materials.summary.search(
    elements=["Li", "O"],
    exclude_elements=["Na", "K"]
)

# Number of elements
docs = mpr.materials.summary.search(
    elements=["O"],
    num_elements=2  # Binary oxides
)

# Formula pattern
docs = mpr.materials.summary.search(formula="ABC3")  # Perovskites
```

## Property Filters

### Band Gap

```python
# Range
docs = mpr.materials.summary.search(band_gap=(0.5, 1.0))

# Minimum only
docs = mpr.materials.summary.search(band_gap=(3, None))

# Maximum only
docs = mpr.materials.summary.search(band_gap=(None, 0.1))

# Exact value
docs = mpr.materials.summary.search(band_gap=(0, 0))  # Metals
```

### Energy and Stability

```python
# Stable materials only
docs = mpr.materials.summary.search(is_stable=True)

# Energy above hull
docs = mpr.materials.summary.search(
    energy_above_hull=(0, 0.05)  # Nearly stable
)

# Formation energy
docs = mpr.materials.summary.search(
    formation_energy_per_atom=(None, -1.0)
)
```

### Structural Properties

```python
# Number of sites
docs = mpr.materials.summary.search(num_sites=(1, 10))

# Crystal system
docs = mpr.materials.summary.search(crystal_system="cubic")

# Space group
docs = mpr.materials.summary.search(spacegroup_number=225)

# Density
docs = mpr.materials.summary.search(density=(5.0, None))

# Volume
docs = mpr.materials.summary.search(volume=(None, 100))
```

### Available Properties

```python
from mp_api.client.core import HasProps

# Materials with specific properties
docs = mpr.materials.summary.search(
    has_props=[
        HasProps.dielectric,
        HasProps.piezoelectric,
        HasProps.elasticity
    ]
)

# Common HasProps values
HasProps.band_structure
HasProps.dos
HasProps.phonon
HasProps.dielectric
HasProps.piezoelectric
HasProps.elasticity
HasProps.magnetism
HasProps.eos
HasProps.xas
```

## Advanced Filtering

### Multiple Criteria

```python
# Combine filters
docs = mpr.materials.summary.search(
    elements=["Si", "O"],
    num_elements=2,
    band_gap=(3, None),
    is_stable=True,
    num_sites=(1, 20),
    crystal_system="cubic"
)
```

### Theoretical vs Experimental

```python
# Theoretical structures only (default)
docs = mpr.materials.summary.search(theoretical=True)

# Experimental structures only
docs = mpr.materials.summary.search(theoretical=False)
```

### Dimensionality

```python
# By dimensional tags
docs = mpr.materials.summary.search(
    elements=["C"],
    num_elements=1
)
# Filter by dimensionality in structure
```

## Field Selection

### Limiting Returned Fields

```python
# All fields (default, slow)
docs = mpr.materials.summary.search(elements=["Li"])

# Specific fields (fast)
docs = mpr.materials.summary.search(
    elements=["Li"],
    fields=["material_id", "formula_pretty", "band_gap"]
)

# Common fields
fields = [
    "material_id",
    "formula_pretty",
    "structure",
    "band_gap",
    "formation_energy_per_atom",
    "energy_above_hull",
    "is_stable",
    "symmetry"
]
```

### Accessing Results

```python
# MPDataDoc objects
docs = mpr.materials.summary.search(
    material_ids=["mp-149"],
    fields=["material_id", "band_gap"]
)

for doc in docs:
    print(doc.material_id)
    print(doc.band_gap)
    print(doc.formula_pretty)
```

## Pagination and Limits

### Chunk Size

```python
# Control pagination (default: 1000)
docs = mpr.materials.summary.search(
    elements=["Li"],
    chunk_size=100
)
```

### All Results

```python
# Get all matching results (auto-paginated)
docs = mpr.materials.summary.search(
    elements=["Li"],
    num_chunks=None  # Get all
)

# Limit chunks
docs = mpr.materials.summary.search(
    elements=["Li"],
    num_chunks=5  # First 5 chunks only
)
```

## Convenience Functions

### Get Structure

```python
# By material ID (most common)
structure = mpr.get_structure_by_material_id("mp-149")

# Multiple structures
structures = mpr.get_structures("Fe2O3")

# By chemical system
structures = mpr.get_structures("Li-Fe-O")
```

### Get Entries

```python
# Single entry
entry = mpr.get_entry_by_material_id("mp-149")

# Entries for phase diagram
entries = mpr.get_entries("Li-Fe-O")
entries = mpr.get_entries_in_chemsys("Li-Fe-O")

# With additional criteria
entries = mpr.get_entries(
    chemsys="Li-Fe-O",
    additional_criteria={"band_gap": (0, None)}
)
```

### Find Structure

```python
from pymatgen.core import Structure

# Find similar structures
structure = Structure(...)
similar = mpr.find_structure(structure)
```

## Query Optimization

### Performance Tips

```python
# 1. Limit fields
docs = mpr.materials.summary.search(
    elements=["Li"],
    fields=["material_id", "formula_pretty"]
)

# 2. Use material IDs when known
docs = mpr.materials.summary.search(
    material_ids=["mp-149", "mp-13"]
)

# 3. Filter early
docs = mpr.materials.summary.search(
    elements=["Li"],
    is_stable=True,
    num_sites=(1, 10)
)

# 4. Disable document model for speed
with MPRester(use_document_model=False) as mpr:
    docs = mpr.materials.summary.search(elements=["Li"])
```

## Common Query Patterns

### Battery Materials

```python
# Lithium-containing stable materials
docs = mpr.materials.summary.search(
    elements=["Li"],
    is_stable=True,
    fields=["material_id", "formula_pretty", "energy_above_hull"]
)
```

### Wide Band Gap Semiconductors

```python
# Band gap > 3 eV
docs = mpr.materials.summary.search(
    band_gap=(3, None),
    is_stable=True,
    fields=["material_id", "formula_pretty", "band_gap"]
)
```

### Binary Oxides

```python
# Two-element compounds with oxygen
docs = mpr.materials.summary.search(
    elements=["O"],
    num_elements=2,
    is_stable=True
)
```

### Perovskites

```python
# ABC3 formula pattern
docs = mpr.materials.summary.search(
    formula="ABC3",
    crystal_system="cubic",
    is_stable=True
)
```

### Magnetic Materials

```python
# Materials with magnetic properties
docs = mpr.materials.summary.search(
    has_props=[HasProps.magnetism],
    is_stable=True
)
```

## Database Mapping

### ICSD Integration

```python
# Get ICSD IDs
docs = mpr.materials.summary.search(material_ids=["mp-149"])
icsd_ids = docs[0].database_IDs.get("icsd", [])

# Map ICSD to MP
from pymatgen.analysis.structure_matcher import StructureMatcher

matcher = StructureMatcher()
# Use matcher to compare structures
```

### Task IDs

```python
# Get underlying calculation IDs
task_ids = mpr.get_task_ids_associated_with_material_id("mp-149")

# Map task ID to material ID
material_id = mpr.get_material_id_from_task_id("mp-12345")
```

## Best Practices

1. **Use field selection**: Dramatically improves performance
2. **Filter early**: Apply all filters in single query
3. **Material IDs**: Fastest query method
4. **Batch queries**: Combine multiple material IDs
5. **Cache results**: Store locally for reuse
6. **Progressive refinement**: Start broad, narrow down
7. **Check result count**: Verify query returns expected results

## Error Handling

```python
try:
    docs = mpr.materials.summary.search(
        elements=["Li"],
        band_gap=(0.5, 1.0)
    )
    
    if not docs:
        print("No results found")
    
    for doc in docs:
        print(doc.material_id)
        
except Exception as e:
    print(f"Query failed: {e}")
```
