# Materials Project Pymatgen Integration Reference

Sources:
- [pymatgen Documentation](https://pymatgen.org/)
- [Materials Project Examples](https://docs.materialsproject.org/downloading-data/using-the-api/examples)

## Overview

Pymatgen (Python Materials Genomics) is the core library powering the Materials Project. MPRester returns pymatgen objects directly for seamless integration.

## Structure Objects

### Creating Structures

```python
from pymatgen.core import Structure, Lattice

# From MPRester
structure = mpr.get_structure_by_material_id("mp-149")

# Create manually
lattice = Lattice.cubic(4.2)
structure = Structure(
    lattice,
    ["Cs", "Cl"],
    [[0, 0, 0], [0.5, 0.5, 0.5]]
)

# From dict
structure = Structure.from_dict(structure_dict)
```

### Structure Properties

```python
# Composition
print(structure.formula)
print(structure.composition)
print(structure.composition.reduced_formula)
print(structure.composition.fractional_composition)

# Lattice
print(structure.lattice.a, structure.lattice.b, structure.lattice.c)
print(structure.lattice.alpha, structure.lattice.beta, structure.lattice.gamma)
print(structure.volume)
print(structure.density)

# Sites
print(structure.num_sites)
for site in structure:
    print(f"{site.species_string}: {site.coords}")

# Elements
print(structure.elements)
print(structure.atomic_numbers)
```

### Structure Modifications

```python
# Add/remove sites
structure.append("Fe", [0.5, 0.5, 0.5])
structure.remove_sites([0])

# Substitution
structure.replace(0, "Au")

# Apply strain
structure.apply_strain(0.01)  # 1% strain

# Perturb
structure.perturb(distance=0.1)

# Sort
structure.sort()
```

## Symmetry Analysis

### Spacegroup Analyzer

```python
from pymatgen.symmetry.analyzer import SpacegroupAnalyzer

structure = mpr.get_structure_by_material_id("mp-149")

# Initialize analyzer
sga = SpacegroupAnalyzer(structure)

# Space group
print(sga.get_space_group_symbol())
print(sga.get_space_group_number())

# Crystal system
print(sga.get_crystal_system())

# Point group
print(sga.get_point_group_symbol())

# Symmetry operations
print(sga.get_symmetry_operations())

# Primitive cell
primitive = sga.get_primitive_standard_structure()

# Conventional cell
conventional = sga.get_conventional_standard_structure()

# Refined structure
refined = sga.get_refined_structure()
```

### Site Symmetry

```python
# Get symmetrized structure
sym_struct = sga.get_symmetrized_structure()

# Equivalent sites
for eq_sites in sym_struct.equivalent_sites:
    print(f"Equivalent sites: {len(eq_sites)}")

# Wyckoff positions
print(sym_struct.wyckoff_symbols)
```

## Structure Comparison

### Structure Matcher

```python
from pymatgen.analysis.structure_matcher import StructureMatcher

# Initialize matcher
matcher = StructureMatcher(
    ltol=0.2,           # Lattice length tolerance
    stol=0.3,           # Site tolerance
    angle_tol=5.0,      # Angle tolerance (degrees)
    primitive_cell=True, # Compare primitive cells
    scale=True,         # Allow scaling
    attempt_supercell=False
)

# Compare structures
structure1 = mpr.get_structure_by_material_id("mp-149")
structure2 = mpr.get_structure_by_material_id("mp-13")

# Check if match
is_match = matcher.fit(structure1, structure2)

# Get RMS distance
if is_match:
    rms = matcher.get_rms_dist(structure1, structure2)
    print(f"RMS distance: {rms}")

# Get transformation
transformation = matcher.get_transformation(structure1, structure2)
```

### Element Comparator

```python
from pymatgen.analysis.structure_matcher import ElementComparator

# Ignore oxidation states
comparator = ElementComparator()
matcher = StructureMatcher(comparator=comparator)
```

## Composition Analysis

### Composition Objects

```python
from pymatgen.core import Composition

# From formula
comp = Composition("Fe2O3")

# Properties
print(comp.reduced_formula)
print(comp.alphabetical_formula)
print(comp.weight)
print(comp.num_atoms)
print(comp.average_electroneg)

# Element amounts
print(comp["Fe"])  # 2.0
print(comp.get_atomic_fraction("Fe"))
```

### Composition Manipulation

```python
# Add compositions
comp1 = Composition("Fe2O3")
comp2 = Composition("FeO")
comp3 = comp1 + comp2

# Multiply
comp4 = comp1 * 2

# Divide
comp5 = comp1 / 2
```

## Transformations

### Standard Transformations

```python
from pymatgen.transformations.standard_transformations import (
    SupercellTransformation,
    SubstitutionTransformation,
    OrderDisorderedStructureTransformation
)

# Create supercell
supercell = SupercellTransformation([[2, 0, 0], [0, 2, 0], [0, 0, 2]])
new_structure = supercell.apply_transformation(structure)

# Substitution
sub = SubstitutionTransformation({"Fe": "Co"})
new_structure = sub.apply_transformation(structure)

# Order disordered structure
order = OrderDisorderedStructureTransformation()
ordered = order.apply_transformation(disordered_structure)
```

### Advanced Transformations

```python
from pymatgen.transformations.advanced_transformations import (
    EnumerateStructureTransformation,
    SubstitutionPredictorTransformation
)

# Enumerate orderings
enum = EnumerateStructureTransformation()
structures = enum.apply_transformation(structure, return_ranked_list=True)
```

## Analysis Tools

### Coordination Number

```python
from pymatgen.analysis.local_env import (
    VoronoiNN,
    CrystalNN,
    MinimumDistanceNN
)

# Voronoi nearest neighbors
vnn = VoronoiNN()
cn = vnn.get_cn(structure, 0)  # Site index 0
neighbors = vnn.get_nn_info(structure, 0)

# CrystalNN
cnn = CrystalNN()
cn = cnn.get_cn(structure, 0)

# Minimum distance
mnn = MinimumDistanceNN()
cn = mnn.get_cn(structure, 0)
```

### Bond Analysis

```python
from pymatgen.analysis.bond_valence import BVAnalyzer

# Bond valence analysis
bva = BVAnalyzer()

# Get valences
valences = bva.get_valences(structure)

# Get structure with oxidation states
structure_ox = bva.get_oxi_state_decorated_structure(structure)
```

### Dimensionality

```python
from pymatgen.analysis.dimensionality import get_dimensionality_larsen

# Get dimensionality
dim = get_dimensionality_larsen(structure)
print(f"Dimensionality: {dim}")  # 0D, 1D, 2D, 3D
```

## Surface Analysis

### Surface Generator

```python
from pymatgen.core.surface import SlabGenerator

# Generate slabs
slabgen = SlabGenerator(
    structure,
    miller_index=(1, 1, 1),
    min_slab_size=10.0,
    min_vacuum_size=10.0
)

# Get all slabs
slabs = slabgen.get_slabs()

for slab in slabs:
    print(slab.miller_index)
    print(slab.surface_area)
```

### Wulff Shape

```python
# Get Wulff shape from MPRester
wulff = mpr.get_wulff_shape("mp-149")

if wulff:
    print(wulff.miller_energy_map)
    print(wulff.surface_area)
    print(wulff.volume)
```

## Electronic Structure Analysis

### Band Structure Analysis

```python
bs = mpr.get_bandstructure_by_material_id("mp-149")

# Properties
print(bs.get_band_gap())
print(bs.is_metal())

# Get specific band
band = bs.bands[0]  # Spin up, band index 0

# K-points
kpoints = bs.kpoints

# Projections (if available)
if bs.projections:
    print(bs.projections)
```

### DOS Analysis

```python
dos = mpr.get_dos_by_material_id("mp-149")

# Complete DOS
complete_dos = dos

# Element DOS
if hasattr(dos, 'get_element_dos'):
    fe_dos = dos.get_element_dos("Fe")
    
# Orbital DOS
if hasattr(dos, 'get_spd_dos'):
    spd = dos.get_spd_dos()
```

## File I/O

### Reading Files

```python
from pymatgen.core import Structure

# CIF
structure = Structure.from_file("structure.cif")

# POSCAR
structure = Structure.from_file("POSCAR")

# JSON
import json
with open("structure.json") as f:
    structure = Structure.from_dict(json.load(f))
```

### Writing Files

```python
# CIF
structure.to(filename="output.cif")

# POSCAR
structure.to(filename="POSCAR", fmt="poscar")

# XYZ
structure.to(filename="output.xyz")

# JSON
import json
with open("output.json", "w") as f:
    json.dump(structure.as_dict(), f)
```

## Molecule Support

### Molecule Objects

```python
from pymatgen.core import Molecule

# Create molecule
mol = Molecule(["C", "H", "H", "H", "H"], 
               [[0, 0, 0], [1, 0, 0], [-1, 0, 0], 
                [0, 1, 0], [0, -1, 0]])

# From file
mol = Molecule.from_file("molecule.xyz")

# Properties
print(mol.formula)
print(mol.charge)
print(mol.spin_multiplicity)
```

## Best Practices

1. **Use symmetry analysis**: Reduce structure to primitive/conventional
2. **Structure matching**: Set appropriate tolerances
3. **Coordinate environments**: Use CrystalNN for robust analysis
4. **File formats**: CIF for crystallographic, POSCAR for VASP
5. **Transformations**: Chain multiple transformations
6. **Validation**: Check structure validity after modifications
7. **Performance**: Use primitive cells for faster operations

## Common Patterns

```python
# Get and analyze structure
structure = mpr.get_structure_by_material_id("mp-149")

# Symmetry
sga = SpacegroupAnalyzer(structure)
print(f"Space group: {sga.get_space_group_symbol()}")
print(f"Crystal system: {sga.get_crystal_system()}")

# Primitive cell
primitive = sga.get_primitive_standard_structure()

# Export
primitive.to(filename="primitive.cif")

# Compare with another structure
structure2 = mpr.get_structure_by_material_id("mp-13")
matcher = StructureMatcher()
is_same = matcher.fit(primitive, structure2)

# Find neighbors
vnn = VoronoiNN()
for i, site in enumerate(structure):
    neighbors = vnn.get_nn_info(structure, i)
    print(f"Site {i} has {len(neighbors)} neighbors")
```
