# RDKit Molecular Sanitization and Aromaticity Reference

Sources:
- [The RDKit Book](https://www.rdkit.org/docs/RDKit_Book.html)
- [rdmolops API](https://www.rdkit.org/docs/source/rdkit.Chem.rdmolops.html)

## Molecular Sanitization

### Overview

Sanitization is the process of validating, correcting, and standardizing molecular structures. It includes kekulization, valence checking, aromaticity perception, and property calculation.

### Full Sanitization

```python
from rdkit import Chem

# Automatic sanitization (default)
mol = Chem.MolFromSmiles('c1ccccc1')  # Auto-sanitized

# Explicit sanitization
mol = Chem.MolFromSmiles('c1ccccc1', sanitize=False)
Chem.SanitizeMol(mol)

# Check sanitization status
try:
    Chem.SanitizeMol(mol)
    print("Sanitization successful")
except Exception as e:
    print(f"Sanitization failed: {e}")
```

### Sanitization Operations

```python
# Individual operations (flags)
Chem.SANITIZE_NONE                  # No sanitization
Chem.SANITIZE_CLEANUP              # Basic cleanup
Chem.SANITIZE_PROPERTIES           # Calculate properties
Chem.SANITIZE_SYMMRINGS            # Symmetrize rings
Chem.SANITIZE_KEKULIZE             # Kekulize aromatic rings
Chem.SANITIZE_FINDRADICALS         # Find radicals
Chem.SANITIZE_SETAROMATICITY       # Assign aromaticity
Chem.SANITIZE_SETCONJUGATION       # Find conjugated bonds
Chem.SANITIZE_SETHYBRIDIZATION     # Assign hybridization
Chem.SANITIZE_CLEANUPCHIRALITY     # Clean invalid chirality
Chem.SANITIZE_ADJUSTHS             # Adjust hydrogens
Chem.SANITIZE_SETVALENCE           # Set valence
Chem.SANITIZE_CLEANUPORGANOMETALLICS # Clean organometallics
Chem.SANITIZE_ALL                  # All operations

# Selective sanitization
mol = Chem.MolFromSmiles('c1ccccc1', sanitize=False)
Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_ALL^Chem.SANITIZE_KEKULIZE)
```

### Sanitization Steps

```python
# Step-by-step sanitization
mol = Chem.MolFromSmiles('c1ccccc1', sanitize=False)

# 1. Cleanup
Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_CLEANUP)

# 2. Set valence
Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_SETVALENCE)

# 3. Kekulize
Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_KEKULIZE)

# 4. Set aromaticity
Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_SETAROMATICITY)

# 5. Set hybridization
Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_SETHYBRIDIZATION)

# 6. Clean chirality
Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_CLEANUPCHIRALITY)
```

### Catching Specific Errors

```python
# Detailed error handling
mol = Chem.MolFromSmiles('invalid', sanitize=False)

try:
    Chem.SanitizeMol(mol)
except Chem.KekulizeException:
    print("Kekulization failed")
except Chem.AtomValenceException:
    print("Invalid valence")
except Chem.AtomKekulizeException:
    print("Atom kekulization failed")
except Exception as e:
    print(f"Sanitization error: {e}")
```

## Aromaticity Perception

### Aromaticity Models

```python
# Default (RDKit model)
Chem.SetAromaticity(mol)

# Simple model (5- and 6-membered rings only)
Chem.SetAromaticity(mol, Chem.AROMATICITY_SIMPLE)

# MDL model (carbon and nitrogen only, fused rings allowed)
Chem.SetAromaticity(mol, Chem.AROMATICITY_MDL)

# RDKit model (explicit)
Chem.SetAromaticity(mol, Chem.AROMATICITY_RDKIT)

# MMFF94 model
Chem.SetAromaticity(mol, Chem.AROMATICITY_MMFF)

# Custom model
Chem.SetAromaticity(mol, Chem.AROMATICITY_CUSTOM)
```

### RDKit Aromaticity Model

```python
# Default aromaticity rules:
# - Aromatic atoms in rings
# - Follows Hückel rule (4n+2 π electrons)
# - Contributions vary by atom type:
#   - C, N, O, S, Se: standard contributions
#   - P, As: special cases

mol = Chem.MolFromSmiles('c1ccccc1')  # Benzene
for atom in mol.GetAtoms():
    if atom.GetIsAromatic():
        print(f"Atom {atom.GetIdx()} is aromatic")

for bond in mol.GetBonds():
    if bond.GetIsAromatic():
        print(f"Bond {bond.GetIdx()} is aromatic")
```

### Simple Aromaticity Model

```python
# Only 5- and 6-membered rings
# No fused rings
mol = Chem.MolFromSmiles('c1ccc2ccccc2c1', sanitize=False)  # Naphthalene
Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_ALL^Chem.SANITIZE_SETAROMATICITY)
Chem.SetAromaticity(mol, Chem.AROMATICITY_SIMPLE)

# Check aromaticity
for atom in mol.GetAtoms():
    print(f"Atom {atom.GetIdx()}: {atom.GetIsAromatic()}")
```

### MDL Aromaticity Model

```python
# Carbon and nitrogen only
# Fused rings allowed
# Different π-electron counting

mol = Chem.MolFromSmiles('c1ccccc1', sanitize=False)
Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_ALL^Chem.SANITIZE_SETAROMATICITY)
Chem.SetAromaticity(mol, Chem.AROMATICITY_MDL)
```

## Kekulization

### Kekulize Molecules

```python
# Convert aromatic to Kekule form
mol = Chem.MolFromSmiles('c1ccccc1')  # Aromatic benzene

Chem.Kekulize(mol)  # Convert to alternating single/double bonds

smiles = Chem.MolToSmiles(mol, kekuleSmiles=True)
print(smiles)  # 'C1=CC=CC=C1'

# Back to aromatic
Chem.SanitizeMol(mol)  # Restores aromaticity
```

### Kekulization Options

```python
# Clear aromatic flags
Chem.Kekulize(mol, clearAromaticFlags=True)

# Handle failures
try:
    Chem.Kekulize(mol)
except Chem.KekulizeException:
    print("Cannot kekulize this molecule")
```

## Valence and Hybridization

### Valence Checking

```python
# Check atom valences
for atom in mol.GetAtoms():
    valence = atom.GetTotalValence()
    expected = atom.GetDefaultValence()
    print(f"Atom {atom.GetSymbol()}: {valence} (expected {expected})")

# Explicit valence
explicit = atom.GetExplicitValence()
implicit = atom.GetImplicitValence()
total = atom.GetTotalValence()
```

### Hybridization

```python
# Get hybridization
for atom in mol.GetAtoms():
    hyb = atom.GetHybridization()
    # SP, SP2, SP3, SP3D, SP3D2, UNSPECIFIED
    print(f"Atom {atom.GetIdx()}: {hyb}")

# Set hybridization manually
atom.SetHybridization(Chem.HybridizationType.SP2)
```

## Radical Detection

```python
# Find radicals
Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_FINDRADICALS)

# Check for radicals
for atom in mol.GetAtoms():
    num_radical = atom.GetNumRadicalElectrons()
    if num_radical > 0:
        print(f"Atom {atom.GetIdx()} has {num_radical} radical electrons")

# Set radical electrons
atom.SetNumRadicalElectrons(1)  # Radical
```

## Ring Perception

### Ring Systems

```python
# Get ring info
ring_info = mol.GetRingInfo()

# Number of rings
num_rings = ring_info.NumRings()

# Atom rings (tuple of tuples)
atom_rings = ring_info.AtomRings()

# Bond rings
bond_rings = ring_info.BondRings()

# Check if atom in ring
is_in_ring = ring_info.IsAtomInRingOfSize(atom_idx, 6)

# Smallest set of smallest rings
from rdkit.Chem import GetSymmSSSR
ssr = GetSymmSSSR(mol)
```

### Ring Properties

```python
# Aromatic rings
for ring in atom_rings:
    atoms = [mol.GetAtomWithIdx(i) for i in ring]
    if all(atom.GetIsAromatic() for atom in atoms):
        print(f"Ring {ring} is aromatic")

# Ring size distribution
from collections import Counter
ring_sizes = Counter(len(ring) for ring in atom_rings)
print(ring_sizes)
```

## Hydrogen Handling

### Explicit vs Implicit

```python
# Add explicit hydrogens
mol = Chem.AddHs(mol)

# Remove explicit hydrogens
mol = Chem.RemoveHs(mol)

# Remove only implicit
mol = Chem.RemoveAllHs(mol)

# Add only to polar atoms
mol = Chem.AddHs(mol, onlyOnAtoms=[1, 3, 5])

# Check hydrogen count
for atom in mol.GetAtoms():
    explicit = atom.GetNumExplicitHs()
    implicit = atom.GetNumImplicitHs()
    total = atom.GetTotalNumHs()
    print(f"{atom.GetSymbol()}: {explicit} explicit, {implicit} implicit")
```

## Conjugation

```python
# Set conjugation
Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_SETCONJUGATION)

# Check conjugated bonds
for bond in mol.GetBonds():
    if bond.GetIsConjugated():
        print(f"Bond {bond.GetIdx()} is conjugated")
```

## Custom Sanitization

```python
def custom_sanitize(mol):
    """Custom sanitization routine"""
    # 1. Basic cleanup
    Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_CLEANUP)
    
    # 2. Set valence
    Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_SETVALENCE)
    
    # 3. Custom aromaticity
    Chem.SetAromaticity(mol, Chem.AROMATICITY_MDL)
    
    # 4. Hybridization
    Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_SETHYBRIDIZATION)
    
    # 5. Properties
    Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_PROPERTIES)
    
    return mol

# Use custom sanitization
mol = Chem.MolFromSmiles('c1ccccc1', sanitize=False)
mol = custom_sanitize(mol)
```

## Best Practices

1. **Sanitize after reading**: Always sanitize molecules from external sources
2. **Handle failures**: Catch sanitization exceptions
3. **Choose aromaticity model**: Pick appropriate model for use case
4. **Kekulize when needed**: For writing certain formats
5. **Validate valence**: Check for chemically valid molecules
6. **Preserve stereochemistry**: Use `cleanupChirality` carefully
7. **Hydrogen management**: Add/remove H consistently

## Common Pitfalls

- **Over-sanitizing**: Sanitizing multiple times
- **Wrong aromaticity model**: MDL vs RDKit differences
- **Failed kekulization**: Not all aromatic systems kekulize
- **Missing sanitization**: Reading with `sanitize=False` and forgetting
- **Valence errors**: Invalid atom valences
- **Radical mishandling**: Not detecting radicals properly
- **Ring perception**: Assuming SSSR is unique
