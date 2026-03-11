# Materials Project Structures and Thermodynamics Reference

Sources:
- [MPRester API](https://materialsproject.github.io/api/_autosummary/mp_api.client.mprester.MPRester.html)
- [API Examples](https://docs.materialsproject.org/downloading-data/using-the-api/examples)

## Structure Retrieval

### By Material ID

```python
# Get final structure
structure = mpr.get_structure_by_material_id("mp-149")

# Get initial structure (before relaxation)
structure = mpr.get_structure_by_material_id("mp-149", final=False)

# Get conventional cell
structure = mpr.get_structure_by_material_id("mp-149", conventional_unit_cell=True)
```

### By Formula or Chemical System

```python
# Get all structures for formula
structures = mpr.get_structures("Fe2O3")

# Get structures in chemical system
structures = mpr.get_structures("Li-Fe-O")

# Limit results
structures = mpr.get_structures("Fe2O3", limit=5)
```

### Structure Finding

```python
from pymatgen.core import Structure

# Find similar structures
your_structure = Structure(...)
similar = mpr.find_structure(your_structure)

# Get material ID for structure
for result in similar:
    print(result["material_id"])
    print(result["normalized_rms_displacement"])
```

## Structure Properties

### Basic Information

```python
structure = mpr.get_structure_by_material_id("mp-149")

# Composition
print(structure.formula)
print(structure.composition)

# Lattice
print(structure.lattice)
print(structure.lattice.abc)  # a, b, c
print(structure.lattice.angles)  # alpha, beta, gamma
print(structure.volume)

# Sites
print(structure.num_sites)
for site in structure:
    print(site.species, site.coords)
```

### Exporting Structures

```python
# To CIF
structure.to(filename="structure.cif")

# To POSCAR
structure.to(filename="POSCAR", fmt="poscar")

# To dict
structure_dict = structure.as_dict()

# To JSON
import json
with open("structure.json", "w") as f:
    json.dump(structure.as_dict(), f)
```

## Thermodynamic Entries

### Get Entries

```python
# Single entry
entry = mpr.get_entry_by_material_id("mp-149")
print(entry.energy)
print(entry.energy_per_atom)
print(entry.composition)

# Entries for chemical system
entries = mpr.get_entries_in_chemsys("Li-Fe-O")

# Entries by formula
entries = mpr.get_entries("Fe2O3")

# With additional criteria
entries = mpr.get_entries(
    chemsys="Li-Fe-O",
    additional_criteria={"is_stable": True}
)
```

### Entry Types

```python
# ComputedEntry (default)
from pymatgen.entries.computed_entries import ComputedEntry

entry = mpr.get_entry_by_material_id("mp-149")
print(type(entry))  # ComputedStructureEntry

# Access properties
print(entry.energy)
print(entry.composition)
print(entry.energy_per_atom)
print(entry.structure)
```

## Phase Diagrams

### Build Phase Diagram

```python
from pymatgen.analysis.phase_diagram import PhaseDiagram, PDPlotter

# Get entries
entries = mpr.get_entries_in_chemsys("Li-Fe-O")

# Build phase diagram
pd = PhaseDiagram(entries)

# Get stable entries
stable_entries = pd.stable_entries

# Check stability
entry = mpr.get_entry_by_material_id("mp-149")
decomp, e_above_hull = pd.get_decomp_and_e_above_hull(entry)
print(f"Energy above hull: {e_above_hull} eV/atom")

# Plot
plotter = PDPlotter(pd)
plotter.show()
```

### Ternary Phase Diagrams

```python
# Ternary system
entries = mpr.get_entries_in_chemsys("Li-Fe-O")
pd = PhaseDiagram(entries)

plotter = PDPlotter(pd, show_unstable=True)
plotter.get_plot().savefig("phase_diagram.png")
```

### Grand Canonical Phase Diagrams

```python
from pymatgen.analysis.phase_diagram import GrandPotentialPhaseDiagram

# Fix chemical potential
entries = mpr.get_entries_in_chemsys("Li-Fe-O")
gcpd = GrandPotentialPhaseDiagram(
    entries,
    chempots={"O": -5.0}  # Fix O chemical potential
)
```

## Formation Energy

### Formation Energy per Atom

```python
# From entry
entry = mpr.get_entry_by_material_id("mp-149")
formation_energy = entry.energy_per_atom

# From search
docs = mpr.materials.summary.search(
    material_ids=["mp-149"],
    fields=["material_id", "formation_energy_per_atom"]
)
print(docs[0].formation_energy_per_atom)
```

### Energy Above Hull

```python
# From search
docs = mpr.materials.summary.search(
    material_ids=["mp-149"],
    fields=["material_id", "energy_above_hull", "is_stable"]
)

for doc in docs:
    print(f"{doc.material_id}: {doc.energy_above_hull} eV/atom")
    print(f"Stable: {doc.is_stable}")

# From phase diagram
entries = mpr.get_entries_in_chemsys("Li-Fe-O")
pd = PhaseDiagram(entries)

entry = mpr.get_entry_by_material_id("mp-149")
e_above_hull = pd.get_e_above_hull(entry)
```

## Cohesive Energy

```python
# Get cohesive energy
cohesive = mpr.get_cohesive_energy("mp-149")
print(cohesive)  # Energy in eV

# Per atom or per formula unit
cohesive_per_atom = mpr.get_cohesive_energy("mp-149", per_atom=True)
```

## Reference States

### Atom Reference Data

```python
# Get reference energies for isolated atoms
ref_data = mpr.get_atom_reference_data()

# For specific functional
ref_data_pbe = mpr.get_atom_reference_data(functional="PBE")
ref_data_scan = mpr.get_atom_reference_data(functional="SCAN")
ref_data_r2scan = mpr.get_atom_reference_data(functional="r2SCAN")
```

## Pourbaix Diagrams

### Electrochemistry

```python
from pymatgen.analysis.pourbaix_diagram import PourbaixDiagram

# Get pourbaix entries
pb_entries = mpr.get_pourbaix_entries(["Fe", "O", "H"])

# Get ion reference data
ion_data = mpr.get_ion_reference_data()

# Get ion entries
ion_entries = mpr.get_ion_entries(["Fe"])

# Build Pourbaix diagram
pb_diagram = PourbaixDiagram(pb_entries)
```

## Advanced Thermodynamics

### Equation of State

```python
# Get EOS data
eos_data = mpr.eos.search(material_ids=["mp-149"])

for doc in eos_data:
    print(doc.bulk_modulus)
    print(doc.shear_modulus)
```

### Thermal Properties

```python
# Search for materials with thermal data
docs = mpr.materials.summary.search(
    material_ids=["mp-149"],
    fields=["material_id", "debye_temperature"]
)
```

## Comparison and Analysis

### Structure Matching

```python
from pymatgen.analysis.structure_matcher import StructureMatcher

# Initialize matcher
matcher = StructureMatcher(
    ltol=0.2,  # Lattice tolerance
    stol=0.3,  # Site tolerance
    angle_tol=5  # Angle tolerance (degrees)
)

# Compare structures
structure1 = mpr.get_structure_by_material_id("mp-149")
structure2 = mpr.get_structure_by_material_id("mp-13")

is_match = matcher.fit(structure1, structure2)
rms = matcher.get_rms_dist(structure1, structure2)
```

### Stability Analysis

```python
# Analyze stability across chemical systems
entries = mpr.get_entries_in_chemsys("Li-Fe-O")

stable_count = sum(1 for e in entries if e.data.get("e_above_hull", 1) < 0.001)
total_count = len(entries)

print(f"Stable: {stable_count}/{total_count}")
```

## Batch Operations

### Multiple Structures

```python
# Get multiple structures efficiently
material_ids = ["mp-149", "mp-13", "mp-22526"]

docs = mpr.materials.summary.search(
    material_ids=material_ids,
    fields=["material_id", "structure"]
)

structures = {doc.material_id: doc.structure for doc in docs}
```

### Multiple Entries

```python
# Get entries for multiple systems
systems = ["Li-O", "Na-O", "K-O"]

all_entries = {}
for system in systems:
    all_entries[system] = mpr.get_entries_in_chemsys(system)
```

## Best Practices

1. **Cache structures**: Store locally for reuse
2. **Use material IDs**: Faster than formula lookups
3. **Batch queries**: Combine multiple requests
4. **Field selection**: Only retrieve needed properties
5. **Structure matching**: Use appropriate tolerances
6. **Phase diagrams**: Cache entries for reuse
7. **File export**: Save structures in portable formats

## Common Patterns

```python
# Get stable phases in system
entries = mpr.get_entries_in_chemsys("Li-Fe-O")
pd = PhaseDiagram(entries)
stable = [e.composition.reduced_formula for e in pd.stable_entries]

# Find low-energy metastable phases
docs = mpr.materials.summary.search(
    chemsys="Li-Fe-O",
    energy_above_hull=(0, 0.05),
    fields=["material_id", "formula_pretty", "energy_above_hull"]
)

# Export all structures in chemical system
docs = mpr.materials.summary.search(
    chemsys="Li-Fe-O",
    is_stable=True,
    fields=["material_id", "structure"]
)

for doc in docs:
    doc.structure.to(filename=f"{doc.material_id}.cif")
```
