# RDKit Advanced Features Reference

Sources:
- [Getting Started in Python](https://www.rdkit.org/docs/GettingStartedInPython.html)
- [The RDKit Book](https://www.rdkit.org/docs/RDKit_Book.html)

## Molecular Drawing

### Basic Drawing

```python
from rdkit.Chem import Draw

# Single molecule
img = Draw.MolToImage(mol, size=(300, 300))
Draw.MolToFile(mol, 'molecule.png')

# Grid of molecules
imgs = Draw.MolsToGridImage(
    mols,
    molsPerRow=4,
    subImgSize=(200, 200),
    legends=['Mol 1', 'Mol 2']
)
```

### Advanced Drawing

```python
from rdkit.Chem.Draw import rdMolDraw2D

# SVG drawing
d = rdMolDraw2D.MolDraw2DSVG(500, 500)
rdMolDraw2D.PrepareAndDrawMolecule(d, mol)
svg = d.GetDrawingText()

# With highlighting
match = mol.GetSubstructMatch(pattern)
d = rdMolDraw2D.MolDraw2DSVG(500, 500)
rdMolDraw2D.PrepareAndDrawMolecule(
    d, mol,
    highlightAtoms=list(match),
    highlightAtomColors={i: (1, 0, 0) for i in match}
)
```

## Molecular Fragmentation

### BRICS

```python
from rdkit.Chem import BRICS

# Decompose
fragments = BRICS.BRICSDecompose(mol)

# Build library
new_mols = BRICS.BRICSBuild(fragments)

# Enumerate products
for new_mol in new_mols:
    print(Chem.MolToSmiles(new_mol))
```

### Recap

```python
from rdkit.Chem import Recap

# Hierarchical decomposition
hierarch = Recap.RecapDecompose(mol)

# Get leaves
leaves = hierarch.GetLeaves()
for leaf in leaves.values():
    print(leaf.smiles)
```

### Maximum Common Substructure

```python
from rdkit.Chem import rdFMCS

# Find MCS
mcs = rdFMCS.FindMCS([mol1, mol2, mol3])
print(mcs.smartsString)

# With parameters
mcs = rdFMCS.FindMCS(
    mols,
    ringMatchesRingOnly=True,
    completeRingsOnly=True,
    timeout=60,
    atomCompare=rdFMCS.AtomCompare.CompareElements,
    bondCompare=rdFMCS.BondCompare.CompareOrder
)

# As molecule
mcs_mol = Chem.MolFromSmarts(mcs.smartsString)
```

## Serialization

### Pickle

```python
import pickle

# Serialize
pkl = pickle.dumps(mol)
with open('mol.pkl', 'wb') as f:
    pickle.dump(mol, f)

# Deserialize
mol = pickle.loads(pkl)
with open('mol.pkl', 'rb') as f:
    mol = pickle.load(f)
```

### Binary

```python
# To binary
bin_str = mol.ToBinary()

# From binary
mol = Chem.Mol(bin_str)
```

### JSON

```python
# To JSON
json_str = Chem.MolToJSON(mol)

# From JSON
mol = Chem.MolFromJSON(json_str)
```

## Molecular Sanitization

```python
# Full sanitization
Chem.SanitizeMol(mol)

# Partial sanitization
Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_ALL^Chem.SANITIZE_KEKULIZE)

# Individual operations
Chem.Kekulize(mol)
Chem.SetAromaticity(mol)
Chem.AssignStereochemistry(mol)
```

## Property Calculations

### Molecular Surface Area

```python
from rdkit.Chem import Descriptors3D

# Requires 3D
mol = Chem.AddHs(mol)
AllChem.EmbedMolecule(mol)

# Surfaces
tpsa = Descriptors.TPSA(mol)  # 2D
lasa = Descriptors3D.LabuteASA(mol)  # 3D
```

### Molecular Volume

```python
# Van der Waals volume
from rdkit.Chem import AllChem, Descriptors3D

mol = Chem.AddHs(mol)
AllChem.EmbedMolecule(mol)

volume = Descriptors3D.CalcMolVolume(mol)
```

## Molecular Similarity

### 2D Similarity

```python
from rdkit import DataStructs
from rdkit.Chem import AllChem

fp1 = AllChem.GetMorganFingerprintAsBitVect(mol1, 2)
fp2 = AllChem.GetMorganFingerprintAsBitVect(mol2, 2)

sim = DataStructs.TanimotoSimilarity(fp1, fp2)
```

### 3D Shape Similarity

```python
from rdkit.Chem import rdShapeHelpers

# Requires aligned 3D conformers
shape_sim = rdShapeHelpers.ShapeTanimotoDist(mol1, mol2)
protrude_sim = rdShapeHelpers.ShapeProtrudeDist(mol1, mol2)
```

## Molecular Filters

### Drug-likeness

```python
from rdkit.Chem import QED, Crippen

# QED score
qed = QED.qed(mol)

# Lipinski
def lipinski(mol):
    return (
        Descriptors.MolWt(mol) <= 500 and
        Crippen.MolLogP(mol) <= 5 and
        Descriptors.NumHDonors(mol) <= 5 and
        Descriptors.NumHAcceptors(mol) <= 10
    )

# Veber rules
def veber(mol):
    return (
        Descriptors.NumRotatableBonds(mol) <= 10 and
        Descriptors.TPSA(mol) <= 140
    )
```

### PAINS Filters

```python
# Load PAINS patterns
from rdkit import RDConfig
import os

pains_file = os.path.join(RDConfig.RDDataDir, 'Pains', 'wehi_pains.csv')
pains = []
with open(pains_file) as f:
    for line in f:
        smarts = line.split(',')[1]
        pains.append(Chem.MolFromSmarts(smarts))

def has_pains(mol):
    return any(mol.HasSubstructMatch(p) for p in pains if p)
```

## Molecular Editing

```python
# Editable molecule
em = Chem.EditableMol(mol)

# Add atom
em.AddAtom(Chem.Atom(6))  # Carbon

# Add bond
em.AddBond(0, 1, Chem.BondType.SINGLE)

# Remove atom
em.RemoveAtom(5)

# Remove bond
em.RemoveBond(2, 3)

# Get modified molecule
new_mol = em.GetMol()
```

## Molecular Standardization

```python
from rdkit.Chem import MolStandardize

# Fragment parent (largest fragment)
uncharger = MolStandardize.fragment.LargestFragmentChooser()
parent = uncharger.choose(mol)

# Neutralize
uncharger = MolStandardize.rdMolStandardize.Uncharger()
neutral = uncharger.uncharge(mol)

# Tautomer enumeration
enumerator = MolStandardize.rdMolStandardize.TautomerEnumerator()
tautomers = enumerator.Enumerate(mol)
```

## Best Practices

1. **Pickle for speed**: Fastest serialization
2. **Sanitize once**: Don't repeat
3. **Add H for 3D**: Required for conformers
4. **Use QED**: Better than Lipinski alone
5. **Filter PAINS**: Remove problematic structures
6. **Standardize**: Consistent molecule representation

## Common Pitfalls

- **Not sanitizing**: Invalid molecules
- **Missing 3D coords**: 3D descriptors fail
- **Over-filtering**: Too strict rules remove good compounds
- **Not handling tautomers**: Same compound, different representations
- **Forgetting fragments**: Keep largest fragment only
- **Ignoring stereochemistry**: Important for activity
