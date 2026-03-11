---
name: rdkit
description: "RDKit cheminformatics toolkit — molecular I/O, descriptors, fingerprints, substructure searching, reactions, and 3D conformer generation"
license: MIT
metadata:
  author: Agent Cluster
  tags: [rdkit, cheminformatics, molecules, smiles, fingerprints, descriptors, reactions]
---

# RDKit Skill

Cheminformatics and machine learning toolkit for molecular operations, property calculation, and chemical analysis.

**Official Sources:**
- [RDKit Documentation](https://www.rdkit.org/docs/)
- [Getting Started in Python](https://www.rdkit.org/docs/GettingStartedInPython.html)
- [The RDKit Book](https://www.rdkit.org/docs/RDKit_Book.html)

## Installation

```bash
# Via conda (recommended)
conda install -c conda-forge rdkit

# Via pip
pip install rdkit

# Verify installation
python -c "from rdkit import Chem; print(Chem.__version__)"
```

## Quick Start

```python
from rdkit import Chem
from rdkit.Chem import AllChem, Descriptors, Draw

# Create molecule from SMILES
mol = Chem.MolFromSmiles('CC(=O)Oc1ccccc1C(=O)O')  # Aspirin

# Calculate properties
mw = Descriptors.MolWt(mol)
logp = Descriptors.MolLogP(mol)

# Generate fingerprint
fp = AllChem.GetMorganFingerprintAsBitVect(mol, radius=2, nBits=2048)

# Draw molecule
img = Draw.MolToImage(mol)
```

## Molecular I/O

```python
from rdkit import Chem

# Read/write single molecules
mol = Chem.MolFromSmiles('c1ccccc1')
mol = Chem.MolFromMolFile('input.mol')
smiles = Chem.MolToSmiles(mol)
Chem.MolToMolFile(mol, 'output.mol')

# Read/write multiple molecules
suppl = Chem.SDMolSupplier('molecules.sdf')
mols = [m for m in suppl if m is not None]

with Chem.SDWriter('output.sdf') as w:
    for mol in mols:
        w.write(mol)
```

## Molecular Operations

```python
# Atoms and bonds
for atom in mol.GetAtoms():
    print(atom.GetSymbol(), atom.GetAtomicNum())
atom = mol.GetAtomWithIdx(0)
bond = mol.GetBondWithIdx(0).GetBondType()

# Rings
atom.IsInRing()
atom.IsInRingSize(6)
ssr = Chem.GetSymmSSSR(mol)

# Modify
mol_with_h = Chem.AddHs(mol)
mol_no_h = Chem.RemoveHs(mol)
Chem.Kekulize(mol)
Chem.SanitizeMol(mol)
```

## Molecular Descriptors

```python
from rdkit.Chem import Descriptors, AllChem

# Common descriptors
Descriptors.MolWt(mol)
Descriptors.MolLogP(mol)
Descriptors.TPSA(mol)
Descriptors.NumHDonors(mol)
Descriptors.NumHAcceptors(mol)
Descriptors.NumRotatableBonds(mol)

# All descriptors
all_desc = Descriptors.CalcMolDescriptors(mol)

# Partial charges
AllChem.ComputeGasteigerCharges(mol)
charge = mol.GetAtomWithIdx(0).GetDoubleProp('_GasteigerCharge')
```

## Fingerprints and Similarity

```python
from rdkit.Chem import AllChem, MACCSkeys, rdMolDescriptors
from rdkit import DataStructs

# Morgan (ECFP)
fp1 = AllChem.GetMorganFingerprintAsBitVect(mol1, radius=2, nBits=2048)
fpgen = AllChem.GetMorganGenerator(radius=2, fpSize=2048)
fp = fpgen.GetFingerprint(mol)

# RDKit fingerprint
fpgen = AllChem.GetRDKitFPGenerator()
fp = fpgen.GetFingerprint(mol)

# Atom pair, topological torsion, MACCS
ap = rdMolDescriptors.GetAtomPairFingerprint(mol)
tt = rdMolDescriptors.GetTopologicalTorsionFingerprint(mol)
maccs = MACCSkeys.GenMACCSKeys(mol)

# Similarity
sim = DataStructs.TanimotoSimilarity(fp1, fp2)
sims = DataStructs.BulkTanimotoSimilarity(fp1, [fp2, fp3])
```

## Substructure Searching

```python
# SMARTS pattern matching
pattern = Chem.MolFromSmarts('c1ccccc1')
has_match = mol.HasSubstructMatch(pattern)
match = mol.GetSubstructMatch(pattern)
matches = mol.GetSubstructMatches(pattern)

# With chirality
matches = mol.GetSubstructMatches(pattern, useChirality=True)

# Functional groups
carboxylic_acid = Chem.MolFromSmarts('C(=O)[OH]')
has_acid = mol.HasSubstructMatch(carboxylic_acid)
```

## Chemical Reactions

```python
from rdkit.Chem import AllChem

# Reaction SMARTS (amide bond formation)
rxn = AllChem.ReactionFromSmarts('[C:1](=[O:2])-[OD1].[N!H0:3]>>[C:1](=[O:2])[N:3]')
acid = Chem.MolFromSmiles('CC(=O)O')
amine = Chem.MolFromSmiles('CN')
products = rxn.RunReactants((acid, amine))

# From file
rxn = AllChem.ReactionFromRxnFile('reaction.rxn')

# Protect atoms
amide = Chem.MolFromSmarts('[N;$(NC=[O,S])]')
for m in mol.GetSubstructMatches(amide):
    mol.GetAtomWithIdx(m[0]).SetProp('_protected', '1')
```

## 2D and 3D Coordinates

```python
from rdkit.Chem import AllChem

# 2D coordinates
AllChem.Compute2DCoords(mol)

# 3D conformers
mol = Chem.AddHs(mol)
AllChem.EmbedMolecule(mol)  # Single conformer
conf_ids = AllChem.EmbedMultipleConfs(mol, numConfs=10)  # Multiple

# Optimize
for conf_id in conf_ids:
    AllChem.UFFOptimizeMolecule(mol, confId=conf_id)

# Align
rms_list = []
AllChem.AlignMolConformers(mol, RMSlist=rms_list)
```

## Drawing Molecules

```python
from rdkit.Chem import Draw
from rdkit.Chem.Draw import rdMolDraw2D

# Basic drawing
img = Draw.MolToImage(mol, size=(300, 300))
Draw.MolToFile(mol, 'molecule.png')
img = Draw.MolsToGridImage(mols, molsPerRow=2, legends=['A', 'B'])

# Highlight substructures
match = mol.GetSubstructMatch(pattern)
d = rdMolDraw2D.MolDraw2DSVG(400, 400)
rdMolDraw2D.PrepareAndDrawMolecule(d, mol, highlightAtoms=list(match))
```

## Molecular Fragmentation

```python
from rdkit.Chem import BRICS, Recap, rdFMCS

# BRICS
fragments = BRICS.BRICSDecompose(mol)
new_mols = BRICS.BRICSBuild(fragments)

# Recap
hierarch = Recap.RecapDecompose(mol)
leaves = hierarch.GetLeaves()

# Maximum common substructure
mcs = rdFMCS.FindMCS([mol1, mol2, mol3], ringMatchesRingOnly=True)
print(mcs.smartsString)
```

## Serialization

```python
import pickle

# Pickle (fast)
pkl = pickle.dumps(mol)
mol2 = pickle.loads(pkl)

# Binary
bin_str = mol.ToBinary()
mol3 = Chem.Mol(bin_str)

# JSON
json_str = Chem.MolToJSON(mol)
mol = Chem.MolFromJSON(json_str)
```

## Performance Tips

1. **Use Suppliers for Large Files**: `SDMolSupplier` instead of loading all molecules
2. **Pickle Molecules**: Much faster than reparsing SMILES/SDF
3. **Remove Hydrogens**: Smaller molecules, faster operations
4. **Sanitize Once**: Don't sanitize multiple times
5. **Reuse Fingerprint Generators**: Create once, use many times
6. **Bulk Operations**: Use `BulkTanimotoSimilarity` for batch calculations

## Common Patterns

```python
# Virtual screening
query_fp = AllChem.GetMorganFingerprintAsBitVect(query, 2, 2048)
hits = []
for mol in Chem.SDMolSupplier('library.sdf'):
    if mol:
        fp = AllChem.GetMorganFingerprintAsBitVect(mol, 2, 2048)
        if DataStructs.TanimotoSimilarity(query_fp, fp) > 0.7:
            hits.append(mol)

# Lipinski's Rule of Five
def passes_lipinski(mol):
    return (Descriptors.MolWt(mol) <= 500 and Descriptors.MolLogP(mol) <= 5 and
            Descriptors.NumHDonors(mol) <= 5 and Descriptors.NumHAcceptors(mol) <= 10)

# PAINS filtering
pains = [Chem.MolFromSmarts(s) for s in ['C1=CC=CC=C1N=NC1=CC=CC=C1']]
clean = [m for m in mols if not any(m.HasSubstructMatch(p) for p in pains)]
```

## References

- **[Molecular I/O](references/molecular-io.md)** - SMILES, SDF, Mol files, suppliers, writers
- **[Descriptors and Fingerprints](references/descriptors-fingerprints.md)** - Molecular properties, fingerprints, similarity
- **[Substructure and Reactions](references/substructure-reactions.md)** - Pattern matching, SMARTS, chemical reactions
- **[3D Conformers](references/3d-conformers.md)** - Coordinate generation, conformer optimization, alignment
- **[Stereochemistry](references/stereochemistry.md)** - Chirality, CIP rules, stereo groups, atropisomers, enumeration
- **[Sanitization and Aromaticity](references/sanitization-aromaticity.md)** - Molecular sanitization, aromaticity models, kekulization
- **[Scaffold Analysis](references/scaffold-analysis.md)** - Murcko scaffolds, generic scaffolds, scaffold-based splits
- **[Advanced Features](references/advanced-features.md)** - Drawing, fragmentation, serialization, MCS
