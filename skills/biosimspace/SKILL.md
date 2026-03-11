---
name: biosimspace
description: "BioSimSpace — engine-agnostic Python framework for biomolecular simulation. Portable workflows across AMBER, GROMACS, NAMD, OpenMM. Parameterisation, solvation, minimisation, equilibration, production MD, free energy perturbation, metadynamics."
license: MIT
metadata:
  author: Agent Cluster
  tags: [biosimspace, molecular-dynamics, amber, gromacs, namd, openmm, free-energy, fep, metadynamics, solvation, parameterisation]
---

# BioSimSpace

Engine-agnostic Python framework for biomolecular simulation. Write portable workflows once — BioSimSpace translates to AMBER, GROMACS, NAMD, OpenMM, or SOMD at runtime.

**Official Sources:**
- [GitHub](https://github.com/OpenBioSim/biosimspace)
- [Documentation](https://biosimspace.openbiosim.org/)
- [Tutorials](https://biosimspace.openbiosim.org/tutorials)
- [API Reference](https://biosimspace.openbiosim.org/api)

## Quick Start

```bash
# Install via conda (recommended)
conda create -n openbiosim -c conda-forge -c openbiosim biosimspace
conda activate openbiosim

# Optional: install MD engines
conda install -c conda-forge ambertools gromacs
```

```python
import BioSimSpace as BSS
```

## Architecture

| Module | Purpose |
|--------|---------|
| `BSS.IO` | Read/write molecules (PDB, MOL2, GRO, PRM7, RST7, SDF, DCD...) |
| `BSS.Parameters` | Parameterise with force fields (ff14SB, GAFF, GAFF2, OpenFF...) |
| `BSS.Solvent` | Solvate in water boxes (TIP3P, SPC/E, TIP4P, OPC...) |
| `BSS.Protocol` | Engine-agnostic simulation protocols |
| `BSS.Process` | Engine-specific process drivers (Amber, Gromacs, Namd, OpenMM, Somd) |
| `BSS.MD` | Auto-select engine and run |
| `BSS.FreeEnergy` | Relative FEP and ATM (alchemical transfer method) |
| `BSS.Align` | Atom mapping, alignment, merging for FEP |
| `BSS.Metadynamics` | Enhanced sampling via PLUMED |
| `BSS.Convert` | Convert between BSS, RDKit, OpenMM, Sire representations |
| `BSS.Box` | Simulation box generators (cubic, rhombic dodecahedron, truncated octahedron) |
| `BSS.Types` | Physical quantities (Length, Temperature, Time, Energy, Pressure...) |
| `BSS.Units` | Unit constants (`BSS.Units.Length.nanometer`, etc.) |
| `BSS.Gateway` | Node I/O for reusable workflow components |
| `BSS.Notebook` | Visualization (plots, 3D views via NGLView) |
| `BSS.Trajectory` | Trajectory analysis wrapper |
| `BSS.Stream` | Serialization/checkpointing |

## Supported Engines

| Engine | Process Class | Detection |
|--------|--------------|-----------|
| AMBER | `BSS.Process.Amber` | `AMBERHOME` env var |
| GROMACS | `BSS.Process.Gromacs` | `gmx`/`gmx_mpi` in PATH or `GROMACSHOME` |
| NAMD | `BSS.Process.Namd` | `namd2`/`namd3` in PATH |
| OpenMM | `BSS.Process.OpenMM` | Python import |
| SOMD | `BSS.Process.Somd` | Part of Sire/OpenBioSim stack |

Check available: `BSS.MD.engines()`

## Core Workflow

### 1. Load Molecules

```python
# From structure files
system = BSS.IO.readMolecules(["protein.top", "protein.crd"])
system = BSS.IO.readMolecules(["complex.gro", "complex.top"])
system = BSS.IO.readMolecules("molecule.pdb")

# Access molecules
molecule = system[0]          # First molecule
n_molecules = system.nMolecules()
```

### 2. Parameterise

```python
# Protein force fields
molecule = BSS.Parameters.ff14SB(molecule).getMolecule()
molecule = BSS.Parameters.ff99SBildn(molecule).getMolecule()

# Small molecule force fields
molecule = BSS.Parameters.gaff2(molecule).getMolecule()

# From SMILES
molecule = BSS.Parameters.gaff("Cc1ccccc1").getMolecule()

# OpenForceField
molecule = BSS.Parameters.openff_unconstrained_2_0_0(molecule).getMolecule()

# Generic (pass FF name as string)
molecule = BSS.Parameters.parameterise(molecule, "ff14SB").getMolecule()
```

### 3. Solvate

```python
# Cubic box with explicit dimensions
solvated = BSS.Solvent.tip3p(
    molecule=molecule,
    box=3 * [5 * BSS.Units.Length.nanometer],
    ion_conc=0.15,  # 150 mM NaCl
)

# Shell-based padding
solvated = BSS.Solvent.spce(
    molecule=molecule,
    shell=1.5 * BSS.Units.Length.nanometer,
)

# Available water models
BSS.Solvent.waterModels()  # ['spc', 'spce', 'tip3p', 'tip4p', 'tip5p', 'opc']
```

### 4. Minimisation

```python
protocol = BSS.Protocol.Minimisation(steps=5000)

# Auto-select engine
process = BSS.MD.run(solvated, protocol)
minimised = process.getSystem(block=True)

# Or use specific engine
process = BSS.Process.Gromacs(solvated, protocol)
process.start()
process.wait()
minimised = process.getSystem()
```

### 5. Equilibration

```python
# NVT heating with backbone restraints
protocol = BSS.Protocol.Equilibration(
    runtime=0.5 * BSS.Units.Time.nanosecond,
    temperature_start=0 * BSS.Units.Temperature.kelvin,
    temperature_end=300 * BSS.Units.Temperature.kelvin,
    restraint="backbone",
)
process = BSS.Process.Amber(minimised, protocol)
process.start()
process.wait()
equilibrated = process.getSystem()

# NPT (constant pressure)
protocol = BSS.Protocol.Equilibration(
    runtime=1 * BSS.Units.Time.nanosecond,
    temperature=300 * BSS.Units.Temperature.kelvin,
    pressure=1 * BSS.Units.Pressure.atm,
    restraint="heavy",
)
```

### 6. Production MD

```python
protocol = BSS.Protocol.Production(
    runtime=10 * BSS.Units.Time.nanosecond,
    temperature=300 * BSS.Units.Temperature.kelvin,
    pressure=1 * BSS.Units.Pressure.atm,
    report_interval=5000,
    restart_interval=50000,
)
process = BSS.Process.OpenMM(equilibrated, protocol)
process.start()

# Monitor while running
process.isRunning()
process.getTime()
process.getTotalEnergy()
BSS.Notebook.plot(process.getTime(), process.getTotalEnergy())

# Get final system
process.wait()
final = process.getSystem()
```

### 7. Save Results

```python
# Save in original format
BSS.IO.saveMolecules("output", final, system.fileFormat())

# Save in specific format
BSS.IO.saveMolecules("output", final, ["prm7", "rst7"])
BSS.IO.saveMolecules("output", final, ["gro87", "grotop"])

# Available formats
BSS.IO.fileFormats()
```

## Free Energy Perturbation

### Relative Binding Free Energy (RBFE)

```python
# 1. Load two ligands
mol0 = BSS.IO.readMolecules("ligand0.pdb")[0]
mol1 = BSS.IO.readMolecules("ligand1.pdb")[0]

# 2. Match atoms between ligands
mapping = BSS.Align.matchAtoms(mol0, mol1)

# 3. Align mol0 to mol1
mol0_aligned = BSS.Align.rmsdAlign(mol0, mol1, mapping)

# 4. Create perturbable (merged) molecule
merged = BSS.Align.merge(mol0_aligned, mol1, mapping)

# 5. Insert into solvated system and run
# (replace original ligand with merged molecule in system)
protocol = BSS.Protocol.FreeEnergy(
    num_lam=11,  # lambda windows
    runtime=2 * BSS.Units.Time.nanosecond,
)
fe = BSS.FreeEnergy.Relative(
    system, protocol, engine="GROMACS", work_dir="fe_calc"
)
fe.run()

# 6. Analyze
pmf, overlap = BSS.FreeEnergy.Relative.analyse("fe_calc")
```

### Perturbation Network

```python
# Generate optimal perturbation graph for multiple ligands
molecules = [BSS.IO.readMolecules(f"ligand_{i}.pdb")[0] for i in range(10)]
network = BSS.Align.generateNetwork(molecules)  # LOMAP-based
# Returns edges (pairs) and mappings for each perturbation
```

### Alchemical Transfer Method (ATM)

```python
atm_setup = BSS.FreeEnergy.ATMSetup(
    receptor=receptor,
    ligand_bound=ligand_bound,
    ligand_free=ligand_free,
)
system = atm_setup.prepare()
# Run ATM protocols: minimise -> equilibrate -> anneal -> production
```

## Metadynamics

```python
# Define collective variable (e.g., torsion angle)
cv = BSS.Metadynamics.CollectiveVariable.Torsion(
    atoms=[0, 1, 2, 3],
    lower_bound=BSS.Metadynamics.Bound(-180*BSS.Units.Angle.degree, force_constant=200),
    upper_bound=BSS.Metadynamics.Bound(180*BSS.Units.Angle.degree, force_constant=200),
    grid=BSS.Metadynamics.Grid(minimum=-180*BSS.Units.Angle.degree,
                                maximum=180*BSS.Units.Angle.degree,
                                num_bins=400),
)

protocol = BSS.Protocol.Metadynamics(
    collective_variable=cv,
    runtime=5 * BSS.Units.Time.nanosecond,
    hill_height=1.5 * BSS.Units.Energy.kj_per_mol,
    hill_frequency=500,
)

# Uses PLUMED internally
process = BSS.Process.OpenMM(system, protocol)
process.start()
```

## Format Conversion

```python
# To RDKit
rdkit_mol = BSS.Convert.to(system[0], "rdkit")

# To OpenMM
openmm_system, openmm_topology, openmm_positions = BSS.Convert.to(system, "openmm")

# From SMILES
mol = BSS.Convert.smiles("Cc1ccccc1")
```

## Nodes (Reusable Workflow Components)

```python
node = BSS.Gateway.Node("Minimise a molecular system")
node.addInput("files", BSS.Gateway.FileSet(help="Input files"))
node.addInput("steps", BSS.Gateway.Integer(help="Steps", default=5000, minimum=1))
node.addOutput("minimised", BSS.Gateway.FileSet(help="Output files"))

node.showControls()  # GUI in Jupyter, argparse on CLI

system = BSS.IO.readMolecules(node.getInput("files"))
protocol = BSS.Protocol.Minimisation(steps=node.getInput("steps"))
process = BSS.MD.run(system, protocol)
node.setOutput("minimised", BSS.IO.saveMolecules("min", process.getSystem(block=True), system.fileFormat()))
node.validate()
```

## Process Monitoring

```python
# All processes expose the same monitoring API
process.isRunning()        # bool
process.isError()          # bool
process.getTime()          # list of time values
process.getTotalEnergy()   # list of energies
process.getTrajectory()    # trajectory object
process.getStdout()        # stdout lines
process.getStderr()        # stderr lines
process.workDir()          # working directory path
process.kill()             # terminate process

# Multi-process runner
runner = BSS.Process.ProcessRunner([proc1, proc2, proc3])
runner.startAll()
runner.isRunning()         # checks all
```

## Units System

```python
# Always use typed quantities — never raw numbers
5 * BSS.Units.Length.nanometer
300 * BSS.Units.Temperature.kelvin
1 * BSS.Units.Pressure.atm
10 * BSS.Units.Time.nanosecond
1.5 * BSS.Units.Energy.kj_per_mol

# Arithmetic works
distance = 5 * BSS.Units.Length.angstrom
distance_nm = distance.nanometers()  # convert
```

## Common Patterns

**Full setup pipeline:**
```python
mol = BSS.IO.readMolecules("protein.pdb")[0]
mol = BSS.Parameters.ff14SB(mol).getMolecule()
system = BSS.Solvent.tip3p(molecule=mol, shell=1.2*BSS.Units.Length.nanometer, ion_conc=0.15)
# Minimise -> Equilibrate (NVT heat) -> Equilibrate (NPT) -> Production
```

**Engine fallback:** `BSS.MD.run()` auto-picks the first available engine. For reproducibility, specify explicitly: `BSS.Process.Gromacs(system, protocol)`.

**Restraints:** Use `"backbone"`, `"heavy"`, `"all"`, or `"none"` in Protocol constructors.

**Checkpointing:** `BSS.Stream.save(system, "checkpoint")` / `BSS.Stream.load("checkpoint.s3")`

## Force Fields

| Category | Available |
|----------|-----------|
| Protein | ff03, ff99, ff99SB, ff99SBildn, ff14SB, ff19SB |
| Small molecule | GAFF, GAFF2 |
| OpenForceField | openff_unconstrained_2_0_0, openff_unconstrained_1_3_1, etc. |
| Water | SPC, SPC/E, TIP3P, TIP4P, TIP5P, OPC |
