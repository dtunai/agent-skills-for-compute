# Materials Project Electronic and Phonon Properties Reference

Sources:
- [MPRester API](https://materialsproject.github.io/api/_autosummary/mp_api.client.mprester.MPRester.html)
- [API Examples](https://docs.materialsproject.org/downloading-data/using-the-api/examples)

## Band Structures

### Get Band Structure

```python
# Band structure by material ID
bs = mpr.get_bandstructure_by_material_id("mp-149")

# Line-mode (default)
bs_line = mpr.get_bandstructure_by_material_id("mp-149", line_mode=True)

# Uniform mode
bs_uniform = mpr.get_bandstructure_by_material_id("mp-149", line_mode=False)
```

### Band Structure Properties

```python
# Band gap
print(bs.get_band_gap())

# Direct vs indirect
print(bs.is_metal())
gap_info = bs.get_band_gap()
print(f"Direct: {gap_info['direct']}")

# Band extrema
cbm = bs.get_cbm()  # Conduction band minimum
vbm = bs.get_vbm()  # Valence band maximum

print(f"CBM energy: {cbm['energy']}")
print(f"VBM energy: {vbm['energy']}")
print(f"CBM k-point: {cbm['kpoint']}")
```

### Plotting Band Structures

```python
from pymatgen.electronic_structure.plotter import BSPlotter

# Create plotter
plotter = BSPlotter(bs)

# Get matplotlib figure
fig = plotter.get_plot()
fig.savefig("bandstructure.png")

# Show plot
plotter.show()

# Customize
fig = plotter.get_plot(
    ylim=(-5, 5),
    vbm_cbm_marker=True,
    zero_to_efermi=True
)
```

## Density of States

### Get DOS

```python
# Complete DOS
dos = mpr.get_dos_by_material_id("mp-149")

# Total DOS
total_dos = dos.densities[dos.efermi]

# Fermi level
print(f"Fermi energy: {dos.efermi}")
```

### DOS Properties

```python
# Band gap from DOS
gap = dos.get_gap()
print(f"Band gap: {gap}")

# CBM and VBM
cbm = dos.get_cbm()
vbm = dos.get_vbm()

# Integrated DOS at energy
integrated = dos.get_interpolated_gap(tol=0.001)
```

### Projected DOS

```python
# Element-projected DOS
if hasattr(dos, 'get_element_dos'):
    fe_dos = dos.get_element_dos("Fe")
    o_dos = dos.get_element_dos("O")

# Orbital-projected DOS
if hasattr(dos, 'get_spd_dos'):
    spd_dos = dos.get_spd_dos()
```

### Plotting DOS

```python
from pymatgen.electronic_structure.plotter import DosPlotter

# Create plotter
plotter = DosPlotter()

# Add DOS
plotter.add_dos("Total", dos)

# Plot
fig = plotter.get_plot()
fig.savefig("dos.png")

# With element projection
plotter.add_dos_dict(dos.get_element_dos())
```

## Combined Band Structure and DOS

```python
from pymatgen.electronic_structure.plotter import BSDOSPlotter

# Get both
bs = mpr.get_bandstructure_by_material_id("mp-149")
dos = mpr.get_dos_by_material_id("mp-149")

# Plot together
plotter = BSDOSPlotter(bs_projection=None, dos_projection=None)
fig = plotter.get_plot(bs, dos)
fig.savefig("bs_dos.png")
```

## Phonon Properties

### Phonon Density of States

```python
# Get phonon DOS
ph_dos = mpr.get_phonon_dos_by_material_id("mp-149")

# Phonon frequencies
print(ph_dos.frequencies)
print(ph_dos.densities)

# Zero-point energy
zpe = ph_dos.zero_point_energy
```

### Phonon Band Structure

```python
# Get phonon band structure
ph_bs = mpr.get_phonon_bandstructure_by_material_id("mp-149")

# Check for imaginary modes (instability)
has_imaginary = ph_bs.has_imaginary_freq()

# Get imaginary mode details
if has_imaginary:
    imaginary_modes = ph_bs.get_imaginary_freqs()
```

### Phonon Plotting

```python
from pymatgen.phonon.plotter import PhononDosPlotter, PhononBSPlotter

# DOS plotting
dos_plotter = PhononDosPlotter()
dos_plotter.add_dos("Material", ph_dos)
dos_plotter.show()

# Band structure plotting
bs_plotter = PhononBSPlotter(ph_bs)
bs_plotter.show()
```

## Electronic Structure Search

```python
# Search by band gap
docs = mpr.electronic_structure.search(
    band_gap=(1.0, 3.0),
    fields=["material_id", "band_gap", "is_gap_direct", "is_metal"]
)

for doc in docs:
    print(f"{doc.material_id}: {doc.band_gap} eV")
    print(f"Direct: {doc.is_gap_direct}")
```

## Dielectric Properties

### Dielectric Tensors

```python
# Get dielectric data
docs = mpr.dielectric.search(material_ids=["mp-149"])

for doc in docs:
    # Electronic dielectric tensor
    print(doc.e_electronic)
    
    # Ionic dielectric tensor
    print(doc.e_ionic)
    
    # Total dielectric tensor
    print(doc.e_total)
    
    # Refractive index
    print(doc.n)
```

### Dielectric Constants

```python
# Search by dielectric constant
docs = mpr.dielectric.search(
    e_total=(10, None),  # High dielectric constant
    fields=["material_id", "e_total", "e_ionic", "e_electronic"]
)
```

## Piezoelectric Properties

### Piezoelectric Tensors

```python
# Get piezoelectric data
docs = mpr.piezoelectric.search(material_ids=["mp-149"])

for doc in docs:
    # Piezoelectric tensor
    print(doc.piezoelectric_tensor)
    
    # e_ij modulus
    print(doc.e_ij_max)
```

## Elastic Properties

### Elastic Tensors

```python
# Get elastic data
docs = mpr.elasticity.search(material_ids=["mp-149"])

for doc in docs:
    # Elastic tensor
    print(doc.elastic_tensor)
    
    # Bulk modulus
    print(doc.k_vrh)  # Voigt-Reuss-Hill average
    
    # Shear modulus
    print(doc.g_vrh)
    
    # Universal anisotropy
    print(doc.universal_anisotropy)
```

## Magnetic Properties

### Magnetism Data

```python
# Get magnetic properties
docs = mpr.magnetism.search(material_ids=["mp-149"])

for doc in docs:
    # Magnetic ordering
    print(doc.ordering)
    
    # Total magnetization
    print(doc.total_magnetization)
    
    # Is magnetic
    print(doc.is_magnetic)
```

## X-Ray Absorption Spectra

### XAS Data

```python
# Get XAS data
docs = mpr.xas.search(
    material_ids=["mp-149"],
    absorbing_element="Fe"
)

for doc in docs:
    # Spectrum data
    print(doc.spectrum)
    
    # Edge
    print(doc.edge)
    
    # Absorbing element
    print(doc.absorbing_element)
```

## Charge Density

### Charge Density Files

```python
# Get charge density data
charge_density = mpr.get_charge_density_from_material_id("mp-149")

# CHGCAR file
if charge_density:
    print(charge_density.data)
```

## VASP Parameters

### Task-Level Data

```python
# Get task IDs
task_ids = mpr.get_task_ids_associated_with_material_id("mp-149")

# Get VASP parameters from specific task
task_docs = mpr.materials.tasks.search(task_ids=[task_ids[0]])

for doc in task_docs:
    # INCAR parameters
    print(doc.input.incar)
    
    # KPOINTS
    print(doc.input.kpoints)
    
    # NELECT (number of electrons)
    if "NELECT" in doc.input.incar:
        print(f"NELECT: {doc.input.incar['NELECT']}")
```

## Fermi Surface

### Fermi Surface Data

```python
# Get Fermi surface data
docs = mpr.materials.fermi_surface.search(material_ids=["mp-149"])

for doc in docs:
    print(doc.data)
```

## Batch Operations

### Multiple Materials

```python
# Get band structures for multiple materials
material_ids = ["mp-149", "mp-13", "mp-22526"]

band_structures = {}
for mid in material_ids:
    try:
        bs = mpr.get_bandstructure_by_material_id(mid)
        band_structures[mid] = bs
    except Exception as e:
        print(f"Failed for {mid}: {e}")
```

## Best Practices

1. **Check availability**: Not all materials have all properties
2. **Handle errors**: Wrap API calls in try-except
3. **Cache results**: Store band structures/DOS locally
4. **Use appropriate k-point paths**: Line-mode for visualization
5. **Normalize DOS**: Use pymatgen normalization functions
6. **Combine plots**: Show band structure and DOS together
7. **Check imaginary phonons**: Indicates structural instability

## Common Patterns

```python
# Screen for semiconductors
docs = mpr.electronic_structure.search(
    band_gap=(0.5, 3.0),
    is_gap_direct=True,
    fields=["material_id", "band_gap"]
)

# Find high-dielectric materials
docs = mpr.dielectric.search(
    e_total=(20, None),
    fields=["material_id", "e_total", "e_ionic"]
)

# Analyze band structure
bs = mpr.get_bandstructure_by_material_id("mp-149")
if not bs.is_metal():
    gap = bs.get_band_gap()
    print(f"Band gap: {gap['energy']} eV")
    print(f"Direct: {gap['direct']}")
    
    # Plot
    from pymatgen.electronic_structure.plotter import BSPlotter
    plotter = BSPlotter(bs)
    plotter.show()
```
