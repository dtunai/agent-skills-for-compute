# RDKit Descriptors and Fingerprints Reference

Sources:
- [Getting Started in Python](https://www.rdkit.org/docs/GettingStartedInPython.html)
- [The RDKit Book](https://www.rdkit.org/docs/RDKit_Book.html)
- [rdMolDescriptors API](https://www.rdkit.org/docs/source/rdkit.Chem.rdMolDescriptors.html)

## Molecular Descriptors

### Common Descriptors

```python
from rdkit.Chem import Descriptors

mol = Chem.MolFromSmiles('CC(=O)Oc1ccccc1C(=O)O')  # Aspirin

# Molecular weight
mw = Descriptors.MolWt(mol)  # 180.16

# LogP (lipophilicity)
logp = Descriptors.MolLogP(mol)

# Topological polar surface area
tpsa = Descriptors.TPSA(mol)

# H-bond donors and acceptors
hbd = Descriptors.NumHDonors(mol)
hba = Descriptors.NumHAcceptors(mol)

# Rotatable bonds
rotatable = Descriptors.NumRotatableBonds(mol)

# Aromatic rings
aromatic = Descriptors.NumAromaticRings(mol)

# Aliphatic rings
aliphatic = Descriptors.NumAliphaticRings(mol)

# Saturated rings
saturated = Descriptors.NumSaturatedRings(mol)

# Heavy atom count
heavy = Descriptors.HeavyAtomCount(mol)

# Molar refractivity
mr = Descriptors.MolMR(mol)
```

### All Descriptors

```python
# Calculate all available descriptors
all_desc = Descriptors.CalcMolDescriptors(mol)

# Returns dictionary
print(all_desc.keys())  # All descriptor names
print(all_desc['MolLogP'])
print(all_desc['TPSA'])

# Iterate
for name, value in all_desc.items():
    print(f"{name}: {value}")
```

### 3D Descriptors

Require 3D coordinates:

```python
from rdkit.Chem import AllChem, Descriptors3D

# Generate 3D coordinates
mol = Chem.AddHs(mol)
AllChem.EmbedMolecule(mol)

# 3D descriptors
pmi1, pmi2, pmi3 = Descriptors3D.PMI1(mol), Descriptors3D.PMI2(mol), Descriptors3D.PMI3(mol)
npr1, npr2 = Descriptors3D.NPR1(mol), Descriptors3D.NPR2(mol)
asphericity = Descriptors3D.Asphericity(mol)
eccentricity = Descriptors3D.Eccentricity(mol)
radius_of_gyration = Descriptors3D.RadiusOfGyration(mol)
inertial_shape_factor = Descriptors3D.InertialShapeFactor(mol)
```

### Partial Charges

```python
# Gasteiger charges
AllChem.ComputeGasteigerCharges(mol)

# Access charges
for atom in mol.GetAtoms():
    charge = atom.GetDoubleProp('_GasteigerCharge')
    print(f"Atom {atom.GetIdx()}: {charge}")

# MMFF94 charges (requires 3D)
props = AllChem.MMFFGetMoleculeProperties(mol)
for i in range(mol.GetNumAtoms()):
    charge = props.GetMMFFPartialCharge(i)
    print(f"Atom {i}: {charge}")
```

## Fingerprints

### Morgan Fingerprints (ECFP/FCFP)

```python
from rdkit.Chem import AllChem
from rdkit import DataStructs

# Morgan fingerprint (ECFP4 equivalent = radius 2)
fp = AllChem.GetMorganFingerprintAsBitVect(mol, radius=2, nBits=2048)

# Count fingerprint (for similarity with counts)
fp_count = AllChem.GetMorganFingerprint(mol, radius=2)

# Using generator (recommended)
fpgen = AllChem.GetMorganGenerator(radius=2, fpSize=2048)
fp = fpgen.GetFingerprint(mol)

# Feature-based (FCFP)
from rdkit.Chem.AllChem import GetMorganFeatureAtomInvGen
invgen = GetMorganFeatureAtomInvGen()
fpgen = AllChem.GetMorganGenerator(radius=2, atomInvariantsGenerator=invgen)
fp = fpgen.GetFingerprint(mol)

# With bit info (identify which bits correspond to which atoms)
info = {}
fp = AllChem.GetMorganFingerprintAsBitVect(mol, radius=2, nBits=2048, bitInfo=info)
# info maps bit indices to (atom_idx, radius) tuples
```

### RDKit Fingerprints

```python
# Topological fingerprint
fpgen = AllChem.GetRDKitFPGenerator()
fp = fpgen.GetFingerprint(mol)

# With parameters
fpgen = AllChem.GetRDKitFPGenerator(
    minPath=1,
    maxPath=7,
    fpSize=2048,
    nBitsPerHash=2
)
fp = fpgen.GetFingerprint(mol)
```

### Atom Pair Fingerprints

```python
from rdkit.Chem import rdMolDescriptors

# Atom pair fingerprint
ap = rdMolDescriptors.GetAtomPairFingerprint(mol)

# As bit vector
ap_bits = rdMolDescriptors.GetHashedAtomPairFingerprintAsBitVect(mol, nBits=2048)
```

### Topological Torsion Fingerprints

```python
# Topological torsion
tt = rdMolDescriptors.GetTopologicalTorsionFingerprint(mol)

# As bit vector
tt_bits = rdMolDescriptors.GetHashedTopologicalTorsionFingerprintAsBitVect(mol, nBits=2048)
```

### MACCS Keys

```python
from rdkit.Chem import MACCSkeys

# 166-bit MACCS keys
maccs = MACCSkeys.GenMACCSKeys(mol)
```

### Pattern Fingerprints

```python
# Pattern fingerprint (substructure keys)
from rdkit.Chem import PatternFingerprint

fp = PatternFingerprint.PatternFingerprintMol(mol)
```

### Layered Fingerprints

```python
# Layered fingerprint
from rdkit.Chem import LayeredFingerprint

fp = LayeredFingerprint.LayeredFingerprintMol(mol)

# With specific layers
fp = LayeredFingerprint.LayeredFingerprintMol(
    mol,
    layerFlags=LayeredFingerprint.substructLayers.LAYERFLAGS_ALL
)
```

## Similarity Calculations

### Tanimoto Similarity

```python
from rdkit import DataStructs

fp1 = AllChem.GetMorganFingerprintAsBitVect(mol1, 2, 2048)
fp2 = AllChem.GetMorganFingerprintAsBitVect(mol2, 2, 2048)

# Tanimoto
sim = DataStructs.TanimotoSimilarity(fp1, fp2)

# Dice
sim = DataStructs.DiceSimilarity(fp1, fp2)

# Cosine
sim = DataStructs.CosineSimilarity(fp1, fp2)

# Sokal
sim = DataStructs.SokalSimilarity(fp1, fp2)

# Kulczynski
sim = DataStructs.KulczynskiSimilarity(fp1, fp2)

# McConnaughey
sim = DataStructs.McConnaugheySimilarity(fp1, fp2)

# Asymmetric
sim = DataStructs.AsymmetricSimilarity(fp1, fp2)
```

### Bulk Similarity

```python
# One vs many
query_fp = AllChem.GetMorganFingerprintAsBitVect(query, 2, 2048)
library_fps = [AllChem.GetMorganFingerprintAsBitVect(m, 2, 2048) for m in library]

# Bulk Tanimoto
sims = DataStructs.BulkTanimotoSimilarity(query_fp, library_fps)

# Bulk Dice
sims = DataStructs.BulkDiceSimilarity(query_fp, library_fps)

# Get indices above threshold
threshold = 0.7
hits = [i for i, sim in enumerate(sims) if sim >= threshold]
```

### Distance Calculations

```python
# Tanimoto distance (1 - similarity)
dist = 1 - DataStructs.TanimotoSimilarity(fp1, fp2)

# All-pairs distances
from rdkit.ML.Cluster import Butina

dists = []
nfps = len(fps)
for i in range(1, nfps):
    sims = DataStructs.BulkTanimotoSimilarity(fps[i], fps[:i])
    dists.extend([1 - sim for sim in sims])

# Distance matrix
import numpy as np
dist_matrix = np.zeros((nfps, nfps))
for i in range(nfps):
    for j in range(i+1, nfps):
        dist = 1 - DataStructs.TanimotoSimilarity(fps[i], fps[j])
        dist_matrix[i, j] = dist
        dist_matrix[j, i] = dist
```

## Fingerprint Analysis

### Bit Vectors

```python
# Fingerprint as bit vector
fp = AllChem.GetMorganFingerprintAsBitVect(mol, 2, 2048)

# Get on bits
on_bits = fp.GetOnBits()
print(list(on_bits))

# Number of on bits
num_on = fp.GetNumOnBits()

# Convert to numpy array
import numpy as np
arr = np.zeros((1,), dtype=np.int8)
DataStructs.ConvertToNumpyArray(fp, arr)

# From numpy array
fp2 = DataStructs.CreateFromBitString(''.join(arr.astype(str)))
```

### Fingerprint Information

```python
# Get bit info (which atoms contribute to which bits)
info = {}
fp = AllChem.GetMorganFingerprintAsBitVect(mol, radius=2, bitInfo=info)

# info maps bit -> list of (atom_idx, radius) tuples
for bit, atom_info in info.items():
    print(f"Bit {bit}: {atom_info}")

# Get environment for specific bit
bit_idx = 123
if bit_idx in info:
    for atom_idx, radius in info[bit_idx]:
        env = Chem.FindAtomEnvironmentOfRadiusN(mol, radius, atom_idx)
        print(f"Atom {atom_idx}, radius {radius}, environment: {env}")
```

## Specialized Descriptors

### Lipinski's Rule of Five

```python
def lipinski_pass(mol):
    mw = Descriptors.MolWt(mol)
    logp = Descriptors.MolLogP(mol)
    hbd = Descriptors.NumHDonors(mol)
    hba = Descriptors.NumHAcceptors(mol)

    conditions = [
        mw <= 500,
        logp <= 5,
        hbd <= 5,
        hba <= 10
    ]

    return all(conditions), {
        'MW': mw, 'LogP': logp, 'HBD': hbd, 'HBA': hba,
        'Pass': all(conditions)
    }
```

### QED (Quantitative Estimate of Drug-likeness)

```python
from rdkit.Chem import QED

# QED score (0-1, higher is more drug-like)
qed_score = QED.qed(mol)

# Detailed properties
qed_props = QED.properties(mol)
print(qed_props.MW)
print(qed_props.ALOGP)
print(qed_props.PSA)
```

### Synthetic Accessibility Score

```python
from rdkit.Chem import RDConfig
import os, sys
sys.path.append(os.path.join(RDConfig.RDContribDir, 'SA_Score'))
import sascorer

# SA score (1-10, lower is easier to synthesize)
sa_score = sascorer.calculateScore(mol)
```

## Performance Optimization

```python
# Reuse generators
fpgen = AllChem.GetMorganGenerator(radius=2, fpSize=2048)
fps = [fpgen.GetFingerprint(mol) for mol in mols]

# Bulk operations
fpgen = AllChem.GetMorganGenerator(radius=2)
fps = [fpgen.GetFingerprint(m) for m in mols]
similarities = [DataStructs.BulkTanimotoSimilarity(fps[0], fps[1:])]

# Parallel fingerprint generation
from multiprocessing import Pool

def gen_fp(mol):
    return AllChem.GetMorganFingerprintAsBitVect(mol, 2, 2048)

with Pool(4) as pool:
    fps = pool.map(gen_fp, mols)
```

## Best Practices

1. **Morgan radius=2** ≈ ECFP4 (common default)
2. **Use generators** for batch operations (more efficient)
3. **Bit info** for interpretability
4. **Bulk similarity** for screening
5. **MACCS keys** for quick screening
6. **Morgan + Tanimoto** for general similarity
7. **QED/SA scores** for drug-likeness filters

## Common Pitfalls

- **Forgetting sanitization**: Some descriptors require sanitized molecules
- **Missing 3D coords**: 3D descriptors fail without coordinates
- **Wrong fingerprint type**: Morgan ≠ RDKit ≠ MACCS
- **Bit vector size**: Must match for similarity calculations
- **Count vs bit vectors**: Choose based on use case
- **Not caching fingerprints**: Recomputing is expensive
