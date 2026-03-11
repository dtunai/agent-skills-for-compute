# RDKit Substructure Searching and Chemical Reactions Reference

Sources:
- [Getting Started in Python](https://www.rdkit.org/docs/GettingStartedInPython.html)
- [The RDKit Book](https://www.rdkit.org/docs/RDKit_Book.html)

## Substructure Searching

### SMARTS Patterns

```python
from rdkit import Chem

# Define patterns
benzene = Chem.MolFromSmarts('c1ccccc1')
phenol = Chem.MolFromSmarts('c1ccccc1O')
carboxylic_acid = Chem.MolFromSmarts('C(=O)[OH]')
amine = Chem.MolFromSmarts('[N;H1,H2]')
ester = Chem.MolFromSmarts('C(=O)O[C,c]')

# Test molecule
mol = Chem.MolFromSmiles('CC(=O)Oc1ccccc1C(=O)O')  # Aspirin

# Check for match
has_benzene = mol.HasSubstructMatch(benzene)  # True
has_acid = mol.HasSubstructMatch(carboxylic_acid)  # True
```

### Getting Matches

```python
# Get first match (tuple of atom indices)
match = mol.GetSubstructMatch(benzene)
print(match)  # (4, 5, 6, 7, 8, 9) for aspirin

# Get all matches
matches = mol.GetSubstructMatches(benzene)
print(len(matches))  # Number of benzene rings

# Unique matches only
matches = mol.GetSubstructMatches(benzene, uniquify=True)

# Maximum matches
matches = mol.GetSubstructMatches(benzene, maxMatches=10)
```

### Chirality and Stereochemistry

```python
# Chiral pattern
chiral_pattern = Chem.MolFromSmarts('[C@H](N)(C)C(=O)O')  # L-alanine

# Match with chirality
mol.HasSubstructMatch(chiral_pattern, useChirality=True)

# Get matches respecting chirality
matches = mol.GetSubstructMatches(chiral_pattern, useChirality=True)
```

### Advanced SMARTS

```python
# Recursive SMARTS
quaternary_N = Chem.MolFromSmarts('[N;X4;+]')  # Quaternary nitrogen
tertiary_amine = Chem.MolFromSmarts('[N;H0;X3;!$(NC=O)]')  # Not amide

# Ring patterns
5_ring = Chem.MolFromSmarts('[r5]')  # Any 5-membered ring
aromatic_6_ring = Chem.MolFromSmarts('[a;r6]')  # Aromatic 6-ring

# Functional groups with environments
primary_alcohol = Chem.MolFromSmarts('[CH2;X4][OH]')
secondary_alcohol = Chem.MolFromSmarts('[CH;X4][OH]')
tertiary_alcohol = Chem.MolFromSmarts('[C;X4;H0][OH]')

# Heteroatoms
any_heteroatom = Chem.MolFromSmarts('[!#6;!#1]')  # Not C or H
```

## Functional Group Filters

### Common Functional Groups

```python
# Define patterns
functional_groups = {
    'carboxylic_acid': Chem.MolFromSmarts('C(=O)[OH]'),
    'ester': Chem.MolFromSmarts('C(=O)O[C,c]'),
    'amide': Chem.MolFromSmarts('C(=O)N'),
    'ketone': Chem.MolFromSmarts('[C;!$(C=O[OH])]=O'),
    'aldehyde': Chem.MolFromSmarts('[CH]=O'),
    'alcohol': Chem.MolFromSmarts('[C;X4][OH]'),
    'phenol': Chem.MolFromSmarts('c[OH]'),
    'primary_amine': Chem.MolFromSmarts('[N;H2;X3]'),
    'secondary_amine': Chem.MolFromSmarts('[N;H1;X3;!$(NC=O)]'),
    'tertiary_amine': Chem.MolFromSmarts('[N;H0;X3;!$(NC=O)]'),
    'nitrile': Chem.MolFromSmarts('C#N'),
    'nitro': Chem.MolFromSmarts('[N+](=O)[O-]'),
    'sulfone': Chem.MolFromSmarts('S(=O)(=O)'),
    'halide': Chem.MolFromSmarts('[F,Cl,Br,I]'),
}

# Check molecule
def get_functional_groups(mol):
    found = []
    for name, pattern in functional_groups.items():
        if mol.HasSubstructMatch(pattern):
            found.append(name)
    return found

# Usage
mol = Chem.MolFromSmiles('CC(=O)Oc1ccccc1C(=O)O')
groups = get_functional_groups(mol)
print(groups)  # ['carboxylic_acid', 'ester']
```

### PAINS Filters

```python
# Pan-Assay Interference Compounds
pains_patterns = [
    'C1=CC=CC=C1N=NC1=CC=CC=C1',  # Azo compounds
    'C1=CC(=O)C=CC1',               # Quinone
    'C1=CC=C(C=C1)S(=O)(=O)N',     # Sulfonamide
    '[#6]1:[#6]:[#6]:[#7]:[#7]:1', # Diazo
]

pains_mols = [Chem.MolFromSmarts(p) for p in pains_patterns]

def has_pains(mol):
    return any(mol.HasSubstructMatch(p) for p in pains_mols)

# Filter molecules
clean_mols = [m for m in mols if not has_pains(m)]
```

## Chemical Reactions

### Reaction SMARTS

```python
from rdkit.Chem import AllChem

# Amide bond formation
rxn = AllChem.ReactionFromSmarts(
    '[C:1](=[O:2])-[OD1].[N!H0:3]>>[C:1](=[O:2])[N:3]'
)

# Reactants
acid = Chem.MolFromSmiles('CC(=O)O')
amine = Chem.MolFromSmiles('CN')

# Run reaction
products = rxn.RunReactants((acid, amine))

# Get products (list of tuples)
for product_set in products:
    for product in product_set:
        print(Chem.MolToSmiles(product))
```

### Common Reactions

```python
# Esterification
ester_rxn = AllChem.ReactionFromSmarts(
    '[C:1](=[O:2])-[OD1].[O:3][C:4]>>[C:1](=[O:2])[O:3][C:4]'
)

# Amidation
amide_rxn = AllChem.ReactionFromSmarts(
    '[C:1](=[O:2])-[OD1].[N!H0:3]>>[C:1](=[O:2])[N:3]'
)

# Sulfonamide formation
sulfonamide_rxn = AllChem.ReactionFromSmarts(
    '[S:1](=[O:2])(=[O:3])[Cl].[N!H0:4]>>[S:1](=[O:2])(=[O:3])[N:4]'
)

# Urea formation
urea_rxn = AllChem.ReactionFromSmarts(
    '[N:1][C:2]=O.[N:3]>>[N:1][C:2](=[O])[N:3]'
)

# Reductive amination
reductive_amination = AllChem.ReactionFromSmarts(
    '[C:1]=[O:2].[N:3]>>[C:1][N:3]'
)
```

### Loading from RXN Files

```python
# From file
rxn = AllChem.ReactionFromRxnFile('reaction.rxn')

# From block
rxn_block = """$RXN

      RDKit

  2  1
$MOL
...
"""
rxn = AllChem.ReactionFromRxnBlock(rxn_block)

# Validate reaction
rxn.Initialize()
if rxn.Validate() != (0, 0):
    print("Reaction validation failed")
```

### Reaction Properties

```python
# Number of reactants and products
num_reactants = rxn.GetNumReactantTemplates()
num_products = rxn.GetNumProductTemplates()

# Get reactant/product templates
reactant_template = rxn.GetReactantTemplate(0)
product_template = rxn.GetProductTemplate(0)

# Run with sanitization
products = rxn.RunReactants((acid, amine), sanitize=True)

# Multiple reactant sets
reactant_sets = [(acid1, amine1), (acid2, amine2)]
for reactants in reactant_sets:
    products = rxn.RunReactants(reactants)
```

### Protecting Groups

```python
# Protect specific atoms
amide_pattern = Chem.MolFromSmarts('[N;$(NC=[O,S])]')

for match in mol.GetSubstructMatches(amide_pattern):
    mol.GetAtomWithIdx(match[0]).SetProp('_protected', '1')

# Run reaction (respects _protected property)
products = rxn.RunReactants((mol,))

# Clear protection
for atom in mol.GetAtoms():
    if atom.HasProp('_protected'):
        atom.ClearProp('_protected')
```

### Enumerating Products

```python
# Combinatorial library
acids = [Chem.MolFromSmiles(s) for s in ['CC(=O)O', 'CCC(=O)O', 'c1ccccc1C(=O)O']]
amines = [Chem.MolFromSmiles(s) for s in ['CN', 'CCN', 'c1cccnc1']]

rxn = AllChem.ReactionFromSmarts('[C:1](=[O:2])-[OD1].[N!H0:3]>>[C:1](=[O:2])[N:3]')

# Generate all products
library = []
for acid in acids:
    for amine in amines:
        products = rxn.RunReactants((acid, amine))
        if products:
            library.append(products[0][0])

print(f"Generated {len(library)} compounds")
```

## Substructure Highlighting

### Highlight Matches in Drawings

```python
from rdkit.Chem import Draw

# Find matches
pattern = Chem.MolFromSmarts('c1ccccc1')
matches = mol.GetSubstructMatches(pattern)

# Highlight all matches
highlight_atoms = []
for match in matches:
    highlight_atoms.extend(match)

# Draw with highlights
img = Draw.MolToImage(mol, highlightAtoms=highlight_atoms)

# With colors
from rdkit.Chem.Draw import rdMolDraw2D

d = rdMolDraw2D.MolDraw2DSVG(500, 500)
rdMolDraw2D.PrepareAndDrawMolecule(
    d, mol,
    highlightAtoms=list(matches[0]),
    highlightAtomColors={idx: (1, 0, 0) for idx in matches[0]}
)
```

## Advanced Substructure Operations

### Recursive Queries

```python
# Atoms in rings
in_ring = Chem.MolFromSmarts('[r]')

# Atoms not in rings
not_in_ring = Chem.MolFromSmarts('[!r]')

# Aromatic atoms
aromatic = Chem.MolFromSmarts('[a]')

# Heteroatoms in aromatic rings
aromatic_hetero = Chem.MolFromSmarts('[a;!#6]')
```

### Generic Queries

```python
from rdkit.Chem import AllChem

# Enable generic matchers
params = Chem.SubstructMatchParameters()
params.useGenericMatchers = True

# Generic groups
alkyl = Chem.MolFromSmarts('[G:ALK]')  # Alkyl
aryl = Chem.MolFromSmarts('[G:ARY]')   # Aryl

mol.GetSubstructMatches(alkyl, params)
```

### Atom Mapping

```python
# Create reaction with atom mapping
rxn = AllChem.ReactionFromSmarts(
    '[C:1](=[O:2])-[O:3].[N:4]>>[C:1](=[O:2])[N:4].[O:3]'
)

# Map numbers preserved in products
products = rxn.RunReactants((acid, amine))
product = products[0][0]

# Get mapped atoms
for atom in product.GetAtoms():
    if atom.HasProp('molAtomMapNumber'):
        map_num = atom.GetIntProp('molAtomMapNumber')
        print(f"Atom {atom.GetIdx()} has map number {map_num}")
```

## Performance Optimization

```python
# Compile patterns once
patterns = {
    'benzene': Chem.MolFromSmarts('c1ccccc1'),
    'acid': Chem.MolFromSmarts('C(=O)[OH]'),
    'ester': Chem.MolFromSmarts('C(=O)O[C,c]'),
}

# Reuse for multiple molecules
for mol in mols:
    for name, pattern in patterns.items():
        if mol.HasSubstructMatch(pattern):
            print(f"{name} found in molecule")

# Parallel substructure search
from multiprocessing import Pool

def has_pattern(mol_smiles):
    mol = Chem.MolFromSmiles(mol_smiles)
    return mol.HasSubstructMatch(pattern) if mol else False

with Pool(4) as pool:
    results = pool.map(has_pattern, smiles_list)
```

## Best Practices

1. **Compile SMARTS once**: Reuse pattern molecules
2. **Use HasSubstructMatch**: Faster than GetSubstructMatches for boolean checks
3. **Sanitize products**: Set sanitize=True for reactions
4. **Validate reactions**: Call Initialize() and Validate()
5. **Protect groups**: Use _protected property to exclude atoms
6. **Test patterns**: Verify SMARTS on known molecules first

## Common Pitfalls

- **Invalid SMARTS**: Always check pattern parsing
- **Missing sanitization**: Products may be invalid without sanitize=True
- **Overlapping matches**: Use uniquify=True if needed
- **Incorrect chirality**: Set useChirality appropriately
- **Reaction validation**: Some reactions may be chemically impossible
- **Atom mapping**: Preserve mapping for tracking transformations
- **Multiple products**: Reactions can produce multiple product sets
