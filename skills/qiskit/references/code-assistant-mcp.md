# Qiskit Code Assistant and MCP Servers Reference

Sources:
- [Qiskit Code Assistant](https://quantum.cloud.ibm.com/docs/en/guides/qiskit-code-assistant)
- [Qiskit MCP Servers GitHub](https://github.com/Qiskit/mcp-servers)
- [Introducing Qiskit Code Assistant Blog](https://www.ibm.com/quantum/blog/qiskit-code-assistant)

## Qiskit Code Assistant

### Overview

**Qiskit Code Assistant** is an AI-powered coding tool providing intelligent quantum code completion, generation, and assistance directly within development environments.

### Key Features

1. **Context-Aware Code Completion**
   - Quantum circuit construction
   - Operator definitions
   - Algorithm implementations
   - Qiskit-specific patterns

2. **Code Generation**
   - Generate circuits from natural language descriptions
   - Create variational ansätze
   - Build quantum algorithms

3. **Documentation Assistance**
   - Inline API documentation
   - Usage examples
   - Best practices suggestions

4. **Error Detection**
   - Qiskit API misuse warnings
   - Dimension mismatch detection
   - Deprecated function alerts

### Supported Environments

- **JupyterLab** (via extension)
- **VS Code** (via extension)
- **PyCharm** (via plugin)
- **Claude Code / Cursor** (via MCP server)

## Installation

### JupyterLab Extension

```bash
pip install qiskit-code-assistant-jupyterlab

# Enable extension
jupyter labextension enable qiskit-code-assistant-jupyterlab
```

**Usage:**
- Type quantum code, get AI suggestions
- Press `Tab` to accept completions
- Use `/assist` command for generation

### VS Code Extension

```
1. Open VS Code
2. Go to Extensions Marketplace
3. Search "Qiskit Code Assistant"
4. Install
5. Authenticate with IBM Quantum account
```

**Usage:**
- Inline completions appear automatically
- Press `Ctrl+Space` for manual trigger
- Right-click → "Qiskit: Generate Circuit"

### Authentication

```python
# First time setup
from qiskit_code_assistant import authenticate

authenticate(token="YOUR_IBM_QUANTUM_TOKEN")
```

Or set environment variable:
```bash
export QISKIT_IBM_TOKEN="YOUR_TOKEN"
```

## MCP Servers for AI Assistants

### Overview

**Model Context Protocol (MCP)** servers enable AI assistants (Claude, Cursor, etc.) to interact with Qiskit and IBM Quantum services.

### Available MCP Servers

| Server | Purpose |
|--------|---------|
| `qiskit-mcp-server` | Core Qiskit library access |
| `qiskit-ibm-runtime-mcp-server` | IBM Quantum Runtime |
| `qiskit-transpiler-mcp-server` | Transpiler service |
| `qiskit-gym-mcp-server` | RL training for circuit synthesis |

### Installation

```bash
# Install MCP server package
pip install qiskit-mcp-servers

# Or individual servers
pip install qiskit-ibm-runtime-mcp-server
```

### Configuration

#### Claude Desktop

Add to `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "qiskit": {
      "command": "python",
      "args": ["-m", "qiskit_mcp_server"],
      "env": {
        "QISKIT_IBM_TOKEN": "YOUR_TOKEN"
      }
    },
    "qiskit-runtime": {
      "command": "python",
      "args": ["-m", "qiskit_ibm_runtime_mcp_server"],
      "env": {
        "QISKIT_IBM_TOKEN": "YOUR_TOKEN"
      }
    }
  }
}
```

#### Cursor

Add to `.cursor/config.json`:

```json
{
  "mcp": {
    "servers": {
      "qiskit": {
        "command": ["python", "-m", "qiskit_mcp_server"],
        "env": {
          "QISKIT_IBM_TOKEN": "YOUR_TOKEN"
        }
      }
    }
  }
}
```

## MCP Server Capabilities

### qiskit-mcp-server

**Tools Provided:**
- `create_circuit`: Build quantum circuits
- `add_gates`: Add gates to circuits
- `transpile_circuit`: Optimize circuits
- `simulate_circuit`: Run statevector simulation
- `visualize_circuit`: Generate circuit diagrams

**Example Usage (via AI Assistant):**

```
User: "Create a 3-qubit GHZ state circuit"

Assistant uses MCP tool:
create_circuit(num_qubits=3)
add_gates(circuit_id=..., gates=[
  {"gate": "h", "qubits": [0]},
  {"gate": "cx", "qubits": [0, 1]},
  {"gate": "cx", "qubits": [0, 2]}
])
```

### qiskit-ibm-runtime-mcp-server

**Tools Provided:**
- `list_backends`: Query available QPUs
- `get_backend_properties`: Retrieve calibration data
- `submit_job`: Execute circuits on hardware
- `get_job_result`: Retrieve results
- `get_job_status`: Monitor job progress

**Example Usage:**

```
User: "Run my circuit on the least busy backend"

Assistant uses MCP tools:
1. list_backends(operational=true, simulator=false)
2. submit_job(circuit=..., backend="ibm_torino")
3. get_job_result(job_id=...)
```

### qiskit-transpiler-mcp-server

**Tools Provided:**
- `transpile`: Cloud-based AI transpilation
- `get_ai_synthesis`: Neural network gate synthesis
- `optimize_circuit`: Advanced optimization

**Example Usage:**

```
User: "Transpile this circuit for ibm_kyiv with AI optimization"

Assistant uses MCP tool:
transpile(circuit=..., backend="ibm_kyiv", ai_optimization=true)
```

### qiskit-gym-mcp-server

**Tools Provided:**
- `train_rl_model`: Train RL agent for circuit synthesis
- `evaluate_model`: Test trained model
- `synthesize_circuit`: Generate circuits with RL

**Example Usage:**

```
User: "Train an RL agent to synthesize Clifford circuits"

Assistant uses MCP tool:
train_rl_model(
  task="clifford_synthesis",
  num_qubits=3,
  episodes=1000
)
```

## Code Assistant Features in Detail

### Natural Language to Circuit

**Prompt:**
```
"Generate a circuit that creates the W state on 4 qubits"
```

**Generated Code:**
```python
from qiskit import QuantumCircuit
from qiskit.circuit.library import WState

qc = QuantumCircuit(4)
w_state = WState(num_qubits=4)
qc.compose(w_state, inplace=True)
qc.draw('mpl')
```

### Algorithm Templates

**Prompt:**
```
"Create a VQE setup for H2 molecule"
```

**Generated Code:**
```python
from qiskit_algorithms import VQE
from qiskit_algorithms.optimizers import SLSQP
from qiskit.circuit.library import TwoLocal
from qiskit.primitives import Estimator
from qiskit.quantum_info import SparsePauliOp

# H2 Hamiltonian
H = SparsePauliOp.from_list([
    ("II", -1.052),
    ("IZ", 0.398),
    ("ZI", -0.398),
    ("ZZ", -0.011),
    ("XX", 0.181)
])

# Ansatz
ansatz = TwoLocal(num_qubits=2, rotation_blocks='ry',
                  entanglement_blocks='cx', reps=2)

# VQE
estimator = Estimator()
optimizer = SLSQP(maxiter=100)
vqe = VQE(estimator, ansatz, optimizer)
result = vqe.compute_minimum_eigenvalue(H)
```

### Circuit Refactoring

**Before:**
```python
qc = QuantumCircuit(3)
qc.h(0)
qc.cx(0, 1)
qc.cx(0, 2)
```

**Assistant Suggestion:**
```python
from qiskit.circuit.library import GHZ

qc = GHZ(num_qubits=3)
# More concise and readable
```

## Advanced MCP Features

### Conversational Circuit Building

```
User: "Start a new circuit with 5 qubits"
Assistant: [Creates circuit]

User: "Add Hadamard on all qubits"
Assistant: [Adds H gates]

User: "Now entangle them in a linear chain"
Assistant: [Adds CX gates in sequence]

User: "Transpile for ibm_torino"
Assistant: [Transpiles and shows result]
```

### Hardware-Aware Suggestions

```
User: "I want to run VQE on ibm_kyiv"