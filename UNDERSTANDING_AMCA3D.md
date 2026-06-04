# Understanding AMCA3D: Physics to Implementation
## A Guide to Cellular Automata Grain Growth Simulation

---

## Table of Contents
1. [Physics Fundamentals](#physics-fundamentals)
2. [Cellular Automata Methodology](#cellular-automata-methodology)
3. [The Rappaz-Gandin 1993 Model](#the-rappaz-gandin-1993-model)
4. [Implementation Architecture](#implementation-architecture)
5. [Main Simulation Loop](#main-simulation-loop)
6. [Key Data Structures](#key-data-structures)
7. [Critical Concepts for Neural Network Surrogate](#critical-concepts-for-neural-network-surrogate)

---

## Physics Fundamentals

### Solidification in Additive Manufacturing

**The Problem**: During additive manufacturing (3D printing with lasers/electron beams), material melts and then solidifies. The microstructure that forms—the size, shape, and orientation of grains—depends on:
- **Temperature field** (from FEM solver)
- **Cooling rate** (temporal temperature derivative)
- **Nucleation** (new grains appearing)
- **Growth** (existing grains expanding)

### Three Key Phenomena

#### 1. **Phase States**
Each cell in the domain exists in one of three states:
- **Liquid** (T ≥ T_melting)
- **Mushy zone** (T_solidus < T < T_melting)  
- **Solid** (T ≤ T_solidus)

#### 2. **Nucleation**
When temperature drops below the solidus and undercooling (ΔT = T_solidus - T) exceeds a critical value, NEW grains spontaneously form.

Nucleation sites are:
- **Discrete locations** in space (predefined at start)
- **Randomly distributed** with a given number density
- **Each has a critical undercooling** drawn from a Gaussian distribution

#### 3. **Growth**
Once a grain is formed, its solid-liquid interface expands into the surrounding liquid/mushy zone. The growth rate depends on undercooling via the **Rappaz-Gandin model** (see below).

---

## Cellular Automata Methodology

### What is Cellular Automata?

A cellular automaton (CA) is a grid-based computational model where:
- **Space** is discretized into a regular lattice (cubic cells in 3D)
- **State** of each cell evolves according to local rules
- **Time** advances in discrete steps
- **Evolution** depends on neighbors (26-neighborhood in 3D)

### Why CA for Grain Growth?

Traditional continuum methods (FEM/FDM) are expensive for microstructure. CA is efficient because:
1. **Fast**: Simple neighbor-checking rules vs. solving PDEs
2. **Parallel**: Each processor gets local domain; ghost cells handle boundaries
3. **Discrete grains**: Naturally represents individual grain entities
4. **Anisotropy**: Can encode crystal orientation effects via geometry (octahedra)

### The Lattice and Neighborhood

```
3D Domain:
┌─────────────────────────────────────┐
│                                     │
│  Cubic cells with (x, y, z) indices │
│  Cell size h (input parameter)      │
│  Total: nx × ny × nz cells          │
│                                     │
└─────────────────────────────────────┘

26-Neighborhood (3D):
     Face neighbors (6): ±x, ±y, ±z
     Edge neighbors (12): (±x, ±y), etc.
     Corner neighbors (8): (±x, ±y, ±z)
     Total: 26 neighbors per cell
```

---

## The Rappaz-Gandin 1993 Model

### The Growth Velocity Law

The fundamental equation in AMCA3D is the **Rappaz-Gandin model**:

$$v(\Delta T) = a_1 + a_2 \cdot \Delta T$$

Where:
- **v** = growth velocity (m/s)
- **ΔT** = undercooling = T_solidus - T (K)
- **a₁, a₂** = material-specific coefficients (loaded from YAML)

### Linear Relationship

This is a **linear approximation** valid in the mushy zone. The model captures:
- **a₁** (intercept): Baseline growth at solidus temperature
- **a₂** (slope): How sensitive growth is to undercooling

### Implementation in Code

```cpp
// File: ProblemPhysics.cpp, lines ~143-150
double ProblemPhysics_RappazGandin1993::compute_growth_velocity_with_temperature_input(double T)
{
    double dT = solidusTemperature_ - T;  // undercooling
    if (dT < 0.0) return 0.0;              // no growth if T > T_s
    return a1_ + a2_ * dT;                 // linear model
}
```

### Material Parameters (from YAML)

```yaml
realms:
  - name: realm1
    type: cellular_automata
    physics_model:
      type: "Rappaz_Gandin_1993"
      melting_temperature: 1700.0          # K
      solidus_temperature: 1600.0          # K
      a1: 0.0001                           # m/s (baseline)
      a2: 0.0005                           # m/(s·K) (sensitivity)
      nucleation_undercooling_mean: 200.0   # K
      nucleation_undercooling_sigma: 50.0  # K (Gaussian spread)
```

---

## Implementation Architecture

### Class Hierarchy and Responsibilities

```
Main Solver
└── CafeSolver
    ├── FiniteElementManager (FEM)
    │   └── Solves heat conduction → Temperature field
    │
    ├── CellularAutomataManager_3D (CA)
    │   ├── Grain management
    │   ├── Nucleation sites
    │   ├── Growth algorithm
    │   └── Octahedral envelopes
    │
    └── MapVoxelManager
        └── Maps FEM nodes to CA cells (interpolation)
```

### Key Classes

#### **1. CafeSolver** (File: CafeSolver.h/cpp)
- **Role**: Orchestrates the coupled simulation
- **Key method**: `time_integration_weak_coupling()`
- **Responsibility**: 
  - Synchronize timesteps between FEM and CA
  - Call FEM solver for temperature field
  - Update CA based on new temperatures
  - Handle file I/O

#### **2. CellularAutomataManager_3D** (File: CellularAutomataManager_3D.h/cpp)
- **Role**: Core CA engine
- **Key methods**:
  - `single_integration_step()` — one CA timestep
  - `calculate_time_step()` — compute dt based on max velocity
  - `nucleate_at_nucleation_sites()` — create new grains
  - `setup_problem()` — initialize domain and cell array

#### **3. Grain** (File: Grain.h/cpp)
- **Role**: Individual grain entity
- **Members**:
  ```cpp
  int id_;                    // Unique grain identifier
  int cell_;                  // Which CA cell contains the nucleus
  double xc_, yc_, zc_;       // Center coordinates
  Orientation * orientation_; // Crystal lattice orientation (Euler angles)
  double length_;             // Half-diagonal of octahedron (grows linearly)
  bool active_;               // Is this grain still growing?
  ```

#### **4. Octahedron** (File: Octahedron.h/cpp)
- **Role**: Defines anisotropic growth envelope
- **Geometry**: Octahedral shape aligned with crystal axes
- **Key method**: `cell_is_captured()` — test if cell center is inside octahedron
- **Purpose**: Encodes crystal anisotropy; different growth in different directions

#### **5. ProblemPhysics** (File: ProblemPhysics.h/cpp)
- **Role**: Material model and physics parameters
- **Subclass**: `ProblemPhysics_RappazGandin1993` — implements growth velocity
- **Key method**: `compute_growth_velocity_with_temperature_input(T)`

#### **6. MapVoxelManager** (File: MapVoxelManager.h/cpp)
- **Role**: Coupling layer between FEM and CA
- **Task**: Interpolate temperature from FEM mesh nodes to CA cell centers
- **Method**: Bilinear/trilinear interpolation

#### **7. ParallelCommManager_3D** (File: ParallelCommManager_3D.h/cpp)
- **Role**: MPI communication handler
- **Tasks**:
  - Exchange ghost cell data between processors
  - Synchronize grain orientations
  - All-reduce operations for global quantities

---

## Main Simulation Loop

### Weak Coupling Strategy

AMCA3D uses **one-way coupling**: Temperature from FEM drives CA, but CA doesn't affect FEM.

```
┌─────────────────────────────────────────────────────────┐
│ For each time step:                                     │
│                                                         │
│ 1. FEM Solver:                                         │
│    - Solve heat conduction equation                    │
│    - Compute temperature field T(x,y,z,t)             │
│                                                         │
│ 2. Temperature Mapping (MapVoxelManager):              │
│    - Interpolate T from FEM nodes to CA cells         │
│                                                         │
│ 3. Phase Update:                                       │
│    - Update each cell's phase (Liquid/Mushy/Solid)   │
│    - Based on T vs T_solidus, T_melting              │
│                                                         │
│ 4. CA Evolution:                                       │
│    - Nucleate new grains where ΔT > ΔT_critical      │
│    - Grow active grains: length += v(ΔT) × dt        │
│    - Capture cells where octahedron now covers       │
│    - Inactivate grains with no uncaptured neighbors  │
│    - Parallel sync via MPI (ghost cells)             │
│                                                         │
│ 5. Timestep Synchronization:                          │
│    - dt = min(dt_FEM, dt_CA) [global]               │
│    - Both solvers advance by same dt                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Pseudocode: time_integration_weak_coupling()

```cpp
// File: CafeSolver.cpp, lines ~177-240

dT_FE = feManager_->get_time_step();        // FEM timestep
caManager_->setup_integration(dT_FE);       // Max allowed CA timestep

time = 0;
iStep = 0;

while (time < finalTime_)
{
    // 1. Update phase state based on current temperature
    caManager_->checkup_current_temperature_to_update_phase_state();
    caManager_->checkup_liquid_cell_neigh_to_growth(time);
    
    // 2. Compute minimum timestep for CA
    dT_CA = caManager_->calculate_time_step();
    
    // 3. Synchronize across all processors
    dT = min(dT_FE, dT_CA);                  // Local candidate
    MPI_Allreduce(&dT, &dT_min, 1, MPI_DOUBLE, MPI_MIN, MPI_COMM_WORLD);
    caManager_->timeStep_ = dT_min;
    
    // 4. Execute one CA step
    caManager_->single_integration_step();
    
    // 5. Execute one FEM step (if applicable)
    feManager_->pseudo_single_integration_step();
    
    // 6. Advance time
    time += dT_min;
    iStep++;
    
    // 7. Output results
    output_results();
}
```

### The CA Single Integration Step

```cpp
// File: CellularAutomataManager_3D.cpp, lines ~517-580

void CellularAutomataManager_3D::single_integration_step()
{
    // Step 1: Grow all active grains
    for (each active grain)
    {
        vel = compute_growth_velocity(grain);
        grain->length_ += vel * dT_;        // Linear growth
    }
    
    // Step 2: Check for cell capture
    for (each active grain)
    {
        for (each neighbor cell)
        {
            if (octahedron contains neighbor cell center)
            {
                capture_cell(neighbor);     // Assign to this grain
                cellGrainID_[neighbor] = grain->id_;
                cellOrientationID_[neighbor] = grain->orientation_->id_;
            }
        }
    }
    
    // Step 3: Inactivate surrounded grains
    for (each active grain)
    {
        bool hasNonCapturedNeighbors = false;
        for (each neighbor cell)
        {
            if (neighbor is liquid/mushy AND not captured)
                hasNonCapturedNeighbors = true;
        }
        if (!hasNonCapturedNeighbors)
            grain->inactivate();             // No more liquid to capture
    }
    
    // Step 4: Parallel synchronization
    commManager_->sync_grain_IDs();         // Ghost cell exchange
    commManager_->sync_global_orientations();
}
```

---

## Key Data Structures

### Cell-Level Arrays (Allocated per Cell)

```cpp
// File: CellularAutomataManager.h, lines ~100-115

int * cellOrientationID_;   // Global grain orientation ID
int * cellGrainID_;         // Local grain ID on this processor
double *cellTemperature_;   // Current temperature
double * cellNucleationDT_; // If >0, this is a nucleation site
bool *cooling_;             // Is cell currently cooling?
std::vector<bool> hasBeenMelted_;    // Has this cell ever melted?
std::vector<bool> cellLiquidNeigh;   // Does cell have liquid neighbors?
```

### Grain Management

```cpp
// File: CellularAutomataManager.h, lines ~120-123

std::vector<Grain *> grainVector_;            // All grains (local)
std::list<Grain *> activeGrainList_;          // Only growing grains (efficient)
std::vector<Orientation *> orientationVector_; // Global orientation catalog
```

Why two grain containers?
- **grainVector_**: Complete history, indexed by grain ID
- **activeGrainList_**: Fast iteration over only growing grains (improves performance)

### Nucleation Sites

```cpp
// File: CellularAutomataManager.h, lines ~126-127

std::list<int> nucleationCellIDList_;         // Cells that are nucleation sites
std::vector<BoundaryNucleal> boundaryNucleals; // Nucleation parameters per site

// BoundaryNucleal stores:
// - Tmax_: Maximum temperature seen at this site
// - Tincubation_: How long undercooling has persisted
// - DeltaTc_: Critical undercooling for this site (random)
// - cellid_: Which cell contains this site
```

---

## Critical Concepts for Neural Network Surrogate

### What is Being Simulated?

The simulator predicts **microstructure evolution**:
- **Input**: Temperature field T(x,y,z,t) from additive manufacturing process
- **Output**: Grain structure: size, orientation, location of each grain at final time

### Key Evolving Quantities

For your neural network, the most important variables are:

| Variable | Symbol | Role | Update Rule |
|----------|--------|------|------------|
| Grain center | (xc, yc, zc) | Position | Constant after nucleation (no grain motion) |
| Grain length | L | Size | L(t+dt) = L(t) + v(ΔT) × dt |
| Grain orientation | θ | Crystal direction | Constant (inherited from nucleation) |
| Undercooling | ΔT | Growth driver | ΔT = T_s - T(cell) |
| Temperature | T | Primary input | From FEM solver |
| Cell phase | φ | Liquid/Mushy/Solid | φ(T) determined by phase boundaries |
| Active status | a | Is grain growing? | a=false if all neighbors captured |

### Parameters for Surrogate Model Training

These are your **inputs** for the neural network:

**Domain/Mesh Properties:**
- Domain size (Lx, Ly, Lz)
- Cell size h (resolution)
- Total cells nx × ny × nz

**Material Properties:**
- T_melting, T_solidus
- a₁, a₂ (growth velocity coefficients)
- Nucleation density (grains/volume)
- Critical undercooling distribution (mean, std)

**Processing Conditions:**
- Cooling rate (dT/dt)
- Scan velocity (if laser-based)
- Peak temperature reached
- Time-temperature history

**Output Targets:**
- Final grain size distribution
- Final grain orientation distribution
- Number of grains nucleated
- Spatial grain map (grain ID at each location)

### Why This is Hard for ML (and Why You Need Understanding)

1. **High dimensionality**: Output is a 3D field (nx × ny × nz values)
2. **Complex nonlinearity**: Phase transitions, nucleation thresholds
3. **Many-to-many mapping**: Many input parameters → multiple outputs
4. **Path dependency**: Final structure depends on entire T(t) history, not just end state
5. **Discrete events**: Nucleation is binary (happens/doesn't happen), hard for smooth NN

### ML Surrogate Strategy Recommendation

**Instead of learning full microstructure directly:**

1. **Stage 1**: Predict nucleation events
   - Input: Local T, dT/dt, position, time
   - Output: Probability of nucleation occurring
   - Architecture: Classification (logistic/softmax)

2. **Stage 2**: Predict final grain sizes
   - Input: Nucleation positions, local cooling rate history
   - Output: Grain volume/radius at end time
   - Architecture: Regression (fully connected or CNN)

3. **Stage 3**: Predict grain boundaries
   - Input: Nucleation map, growth velocities
   - Output: Voronoi-like grain partition
   - Architecture: Graph neural network or segmentation CNN

### Critical Simulation Parameters to Log

When running AMCA3D for training data, save:

```yaml
outputs:
  - temperature_history       # T(x,y,z,t) at all times
  - grain_nucleation_events   # When/where each grain forms
  - grain_growth_history      # L(t) for each grain
  - final_grain_map           # cellGrainID_ at end
  - final_orientations        # cellOrientationID_ at end
  - cooling_rate_field        # dT/dt(x,y,z,t)
```

---

## Summary: The Algorithm in One Diagram

```
AMCA3D Simulation Cycle:

t=0                t=t+dt              t=final
│                   │                   │
└──┬─────────┬──────┴──────┬────────────┴──┐
   │         │             │                │
   ▼         ▼             ▼                ▼
Start      FEM Solve    Temperature    Phase Update
Domain     →Heat Eq.    Mapping        Nucleate/Grow
           →T(x,y,z)    CA→Cells       Capture Cells
                        Interpolate    Sync Parallel
                                       
                        ▲───┬──────────┘
                            │
                         Check:
                    - Max velocity
                    - dt = min(dt_FE, dt_CA)
                    - Global reduction (MPI)
```

---

## File Reference Guide

For deep dives into specific topics:

| Topic | File | Lines | Content |
|-------|------|-------|---------|
| Main loop | CafeSolver.cpp | 177-240 | Weak coupling algorithm |
| CA timestep | CellularAutomataManager_3D.cpp | 467-510 | Growth velocity & dt calc |
| Single step | CellularAutomataManager_3D.cpp | 517-580 | Grow, capture, inactivate |
| Growth model | ProblemPhysics.cpp | 115-150 | Rappaz-Gandin implementation |
| Octahedron | Octahedron.cpp | 131-270 | Cell capture geometry |
| Nucleation | CellularAutomataManager.cpp | ~400s | Site creation & activation |
| Parallel comm | ParallelCommManager_3D.cpp | All | MPI ghost cell exchange |
| FEM coupling | MapVoxelManager.cpp | All | Temperature interpolation |

---

## Questions This Helps You Answer

After studying this document, you should understand:

1. ✅ Why cellular automata is efficient for grain growth
2. ✅ How the Rappaz-Gandin model controls growth
3. ✅ Why timesteps must be synchronized between FEM and CA
4. ✅ What determines when a grain becomes inactive
5. ✅ How MPI parallelization handles domain decomposition
6. ✅ Why octahedral envelopes are used (anisotropy)
7. ✅ What data flows from FEM to CA at each step
8. ✅ How to extract features for a neural network surrogate

---

## Next Steps for Your Neural Network

1. **Generate training data**: Run AMCA3D with varied processing conditions; save intermediate states
2. **Feature engineering**: Extract relevant inputs (cooling rate, local T history, nucleation patterns)
3. **Target definition**: Decide if surrogate predicts final structure or full time evolution
4. **Model validation**: Compare surrogate predictions against full AMCA3D runs
5. **Acceleration**: Use trained model for inverse design (optimize process for desired microstructure)

---

*This document encapsulates the physics, algorithms, and implementation of AMCA3D. Use it as a reference while reading the source code to deepen your understanding.*
