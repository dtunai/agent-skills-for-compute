# RDKit Stereochemistry Reference

Sources:
- [The RDKit Book](https://www.rdkit.org/docs/RDKit_Book.html)
- [Stereo Groups and Enhanced Stereochemistry](https://greglandrum.github.io/rdkit-blog/posts/2023-11-19-explaining-stereo-groups.html)
- [EnumerateStereoisomers API](https://www.rdkit.org/docs/source/rdkit.Chem.EnumerateStereoisomers.html)

## Overview

RDKit provides comprehensive stereochemistry support including tetrahedral centers, double bond geometry, enhanced stereochemistry (stereo groups), and atropisomers.

## Tetrahedral Stereochemistry

### Chiral Centers

```python
from rdkit import Chem

# R/S configuration
mol = Chem.MolFromSmiles('C[C@H](O)CC')  # S-configuration
mol = Chem.MolFromSmiles('C[C@@H](O)CC') # R-configuration

# Check chirality
for atom in mol.GetAtoms():
    if atom.HasProp('_CIPCode'):
        cip = atom.GetProp('_CIPCode')
        print(f"Atom {atom.GetIdx()}: {cip}")

# Assign stereochemistry
from rdkit.Chem import AllChem
Chem.AssignStereochemistry(mol, cleanIt=True, force=True)

# Get chiral tag
atom = mol.GetAtomWithIdx(1)
tag = atom.GetChiralTag()
# CHI_UNSPECIFIED, CHI_TETRAHEDRAL_CW, CHI_TETRAHEDRAL_CCW
```

### CIP Assignment

```python
# Legacy approximate method
Chem.AssignStereochemistry(mol, cleanIt=True, force=True, flagPossibleStereoCenters=True)

# New accurate method
Chem.AssignCIPLabels(mol)

# Check for potential stereocenters
for atom in mol.GetAtoms():
    if atom.HasProp('_ChiralityPossible'):
        print(f"Atom {atom.GetIdx()} is potentially chiral")
```

## Double Bond Stereochemistry

### E/Z Configuration

```python
# E/Z bonds
mol = Chem.MolFromSmiles('C/C=C/C')  # E
mol = Chem.MolFromSmiles('C/C=C\\C') # Z

# Get bond stereo
bond = mol.GetBondWithIdx(1)
stereo = bond.GetStereo()
# STEREONONE, STEREOANY, STEREOZ, STEREOE, STEREOCIS, STEREOTRANS

# Assign E/Z
Chem.AssignStereochemistry(mol, cleanIt=True)

# Check assignment
for bond in mol.GetBonds():
    if bond.GetBondType() == Chem.BondType.DOUBLE:
        if bond.HasProp('_CIPCode'):
            print(f"Bond {bond.GetIdx()}: {bond.GetProp('_CIPCode')}")
```

## Enhanced Stereochemistry (Stereo Groups)

### Stereo Group Types

```python
from rdkit.Chem import rdchem

# ABS: Absolute stereochemistry (single substance)
# AND: Mixture of enantiomers (racemic mixture)
# OR: Unknown single stereoisomer

# Create stereo group
mol = Chem.MolFromSmiles('C[C@H](O)CC')
Chem.AssignStereochemistry(mol)

# Add stereo group
stereo_group = rdchem.StereoGroup(
    rdchem.StereoGroupType.STEREO_ABSOLUTE,
    [mol.GetAtomWithIdx(1)],  # Atoms in group
    0  # Group ID
)
mol.SetStereoGroups([stereo_group])

# OR group (unknown configuration)
or_group = rdchem.StereoGroup(
    rdchem.StereoGroupType.STEREO_OR,
    [mol.GetAtomWithIdx(1)],
    0
)

# AND group (racemic mixture)
and_group = rdchem.StereoGroup(
    rdchem.StereoGroupType.STEREO_AND,
    [mol.GetAtomWithIdx(1)],
    0
)
```

### CXSMILES with Stereo Groups

```python
# Write CXSMILES with stereo groups
mol = Chem.MolFromSmiles('N[C@H](C)C(=O)O |&1:1|')  # AND group
cxsmiles = Chem.MolToCXSmiles(mol)

# Read CXSMILES
mol = Chem.MolFromSmiles('N[C@H](C)C(=O)O |a:1|')  # ABS group

# Get stereo groups
stereo_groups = mol.GetStereoGroups()
for sg in stereo_groups:
    print(f"Type: {sg.GetGroupType()}")
    print(f"Atoms: {[a.GetIdx() for a in sg.GetAtoms()]}")
```

### Multiple Stereocenters with Relative Stereochemistry

```python
# Two stereocenters with AND group (diastereomer mixture)
mol = Chem.MolFromSmiles('[C@H]1CC[C@H](O)CC1 |&1:0,3|')

# OR group for unknown relative configuration
mol = Chem.MolFromSmiles('[C@H]1CC[C@H](O)CC1 |o1:0,3|')

# Multiple groups
mol = Chem.MolFromSmiles('[C@H]1CC[C@H](O)C[C@H]1C |&1:0,3,&2:6|')
```

## Stereoisomer Enumeration

### Enumerate All Stereoisomers

```python
from rdkit.Chem.EnumerateStereoisomers import EnumerateStereoisomers, StereoEnumerationOptions

# Molecule with unspecified stereocenters
mol = Chem.MolFromSmiles('CC(O)C(O)C')

# Enumerate all stereoisomers
isomers = tuple(EnumerateStereoisomers(mol))
print(f"Generated {len(isomers)} stereoisomers")

for isomer in isomers:
    print(Chem.MolToSmiles(isomer))

# With options
opts = StereoEnumerationOptions(unique=True, maxIsomers=16)
isomers = tuple(EnumerateStereoisomers(mol, options=opts))
```

### Enumerate from Stereo Groups

```python
# Molecule with stereo groups
mol = Chem.MolFromSmiles('[C@H]1CC[C@H](O)CC1 |&1:0,3|')

# Enumerate only stereo groups
opts = StereoEnumerationOptions(onlyStereoGroups=True)
isomers = tuple(EnumerateStereoisomers(mol, options=opts))

# Each isomer has explicit stereochemistry
for isomer in isomers:
    print(Chem.MolToSmiles(isomer))
```

## Atropisomers

### Restricted Rotation Bonds

```python
# Atropisomeric bonds (restricted rotation)
mol = Chem.MolFromSmiles('Cc1ccccc1-c1c(C)cccc1C')

# Generate 3D conformer (required for atropisomer detection)
from rdkit.Chem import AllChem
mol = Chem.AddHs(mol)
AllChem.EmbedMolecule(mol)

# Detect and assign atropisomers
Chem.FindPotentialStereo(mol)
Chem.AssignStereochemistry(mol)

# Check for atropisomeric bonds
for bond in mol.GetBonds():
    stereo = bond.GetStereo()
    if stereo in [Chem.BondStereo.STEREOATROPCCW, Chem.BondStereo.STEREOATROPCW]:
        print(f"Bond {bond.GetIdx()} is atropisomeric: {stereo}")

# Clean up invalid atropisomers
Chem.cleanupAtropisomers(mol)
```

### Atropisomer Notation

```python
# From SMILES with atropisomer notation
mol = Chem.MolFromSmiles('Cc1ccccc1-c1c(C)cccc1C |(-1.0,0,;0,0,;...)|')

# To CXSMILES with coordinates
cxsmiles = Chem.MolToCXSmiles(mol)
```

## Stereochemistry in Mol Files

### 2D with Wedge Bonds

```python
# Mol file with wedge bonds
mol_block = """
  Mrv0541 02251512282D

  5  4  0  0  0  0            999 V2000
    0.0000    0.0000    0.0000 C   0  0  0  0  0  0  0  0  0  0  0  0
    1.5000    0.0000    0.0000 C   0  0  1  0  0  0  0  0  0  0  0  0
    2.2500    1.2990    0.0000 O   0  0  0  0  0  0  0  0  0  0  0  0
    2.2500   -1.2990    0.0000 C   0  0  0  0  0  0  0  0  0  0  0  0
    3.0000    0.0000    0.0000 H   0  0  0  0  0  0  0  0  0  0  0  0
  1  2  1  0  0  0  0
  2  3  1  0  0  0  0
  2  4  1  0  0  0  0
  2  5  1  1  0  0  0
M  END
"""

mol = Chem.MolFromMolBlock(mol_block)
Chem.AssignStereochemistry(mol)
```

### 3D Coordinates

```python
# 3D mol file (stereochemistry from coordinates)
mol = Chem.MolFromMolFile('molecule_3d.mol')

# Assign from 3D coordinates
Chem.AssignStereochemistryFrom3D(mol)
```

### V3000 with Enhanced Stereo

```python
# V3000 format with stereo groups
mol = Chem.MolFromMolBlock(v3000_block)

# Get stereo groups
stereo_groups = mol.GetStereoGroups()

# Write V3000
mol_block = Chem.MolToV3KMolBlock(mol)
```

## Stereochemistry in Reactions

### Preserving Stereochemistry

```python
from rdkit.Chem import AllChem

# Reaction that preserves stereochemistry
rxn = AllChem.ReactionFromSmarts('[C:1]=[O:2].[C@H:3]>>[C:1][O:2][C@H:3]')

# Run reaction
reactants = (Chem.MolFromSmiles('CC=O'), Chem.MolFromSmiles('C[C@H](O)C'))
products = rxn.RunReactants(reactants)

# Product maintains stereochemistry
product = products[0][0]
print(Chem.MolToSmiles(product))
```

### Inverting Stereochemistry

```python
# Reaction that inverts configuration
rxn = AllChem.ReactionFromSmarts('[C@H:1]>>[C@@H:1]')

# Inversion at chiral center
reactant = Chem.MolFromSmiles('C[C@H](O)CC')
products = rxn.RunReactants((reactant,))
product = products[0][0]
print(Chem.MolToSmiles(product))  # Configuration inverted
```

### Creating New Stereocenters

```python
# Reaction creates new stereocenter
rxn = AllChem.ReactionFromSmarts('[C:1]=[C:2]>>[C@H:1][C:2]')

# Product has new stereocenter
reactant = Chem.MolFromSmiles('C=C')
products = rxn.RunReactants((reactant,))
```

## Substructure Matching with Stereochemistry

### Chiral Matching

```python
# Pattern with chirality
pattern = Chem.MolFromSmarts('[C@H](O)(C)CC')

# Match with chirality
mol1 = Chem.MolFromSmiles('C[C@H](O)CC')   # Matches
mol2 = Chem.MolFromSmiles('C[C@@H](O)CC')  # Doesn't match

match1 = mol1.HasSubstructMatch(pattern, useChirality=True)  # True
match2 = mol2.HasSubstructMatch(pattern, useChirality=True)  # False
```

### Enhanced Stereo Matching

```python
# Matching with stereo groups
from rdkit.Chem import rdChemReactions

# Query with ABS group matches molecules with ABS
# Query with OR matches both configurations
# Achiral query matches everything
```

## Stereochemistry Properties

### Atom Properties

```python
# Get chirality properties
atom = mol.GetAtomWithIdx(1)

# Chiral tag
tag = atom.GetChiralTag()

# CIP code
if atom.HasProp('_CIPCode'):
    cip = atom.GetProp('_CIPCode')  # 'R' or 'S'

# Chirality possible
if atom.HasProp('_ChiralityPossible'):
    possible = atom.GetBoolProp('_ChiralityPossible')
```

### Bond Properties

```python
# Bond stereochemistry
bond = mol.GetBondWithIdx(1)

# Stereo
stereo = bond.GetStereo()

# CIP code for double bonds
if bond.HasProp('_CIPCode'):
    cip = bond.GetProp('_CIPCode')  # 'E' or 'Z'
```

## Best Practices

1. **Always assign stereochemistry**: Call `AssignStereochemistry()` after reading
2. **Use CIP labels**: Call `AssignCIPLabels()` for accurate R/S assignment
3. **Handle unspecified**: Use stereo groups or enumeration
4. **3D for atropisomers**: Requires coordinates for detection
5. **Preserve in reactions**: Use atom mapping with chirality notation
6. **Match with chirality**: Set `useChirality=True` when needed
7. **Enumerate systematically**: Use `EnumerateStereoisomers()` for libraries

## Common Pitfalls

- **Missing assignment**: Stereochemistry not automatically assigned
- **Approximate CIP**: Legacy method may give wrong labels
- **2D vs 3D**: 3D coordinates override wedge bonds
- **Unspecified handling**: Unclear whether achiral or unknown
- **Reaction stereochemistry**: Must specify in products or use mapping
- **Stereo group compatibility**: Not all formats support enhanced stereo
- **Atropisomer detection**: Requires explicit call and 3D coordinates
