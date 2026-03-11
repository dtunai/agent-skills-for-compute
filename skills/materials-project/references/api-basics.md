# Materials Project API Basics Reference

Sources:
- [Getting Started](https://docs.materialsproject.org/downloading-data/using-the-api/getting-started)
- [MPRester API](https://materialsproject.github.io/api/_autosummary/mp_api.client.mprester.MPRester.html)

## Installation

### pip Installation

```bash
# Basic installation
pip install mp_api

# With pymatgen
pip install mp_api pymatgen

# Development installation
git clone https://github.com/materialsproject/api
cd api
pip install -e .
```

### Requirements

- Python 3.8+
- Internet connection for API access
- Materials Project account for API key

## API Key

### Obtaining API Key

1. Create account at https://materialsproject.org/
2. Navigate to https://next-gen.materialsproject.org/api
3. Copy your unique API key from profile dashboard

### API Key Configuration

```python
# Method 1: Direct initialization
from mp_api.client import MPRester
with MPRester("your_api_key_here") as mpr:
    pass

# Method 2: Environment variable
import os
os.environ["MP_API_KEY"] = "your_api_key"
with MPRester() as mpr:
    pass

# Method 3: Shell configuration (~/.bashrc or ~/.zshrc)
# export MP_API_KEY="your_api_key"
```

## MPRester Initialization

### Basic Initialization

```python
from mp_api.client import MPRester

# With context manager (recommended)
with MPRester(api_key="your_key") as mpr:
    # API calls here
    pass

# Without context manager
mpr = MPRester(api_key="your_key")
try:
    # API calls
    pass
finally:
    mpr.session.close()
```

### Configuration Parameters

```python
MPRester(
    api_key=None,                    # API key (or use MP_API_KEY env var)
    endpoint="https://api.materialsproject.org",  # API endpoint
    notify_db_version=True,          # Log database version changes
    include_user_agent=True,         # Include system info in requests
    use_document_model=True,         # Validate data, enable autocomplete
    session=None,                    # Custom HTTP session
    mute_progress_bars=False         # Suppress progress indicators
)
```

### use_document_model

```python
# True (default): Returns validated pymatgen objects
with MPRester(use_document_model=True) as mpr:
    structure = mpr.get_structure_by_material_id("mp-149")
    # Returns pymatgen.core.Structure object

# False: Returns raw dictionaries (faster)
with MPRester(use_document_model=False) as mpr:
    structure = mpr.get_structure_by_material_id("mp-149")
    # Returns dict
```

## Database Information

### Database Version

```python
# Get current database version
version = mpr.get_database_version()
print(version)  # Format: YYYY_MM_DD
```

### Database Contents

- **150,000+ inorganic compounds**
- **35,000+ molecules**
- **DFT-calculated properties** (PBE, SCAN, r2SCAN functionals)
- **Experimental structures** from ICSD, COD
- **Computed properties**: thermodynamics, electronic, phonon, elastic, piezoelectric, dielectric

## Available Endpoints

### Materials Routes

```python
# Summary (default)
mpr.materials.summary.search()

# Specific routes
mpr.materials.tasks           # Calculation tasks
mpr.materials.robocrystallographer  # Structure descriptions
mpr.materials.bonds           # Bond analysis
mpr.materials.substrates      # Substrate matching
mpr.materials.grain_boundaries # Grain boundary structures
mpr.materials.fermi_surface   # Fermi surface data
```

### Property Routes

```python
mpr.thermo              # Thermodynamic properties
mpr.electronic_structure # Band structures, DOS
mpr.phonon              # Phonon properties
mpr.dielectric          # Dielectric tensors
mpr.piezoelectric       # Piezoelectric tensors
mpr.elasticity          # Elastic tensors
mpr.magnetism           # Magnetic properties
mpr.xas                 # X-ray absorption spectra
mpr.eos                 # Equations of state
mpr.oxidation_states    # Oxidation states
mpr.provenance          # Data provenance
mpr.similarity          # Structure similarity
mpr.surface_properties  # Surface energies
mpr.grain_boundaries    # Grain boundary properties
mpr.synthesis           # Synthesis recipes
mpr.insertion_electrodes # Battery electrode materials
mpr.molecules           # Molecular database
```

## Session Management

### Context Manager (Recommended)

```python
# Automatic session cleanup
with MPRester() as mpr:
    data = mpr.materials.summary.search(elements=["Li"])
# Session automatically closed
```

### Manual Session Management

```python
mpr = MPRester()
try:
    data = mpr.materials.summary.search(elements=["Li"])
finally:
    mpr.session.close()
```

### Custom Session

```python
import requests

session = requests.Session()
session.headers.update({"Custom-Header": "value"})

with MPRester(session=session) as mpr:
    data = mpr.materials.summary.search(elements=["Li"])
```

## Error Handling

### API Errors

```python
from mp_api.client.core import MPRestError

try:
    with MPRester() as mpr:
        structure = mpr.get_structure_by_material_id("invalid-id")
except MPRestError as e:
    print(f"API error: {e}")
except Exception as e:
    print(f"Unexpected error: {e}")
```

### Rate Limiting

```python
# Materials Project implements rate limiting
# Best practices:
# 1. Use context manager for proper cleanup
# 2. Batch queries when possible
# 3. Cache results locally
# 4. Limit fields to reduce data transfer
```

## Progress Bars

```python
# Enable progress bars (default)
with MPRester() as mpr:
    docs = mpr.materials.summary.search(elements=["Li"])

# Disable progress bars
with MPRester(mute_progress_bars=True) as mpr:
    docs = mpr.materials.summary.search(elements=["Li"])
```

## Best Practices

1. **Use environment variables**: Store API key securely
2. **Context manager**: Always use `with` statement
3. **Limit fields**: Only retrieve needed data
4. **Cache results**: Store frequently used data locally
5. **Batch queries**: Combine multiple requests
6. **Error handling**: Catch and handle API errors
7. **Version tracking**: Monitor database version changes
8. **Document model**: Use for development, disable for production speed

## Quick Reference

```python
from mp_api.client import MPRester

# Initialize
with MPRester(api_key="your_key") as mpr:
    # Get database version
    version = mpr.get_database_version()
    
    # Basic query
    docs = mpr.materials.summary.search(elements=["Li"])
    
    # Get structure
    structure = mpr.get_structure_by_material_id("mp-149")
    
    # Access different routes
    thermo_docs = mpr.thermo.search(material_ids=["mp-149"])
    
    # Performance optimization
    docs = mpr.materials.summary.search(
        elements=["Li"],
        fields=["material_id", "formula_pretty"]
    )
```
