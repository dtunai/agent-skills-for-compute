---
title: Physics Constraints
impact: MEDIUM
tags: pinn, physics-informed, sym, constraints, pde, loss, boundary-conditions
---

# Skill: Physics Constraints

Add physics-informed constraints to neural networks using PhysicsNeMo-Sym.

## Quick Command

```bash
# Install PhysicsNeMo-Sym
pip install Cython
pip install nvidia-physicsnemo-sym --no-build-isolation

# Run Helmholtz example
git clone https://github.com/NVIDIA/physicsnemo-sym.git
cd physicsnemo-sym/examples/helmholtz/
python helmholtz.py
```

## When to Use

- Solving PDEs with physics-informed neural networks (PINNs)
- Adding boundary condition constraints to training
- Enforcing conservation laws in neural surrogates
- Solving inverse problems with physics priors
- Problems with limited training data (physics compensates)

## Step-by-Step Instructions

### 1. PhysicsNeMo-Sym Key API

The Key API maps named physical variables through the network:

```python
from physicsnemo.sym.key import Key
from physicsnemo.sym.models.arch import Arch

class PhysicsModel(Arch):
    def __init__(self):
        super().__init__(
            input_keys=[Key("x"), Key("y"), Key("t")],   # Spatial + time
            output_keys=[Key("u"), Key("v"), Key("p")],   # Velocity + pressure
        )
        self.net = ...  # Your neural network

    def forward(self, dict_tensor):
        x = self.concat_input(dict_tensor, self.input_key_dict,
                              detach_dict=None, dim=1)
        out = self.net(x)
        return self.split_output(out, self.output_key_dict, dim=1)
```

### 2. PINN Loss Construction

Physics-informed loss combines data loss and PDE residual:

```python
import torch

def physics_loss(model, collocation_points, boundary_data, data_points=None):
    # Enable gradients for collocation points
    collocation_points.requires_grad_(True)

    # Forward pass
    predictions = model(collocation_points)

    # PDE residual (e.g., Laplace equation: nabla^2 u = 0)
    u = predictions["u"]
    du_dx = torch.autograd.grad(u, collocation_points,
        grad_outputs=torch.ones_like(u), create_graph=True)[0]
    d2u_dx2 = torch.autograd.grad(du_dx, collocation_points,
        grad_outputs=torch.ones_like(du_dx), create_graph=True)[0]

    pde_residual = d2u_dx2.sum(dim=1)  # Laplacian
    pde_loss = (pde_residual ** 2).mean()

    # Boundary condition loss
    bc_pred = model(boundary_data["points"])
    bc_loss = ((bc_pred["u"] - boundary_data["values"]) ** 2).mean()

    # Data loss (if available)
    data_loss = 0.0
    if data_points is not None:
        data_pred = model(data_points["points"])
        data_loss = ((data_pred["u"] - data_points["values"]) ** 2).mean()

    # Weighted combination
    total_loss = pde_loss + 10.0 * bc_loss + data_loss
    return total_loss
```

### 3. Common PDE Patterns

**Helmholtz equation** (wave propagation):
```python
# nabla^2 u + k^2 u = f
def helmholtz_residual(u, x, y, k, f):
    du_dx = torch.autograd.grad(u, x, torch.ones_like(u), create_graph=True)[0]
    d2u_dx2 = torch.autograd.grad(du_dx, x, torch.ones_like(du_dx), create_graph=True)[0]
    du_dy = torch.autograd.grad(u, y, torch.ones_like(u), create_graph=True)[0]
    d2u_dy2 = torch.autograd.grad(du_dy, y, torch.ones_like(du_dy), create_graph=True)[0]
    return d2u_dx2 + d2u_dy2 + k**2 * u - f
```

**Navier-Stokes (2D steady, incompressible)**:
```python
# Continuity: du/dx + dv/dy = 0
# Momentum-x: u*du/dx + v*du/dy = -dp/dx + nu*(d2u/dx2 + d2u/dy2)
def navier_stokes_residual(u, v, p, x, y, nu):
    # Compute all needed derivatives via autograd
    du_dx = grad(u, x)
    du_dy = grad(u, y)
    dv_dx = grad(v, x)
    dv_dy = grad(v, y)
    dp_dx = grad(p, x)
    dp_dy = grad(p, y)

    continuity = du_dx + dv_dy
    momentum_x = u * du_dx + v * du_dy + dp_dx - nu * (grad(du_dx, x) + grad(du_dy, y))
    momentum_y = u * dv_dx + v * dv_dy + dp_dy - nu * (grad(dv_dx, x) + grad(dv_dy, y))

    return continuity, momentum_x, momentum_y
```

### 4. Loss Weighting Strategies

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| Fixed weights | Manual λ values | Known relative importance |
| Learning rate annealing | Decay data weight over time | Transition from data to physics |
| Gradient balancing | Normalize by gradient magnitude | Multi-scale physics |
| Curriculum | Gradually increase PDE complexity | Hard-to-optimize PDEs |

```python
# Example: gradient-balanced weighting
def balanced_loss(pde_loss, bc_loss, data_loss):
    pde_grad = torch.autograd.grad(pde_loss, model.parameters(), retain_graph=True)
    bc_grad = torch.autograd.grad(bc_loss, model.parameters(), retain_graph=True)

    pde_norm = sum(g.norm() for g in pde_grad)
    bc_norm = sum(g.norm() for g in bc_grad)

    weight = pde_norm / (bc_norm + 1e-8)
    return pde_loss + weight * bc_loss + data_loss
```

### 5. Full PhysicsNeMo-Sym Workflow (Solver + Domain)

Complete PINN using the Sym framework with geometry, constraints, and solver:

