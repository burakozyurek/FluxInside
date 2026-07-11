# FluxInside v0.2 Alpha

**Author:** Burak ÖZYÜREK

## User Manual

# 1. Overview
FluxInside is a high-performance **Flux Balance Analysis (FBA)** extension designed for **Agent-Based Models (ABMs)**. It enables researchers to integrate constraint-based metabolic calculations into ABMs, allowing metabolically aware simulations for individual agents.

FluxInside utilizes the **GNU Linear Programming Kit (GLPK)** to solve linear programming optimization problems efficiently.

# 2. Requirements
Although FluxInside is primarily designed for agent-based modeling platforms, it can also be used as a standalone C++ library by following a similar integration approach.

### Dependencies
Make sure following libraries are isntalled on your compiler:

- **GLPK (GNU Linear Programming Kit)** Linear programming solver.

- **Matio** Library for reading MATLAB (`.mat`) files directly into memory.

# 3. Installation
Clone or download FluxInside from its GitHub repository:

> https://github.com/burakozyurek/FluxInside

Include the FluxInside source files in your C++ project and import the library using:

```cpp
#include "intracellularFBA.hpp"
```

# 4. Implementation
## Supported Model Files
FluxInside does not currently provide tools to construct genome-scale metabolic models (GEMs) from scratch.

Models should first be configured and exported as MATLAB (`.mat`) file using one of the constraint based optimization tool such as:

- COBRApy
- COBRA Toolbox
- RAVEN Toolbox

### Supported MATLAB Formats

- v4
- v5
- v7.3

### Required Fields

The model must contain the following data as indicated:

- Stoichiometric matrix (`S`)
- Lower bounds (`lb`)
- Upper bounds (`ub`)
- Objective coefficients (`c`)
- Reaction names (`rxnNames`)

FluxInside automatically detects whether the stoichiometric matrix is **dense** or **sparse**. Sparse matrices are internally converted into GLPK's triplet representation during model loading.

## IntracellularFBA Class
### `buildModelFromMat(const char* file_path)`

Reads the structural components of a GEM from a MATLAB file and constructs the internal GLPK model.

> **Important:** This function should be called **only once** during initialization.

**Parameters**

| Parameter | Description |
|------------|-------------|
| `file_path` | Path to the `.mat` model |

### `changeBoundByName(const std::string& rx_name, double lb, double ub)`

Updates the lower and upper bounds of a reaction without rebuilding the optimization model.

Typically used for dynamically changing transport reaction constraints according to extracellular metabolite concentrations.

| Parameter | Description |
|------------|-------------|
| `rx_name` | Reaction ID |
| `lb` | Lower bound |
| `ub` | Upper bound |

### `setObjectiveByName(const std::string& rx_name, double coef)`

Changes the optimization objective.

| Parameter | Description |
|------------|-------------|
| `rx_name` | Reaction ID |
| `coef` | Objective coefficient (`1` = maximize, `-1` = minimize) |

### `optimizeModel()`

Executes the GLPK simplex algorithm using a warm-start and stores the resulting flux distribution in memory.

### `getFluxByName(const std::string& name)`

Returns the optimized flux value for a specified reaction.


# 5. Integrating with PhysiCell & BioFVM
## Step 1 — Install PhysiCell
Download and install PhysiCell following its official documentation.

## Step 2 — Create a Template Project
Inside the PhysiCell root directory:

```bash
cd <PhysiCell folder>

make template

make
```

## Step 3 — Copy FluxInside Folder
Copy the FluxInside folder containing following files

```
intracellularFBA.h
intracellularFBA.hpp
```

into the following folder(recomended):

```
./addons/
```


Also place your metabolic model (`.mat`) inside:

```
./config/
```

## Step 4 — Include FluxInside
```cpp
#include "../addons/FluxInside/intracellularFBA.hpp"
```

## Step 5 — Modify the Makefile
Append the following linker flags near the compile command (approximately line 77):

```make
-lglpk -lmatio
```

## Step 6 — Create a Global FBA Object
```cpp
IntracellularFBA ecoli_fba;

static std::mutex fba_mutex;
```


## Step 7 — Load the GEM
Inside `create_cell_types()` (recommended):

```cpp
ecoli_fba.buildModelFromMat("../config/ecolimat.mat");
```

> **Recommendation:** Load the model only once during initialization to avoid unnecessary file parsing and model reconstruction.

## Step 8 — Compile and run
```bash
make; ./project
```

## Expected Initialization Output
```
Executable name is project

Using config file ./config/PhysiCell_settings.xml ...

Reading MAT file...

MAT file version: v5

Reading variables from:

./config/e_coli_core.mat

Variable name: e_coli_core

Number of fields: 17

Field 0: mets
Field 1: metNames
Field 2: metFormulas
Field 3: metCharge
Field 4: genes
Field 5: rxnGeneMat
Field 6: grRules
Field 7: rxns
Field 8: rxnNames
Field 9: subSystems
Field 10: S
Field 11: lb
Field 12: ub
Field 13: b
Field 14: c
Field 15: rev
Field 16: description

Dense matrix detected.

S_matrix size: 360 x 3

Dense size: 72 x 95

BIOMASS_Ecoli_core_w_GAM, obj coef: 1

Optimization took 0.0018423 seconds.
```

## Running FBA During Simulation
Inside a phenotype function:

```cpp
std::lock_guard<std::mutex> lock(fba_mutex);

ecoli_fba.changeBoundByName(
    "EX_o2_e",
    -UniformRandom() * 5,
    0
);

ecoli_fba.optimizeModel();

std::cout
    << "Biomass for cell "
    << pCell->ID
    << ": "
    << ecoli_fba.getFluxByName("BIOMASS_Ecoli_core_w_GAM")
    << std::endl;
```

## Example Output
```
...
current simulated time: 540 min (max: 7200 min)
total agents: 5
total wall time: 0 days, 0 hours, 0 minutes,and 4.31966 seconds

Biomass for cell 0: 0.00458103
Biomass for cell 1: 0.0257217
Biomass for cell 2: -0.0568624
Biomass for cell 3: 0.129544
Biomass for cell 4: 0.215374
Biomass for cell 0: 0.146727
Biomass for cell 1: -0.0560702
Biomass for cell 2: -0.0036188
Biomass for cell 3: 0.25905
Biomass for cell 4: 0.230496
Biomass for cell 0: 0.0845726
Biomass for cell 1: 0.0178405
Biomass for cell 2: 0.226161
Biomass for cell 3: 0.0139243
Biomass for cell 4: 0.160807
Biomass for cell 0: 0.00896789
...
```

# 6. Troubleshooting & Known Issues
## Numerical Instability with Warm Starts
Large genome-scale metabolic models such as **HumanGEM** may gradually accumulate numerical instability when repeatedly solved using warm starts without resetting the internal LP state.

This behavior has **not** been observed in smaller models such as:

- Ecoli_Core
- Yeast8

### Resolution

Disable warm starts when repeatedly solving very large metabolic models.

# Repository

**GitHub**

https://github.com/burakozyurek/FluxInside
