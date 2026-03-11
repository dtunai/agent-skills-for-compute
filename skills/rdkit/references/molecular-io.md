# RDKit Molecular I/O Reference

Sources:
- [Getting Started in Python](https://www.rdkit.org/docs/GettingStartedInPython.html)
- [The RDKit Book](https://www.rdkit.org/docs/RDKit_Book.html)

## Overview

RDKit supports multiple molecular file formats and provides efficient readers/writers for both single molecules and large datasets.

**Supported Formats**:
- SMILES (Simplified Molecular Input Line Entry System)
- SMARTS (SMiles ARbitrary Target Specification)
- Mol/SDF (MDL Molfile/Structure Data File)
- Mol2 (Tripos Mol2)
- PDB (Protein Data Bank)
- XYZ coordinates
- InChI (IUPAC International Chemical Identifier)

## Reading Single Molecules

### From SMILES

```python
from rdkit import Chem

# Basic SMILES
mol = Chem.MolFromSmiles('c1ccccc1')  # Benzene
mol = Chem.MolFromSmiles('CCO')        # Ethanol
mol = Chem.MolFromSmiles('CC(=O)O')    # Acetic acid

# Check for failure
if mol is None:
    print("Failed to parse SMILES")

# Suppress sanitization
mol = Chem.MolFromSmiles('c1ccccc1', sanitize=False)
```

### From SMARTS

```python
# SMARTS pattern
pattern = Chem.MolFromSmarts('[OH]')    # Hydroxyl
pattern = Chem.MolFromSmarts('c1ccccc1') # Aromatic 6-ring
pattern = Chem.MolFromSmarts('[$(C=O)]') # Carbonyl

# SMARTS are used for substructure searching, not molecule representation
```

### From Mol Files

```python
# Read from file
mol = Chem.MolFromMolFile('molecule.mol')

# Suppress sanitization
mol = Chem.MolFromMolFile('molecule.mol', sanitize=False)

# Remove hydrogens
mol = Chem.MolFromMolFile('molecule.mol', removeHs=True)

# Strict parsing
mol = Chem.MolFromMolFile('molecule.mol', strictParsing=True)
```

### From Mol Block (String)

```python
# Mol block as string
mol_block = """
  Mrv0541 02251512282D

  4  3  0  0  0  0            999 V2000
   -0.7145    0.2063    0.0000 C   0  0  0  0  0  0  0  0  0  0  0  0
   -0.0000   -0.2062    0.0000 C   0  0  0  0  0  0  0  0  0  0  0  0
    0.7145    0.2063    0.0000 O   0  0  0  0  0  0  0  0  0  0  0  0
    0.0000   -1.0312    0.0000 O   0  0  0  0  0  0  0  0  0  0  0  0
  1  2  1  0  0  0  0
  2  3  1  0  0  0  0
  2  4  2  0  0  0  0
M  END
"""

mol = Chem.MolFromMolBlock(mol_block)
```

### From Other Formats

```python
# Mol2
mol = Chem.MolFromMol2File('molecule.mol2')
mol = Chem.MolFromMol2Block(mol2_block)

# PDB
mol = Chem.MolFromPDBFile('protein.pdb')
mol = Chem.MolFromPDBBlock(pdb_block)

# XYZ
mol = Chem.MolFromXYZFile('coords.xyz')
mol = Chem.MolFromXYZBlock(xyz_block)

# InChI
mol = Chem.MolFromInchi('InChI=1S/C6H6/c1-2-4-6-5-3-1/h1-6H')
```

## Writing Single Molecules

### To SMILES

```python
# Canonical SMILES
smiles = Chem.MolToSmiles(mol)

# Isomeric SMILES (includes stereochemistry)
smiles = Chem.MolToSmiles(mol, isomericSmiles=True)

# Non-isomeric
smiles = Chem.MolToSmiles(mol, isomericSmiles=False)

# Kekule SMILES
Chem.Kekulize(mol)
smiles = Chem.MolToSmiles(mol, kekuleSmiles=True)

# Canonical atom order
smiles = Chem.MolToSmiles(mol, canonical=True)

# CXSMILES (extended SMILES with coordinates)
smiles = Chem.MolToCXSmiles(mol)
```

### To Mol Block

```python
# V2000 format (default)
mol_block = Chem.MolToMolBlock(mol)

# V3000 format
mol_block = Chem.MolToV3KMolBlock(mol)

# With 2D coordinates
from rdkit.Chem import AllChem
AllChem.Compute2DCoords(mol)
mol_block = Chem.MolToMolBlock(mol)

# With 3D coordinates
AllChem.EmbedMolecule(mol)
mol_block = Chem.MolToMolBlock(mol)
```

### To Mol File

```python
# Write to file
Chem.MolToMolFile(mol, 'output.mol')

# V3000 format
Chem.MolToMolFile(mol, 'output.mol', forceV3000=True)

# Include properties
mol.SetProp('Activity', '10.5')
Chem.MolToMolFile(mol, 'output.mol')
```

### To Other Formats

```python
# PDB
pdb = Chem.MolToPDBBlock(mol)
Chem.MolToPDBFile(mol, 'output.pdb')

# XYZ
xyz = Chem.MolToXYZBlock(mol)
Chem.MolToXYZFile(mol, 'output.xyz')

# InChI
inchi = Chem.MolToInchi(mol)
inchi_key = Chem.MolToInchiKey(mol)
```

## Reading Multiple Molecules

### SDF Supplier

```python
# Basic usage
suppl = Chem.SDMolSupplier('molecules.sdf')

for mol in suppl:
    if mol is None:
        continue
    print(Chem.MolToSmiles(mol))

# With options
suppl = Chem.SDMolSupplier(
    'molecules.sdf',
    sanitize=True,
    removeHs=True,
    strictParsing=False
)

# Access by index
mol = suppl[0]
mol = suppl[10]

# Number of molecules
print(len(suppl))
```

### Forward SDF Supplier

For streaming large files or compressed files:

```python
import gzip

# Compressed SDF
with gzip.open('molecules.sdf.gz', 'rb') as f:
    with Chem.ForwardSDMolSupplier(f) as suppl:
        for mol in suppl:
            if mol is None:
                continue
            process(mol)

# File-like objects
with open('molecules.sdf', 'rb') as f:
    suppl = Chem.ForwardSDMolSupplier(f)
    mols = [m for m in suppl if m is not None]
```

### SMILES Supplier

```python
# From file
with Chem.SmilesMolSupplier('molecules.smi') as suppl:
    for mol in suppl:
        if mol is None:
            continue
        print(mol.GetNumAtoms())

# With delimiter and title line
suppl = Chem.SmilesMolSupplier(
    'molecules.smi',
    delimiter=' ',
    titleLine=True
)

# From stream
with Chem.SmilesMolSupplier('molecules.smi', titleLine=False) as suppl:
    mols = [m for m in suppl if m is not None]
```

### Mol2 and PDB Suppliers

```python
# Mol2
suppl = Chem.MolFromMol2File('molecules.mol2')

# PDB (single protein)
mol = Chem.MolFromPDBFile('protein.pdb')
```

## Writing Multiple Molecules

### SDF Writer

```python
# Basic usage
with Chem.SDWriter('output.sdf') as w:
    for mol in mols:
        w.write(mol)

# With properties
with Chem.SDWriter('output.sdf') as w:
    for mol in mols:
        mol.SetProp('ID', str(i))
        mol.SetProp('Activity', str(activity))
        w.write(mol)

# V3000 format
with Chem.SDWriter('output.sdf') as w:
    w.SetForceV3000(True)
    for mol in mols:
        w.write(mol)
```

### SMILES Writer

```python
# Basic usage
with Chem.SmilesWriter('output.smi') as w:
    for mol in mols:
        w.write(mol)

# With name column
with Chem.SmilesWriter('output.smi', includeHeader=True) as w:
    for mol in mols:
        w.write(mol)

# Isomeric SMILES
with Chem.SmilesWriter('output.smi', isomericSmiles=True) as w:
    for mol in mols:
        w.write(mol)

# Kekule SMILES
with Chem.SmilesWriter('output.smi', kekuleSmiles=True) as w:
    for mol in mols:
        w.write(mol)
```

### TDT Writer

```python
# TDT format (Thor Data Tree)
with Chem.TDTWriter('output.tdt') as w:
    for mol in mols:
        w.write(mol)
```

## Property Handling

### Reading Properties from SDF

```python
suppl = Chem.SDMolSupplier('molecules.sdf')
for mol in suppl:
    if mol is None:
        continue

    # Get all properties
    props = mol.GetPropsAsDict()
    print(props)

    # Get specific property
    if mol.HasProp('Activity'):
        activity = mol.GetProp('Activity')
        print(activity)

    # Get numeric property
    if mol.HasProp('MW'):
        mw = float(mol.GetProp('MW'))
```

### Writing Properties to SDF

```python
with Chem.SDWriter('output.sdf') as w:
    for mol in mols:
        # Set string property
        mol.SetProp('ID', 'MOL_001')

        # Set numeric property (as string)
        mol.SetProp('Activity', '10.5')
        mol.SetProp('MW', str(Descriptors.MolWt(mol)))

        # Set integer property
        mol.SetIntProp('NumAtoms', mol.GetNumAtoms())

        # Set double property
        mol.SetDoubleProp('LogP', Descriptors.MolLogP(mol))

        # Set boolean property
        mol.SetBoolProp('Active', True)

        w.write(mol)
```

## Error Handling

```python
# Check for parsing errors
mol = Chem.MolFromSmiles('invalid_smiles')
if mol is None:
    print("Failed to parse")

# Supplier errors
suppl = Chem.SDMolSupplier('molecules.sdf')
for i, mol in enumerate(suppl):
    if mol is None:
        print(f"Failed to parse molecule at index {i}")
        continue
    process(mol)

# Get error messages
Chem.WrapLogs()  # Capture error messages
mol = Chem.MolFromSmiles('invalid')
if mol is None:
    errors = Chem.ErrorLogger.GetLog()
    print(errors)
```

## Performance Optimization

### Lazy Loading

```python
# Use ForwardSDMolSupplier for streaming
with gzip.open('large.sdf.gz', 'rb') as f:
    suppl = Chem.ForwardSDMolSupplier(f)
    for mol in suppl:
        if mol:
            process(mol)
```

### Parallel Processing

```python
from multiprocessing import Pool

def process_mol(smiles):
    mol = Chem.MolFromSmiles(smiles)
    if mol:
        return Descriptors.MolWt(mol)
    return None

# Parallel processing
with open('smiles.txt') as f:
    smiles_list = [line.strip() for line in f]

with Pool(4) as pool:
    results = pool.map(process_mol, smiles_list)
```

### Memory-Efficient Reading

```python
# Process in batches
batch_size = 1000
batch = []

suppl = Chem.SDMolSupplier('large.sdf')
for i, mol in enumerate(suppl):
    if mol is None:
        continue

    batch.append(mol)

    if len(batch) >= batch_size:
        process_batch(batch)
        batch = []

# Process remaining
if batch:
    process_batch(batch)
```

## Best Practices

1. **Check for None**: Always check if `mol is None` after reading
2. **Use Suppliers**: For large datasets, use suppliers instead of loading all molecules
3. **Context Managers**: Use `with` statements for writers to ensure proper closure
4. **Sanitization**: Control sanitization based on needs (faster without, safer with)
5. **Compression**: Use gzip for large SDF files
6. **Properties**: Use typed property setters (`SetDoubleProp`, `SetIntProp`) for clarity
7. **Streaming**: Use `ForwardSDMolSupplier` for very large files
8. **Error Logging**: Enable error logging for debugging parsing issues

## Common Pitfalls

- **Not checking for None**: Leads to crashes on invalid molecules
- **Loading entire large files**: Use suppliers for memory efficiency
- **Forgetting to close writers**: Use context managers
- **Over-sanitizing**: Sanitizing multiple times is redundant
- **Wrong format assumptions**: SDF != SMILES, check file format
- **Missing coordinates**: Some operations require 2D/3D coords
- **Property types**: SDF properties are strings, cast appropriately
- **Encoding issues**: Specify encoding for non-ASCII characters