```python
from physicsnemo.sym.hydra import instantiate_arch, PhysicsNeMoConfig
from physicsnemo.sym.solver import Solver
from physicsnemo.sym.domain import Domain
from physicsnemo.sym.geometry.primitives_2d import Rectangle
from physicsnemo.sym.domain.constraint import (
    PointwiseBoundaryConstraint, PointwiseInteriorConstraint,
)
from physicsnemo.sym.key import Key
from physicsnemo.sym.eq.pdes.navier_stokes import NavierStokes

@physicsnemo.sym.main(config_path="conf", config_name="config")
def run(cfg: PhysicsNeMoConfig) -> None:
    ns = NavierStokes(nu=0.01, rho=1.0, dim=2, time=False)

    flow_net = instantiate_arch(
        input_keys=[Key("x"), Key("y")],
        output_keys=[Key("u"), Key("v"), Key("p")],
        cfg=cfg.arch.fully_connected,
    )
    nodes = ns.make_nodes() + [flow_net.make_node(name="flow_network")]

    rec = Rectangle((-0.05, -0.05), (0.05, 0.05))
    domain = Domain()

    # Boundary: top wall moving at u=1
    domain.add_constraint(PointwiseBoundaryConstraint(
        nodes=nodes, geometry=rec,
        outvar={"u": 1.0, "v": 0}, batch_size=1000,
        criteria=Eq(Symbol("y"), 0.05),
    ), "top_wall")

    # Interior: enforce PDE residuals
    domain.add_constraint(PointwiseInteriorConstraint(
        nodes=nodes, geometry=rec,
        outvar={"continuity": 0, "momentum_x": 0, "momentum_y": 0},
        batch_size=4000,
        lambda_weighting={
            "continuity": Symbol("sdf"),
            "momentum_x": Symbol("sdf"),
            "momentum_y": Symbol("sdf"),
        },
    ), "interior")

    solver = Solver(cfg, domain=domain)
    solver.solve()
```

### 6. Custom PDE Definition

```python
from sympy import Symbol, Function, Number
from physicsnemo.sym.eq.pde import PDE

class WaveEquation1D(PDE):
    name = "WaveEquation1D"
    def __init__(self, c=1.0):
        x, t = Symbol("x"), Symbol("t")
        u = Function("u")(x, t)
        if type(c) is str:
            c = Function(c)(x, t)
        elif type(c) in [float, int]:
            c = Number(c)
        self.equations = {"wave_equation": u.diff(t, 2) - (c**2 * u.diff(x)).diff(x)}
```

Built-in PDEs: `NavierStokes`, `AdvectionDiffusion`, `Diffusion`, `WaveEquation`, `HelmholtzEquation`, `MaxwellFreqReal`, `LinearElasticity`, `ZeroEquation`.

### 7. Geometry (CSG + STL)

```python
from physicsnemo.sym.geometry.primitives_2d import Rectangle, Circle
from physicsnemo.sym.geometry.primitives_3d import Box, Sphere, Cylinder
from physicsnemo.sym.geometry.tessellation import Tessellation

# CSG operations
geo = (Box((-1,-1,-1), (1,1,1)) & Sphere((0,0,0), 1.2)) - Cylinder((0,0,0), 0.5, 2)

# STL import
geo_stl = Tessellation.from_stl("./mesh.stl", airtight=True)

# Sampling
boundary_pts = geo.sample_boundary(nr_points=100000)
interior_pts = geo.sample_interior(nr_points=100000, compute_sdf_derivatives=True)
```

### 8. Sym Constraint Types

| Constraint | Purpose |
|------------|---------|
| `PointwiseBoundaryConstraint` | Enforce BCs on geometry surface |
| `PointwiseInteriorConstraint` | PDE residuals in domain interior |
| `IntegralBoundaryConstraint` | Monte Carlo integral conservation |
| `PointwiseConstraint.from_numpy()` | Data assimilation from arrays |

### 9. Inverse Problems

Use `detach_names` to freeze known variables, optimize unknown parameters:

```python
# Separate networks for unknown material properties
nu_net = instantiate_arch(
    input_keys=[Key("x"), Key("y")],
    output_keys=[Key("nu")],
    cfg=cfg.arch.fully_connected,
)
# Data constraint with observed fields
PointwiseConstraint.from_numpy(
    nodes=nodes, invar=observed_data, outvar=observed_fields,
    batch_size=cfg.batch_size.data,
)
```

## Physics Domains in PhysicsNeMo

| Domain | PDE | Example |
|--------|-----|---------|
| Heat transfer | Diffusion equation | Thermal simulation |
| Fluid dynamics | Navier-Stokes | CFD surrogate |
| Electromagnetics | Maxwell's equations | Antenna design |
| Structural | Elasticity equations | Stress analysis |
| Acoustics | Helmholtz equation | Sound propagation |
| Weather | Atmospheric dynamics | FourCastNet |

## Common Pitfalls

- **Stiff PDE residuals**: Scale PDE terms to similar magnitudes. Use loss balancing.
- **Vanishing gradients in autograd**: Use `create_graph=True` for higher-order derivatives.
- **Not enough collocation points**: PDE residual needs dense sampling in the domain interior.
- **Ignoring boundary conditions**: BC loss should be weighted higher (10-100x) than PDE loss initially.
- **Wrong coordinate system**: Ensure autograd variables match the PDE spatial dimensions.

## Related Skills

- [training-recipes.md](./training-recipes.md) - Integrate physics loss into training
- [models-and-architectures.md](./models-and-architectures.md) - Sym model wrapper
- [data-pipelines.md](./data-pipelines.md) - Collocation point generation
