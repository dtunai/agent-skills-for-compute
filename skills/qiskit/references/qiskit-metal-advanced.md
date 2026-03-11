# Qiskit Metal Advanced Reference

Sources:
- [Qiskit Metal Tutorials](https://qiskit-community.github.io/qiskit-metal/)
- [EPRanalysis Documentation](https://qiskit-community.github.io/qiskit-metal/apidocs/qiskit_metal.analyses.EPRanalysis.html)
- [Eigenmode and EPR Analysis Tutorial](https://qiskit-community.github.io/qiskit-metal/tut/4-Analysis/4.02-Eigenmode-and-EPR.html)
- [Analyzing Transmon and Resonator](https://qiskit.org/documentation/metal/tut/4-Analysis/4.13-Analyze-transmon-and-resonator.html)

## Analysis Workflows

### EPR Analysis (Energy Participation Ratio)

**Purpose:** Extract qubit frequencies, coupling strengths, and anharmonicities from electromagnetic simulations.

**Requirements:**
- Ansys HFSS installed and licensed
- Python connection to Ansys

### Complete EPR Workflow

```python
from qiskit_metal import designs, MetalGUI
from qiskit_metal.qlibrary.qubits.transmon_pocket import TransmonPocket
from qiskit_metal.analyses.quantization import EPRanalysis

# 1. Create design
design = designs.DesignPlanar()
gui = MetalGUI(design)

# 2. Add transmon qubit
qubit_opts = dict(
    pos_x='0mm',
    pos_y='0mm',
    pad_width='425um',
    pad_height='90um',
    pocket_width='650um',
    pocket_height='650um'
)
q1 = TransmonPocket(design, 'Q1', options=qubit_opts)

gui.rebuild()
gui.autoscale()

# 3. Initialize EPR analysis
epr = EPRanalysis(design, "hfss")

# 4. Connect to Ansys
epr.sim.renderer.connect_ansys()

# 5. Render design in HFSS
epr.sim.render_design(['Q1'], [])

# 6. Setup simulation
epr.sim.setup.max_passes = 15
epr.sim.setup.min_converged_passes = 2
epr.sim.setup.pct_refinement = 30
epr.sim.setup.basis_order = 1

# 7. Analyze specific modes
epr.sim.setup.n_modes = 2
epr.sim.setup.min_freq_ghz = 3
epr.sim.setup.max_freq_ghz = 8

# 8. Run simulation
epr.sim.run()

# 9. Get results
frequencies = epr.get_frequencies()
print(f"Mode frequencies: {frequencies} GHz")

# 10. EPR analysis (requires junction dictionary)
epr.sim.setup_junction_analysis()
epr_results = epr.get_epr()

print(f"Participation ratios: {epr_results}")

# 11. Extract qubit parameters
qubit_freq = epr.get_qubit_frequencies()
anharmonicity = epr.get_anharmonicity()
coupling = epr.get_coupling_matrix()

print(f"Qubit frequency: {qubit_freq} GHz")
print(f"Anharmonicity: {anharmonicity} MHz")
print(f"Coupling matrix:\n{coupling}")
```

### Junction Configuration

**Critical Step:** Define junction locations and properties.

```python
# Junction dictionary
junctions = {
    'junc_rect': {
        'Lj_variable': 'Lj',
        'Lj': '12nH',
        'rect': 'JJ_rect_Lj_Q1_rect_jj'
    }
}

epr.sim.setup.junctions = junctions
```

### EPR Results Interpretation

**Participation Ratio:**
Fraction of total electric energy stored in each component.

```python
epr_data = epr.get_epr()

# Example output:
# {
#   'Q1_pad': 0.45,     # 45% in qubit pads
#   'Q1_pocket': 0.03,  # 3% in pocket
#   'substrate': 0.52   # 52% in substrate
# }
```

**Use Cases:**
- Optimize qubit design to maximize pad participation
- Minimize substrate loss
- Balance coupling vs. isolation

## Capacitance Matrix Analysis

### LOM (Lumped Oscillator Model) Extraction

**Purpose:** Extract capacitance matrix for equivalent circuit model.

**Workflow:**

```python
from qiskit_metal.analyses.quantization import LOManalysis

# 1. Initialize LOM
lom = LOManalysis(design, "q3d")

# 2. Connect to Ansys Q3D
lom.sim.renderer.connect_ansys()

# 3. Render design
lom.sim.render_design(['Q1', 'Q2'], [])

# 4. Setup simulation
lom.sim.setup.max_passes = 10
lom.sim.setup.min_converged_passes = 2
lom.sim.setup.pct_error = 1.0

# 5. Run simulation
lom.sim.run()

# 6. Get capacitance matrix
C_matrix = lom.get_capacitance_matrix()

print(f"Capacitance matrix:\n{C_matrix}")
```

### Capacitance Matrix Format

```
C_matrix (in fF):
       Q1_pad  Q1_ground  Q2_pad  Q2_ground
Q1_pad    100       -20      -5        -75
Q1_ground -20        95     -10        -65
Q2_pad     -5       -10     100        -85
Q2_ground -75       -65     -85        225
```

**Interpretation:**
- Diagonal: Self-capacitance
- Off-diagonal: Mutual capacitance (coupling)
- Sum of row = net capacitance to ground

## Parametric Sweeps

### Sweep Qubit Parameters

```python
import numpy as np

# Define sweep parameters
pad_widths = np.linspace(400, 500, 10)  # µm

results = []

for width in pad_widths:
    # Update design
    q1.options['pad_width'] = f'{width}um'
    design.rebuild()

    # Run EPR
    epr.sim.render_design(['Q1'], [])
    epr.sim.run()

    freq = epr.get_frequencies()[0]
    results.append((width, freq))

# Plot results
import matplotlib.pyplot as plt
widths, freqs = zip(*results)
plt.plot(widths, freqs)
plt.xlabel('Pad Width (µm)')
plt.ylabel('Frequency (GHz)')
plt.title('Qubit Frequency vs Pad Width')
plt.show()
```

### Automated Sweep Script

```python
def sweep_parameter(component, param_name, values):
    """Generic parameter sweep."""
    results = []

    for val in values:
        # Update parameter
        component.options[param_name] = f'{val}um'
        design.rebuild()

        # Run analysis
        epr.sim.render_design([component.name], [])
        epr.sim.run()

        # Extract results
        freq = epr.get_frequencies()[0]
        epr_data = epr.get_epr()

        results.append({
            'value': val,
            'frequency': freq,
            'epr': epr_data
        })

    return results

# Example usage
results = sweep_parameter(q1, 'pad_width', np.linspace(400, 500, 10))
```

## Multi-Qubit Analysis

### Two-Qubit Coupling

```python
from qiskit_metal.qlibrary.qubits.transmon_pocket import TransmonPocket

# Create two qubits
q1 = TransmonPocket(design, 'Q1', options={'pos_x': '0mm', 'pos_y': '0mm'})
q2 = TransmonPocket(design, 'Q2', options={'pos_x': '2mm', 'pos_y': '0mm'})

gui.rebuild()

# EPR analysis
epr = EPRanalysis(design, "hfss")
epr.sim.render_design(['Q1', 'Q2'], [])
epr.sim.setup.n_modes = 4  # Two qubits × 2 modes each
epr.sim.run()

# Get coupling matrix
coupling = epr.get_coupling_matrix()

print(f"Q1-Q2 coupling: {coupling[0, 1]} MHz")
```

### Coupling Interpretation

**Coupling Strength:**
```
g = |coupling[i, j]| MHz
```

**Coupling Regimes:**
- **g < 1 MHz**: Weak coupling (isolated qubits)
- **1 MHz < g < 10 MHz**: Moderate coupling (controlled entanglement)
- **g > 10 MHz**: Strong coupling (always-on interaction)

## Resonator Design and Analysis

### Meandered Resonator

```python
from qiskit_metal.qlibrary.resonator.resonator_coil_rect import ResonatorCoilRect

# Create resonator
res_opts = dict(
    pos_x='1mm',
    pos_y='0mm',
    orientation='0',
    total_length='6mm',
    trace_width='10um',
    trace_gap='6um',
    n_turns=5
)

res = ResonatorCoilRect(design, 'Res1', options=res_opts)

gui.rebuild()

# Analyze resonator mode
epr = EPRanalysis(design, "hfss")
epr.sim.render_design(['Res1'], [])
epr.sim.setup.n_modes = 1
epr.sim.setup.min_freq_ghz = 5
epr.sim.setup.max_freq_ghz = 8
epr.sim.run()

res_freq = epr.get_frequencies()[0]
print(f"Resonator frequency: {res_freq} GHz")
```

### Qubit-Resonator Coupling

```python
# Design with qubit and resonator
q1 = TransmonPocket(design, 'Q1', options={...})
res = ResonatorCoilRect(design, 'Res1', options={...})

gui.rebuild()

# Analyze coupled system
epr = EPRanalysis(design, "hfss")
epr.sim.render_design(['Q1', 'Res1'], [])
epr.sim.setup.n_modes = 3  # Qubit + resonator modes
epr.sim.run()

# Dispersive coupling
chi = epr.get_dispersive_coupling()
print(f"Dispersive shift χ: {chi} MHz")
```

## GDS Export for Fabrication

### Export Full Design

```python
from qiskit_metal.renderers.renderer_gds import QGDSRenderer

# Initialize GDS renderer
gds = QGDSRenderer(design)

# Configure layers
gds.options['ground_plane'] = True
gds.options['cheese']['datatype'] = 100
gds.options['path_filename'] = './qubits_chip.gds'

# Export
gds.export_to_gds('chip_v1.gds')
```

### Layer Configuration

```python
# Custom layer mapping
gds.options['layers'] = {
    'qubit_pads': {'layer': 1, 'datatype': 0},
    'routing': {'layer': 2, 'datatype': 0},
    'ground': {'layer': 3, 'datatype': 0},
    'airbridges': {'layer': 4, 'datatype': 0}
}
```

### Fabrication-Ready Checklist

1. **Layer assignment**: All features on correct layers
2. **Design rule check (DRC)**: Minimum feature sizes
3. **Ground plane**: Proper tie-downs and vias
4. **Alignment marks**: For multi-layer lithography
5. **Dicing streets**: Chip separation
6. **Bonding pads**: Wire bond locations

## Advanced Rendering

### Gmsh Export

```python
from qiskit_metal.renderers.renderer_gmsh import QGmshRenderer

gmsh = QGmshRenderer(design)
gmsh.render_design()
gmsh.export_mesh('design.msh')
```

### Custom Matplotlib Visualization

```python
from qiskit_metal.renderers.renderer_mpl import QMplRenderer

mpl = QMplRenderer(design)

# Configure plot
mpl.options['metal_color'] = 'gold'
mpl.options['background_color'] = 'black'
mpl.options['show_labels'] = True

# Render
fig, ax = mpl.render()
plt.show()
```

## Best Practices for Analysis

1. **Convergence**: Ensure HFSS converges (check delta S)
2. **Mesh quality**: Increase passes for accurate results
3. **Boundary conditions**: Verify radiation/PEC boundaries
4. **Junction placement**: Accurate junction location critical
5. **Parameter sweeps**: Automate for design optimization
6. **Result validation**: Cross-check with analytical models
7. **Save frequently**: Design files and analysis results
8. **Version control**: Track design iterations
9. **Documentation**: Record parameter choices and results
10. **Calibration**: Compare simulations to fabricated devices

## Common Pitfalls

- **Insufficient mesh refinement**: Inaccurate frequencies
- **Wrong junction parameters**: Unrealistic anharmonicity
- **Missing ground plane**: Poor field confinement
- **Overlapping components**: Mesh errors
- **Incorrect material properties**: Wrong loss tangent/permittivity
- **Not simplifying geometry**: Slow simulations
- **Forgetting to rebuild**: Stale design in renderer
- **Boundary too close**: Affects mode structure

## Performance Optimization

### Reduce Simulation Time

1. **Simplify geometry**: Remove unnecessary features
2. **Use symmetry**: Exploit design symmetry
3. **Coarse initial mesh**: Refine iteratively
4. **Limit modes**: Only analyze required modes
5. **Parallel simulation**: Use HFSS distributed solve

### GPU Acceleration

```python
# Enable GPU solving (requires HFSS GPU license)
epr.sim.setup.use_gpu = True
epr.sim.setup.gpu_ids = [0, 1]  # Use first two GPUs
```

## Advanced Topics

### Purcell Effect Analysis

Calculate decay rate into transmission line.

```python
# Define transmission line ports
epr.sim.setup.ports = {
    'port1': {'impedance': 50, 'renormalize': True}
}

# Run eigenmode with ports
epr.sim.run()

# Get Purcell decay rate
kappa_purcell = epr.get_purcell_decay()
print(f"Purcell decay: {kappa_purcell} MHz")
```

### T1 Estimation

Estimate relaxation time from dielectric loss.

```python
# Set material loss tangent
design.chips['main'].material = 'silicon'
design.chips['main'].loss_tangent = 1e-4

# Run EPR
epr.sim.run()

# Estimate T1
T1_est = epr.estimate_t1()
print(f"Estimated T1: {T1_est} µs")
```

## Resources

- **Tutorials**: Full chip design examples in `tutorials/Appendix A`
- **API Docs**: Component and analysis API reference
- **Discord**: Community support at discord.gg/kaZ3UFuq
- **GitHub Issues**: Bug reports and feature requests
- **QDW Conference**: Annual quantum device workshop
