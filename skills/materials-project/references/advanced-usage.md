# Materials Project Advanced Usage Reference

Sources:
- [MPRester API](https://materialsproject.github.io/api/_autosummary/mp_api.client.mprester.MPRester.html)
- [API Examples](https://docs.materialsproject.org/downloading-data/using-the-api/examples)

## All Available Routes

### Materials Routes

```python
with MPRester() as mpr:
    # Summary (main endpoint)
    mpr.materials.summary
    
    # Tasks (calculation details)
    mpr.materials.tasks
    
    # Robocrystallographer (descriptions)
    mpr.materials.robocrystallographer
    
    # Bonds
    mpr.materials.bonds
    
    # Substrates
    mpr.materials.substrates
    
    # Grain boundaries
    mpr.materials.grain_boundaries
    
    # Fermi surface
    mpr.materials.fermi_surface
```

### Property Routes

```python
# Thermodynamics
mpr.thermo

# Electronic structure
mpr.electronic_structure

# Phonon
mpr.phonon

# Dielectric
mpr.dielectric

# Piezoelectric
mpr.piezoelectric

# Elasticity
mpr.elasticity

# Magnetism
mpr.magnetism

# X-ray absorption
mpr.xas

# Equation of state
mpr.eos

# Oxidation states
mpr.oxidation_states

# Provenance
mpr.provenance

# Similarity
mpr.similarity

# Surface properties
mpr.surface_properties

# Synthesis
mpr.synthesis

# Battery electrodes
mpr.insertion_electrodes

# Molecules
mpr.molecules
```

## Batch Operations

### Parallel Queries

```python
from concurrent.futures import ThreadPoolExecutor

def get_structure(material_id):
    with MPRester() as mpr:
        return mpr.get_structure_by_material_id(material_id)

material_ids = ["mp-149", "mp-13", "mp-22526", "mp-1234"]

with ThreadPoolExecutor(max_workers=4) as executor:
    structures = list(executor.map(get_structure, material_ids))
```

### Bulk Retrieval

```python
# Get many materials at once
material_ids = [f"mp-{i}" for i in range(1, 101)]

docs = mpr.materials.summary.search(
    material_ids=material_ids,
    fields=["material_id", "formula_pretty", "band_gap"]
)

# Process in batches
batch_size = 10
for i in range(0, len(material_ids), batch_size):
    batch = material_ids[i:i+batch_size]
    docs = mpr.materials.summary.search(material_ids=batch)
```

## Custom Queries

### Complex Filters

```python
# Multiple property constraints
docs = mpr.materials.summary.search(
    elements=["Li", "Co", "O"],
    num_elements=3,
    is_stable=True,
    band_gap=(0, 0.5),
    num_sites=(1, 20),
    crystal_system="cubic",
    spacegroup_number=225,
    fields=["material_id", "formula_pretty", "band_gap", "energy_above_hull"]
)
```

### Using HasProps

```python
from mp_api.client.core import HasProps

# Materials with all specified properties
docs = mpr.materials.summary.search(
    has_props=[
        HasProps.band_structure,
        HasProps.dos,
        HasProps.dielectric,
        HasProps.piezoelectric,
        HasProps.elasticity
    ]
)

# Available HasProps
# HasProps.band_structure
# HasProps.dos
# HasProps.phonon
# HasProps.dielectric
# HasProps.piezoelectric
# HasProps.elasticity
# HasProps.magnetism
# HasProps.eos
# HasProps.xas
# HasProps.grain_boundaries
# HasProps.surface_properties
```

## Robocrystallographer

### Structure Descriptions

```python
# Get natural language descriptions
docs = mpr.materials.robocrystallographer.search(
    material_ids=["mp-149"]
)

for doc in docs:
    print(doc.description)
    # "Silicon adopts the cubic diamond structure..."
```

## Synthesis Information

### Synthesis Recipes

```python
# Get synthesis data
docs = mpr.synthesis.search(keywords=["solid state"])

for doc in docs:
    print(doc.target_formula)
    print(doc.synthesis_type)
    print(doc.operations)
```

## Battery Electrode Materials

### Insertion Electrodes

```python
# Battery materials
docs = mpr.insertion_electrodes.search(
    elements=["Li"],
    battery_ids=["mp-149"]
)

for doc in docs:
    print(doc.battery_id)
    print(doc.max_voltage_step)
    print(doc.capacity_grav)
```

## Molecules Database

### Molecular Queries

```python
# Search molecules
docs = mpr.molecules.search(
    formula="C6H6",
    fields=["molecule_id", "formula", "charge", "spin_multiplicity"]
)

# By SMILES
docs = mpr.molecules.search(
    smiles="c1ccccc1"
)
```

## Provenance and Citations

### Data Provenance

```python
# Get provenance
docs = mpr.provenance.search(material_ids=["mp-149"])

for doc in docs:
    print(doc.created_at)
    print(doc.database_IDs)
```

### Citations

```python
# Get references
refs = mpr.get_material_id_references(["mp-149"])

for ref in refs:
    print(ref.doi)
    print(ref.bibtex)
```

## Download Raw VASP Files

### NoMaD Repository

```python
# Get download URLs
download_info = mpr.get_download_info(
    material_ids=["mp-149"],
    task_ids=None
)

for info in download_info:
    print(info["url"])
    # Download from NoMaD repository
```

## Grain Boundaries

### Grain Boundary Structures

```python
# Get grain boundary data
docs = mpr.grain_boundaries.search(
    material_ids=["mp-149"],
    sigma=3
)

for doc in docs:
    print(doc.sigma)
    print(doc.rotation_axis)
    print(doc.gb_energy)
```

## Surface Properties

### Surface Energies

```python
# Get surface data
docs = mpr.surface_properties.search(
    material_ids=["mp-149"]
)

for doc in docs:
    print(doc.surface_energy)
    print(doc.miller_index)
```

## Structure Similarity

### Similar Structures

```python
# Find similar structures
docs = mpr.similarity.search(
    material_ids=["mp-149"]
)

for doc in docs:
    print(doc.similar_material_id)
    print(doc.similarity_score)
```

## Oxidation State Assignment

### Oxidation States

```python
# Get oxidation states
docs = mpr.oxidation_states.search(
    material_ids=["mp-149"]
)

for doc in docs:
    print(doc.possible_species)
    print(doc.method)
```

## Performance Optimization

### Caching Results

```python
import pickle
import os

def get_cached_or_fetch(material_id, cache_dir="cache"):
    os.makedirs(cache_dir, exist_ok=True)
    cache_file = os.path.join(cache_dir, f"{material_id}.pkl")
    
    if os.path.exists(cache_file):
        with open(cache_file, "rb") as f:
            return pickle.load(f)
    
    with MPRester() as mpr:
        structure = mpr.get_structure_by_material_id(material_id)
    
    with open(cache_file, "wb") as f:
        pickle.dump(structure, f)
    
    return structure

# Usage
structure = get_cached_or_fetch("mp-149")
```

### Disabling Document Model

```python
# Faster but returns dicts instead of objects
with MPRester(use_document_model=False) as mpr:
    docs = mpr.materials.summary.search(
        elements=["Li"],
        fields=["material_id", "formula_pretty"]
    )
    
    # docs are dicts, not objects
    for doc in docs:
        print(doc["material_id"])
```

### Field Selection Strategy

```python
# Minimal fields (fastest)
minimal_fields = ["material_id", "formula_pretty"]

# Common fields
common_fields = [
    "material_id",
    "formula_pretty",
    "band_gap",
    "formation_energy_per_atom",
    "energy_above_hull",
    "is_stable"
]

# All fields (slowest)
docs = mpr.materials.summary.search(elements=["Li"])
```

## Database Version Tracking

### Version Management

```python
# Get current version
version = mpr.get_database_version()
print(f"Database version: {version}")

# Store version with results
results = {
    "database_version": version,
    "query_date": "2026-02-15",
    "data": []
}

# Check if version changed
if mpr.get_database_version() != version:
    print("Database updated - may need to re-query")
```

## Error Handling

### Robust API Calls

```python
from mp_api.client.core import MPRestError
import time

def robust_query(mpr, material_id, max_retries=3):
    for attempt in range(max_retries):
        try:
            return mpr.get_structure_by_material_id(material_id)
        except MPRestError as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            raise
        except Exception as e:
            print(f"Unexpected error: {e}")
            return None
    return None

# Usage
with MPRester() as mpr:
    structure = robust_query(mpr, "mp-149")
```

## Best Practices

1. **Cache aggressively**: Store results locally
2. **Batch queries**: Combine multiple material IDs
3. **Limit fields**: Only retrieve what you need
4. **Use material IDs**: Fastest query method
5. **Handle errors**: Implement retry logic
6. **Track versions**: Store database version with results
7. **Parallel requests**: Use threading for independent queries
8. **Progress bars**: Enable for long-running queries
9. **Document model**: Disable for production speed
10. **Respect rate limits**: Don't hammer the API

## Complete Workflow Example

```python
from mp_api.client import MPRester
from pymatgen.analysis.phase_diagram import PhaseDiagram
from pymatgen.analysis.structure_matcher import StructureMatcher
import pickle
import os

# 1. Query materials
with MPRester() as mpr:
    # Get all stable Li-Fe-O compounds
    docs = mpr.materials.summary.search(
        chemsys="Li-Fe-O",
        is_stable=True,
        fields=["material_id", "formula_pretty", "structure", "energy_above_hull"]
    )
    
    print(f"Found {len(docs)} stable materials")
    
    # 2. Get entries for phase diagram
    entries = mpr.get_entries_in_chemsys("Li-Fe-O")
    
    # 3. Build phase diagram
    pd = PhaseDiagram(entries)
    
    # 4. Get electronic properties for stable phases
    stable_ids = [doc.material_id for doc in docs]
    
    band_gaps = {}
    for mid in stable_ids:
        try:
            bs = mpr.get_bandstructure_by_material_id(mid)
            if bs and not bs.is_metal():
                band_gaps[mid] = bs.get_band_gap()
        except:
            pass
    
    # 5. Export results
    for doc in docs:
        # Save structure
        doc.structure.to(filename=f"{doc.material_id}.cif")
        
        # Save band gap if available
        if doc.material_id in band_gaps:
            print(f"{doc.formula_pretty}: {band_gaps[doc.material_id]['energy']} eV")

print("Analysis complete!")
```
