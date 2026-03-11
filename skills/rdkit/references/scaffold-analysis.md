# RDKit Scaffold Analysis and Murcko Decomposition Reference

Sources:
- [MurckoScaffold API](https://rdkit.org/docs/source/rdkit.Chem.Scaffolds.MurckoScaffold.html)
- [Murcko Scaffolds Tutorial](https://greglandrum.github.io/rdkit-blog/posts/2025-06-06-murcko-scaffolds.html)
- [The RDKit Book](https://www.rdkit.org/docs/RDKit_Book.html)

## Murcko Scaffolds

### Overview

Murcko scaffolds (Bemis-Murcko scaffolds) represent the core ring systems and linkers of molecules, removing all side chains and substituents. Useful for clustering, diversity analysis, and scaffold-based splits.

### Basic Scaffold Extraction

```python
from rdkit import Chem
from rdkit.Chem.Scaffolds import MurckoScaffold

# Get scaffold
mol = Chem.MolFromSmiles('c1ccc(CCO)cc1')  # Phenethyl alcohol
scaffold = MurckoScaffold.GetScaffoldForMol(mol)
scaffold_smiles = Chem.MolToSmiles(scaffold)
print(scaffold_smiles)  # 'c1ccccc1' (benzene)

# Direct SMILES
scaffold_smiles = MurckoScaffold.MurckoScaffoldSmiles(mol=mol)

# From SMILES string
scaffold_smiles = MurckoScaffold.MurckoScaffoldSmilesFromSmiles('c1ccc(CCO)cc1')
```

### Generic Scaffolds

```python
# Make scaffold generic (all atoms -> C, all bonds -> single)
mol = Chem.MolFromSmiles('c1ncccc1')  # Pyridine
scaffold = MurckoScaffold.GetScaffoldForMol(mol)
generic = MurckoScaffold.MakeScaffoldGeneric(scaffold)
generic_smiles = Chem.MolToSmiles(generic)
print(generic_smiles)  # 'C1CCCCC1' (cyclohexane)

# Both benzene and pyridine have same generic scaffold
benzene = Chem.MolFromSmiles('c1ccccc1')
pyridine = Chem.MolFromSmiles('c1ncccc1')

gen_benzene = MurckoScaffold.MakeScaffoldGeneric(MurckoScaffold.GetScaffoldForMol(benzene))
gen_pyridine = MurckoScaffold.MakeScaffoldGeneric(MurckoScaffold.GetScaffoldForMol(pyridine))

assert Chem.MolToSmiles(gen_benzene) == Chem.MolToSmiles(gen_pyridine)
```

### Preserving Chirality

```python
# With chirality
mol = Chem.MolFromSmiles('C[C@H](O)c1ccccc1')
scaffold = MurckoScaffold.MurckoScaffoldSmiles(mol=mol, includeChirality=True)
print(scaffold)  # Includes chirality

# Without chirality (default)
scaffold = MurckoScaffold.MurckoScaffoldSmiles(mol=mol, includeChirality=False)
```

## Scaffold Analysis

### Scaffold Counting

```python
from collections import Counter

# Analyze scaffold distribution
mols = [Chem.MolFromSmiles(s) for s in smiles_list]
scaffolds = [MurckoScaffold.MurckoScaffoldSmilesFromSmiles(s) for s in smiles_list]

scaffold_counts = Counter(scaffolds)

# Most common scaffolds
for scaffold, count in scaffold_counts.most_common(10):
    print(f"{scaffold}: {count}")
```

### Scaffold Trees

```python
# Build scaffold tree (hierarchical decomposition)
from rdkit.Chem.Scaffolds import MurckoScaffold

def get_scaffold_hierarchy(mol):
    """Get hierarchy of scaffolds"""
    scaffolds = []
    current = mol
    
    while True:
        scaffold = MurckoScaffold.GetScaffoldForMol(current)
        if scaffold is None or scaffold.GetNumAtoms() == 0:
            break
        
        smiles = Chem.MolToSmiles(scaffold)
        if smiles in scaffolds:
            break
        
        scaffolds.append(smiles)
        current = scaffold
    
    return scaffolds

# Example
mol = Chem.MolFromSmiles('c1ccc(CCOc2ccccc2)cc1')
hierarchy = get_scaffold_hierarchy(mol)
for i, scaffold in enumerate(hierarchy):
    print(f"Level {i}: {scaffold}")
```

## Scaffold Clustering

### Group by Scaffold

```python
from collections import defaultdict

# Group molecules by scaffold
scaffold_to_mols = defaultdict(list)

for mol in mols:
    scaffold = MurckoScaffold.MurckoScaffoldSmilesFromSmiles(Chem.MolToSmiles(mol))
    scaffold_to_mols[scaffold].append(mol)

# Analyze clusters
for scaffold, mol_list in scaffold_to_mols.items():
    print(f"Scaffold {scaffold}: {len(mol_list)} molecules")
```

### Generic Scaffold Clustering

```python
# Cluster by generic scaffold
generic_scaffold_to_mols = defaultdict(list)

for mol in mols:
    scaffold = MurckoScaffold.GetScaffoldForMol(mol)
    generic = MurckoScaffold.MakeScaffoldGeneric(scaffold)
    generic_smiles = Chem.MolToSmiles(generic)
    generic_scaffold_to_mols[generic_smiles].append(mol)
```

## Scaffold-Based Splits

### Train/Test Split by Scaffold

```python
import random

def scaffold_split(mols, test_size=0.2, random_seed=42):
    """Split molecules by scaffold"""
    random.seed(random_seed)
    
    # Group by scaffold
    scaffold_to_mols = defaultdict(list)
    for mol in mols:
        scaffold = MurckoScaffold.MurckoScaffoldSmilesFromSmiles(Chem.MolToSmiles(mol))
        scaffold_to_mols[scaffold].append(mol)
    
    # Shuffle scaffolds
    scaffolds = list(scaffold_to_mols.keys())
    random.shuffle(scaffolds)
    
    # Split
    train_mols = []
    test_mols = []
    test_count = int(len(mols) * test_size)
    
    for scaffold in scaffolds:
        if len(test_mols) < test_count:
            test_mols.extend(scaffold_to_mols[scaffold])
        else:
            train_mols.extend(scaffold_to_mols[scaffold])
    
    return train_mols, test_mols

# Usage
train, test = scaffold_split(mols, test_size=0.2)
print(f"Train: {len(train)}, Test: {len(test)}")
```

## Scaffold Diversity

### Calculate Diversity

```python
# Scaffold diversity (unique scaffolds / total molecules)
unique_scaffolds = len(set(scaffolds))
total_molecules = len(scaffolds)
diversity = unique_scaffolds / total_molecules

print(f"Scaffold diversity: {diversity:.2%}")

# Generic scaffold diversity
generic_scaffolds = [
    Chem.MolToSmiles(MurckoScaffold.MakeScaffoldGeneric(
        MurckoScaffold.GetScaffoldForMol(mol)
    ))
    for mol in mols
]
generic_diversity = len(set(generic_scaffolds)) / len(mols)
```

### Scaffold Similarity

```python
from rdkit import DataStructs
from rdkit.Chem import AllChem

# Compare scaffolds by fingerprint similarity
scaffold1 = MurckoScaffold.GetScaffoldForMol(mol1)
scaffold2 = MurckoScaffold.GetScaffoldForMol(mol2)

fp1 = AllChem.GetMorganFingerprintAsBitVect(scaffold1, 2)
fp2 = AllChem.GetMorganFingerprintAsBitVect(scaffold2, 2)

similarity = DataStructs.TanimotoSimilarity(fp1, fp2)
```

## Advanced Scaffold Operations

### Scaffold Enumeration

```python
# Enumerate scaffolds in library
scaffolds_with_counts = []

for scaffold, mol_list in scaffold_to_mols.items():
    scaffold_mol = Chem.MolFromSmiles(scaffold)
    scaffolds_with_counts.append({
        'smiles': scaffold,
        'mol': scaffold_mol,
        'count': len(mol_list),
        'examples': mol_list[:3]  # First 3 examples
    })

# Sort by count
scaffolds_with_counts.sort(key=lambda x: x['count'], reverse=True)
```

### Scaffold Filtering

```python
# Filter molecules by scaffold properties
def filter_by_scaffold(mols, min_rings=2, max_rings=4):
    """Keep molecules with scaffold in ring count range"""
    filtered = []
    
    for mol in mols:
        scaffold = MurckoScaffold.GetScaffoldForMol(mol)
        if scaffold is None:
            continue
        
        ring_info = scaffold.GetRingInfo()
        num_rings = ring_info.NumRings()
        
        if min_rings <= num_rings <= max_rings:
            filtered.append(mol)
    
    return filtered
```

### Scaffold Decorations

```python
# Analyze decorations (R-groups) for each scaffold
def get_scaffold_decorations(mol):
    """Get decorations (side chains) for scaffold"""
    scaffold = MurckoScaffold.GetScaffoldForMol(mol)
    
    if scaffold is None:
        return []
    
    # Find atoms in scaffold
    scaffold_match = mol.GetSubstructMatch(scaffold)
    
    # Find atoms not in scaffold (decorations)
    decoration_atoms = set(range(mol.GetNumAtoms())) - set(scaffold_match)
    
    return list(decoration_atoms)

# Analyze
decorations = get_scaffold_decorations(mol)
print(f"Decoration atoms: {decorations}")
```

## Integration with Other Analyses

### Scaffold + Fingerprint

```python
# Scaffold-aware similarity
def scaffold_aware_similarity(mol1, mol2):
    """Check if same scaffold, then calculate similarity"""
    scaffold1 = MurckoScaffold.MurckoScaffoldSmilesFromSmiles(Chem.MolToSmiles(mol1))
    scaffold2 = MurckoScaffold.MurckoScaffoldSmilesFromSmiles(Chem.MolToSmiles(mol2))
    
    if scaffold1 != scaffold2:
        return 0.0  # Different scaffolds
    
    # Same scaffold, calculate similarity
    fp1 = AllChem.GetMorganFingerprintAsBitVect(mol1, 2)
    fp2 = AllChem.GetMorganFingerprintAsBitVect(mol2, 2)
    
    return DataStructs.TanimotoSimilarity(fp1, fp2)
```

### Scaffold + Descriptors

```python
from rdkit.Chem import Descriptors

# Analyze properties by scaffold
scaffold_properties = defaultdict(list)

for mol in mols:
    scaffold = MurckoScaffold.MurckoScaffoldSmilesFromSmiles(Chem.MolToSmiles(mol))
    
    properties = {
        'MW': Descriptors.MolWt(mol),
        'LogP': Descriptors.MolLogP(mol),
        'TPSA': Descriptors.TPSA(mol)
    }
    
    scaffold_properties[scaffold].append(properties)

# Average properties per scaffold
for scaffold, props_list in scaffold_properties.items():
    avg_mw = sum(p['MW'] for p in props_list) / len(props_list)
    print(f"{scaffold}: Avg MW = {avg_mw:.2f}")
```

## Best Practices

1. **Use for clustering**: Group similar molecules by scaffold
2. **Generic scaffolds**: Normalize heteroatom differences
3. **Scaffold splits**: Prevent data leakage in ML
4. **Diversity analysis**: Measure library coverage
5. **Hierarchical decomposition**: Build scaffold trees
6. **Filter early**: Remove unwanted scaffolds early in pipeline

## Common Pitfalls

- **Empty scaffolds**: Some molecules have no scaffold (acyclic)
- **Over-generalization**: Generic scaffolds lose information
- **Scaffold uniqueness**: Different molecules can have same scaffold
- **Split imbalance**: Scaffold splits may be unbalanced
- **Chirality**: Decide whether to preserve stereochemistry
- **Linker definition**: What counts as linker varies
