---
name: materials-project
description: "Materials Project API — access 150k+ computed materials, crystal structures, thermodynamics, electronic/phonon properties via MPRester and pymatgen"
license: MIT
metadata:
  author: Agent Cluster
  tags: [materials-project, mp-api, mprester, pymatgen, dft, crystal-structures, materials-science]
---

# Materials Project Skill

Python API client for accessing Materials Project computational materials database with 150,000+ inorganic compounds and 35,000+ molecules.

**Official Sources:**
- [Materials Project Documentation](https://docs.materialsproject.org/)
- [API Getting Started](https://docs.materialsproject.org/downloading-data/using-the-api/getting-started)
- [API Examples](https://docs.materialsproject.org/downloading-data/using-the-api/examples)
- [MPRester API Reference](https://materialsproject.github.io/api/_autosummary/mp_api.client.mprester.MPRester.html)
- [pymatgen Documentation](https://pymatgen.org/)

## Installation

```bash
# Install mp-api client
pip install mp_api

# With pymatgen for structure analysis
pip install mp_api pymatgen

# From source
git clone https://github.com/materialsproject/api
cd api
pip install -e .
```

## Quick Start

```python
from mp_api.client import MPRester

# Initialize with API key (get from https://next-gen.materialsproject.org/api)
with MPRester("your_api_key_here") as mpr:
    # Get structure by material ID
    structure = mpr.get_structure_by_material_id("mp-149")

    # Search materials
    docs = mpr.materials.summary.search(
        elements=["Si", "O"],
        band_gap=(0.5, 1.0)
    )

    # Get entries for phase diagram
    entries = mpr.get_entries("Li-Fe-O")
```

## API Key Setup

```python
# Method 1: Pass directly
with MPRester("your_api_key") as mpr:
    pass

# Method 2: Environment variable (recommended)
import os
os.environ["MP_API_KEY"] = "your_api_key"
with MPRester() as mpr:
    pass

# Method 3: .bashrc/.zshrc
# export MP_API_KEY="your_api_key"
with MPRester() as mpr:
    pass
```

## Basic Queries

```python
with MPRester() as mpr:
    # By material ID
    structure = mpr.get_structure_by_material_id("mp-149")

    # By formula
    structures = mpr.get_structures("Fe2O3")

    # By chemical system
    docs = mpr.materials.summary.search(chemsys="Li-Fe-O")

    # By elements (at least these)
    docs = mpr.materials.summary.search(elements=["Si", "O"])

    # Get all material IDs for formula
    ids = mpr.get_material_ids("TiO2")
```

## Filtering and Search

```python
# Property filters
docs = mpr.materials.summary.search(
    elements=["Si", "O"],
    band_gap=(3, None),  # >= 3 eV
    num_sites=(1, 10),
    is_stable=True
)

# By properties available
from mp_api.client.core import HasProps
docs = mpr.materials.summary.search(
    has_props=[HasProps.dielectric, HasProps.piezoelectric]
)

# Formula pattern
docs = mpr.materials.summary.search(formula="ABC3")

# Limit returned fields (performance)
docs = mpr.materials.summary.search(
    elements=["Li"],
    fields=["material_id", "formula_pretty", "band_gap"]
)
```

## Structure Operations

```python
from pymatgen.core import Structure

# Get structure
structure = mpr.get_structure_by_material_id("mp-149")

# Structure info
print(structure.formula)
print(structure.lattice)
print(structure.composition)

# Find similar structures
similar = mpr.find_structure(structure)

# Export
structure.to(filename="structure.cif")
structure.to(filename="structure.vasp", fmt="poscar")
```

## Thermodynamics

```python
# Get entries for phase diagram
entries = mpr.get_entries_in_chemsys("Li-Fe-O")

# Single entry
entry = mpr.get_entry_by_material_id("mp-149")
print(entry.energy_per_atom)

# Cohesive energy
cohesive = mpr.get_cohesive_energy("mp-149")

# Build phase diagram
from pymatgen.analysis.phase_diagram import PhaseDiagram
pd = PhaseDiagram(entries)
```

## Electronic Structure

```python
# Band structure
bs = mpr.get_bandstructure_by_material_id("mp-149")
print(bs.get_band_gap())
bs.get_cbm()  # Conduction band minimum
bs.get_vbm()  # Valence band maximum

# Density of states
dos = mpr.get_dos_by_material_id("mp-149")
print(dos.get_gap())
dos.get_cbm()

# Plot
from pymatgen.electronic_structure.plotter import BSPlotter, DosPlotter
BSPlotter(bs).get_plot().savefig("bandstructure.png")
DosPlotter().add_dos("Total", dos).get_plot().savefig("dos.png")
```

## Phonon Properties

```python
# Phonon DOS
ph_dos = mpr.get_phonon_dos_by_material_id("mp-149")

# Phonon band structure
ph_bs = mpr.get_phonon_bandstructure_by_material_id("mp-149")
```

## Database Routes

```python
with MPRester() as mpr:
    # Materials summary (default)
    mpr.materials.summary

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

    # XAS (X-ray absorption)
    mpr.xas

    # Synthesis
    mpr.synthesis

    # Molecules
    mpr.molecules
```

## Advanced Filtering

```python
# Stable materials only
docs = mpr.materials.summary.search(
    is_stable=True,
    energy_above_hull=(0, 0)
)

# By crystal system
docs = mpr.materials.summary.search(
    crystal_system="cubic"
)

# By space group
docs = mpr.materials.summary.search(
    spacegroup_number=225  # Fm-3m
)

# By dimensionality
docs = mpr.materials.summary.search(
    num_elements=2,
    theoretical=False  # Experimental structures only
)
```

## Convenience Functions

```python
# Get database version
version = mpr.get_database_version()

# Map ICSD to MP IDs
docs = mpr.materials.summary.search(material_ids=["mp-149"])
icsd_ids = docs[0].database_IDs.get("icsd", [])

# Get task IDs
task_ids = mpr.get_task_ids_associated_with_material_id("mp-149")

# Download raw VASP files
download_info = mpr.get_download_info(material_ids=["mp-149"])
```

## Pymatgen Integration

```python
from pymatgen.core import Structure, Lattice
from pymatgen.analysis.structure_matcher import StructureMatcher
from pymatgen.symmetry.analyzer import SpacegroupAnalyzer

# Structure analysis
structure = mpr.get_structure_by_material_id("mp-149")

# Symmetry
sga = SpacegroupAnalyzer(structure)
print(sga.get_space_group_symbol())
print(sga.get_crystal_system())

# Compare structures
matcher = StructureMatcher()
is_match = matcher.fit(structure1, structure2)

# Phase diagrams
from pymatgen.analysis.phase_diagram import PhaseDiagram
entries = mpr.get_entries_in_chemsys("Li-Fe-O")
pd = PhaseDiagram(entries)
```

## Performance Tips

1. **Limit fields**: Use `fields=` to retrieve only needed data
2. **Use context manager**: Ensures proper session cleanup
3. **Batch queries**: Get multiple materials in single call
4. **Cache results**: Store frequently used data locally
5. **Use material IDs**: Faster than searching by formula
6. **Disable validation**: Set `use_document_model=False` for speed

## Common Patterns

```python
# Find stable binary oxides
docs = mpr.materials.summary.search(
    elements=["O"],
    num_elements=2,
    is_stable=True,
    fields=["material_id", "formula_pretty", "formation_energy_per_atom"]
)

# Wide band gap semiconductors
docs = mpr.materials.summary.search(
    band_gap=(3, None),
    is_stable=True,
    fields=["material_id", "formula_pretty", "band_gap"]
)

# Perovskites
docs = mpr.materials.summary.search(
    formula="ABC3",
    crystal_system="cubic"
)
```

## References

- **[API Basics](references/api-basics.md)** - Installation, authentication, MPRester configuration
- **[Querying Data](references/querying-data.md)** - Search methods, filtering, pagination, fields
- **[Structures and Thermodynamics](references/structures-thermo.md)** - Structure retrieval, phase diagrams, entries
- **[Electronic and Phonon Properties](references/electronic-phonon.md)** - Band structures, DOS, phonons
- **[Pymatgen Integration](references/pymatgen-integration.md)** - Structure analysis, symmetry, comparisons
- **[Advanced Usage](references/advanced-usage.md)** - Routes, batch operations, performance optimization
