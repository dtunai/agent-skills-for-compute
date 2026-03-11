# RDKit 3D Conformer Generation Reference

Sources:
- [Getting Started in Python](https://www.rdkit.org/docs/GettingStartedInPython.html)
- [The RDKit Book](https://www.rdkit.org/docs/RDKit_Book.html)

## 2D Coordinates

```python
from rdkit.Chem import AllChem

# Generate 2D coordinates
AllChem.Compute2DCoords(mol)

# Coordgen (higher quality 2D)
from rdkit.Chem import rdDepictor
rdDepictor.Compute2DCoords(mol)

# With specific template
template = Chem.MolFromSmiles('c1ccccc1')
AllChem.Compute2DCoords(template)
rdDepictor.Compute2DCoords(mol, canonOrient=True, coordMap={})
```

## 3D Conformer Generation

### Basic 3D

```python
# Add hydrogens (required)
mol = Chem.AddHs(mol)

# Embed molecule (ETKDG algorithm)
AllChem.EmbedMolecule(mol)

# Check success
if mol.GetNumConformers() == 0:
    print("Embedding failed")

# With random seed
AllChem.EmbedMolecule(mol, randomSeed=42)

# Multiple attempts
AllChem.EmbedMolecule(mol, maxAttempts=10)
```

### Multiple Conformers

```python
# Generate multiple conformers
mol = Chem.AddHs(mol)
conf_ids = AllChem.EmbedMultipleConfs(mol, numConfs=10)

# With parameters
conf_ids = AllChem.EmbedMultipleConfs(
    mol,
    numConfs=50,
    maxAttempts=100,
    randomSeed=42,
    pruneRmsThresh=0.5,  # Prune similar conformers
    numThreads=0  # Use all cores
)

# ETKDG version 3 (default since 2020.03)
params = AllChem.ETKDGv3()
params.randomSeed = 42
conf_ids = AllChem.EmbedMultipleConfs(mol, numConfs=10, params=params)
```

### Force Field Optimization

```python
# UFF optimization
for conf_id in conf_ids:
    AllChem.UFFOptimizeMolecule(mol, confId=conf_id)

# MMFF94 optimization
for conf_id in conf_ids:
    AllChem.MMFFOptimizeMolecule(mol, confId=conf_id)

# Get energy
props = AllChem.MMFFGetMoleculeProperties(mol)
ff = AllChem.MMFFGetMoleculeForceField(mol, props, confId=0)
energy = ff.CalcEnergy()
```

## Conformer Alignment

```python
# Align all conformers to first
rms_list = []
AllChem.AlignMolConformers(mol, RMSlist=rms_list)

# Align with reference
ref_conf_id = 0
for conf_id in range(1, mol.GetNumConformers()):
    rms = AllChem.AlignMol(mol, mol, prbCid=conf_id, refCid=ref_conf_id)
    print(f"RMSD conf {conf_id}: {rms}")

# Get RMSD between conformers
rms = AllChem.GetConformerRMS(mol, 0, 1)

# Best RMS
best_rms = AllChem.GetBestRMS(mol, mol)
```

## Conformer Properties

```python
# Get conformer
conf = mol.GetConformer(0)

# Atom positions
for atom_idx in range(mol.GetNumAtoms()):
    pos = conf.GetAtomPosition(atom_idx)
    print(f"Atom {atom_idx}: ({pos.x}, {pos.y}, {pos.z})")

# Set position
from rdkit.Geometry import Point3D
conf.SetAtomPosition(0, Point3D(1.0, 2.0, 3.0))

# Distance between atoms
dist = rdMolTransforms.GetBondLength(conf, 0, 1)

# Angle
angle = rdMolTransforms.GetAngleDeg(conf, 0, 1, 2)

# Dihedral
dihedral = rdMolTransforms.GetDihedralDeg(conf, 0, 1, 2, 3)

# Set dihedral
rdMolTransforms.SetDihedralDeg(conf, 0, 1, 2, 3, 180.0)
```

## Conformer Clustering

```python
from rdkit.ML.Cluster import Butina

# Calculate distance matrix
dists = []
for i in range(1, mol.GetNumConformers()):
    for j in range(i):
        dists.append(AllChem.GetConformerRMS(mol, i, j))

# Butina clustering
clusters = Butina.ClusterData(dists, mol.GetNumConformers(), 2.0, isDistData=True)

# Get representative conformers
representatives = [cluster[0] for cluster in clusters]
```

## Constrained Embedding

```python
# Constrain atom positions
coords_map = {0: Point3D(0, 0, 0), 1: Point3D(1.5, 0, 0)}
AllChem.EmbedMolecule(mol, coordMap=coords_map)

# Constrain to template
template = Chem.MolFromSmiles('c1ccccc1')
AllChem.Compute2DCoords(template)
match = mol.GetSubstructMatch(template)
coord_map = {}
for i, atom_idx in enumerate(match):
    coord_map[atom_idx] = template.GetConformer().GetAtomPosition(i)
AllChem.EmbedMolecule(mol, coordMap=coord_map)
```

## Energy Calculations

```python
# MMFF energy
props = AllChem.MMFFGetMoleculeProperties(mol)
energies = []
for conf_id in range(mol.GetNumConformers()):
    ff = AllChem.MMFFGetMoleculeForceField(mol, props, confId=conf_id)
    energies.append(ff.CalcEnergy())

# Find lowest energy conformer
min_energy_id = energies.index(min(energies))

# UFF energy
for conf_id in range(mol.GetNumConformers()):
    ff = AllChem.UFFGetMoleculeForceField(mol, confId=conf_id)
    energy = ff.CalcEnergy()
```

## Best Practices

1. **Add hydrogens**: Always add explicit H for 3D
2. **ETKDG**: Use ETKDGv3 for best results
3. **Optimize**: Always optimize with force field
4. **Multiple conformers**: Generate 10-50 depending on flexibility
5. **Prune**: Use pruneRmsThresh to remove similar conformers
6. **Align**: Align before RMSD calculations
7. **Check energies**: Use lowest energy conformer for properties

## Common Pitfalls

- **Missing hydrogens**: 3D embedding fails without H
- **No conformers**: Check embedding success
- **Not optimizing**: Raw embedded coords may be poor
- **Wrong RMSD**: Align before calculating
- **Over-generating**: Too many conformers wastes time
- **Ignoring energies**: Lowest energy often most relevant
