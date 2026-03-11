# Qiskit Metal Reference

Source: [Qiskit Metal Documentation](https://qiskit-community.github.io/qiskit-metal/)

## Overview

**Quantum Metal** (formerly Qiskit Metal) is an open-source quantum EDA platform for designing and analyzing superconducting quantum chips.

**Key Feature:** "Design and analyze superconducting quantum chips with a Python API + GUI that plugs into your favorite tools."

## Project Status

- **Current Version**: 0.5 (transition underway)
- **Breaking Changes**: Expected through v0.6
- **Installation**: Source installation required for v0.5+
- **PyPI Package**: Archived at pre-0.5 state

## Installation

```bash
git clone https://github.com/qiskit-community/qiskit-metal.git
cd qiskit-metal
pip install -e .
```

## Core Architecture

### Four Foundational Elements

1. **Quantum Device Design** (`QDesign`)
   - Central framework integrating all components
   - Handles design rules and chip specifications
   - Manages component hierarchy

2. **Quantum Device Components** (`QComponent`)
   - Core class library
   - Base classes: `QComponent`, `BaseQubit`
   - Routing systems for transmission lines

3. **Quantum Renderer** (`QRenderer`)
   - Export/visualization tools
   - Formats: GDS, Ansys, matplotlib
   - Integration with external EDA tools

4. **Quantum Analysis**
   - Electromagnetic simulation
   - Device characterization
   - Parameter extraction

## Component Library

### Qubit Components

| Component | Description |
|-----------|-------------|
| `TransmonPocket` | Pocket-style transmon with tunable coupling |
| `TransmonCross` | Cross-shaped transmon geometry |
| `TransmonInterdigitated` | Interdigital capacitor design |

### Resonator Components

| Component | Description |
|-----------|-------------|
| `ResonatorLumped` | Lumped element resonator |
| `ResonatorCoplanar` | CPW resonator |
| `ResonatorMeandered` | Meandered resonator for compact layout |

### Routing Components

| Component | Description |
|-----------|-------------|
| `RouteMeander` | Meandered transmission line (CPW) |
| `RouteStraight` | Straight CPW transmission line |
| `RoutePathfinder` | Auto-routed connections |

### Passive Components

| Component | Description |
|-----------|-------------|
| `CapNInterdigital` | Interdigital capacitor |
| `JunctionJosephson` | Josephson junction |
| `LaunchpadWirebond` | Wirebond pad for I/O |

## Design Workflow

### 1. Create Design
```python
from qiskit_metal import designs, MetalGUI

design = designs.DesignPlanar()
gui = MetalGUI(design)
```

### 2. Add Components
```python
from qiskit_metal.qlibrary.qubits.transmon_pocket import TransmonPocket

qubit_opts = dict(
    pos_x='0mm',
    pos_y='0mm',
    connection_pads=dict(
        a=dict(loc_W=+1, loc_H=+1),
        b=dict(loc_W=-1, loc_H=-1)
    )
)

q1 = TransmonPocket(design, 'Q1', options=qubit_opts)
```

### 3. Add Routing
```python
from qiskit_metal.qlibrary.tlines.meandered import RouteMeander

route_opts = dict(
    pin_inputs=dict(
        start_pin=dict(component='Q1', pin='a'),
        end_pin=dict(component='Q1', pin='b')
    ),
    total_length='6mm'
)

route = RouteMeander(design, 'route1', options=route_opts)
```

### 4. Visualize
```python
gui.rebuild()
gui.autoscale()
gui.screenshot()
```

## Renderers

### GDS Export
```python
from qiskit_metal.renderers.renderer_gds import QGDSRenderer

gds = QGDSRenderer(design)
gds.options['fabricate'] = True
gds.export_to_gds('my_chip.gds')
```

### Ansys Integration
```python
from qiskit_metal.renderers.renderer_ansys import QAnsysRenderer

ansys = QAnsysRenderer(design, "hfss")
ansys.connect()
ansys.render_design()
```

### Matplotlib Visualization
```python
from qiskit_metal.renderers.renderer_mpl import QMplRenderer

mpl = QMplRenderer(design)
mpl.render()
```

### Gmsh Support
```python
from qiskit_metal.renderers.renderer_gmsh import QGmshRenderer

gmsh = QGmshRenderer(design)
gmsh.render_design()
```

## Analysis Tools

### EPR Analysis (Eigenmode Participation Ratio)

Requires Ansys HFSS installation.

```python
from qiskit_metal.analyses.quantization import EPRanalysis

epr = EPRanalysis(design, "hfss")
epr.sim.setup.max_passes = 15
epr.sim.setup.min_converged_passes = 2
epr.sim.setup.pct_refinement = 30

# Run simulation
epr.sim.run()

# Extract results
frequencies = epr.get_frequencies()
coupling_matrix = epr.get_coupling_matrix()
participation_ratios = epr.get_participations()
```

### Capacitance Matrix Extraction
```python
from qiskit_metal.analyses.quantization import LOManalysis

lom = LOManalysis(design, "q3d")
lom.sim.setup.max_passes = 10
lom.sim.run()

cap_matrix = lom.get_capacitance_matrix()
```

### Impedance Analysis
```python
from qiskit_metal.analyses.impedance import Impedance_analysis

imp = Impedance_analysis(design)
imp.run()
Z0 = imp.get_impedance()
```

## Parametric Design

All components support parametric options:

```python
qubit_opts = dict(
    pos_x='0mm',
    pos_y='0mm',
    pad_width='425um',
    pad_height='90um',
    pad_gap='30um',
    pocket_width='650um',
    pocket_height='650um'
)

# Update options dynamically
q1.options['pad_width'] = '500um'
design.rebuild()
```

## Batch Optimization

```python
import numpy as np

# Sweep parameter
pad_widths = np.linspace(400, 500, 10)

results = []
for width in pad_widths:
    q1.options['pad_width'] = f'{width}um'
    design.rebuild()

    epr.sim.run()
    freq = epr.get_frequencies()[0]
    results.append((width, freq))
```

## Community & Support

- **Discord**: Primary community platform ([discord.gg/kaZ3UFuq](https://discord.gg/kaZ3UFuq))
- **Slack**: Legacy support channel
- **GitHub**: [qiskit-community/qiskit-metal](https://github.com/qiskit-community/qiskit-metal)
- **Annual Conference**: Quantum Device Workshop (QDW) at UCLA/USC
- **Governance**: Quantum Device Consortium (QDC)

## Tutorial Resources

Available in GitHub repository:

- `tutorials/1-Intro-to-qiskit-metal/`
- `tutorials/2-From-components-to-chip/`
- `tutorials/3-Renderers/`
- `tutorials/4-Analysis/`

## Automation Capabilities

- **Parametric Design**: All components fully parametrized
- **Batch Optimization**: Sweep parameters programmatically
- **Hierarchical Composition**: Nest components for reusability
- **Auto-routing**: Pathfinding algorithms for transmission lines
- **Design Rule Checking**: Built-in DRC integration

## Best Practices

1. **Use parametric options** instead of hard-coded geometries
2. **Rebuild design** after option changes: `design.rebuild()`
3. **Save frequently**: `design.save_design('my_design.metal')`
4. **Version control GDS exports** for fabrication tracking
5. **Run convergence checks** in Ansys simulations (max_passes, min_converged_passes)
6. **Use hierarchical design** for complex multi-qubit chips
7. **Check port impedances** before fabrication
8. **Validate routing lengths** for frequency control

## Known Limitations

- **v0.5 Transition**: Breaking API changes expected
- **Ansys License**: Required for EM analysis (commercial software)
- **GUI Performance**: Can slow with >50 components
- **GDS Precision**: Verify layer mappings for fab house requirements
