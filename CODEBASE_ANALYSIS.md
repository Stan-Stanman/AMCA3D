# AMCA3D Codebase Structure Analysis

**Project**: AMCA3D - Additive Manufacturing Cellular Automata 3D  
**Purpose**: A grain growth simulation code targeting additive manufacturing (AM) using coupled cellular automata (CA) and finite element method (FEM)  
**Language**: C++  
**Parallelization**: MPI-based distributed memory parallelization

---

## 1. Main Classes and Their Responsibilities

### 1.1 **Simulation** ([Simulation.h](CA/include/Simulation.h), [Simulation.cpp](CA/src/Simulation.cpp))
- **Responsibility**: Top-level orchestrator and entry point for the simulation
- **Key Functions**:
  - `load()`: Parses YAML configuration to determine which solvers to activate
  - `initialize()`: Delegates initialization to the selected solver
  - `run()`: Executes simulation and triggers finalization
- **Key Members**:
  - `solver_`: Pointer to concrete Solver implementation
  - `cellularAutomata_`, `finiteElementMethod_`: Boolean flags indicating active solvers
- **Decision Logic**: Determines solver type based on input configuration:
  - CA only → `CellularAutomataSolver`
  - FEM only → `FiniteElementMethodSolver`
  - Both → `CafeSolver` (weak coupling)

### 1.2 **CafeSolver** ([CafeSolver.h](CA/include/CafeSolver.h), [CafeSolver.cpp](CA/src/CafeSolver.cpp))
- **Responsibility**: Manages coupled CA-FEM simulation with weak time integration coupling
- **Key Functions**:
  - `load()`: Loads realm configurations for CA and FE managers
  - `initialize()`: Sets up FEM, CA, and voxel mapping between them
  - `execute()`: Runs the weak coupling time integration loop
  - `time_integration_weak_coupling()`: Core coupling algorithm (see workflow below)
- **Key Members**:
  - `caManager_`: CellularAutomataManager (abstract, instantiated as 2D/3D variant)
  - `feManager_`: FiniteElementManager for heat conduction
  - `mapVoxelManager_`: Handles temperature field interpolation from FE to CA
  - `problemPhysics`: Physics model (Rappaz-Gandin 1993)
  - Time parameters: `finalTime_`, `timeStepFactorFE_`, `maxDt_`, `maxSteps_`
- **Coupling Strategy**: One-way coupling without CA time step subcycling
  - Both solvers advance by the same minimum timestep
  - FEM solves heat conduction; results mapped to CA cells
  - CA computes grain growth independent of FEM

### 1.3 **CellularAutomataManager** (Base Class)
- **Responsibility**: Abstract base managing grain nucleation and growth
- **Key Pure Virtual Methods** (implemented in 2D/3D subclasses):
  - `load()`, `setup_problem()`, `setup_cells()`: Initialization
  - `calculate_time_step()`: Adaptive timestep calculation
  - `single_integration_step()`: Advance simulation by one timestep
  - `get_cell_center()`, `get_cell_neighbors()`: Geometric queries
  - `convert_id_*()`: ID mapping utilities (local ↔ global ↔ local-with-ghost)
- **Key Data Members**:
  - **Domain**: `nDim_`, `nx_`, `ny_`, `nz_`, `h_` (cell size)
  - **Local decomposition**: `nxLocal_`, `nyLocal_`, `nzLocal_` (+ ghost variants)
  - **Cell data**: `cellTemperature_`, `cellGrainID_`, `cellOrientationID_`
  - **Nucleation**: `nucleationCellIDList_`, `cellNucleationDT_` (critical undercooling)
  - **Grains**: `grainVector_`, `activeGrainList_`
  - **Orientations**: `orientationVector_` (globally synchronized)
- **Key Physics Methods**:
  - `create_nucleation_sites()`: Stochastic placement with critical undercooling
  - `nucleate_at_nucleation_sites()`: Activate grains when T < T_nucleation
  - `checkup_liquid_cell_neigh_to_growth()`: Grain growth into mushy zone
  - `global_mapped_grain_ID()`: Communicate orientation data across MPI ranks

### 1.4 **CellularAutomataManager_3D** ([CellularAutomataManager_3D.h](CA/include/CellularAutomataManager_3D.h), [CellularAutomataManager_3D.cpp](CA/src/CellularAutomataManager_3D.cpp))
- **Responsibility**: 3D specialization of CA manager
- **Key Differences from 2D**:
  - Uses `Octahedron` envelope (crystal growth shape in 3D)
  - Implements 26-neighbor connectivity (3D grid neighbors)
  - Handles `(x, y, z)` coordinates and 3D cell indexing
- **Key Methods**:
  - `capture_cell()`: Assigns cell to grain if inside octahedral envelope
  - `get_cell_neighbors()`: Returns up to 26 neighboring cells
  - `cell_is_captured()`: Tests if cell center is within grain octahedron
- **MPI Decomposition**:
  - Distributes domain across processors in 3D Cartesian topology
  - `commManager_` handles ghost cell communication
  - Each rank owns a `(nxLocal, nyLocal, nzLocal)` subdomain + 1-cell ghost layer

### 1.5 **Grain** ([Grain.h](CA/include/Grain.h))
- **Responsibility**: Represents a single crystalline grain nucleated at a cell
- **Key Members**:
  - `id_`: Unique grain ID (local to processor)
  - `cell_`: Local ID of nucleation cell
  - `(xc_, yc_, zc_)`: Center coordinates (used for growth)
  - `orientation_`: Pointer to `Orientation` object (defines crystal lattice direction)
  - `length_`: Grain size (diagonal of octahedron in 3D)
  - `active_`: Boolean flag (true if growing, false if surrounded)
- **Key Methods**:
  - `grow(velocity, dt)`: Increments `length_` by `velocity * dt`
  - `reset_grain_envelope()`: Updates center coordinates

### 1.6 **Orientation** ([Orientation.h](CA/include/Orientation.h))
- **Responsibility**: Encodes crystallographic orientation
- **Key Members**:
  - `id_`: Unique global orientation ID
  - `angle_`: Single orientation angle (2D)
  - `EulerAngle_[3]`: Full Euler angles (3D: φ₁, Φ, φ₂)
- **Scope**: Globally synchronized across all MPI ranks
- **Purpose**: Enables multiple grains to share same orientation ID for efficient storage

### 1.7 **Envelope** (Abstract) / **Octahedron** (3D Implementation)
- **Responsibility**: Defines grain shape and capture logic
- **Key Methods**:
  - `cell_is_captured()`: Tests if `(x, y, z)` is inside grain envelope
  - `capture_cell()`: Computes new grain's center and envelope for captured cell
  - `set_coordinate_transformation_matrix()`: Orients envelope based on grain orientation
  - `set_truncation_coefficient()`: Truncates envelope to avoid over-prediction
- **Physical Meaning**: Represents crystal anisotropy; octahedron reflects crystal symmetry

### 1.8 **ProblemPhysics** / **ProblemPhysics_RappazGandin1993**
- **Responsibility**: Physics model for undercooling-based grain growth
- **Key Methods**:
  - `compute_growth_velocity()`: Growth velocity as function of grain and time
  - `compute_growth_velocity_with_temperature_input()`: Velocity from temperature
  - `compute_cell_temperature()`: Current temperature at cell (from FEM)
- **Rappaz-Gandin Model**: Growth velocity proportional to undercooling
  - `v = a₁ + a₂*ΔT` (with temperature-dependent adjustments)
- **Key Parameters** (from ProblemPhysics.h):
  - `meltingTemperature_`: Liquidus (T_L)
  - `solidusTemperature_`: Solidus (T_S)
  - `minTemperatureToNucleate_`: Nucleation undercooling threshold

### 1.9 **FiniteElementManager** ([FiniteElementManager.h](CA/include/FiniteElementManager.h))
- **Responsibility**: Solves transient heat conduction on FE mesh
- **Key Members**:
  - `nodeVector_`, `elementVector_`: FE mesh data
  - `thetaArray_`: Nodal temperatures
  - Time integration: `dT_`, `time_`, `iStep_`, `timeStepFactor_`
- **Key Methods**:
  - `load()`: Reads FE mesh and boundary conditions
  - `initialize()`: Sets initial temperature field
  - `pseudo_single_integration_step()`: Advances thermal solution
  - `setup_time_step_for_coupling()`: Receives timestep from CA
- **Note**: Primarily provides temperature field to CA; not bidirectionally coupled

### 1.10 **Element** / **HexahedralElement** / **QuadrilateralElement**
- **Responsibility**: FE element definitions for variational formulation
- **Key Methods**:
  - `shape_fcn()`: Element shape functions N_i(ξ,η,ζ)
  - `element_level_stiffness_matrix()`: Computes k^e
  - `element_level_time_step()`: Stability-limited timestep
  - `integration_over_the_element_domain()`: Gaussian quadrature

### 1.11 **Node** ([Node.h](CA/include/Node.h))
- **Responsibility**: FE node with temperature and connectivity
- **Key Members**:
  - `coordinates_[3]`: Spatial location
  - `theta_`: Vector of nodal temperatures (one per time step stored)
  - `velocity_[3]`: Nodal velocity (for mechanics coupling if added)
  - `fixTheta_`: Boundary condition flag
  - `nEle_`: Connected elements

### 1.12 **MapVoxelManager**
- **Responsibility**: Interpolation bridge from FE mesh to CA grid
- **Function**: Maps FE nodal temperatures → CA cell temperatures
- **Algorithm**: Likely cell-centered FE to CA voxel nearest-neighbor or shape function interpolation
- **Key Method**: `execute_with_focus_on_CA_region()` - focuses computation on CA-active cells

---

## 2. Overall Simulation Workflow / Main Loop

### 2.1 Program Flow Sequence

```
main() [Cafe.cpp]
  │
  ├─ MPI_Init() - Initialize MPI environment
  ├─ Parse command line arguments (input file, log file)
  ├─ Read YAML input file
  │
  ├─ Simulation::Simulation(yaml_root) - Constructor
  ├─ Simulation::load(yaml_root) - Parse solver types
  │   └─ Create Solver: CafeSolver (if CA+FEM), or CA-only, or FEM-only
  │
  ├─ Simulation::initialize()
  │   └─ Solver::initialize()
  │       ├─ CafeSolver::load(yaml_node) - Load CA & FEM realm configs
  │       │   ├─ CellularAutomataManager_3D::load() - Parse CA domain, cells, nucleation
  │       │   ├─ ProblemPhysics_RappazGandin1993::load() - Load physics params
  │       │   └─ FiniteElementManager::load() - Load FE mesh & BCs
  │       │
  │       └─ CafeSolver::initialize() - Initialize all components
  │           ├─ feManager_->initialize() - Compute FE stiffness, set T₀
  │           ├─ caManager_->initialize() - Set up CA cells, allocate arrays
  │           └─ mapVoxelManager_->initialize() - Build interpolation structure
  │
  ├─ Simulation::run()
  │   └─ Solver::execute() (CafeSolver::execute())
  │       ├─ mapVoxelManager_->execute_with_focus_on_CA_region()
  │       ├─ feManager_->set_grain_around_node_vector()
  │       │
  │       └─ CafeSolver::time_integration_weak_coupling()
  │           │
  │           └─ MAIN LOOP: while (time < finalTime)
  │               │
  │               ├─ [CA Phase] Timer start
  │               ├─ caManager_->checkup_current_temperature_to_update_phase_state()
  │               │   └─ Update phase (solid/liquid) for each CA cell based on T(t)
  │               ├─ caManager_->checkup_liquid_cell_neigh_to_growth()
  │               │   └─ Enable growth into mushy zone (T_s < T < T_l)
  │               │
  │               ├─ [Time step calculation]
  │               ├─ dT_CA = caManager_->calculate_time_step()
  │               ├─ dT = min(dT_FE, dT_CA) - Restrict to smaller timestep
  │               ├─ MPI_Allreduce(dT → dT_min) - Global minimum across ranks
  │               ├─ caManager_->timeStep_ = dT_min
  │               │
  │               ├─ [FEM Phase] Timer start
  │               ├─ feManager_->setup_time_step_for_coupling(dT_min)
  │               │
  │               ├─ [CA Integration] Timer start
  │               ├─ caManager_->setup_time_step_for_coupling(dT_min)
  │               ├─ caManager_->single_integration_step()
  │               │   ├─ nucleate_at_nucleation_sites(time)
  │               │   │   └─ For each nucleation cell: if T < T_nucleation, create grain
  │               │   ├─ For each active grain:
  │               │   │   ├─ compute_growth_velocity(grain, time)
  │               │   │   ├─ grain->grow(velocity, dT_min)
  │               │   │   ├─ Capture cells inside octahedral envelope
  │               │   │   └─ Mark cells as belonging to grain orientation
  │               │   └─ Communicate orientation updates across MPI ranks
  │               │
  │               ├─ time += dT_min
  │               ├─ iStep++
  │               │
  │               ├─ [FEM Integration]
  │               ├─ feManager_->pseudo_single_integration_step()
  │               │   └─ Advance thermal solution (likely using history data)
  │               │
  │               └─ Update dT_FE from FEM
  │
  │   └─ Solver::finalize() (CafeSolver::finalize())
  │       ├─ Stop global timer
  │       ├─ Count final number of grains
  │       ├─ MPI_Allreduce() timing statistics
  │       └─ Output timing summary
  │
  └─ MPI_Finalize()
```

### 2.2 Key Control Flow Details

**Time Stepping**:
- **FEM time step** `dT_FE`: Set from input (e.g., 0.001 s) based on thermal stability
- **CA time step** `dT_CA`: Computed adaptively based on grain growth rates
- **Coupling constraint**: Both advance by `dT = min(dT_FE, dT_CA)`
- **Global synchronization**: `MPI_Allreduce` ensures all ranks use same minimum timestep

**Weak Coupling**:
- FEM → CA: Temperature field interpolated each timestep (one-way)
- CA → FEM: No feedback (decoupled heat conduction from grain growth)
- **Assumption**: Grain growth doesn't significantly affect thermal properties

**Phase State Tracking**:
- Cells transition from liquid → mushy → solid based on temperature thresholds
- Nucleation occurs when T drops below critical undercooling
- Growth only occurs in solid and mushy regions (T ≤ T_s)

---

## 3. Physics Parameters Being Tracked and Evolved

### 3.1 **Temperature Field**
- **Source**: FEM heat conduction solver
- **Spatial**: Defined at FE nodes, interpolated to CA cell centers
- **Temporal**: Evolved each FEM timestep according to transient heat equation
- **Role**: Drives all CA physics (nucleation, growth rate)

### 3.2 **Phase State** (Liquid/Solid/Mushy)
- **Liquid**: T > T_melting
- **Mushy**: T_solidus < T < T_melting (active growth zone)
- **Solid**: T < T_solidus
- **Updated**: Each timestep via `checkup_current_temperature_to_update_phase_state()`

### 3.3 **Grain Microstructure**
- **Grain ID**: Local ID on each processor (not globally unique)
- **Grain Orientation**: Global orientation ID (synchronized across ranks)
- **Grain Center**: `(x_c, y_c, z_c)` - tracks nucleation point
- **Grain Size**: `length` - diagonal of octahedral envelope
- **Cell Assignment**: `cellGrainID_[i]` - which grain owns each CA cell

### 3.4 **Growth Dynamics**

| Parameter | Symbol | Role | Evolution |
|-----------|--------|------|-----------|
| Undercooling | ΔT = T_s - T | Drives growth | Computed from T(t) |
| Growth Velocity | v(ΔT) | Rate of grain expansion | v = a₁ + a₂·ΔT (Rappaz-Gandin) |
| Grain Length | L | Radius of octahedron | dL/dt = v(ΔT, t) |
| Time Step Factor | TimeStep = Factor × h/v_max | Stability constraint | Adaptively limited by max growth rate |

### 3.5 **Nucleation Parameters**
- **Critical Undercooling**: ΔT_c ~ N(μ, σ) per nucleation site (Gaussian distribution)
- **Site Density**: Number of potential nucleation sites per unit volume
  - Surface population: `ns_` sites (lower density)
  - Bulk population: `nv_` sites (higher density)
- **Nucleation Condition**: T < T_s - ΔT_c triggers grain birth

### 3.6 **Domain Partition Data** (For MPI)
- **Global mesh**: `(nx, ny, nz)` total cells
- **Local mesh**: `(nxLocal, nyLocal, nzLocal)` cells per rank
- **Ghost layers**: 1-cell halo around each rank's subdomain
- **Cell mapping**: `cellID_local ↔ cellID_locghost ↔ cellID_global`

---

## 4. How the Cellular Automata Algorithm Works

### 4.1 **CA Lattice Structure**

**3D Cubic Lattice**:
- Each cell: cubic volume element with side length `h`
- **Cell indexing**: 
  - Local: 0 to `nxLocal*nyLocal*nzLocal - 1` (owned by processor)
  - Local-with-ghost: 0 to `nxLocGhost*nyLocGhost*nzLocGhost - 1` (includes boundary)
  - Global: 0 to `nx*ny*nz - 1` (entire domain)

**Cell Data**:
```
cellTemperature_[i]     → T at cell i (from FEM interpolation)
cellGrainID_[i]         → Grain index owning cell i (local, -1 if unowned)
cellOrientationID_[i]   → Crystallographic orientation ID (global, shared)
cellNucleationDT_[i]    → Critical undercooling for nucleation at site i
cooling_[i]             → Boolean flag (temp decreasing?)
hasBeenMelted_[i]       → Flag for remelting problems
```

**Connectivity**:
- **2D**: 8 neighbors (von Neumann + diagonal)
- **3D**: 26 neighbors (face + edge + corner neighbors)
- Retrieved via: `get_cell_neighbors(cellID, neighborVector)`

### 4.2 **CA Kernel: Single Integration Step**

```cpp
void single_integration_step():
  // Step 1: Nucleation
  for each nucleation site i:
    if cellTemperature_[i] < T_liquidus - cellNucleationDT_[i]:
      Create new Grain at cell i with random orientation
      Add to activeGrainList_
      Mark cellGrainID_[i] = grain->id_
  
  // Step 2: Growth
  for each active grain g:
    // Compute growth parameters
    deltaT = T_liquidus - cellTemperature_[g->cell_]
    v_g = compute_growth_velocity(g, deltaT)  // Rappaz-Gandin
    g->grow(v_g, dT)  // Increment length by v·dT
    
    // Capture cells inside envelope
    for each cell j in domain:
      centerCoord = get_cell_center(j)
      if envelope_->cell_is_captured(g, centerCoord):
        if cellGrainID_[j] == -1:  // Unowned cell
          cellGrainID_[j] = g->id_
          cellOrientationID_[j] = g->orientation_->id_
        elif cellGrainID_[j] == g->id_:  // Already owned, skip
          continue
        else:  // Contested cell (growth from multiple grains)
          // Resolution: cell goes to grain reaching it first
          // (in practice: compare distances or use priority)
  
  // Step 3: Deactivate grains
  for each active grain g:
    if all_neighbors_of_g_are_captured_or_liquid:
      g->inactivate()
  
  // Step 4: Parallel Communication
  communicate_orientation_updates()  // MPI sync of cellOrientationID_
  update_global_orientation_list()   // Merge local orientations globally
```

### 4.3 **Grain Growth Envelope (Octahedron in 3D)**

**Geometry**:
- Octahedron with half-diagonal length `L` (from grain center)
- 8 faces aligned with crystal axes (after rotation by Euler angles)
- Shape encodes crystal anisotropy (growth faster along certain directions)

**Cell Capture Logic**:
```
Octahedron::cell_is_captured(grain, cellCenter):
  // Transform cell center to grain-local coordinates
  localCoord = rotate(cellCenter - grain->center, -grain->orientation)
  
  // Test containment in axis-aligned octahedron
  if |x_local| + |y_local| + |z_local| <= L:
    return true
  return false
```

**Coordinate Transformation**:
- Grain orientation stored as Euler angles or rotation matrix
- Enables anisotropic growth (different rates along different crystal axes)

### 4.4 **Adaptive Time Stepping**

```cpp
double calculate_time_step():
  v_max = 0
  for each active grain g:
    v = compute_growth_velocity(g, T_current)
    v_max = max(v_max, v)
  
  if v_max > 0:
    // Ensure cell doesn't grow more than ~1 cell per timestep
    dT = timeStepFactor * h / v_max
  else:
    dT = maxTimestep  // Solid region, use max
  
  return min(dT, maxTimestep)
```
- **Stability**: Avoids spurious oscillations from excessively fast growth
- **Efficiency**: Larger steps when growth is slow

### 4.5 **Parallel Decomposition (MPI)**

**Domain Decomposition**:
- Each rank owns `(nxLocal, nyLocal, nzLocal)` cells
- **Ghost cells**: 1-layer halo around subdomain
  - Needed for neighbor queries at boundaries
  - Communicated after each grain growth step

**ID Mapping**:
```
convert_id_locghost_to_global(locGhostID):
  (ix, iy, iz) = decompose(locGhostID)
  ix_global = xStart_ + ix - 1  // -1 removes ghost offset
  iy_global = yStart_ + iy - 1
  iz_global = zStart_ + iz - 1
  return ix_global * ny * nz + iy_global * nz + iz_global

convert_id_global_to_locghost(globalID):
  (ix, iy, iz) = decompose(globalID)
  ix_local = ix - xStart_ + 1  // +1 adds ghost offset
  iy_local = iy - yStart_ + 1
  iz_local = iz - zStart_ + 1
  return ix_local * (nyLocal+2) * (nzLocal+2) + ...
```

**Communication Pattern** (in `time_integration_weak_coupling()`):
1. Each rank advances CA independently for `dT_min`
2. After growth, communicate new `cellOrientationID_` values to neighbors
3. Synchronize global orientation list across all ranks: `MPI_Allgather`
4. Ensures consistent grain IDs for post-processing

---

## 5. Input/Output Mechanisms

### 5.1 **Input Configuration** (YAML-based)

**File Format**: `*.i` (example: [example/nucleation/Inputfile.i](example/nucleation/Inputfile.i))

**Top-level Sections**:

```yaml
simulations:
  - name: test
    time_integrator: ti_1

solvers:
  - cellular_automata    # or: [cellular_automata, finite_element_method]

realms:
  - name: realm1
    type: cellular_automata  # or: finite_element, cellular_automata_remelting
    dimension: 3
    
    domain:
      type: cubic
      original_point: [x₀, y₀, z₀]
      lateral_sizes: [Lx, Ly, Lz]
    
    discretization:
      cell_size: h            # CA cell size
      number_cells: [nx, ny, nz]
    
    nucleation_rules:
      - surface:
          type: Gaussian
          site_density: ns
          mean: μ_s
          standard_deviation: σ_s
      - bulk:
          type: Gaussian
          site_density: nv
          mean: μ_v
          standard_deviation: σ_v
    
    problem_physics:
      type: RappazGandin
      initial_temperature: T₀
      melting_temperature: T_L
      t_dot: dT/dt (cooling rate)
      a1, a2, a3: Growth coefficients
    
    solution_options:
      options:
        - random_type: rejection_sampling
        - cell_grain_length_cutoff_factor: (truncation)
        - output_microstructure_information: file_name
        - parallel_specification: y  # preferred distribution axis
    
    output:
      output_data_base_name: ./Results/Grains
      output_frequency: 100
      output_variables: [grain_ID, temperature, ...]

time_integrators:
  - standard_time_integrator:
      name: ti_1
      termination_time: T_end
      termination_step_count: N_steps_max
      time_step: Δt_FEM
      time_step_factor_ca: Factor_CA  # Multiplier for CA timestep
      realms: [realm1]
```

**Parsing** (in [CafeSolver.cpp](CA/src/CafeSolver.cpp) via YAML-cpp library):
- Uses `get_required()`, `get_if_present()` helpers
- Validates critical parameters; throws exceptions on parse errors
- Supports multiple realms (CA + FEM) for coupled runs

### 5.2 **Command-line Interface**

[Cafe.cpp](CA/src/Cafe.cpp) handles:
```bash
./Cafe -i input_file.i -o log_file.log
./Cafe --help
./Cafe --version
```
- `-i` (default: `cafe.i`): Input configuration file
- `-o`: Optional log file name (default: `{input_file}.log`)

### 5.3 **Output Mechanisms**

#### **VTK/VTU Output** (VtuManager, VtrManager)
- **Purpose**: Paraview-compatible mesh visualization
- **Variables** (configured in YAML):
  - `temperature`: Cell temperature field
  - `grain_ID`: Grain ownership (color-coded)
  - `grain_orientation`: Euler angles (3D vectors)
  - `element_birth`: Time when cell solidified
  - `melted_flag`: Boolean phase state
  - `phase_ID`: Liquid/solid/mushy state code
- **Frequency**: Every `output_frequency` timesteps
- **File naming**: `{output_data_base_name}_{iStep}.vtu`

#### **Microstructure Information File** (Text)
- Filename: Configured via `output_microstructure_information`
- **Content** (per file `{name}_{procID}.txt`):
  ```
  numCells  numOrientations  numLocalGrains  numGlobalExpectedGrains  ...  dCell
  cellID  grainID  orientationID  xc  yc  zc  length  ...
  ...
  ```
- **Purpose**: For post-processing, remelting simulations, or restart capability

#### **Timing Statistics** (stdout / log)
- **Per-rank timing**:
  - CA computation time
  - FEM computation time
  - MPI communication overhead
- **Global statistics**:
  - Max/min across all ranks
  - Identifies load imbalance
- **Final summary**: Total runtime, grain count

#### **Console Output** (stdout + optional log file)
- **High-level banner**: CAFE ASCII art logo
- **Configuration summary**: Domain size, cell count, nucleation params
- **Time step diagnostics**: dT_CA, dT_FE, selected dT_min
- **Nucleation events**: (If verbose logging enabled)
- **Final statistics**: Number of grains at termination

### 5.4 **File Organization**

```
AMCA3D/
├── CA/
│   ├── include/           # Header files
│   ├── src/              # Implementation (.cpp)
│   └── CMakeLists.txt    # Build config
├── example/
│   ├── nucleation/       # Pure nucleation+growth test
│   │   └── Inputfile.i
│   └── PBF_AM/           # Powder bed fusion example
│       └── PBF_x/, PBF_y/
├── Results/              # Output directory (created at runtime)
│   └── Grains_*.vtu      # VTK unstructured mesh files
└── README.md, LICENSE, CMakeLists.txt
```

---

## 6. Data Flow Architecture

### 6.1 **High-level Data Flow Diagram**

```
┌─────────────────────────────────────────────────────────────────┐
│                     CAFE SIMULATION LOOP                         │
└─────────────────────────────────────────────────────────────────┘

INPUT: cafe.i (YAML)
  │
  ├─→ CafeSolver
  │    ├─→ FiniteElementManager
  │    │    ├─ Load FE mesh (nodes, elements)
  │    │    ├─ Read boundary conditions (Dirichlet/Neumann on temperature)
  │    │    └─ Set initial temperature T₀
  │    │
  │    └─→ CellularAutomataManager_3D
  │         ├─ Allocate CA domain (nx, ny, nz cells)
  │         ├─ Create nucleation sites (random sampling)
  │         ├─ Initialize cell arrays: T, grainID, orientationID
  │         └─ Set up MPI domain decomposition
  │
  ├─→ MapVoxelManager
  │    └─ Build interpolation weights: FE nodes → CA cell centers
  │
  └──────────────────────┐
                         │
          TIME LOOP: while (t < t_end)
          │
          ├─ [Read FEM solution at time t]
          │  FEM load history file or compute from previous step
          │  Extract T(nodes, t)
          │
          ├─ [Interpolate to CA]
          │  MapVoxelManager::interpolate()
          │  cellTemperature_[i] = Σ N_A(cellCenter) * T_A(nodes)
          │
          ├─ [Update CA phase state]
          │  For each cell i:
          │    if T[i] > T_L:        liquid
          │    elif T_S < T[i] ≤ T_L: mushy
          │    else:                  solid
          │
          ├─ [Compute adaptive timestep]
          │  dT_CA = min(dT_max, Factor * h / v_max)
          │  dT = MPI_Allreduce(min, dT_CA)
          │
          ├─ [Nucleate new grains]
          │  For i in nucleationCellIDList_:
          │    if T[i] < T_S - ΔT_c[i]:
          │      grain = create_new_grain(i, orientation_random)
          │      grainVector_.push_back(grain)
          │      activeGrainList_.push_back(grain)
          │      cellGrainID_[i] = grain.id
          │
          ├─ [Grow existing grains]
          │  For each grain g in activeGrainList_:
          │    v_g = compute_growth_velocity(g, T_S - T[g.cell])
          │    g.length += v_g * dT
          │    For each cell j in domain:
          │      centerCoord = get_cell_center(j)
          │      if envelope_octahedron->cell_is_captured(g, centerCoord):
          │        if cellGrainID_[j] < 0:
          │          cellGrainID_[j] = g.id
          │          cellOrientationID_[j] = g.orientation.id
          │
          ├─ [Parallel communication]
          │  MPI ghost cell exchange: cellGrainID_, cellOrientationID_
          │  MPI_Allgather: Synchronize global orientation vector
          │
          ├─ [Output diagnostics]
          │  Print time, step, grains_active, dT, v_max, ...
          │  If (step % output_freq == 0):
          │    VtuManager::write_mesh(cellGrainID_, cellTemp, ..., "step_i.vtu")
          │
          ├─ [Advance time]
          │  t += dT
          │  iStep += 1
          │
          └─ [Update FEM field]
             FEM advances to time t (one-way: no feedback from CA)
             (Implementation: pseudo_single_integration_step reads history)

OUTPUT: Results/
  ├─ Grains_*.vtu      (VTK unstructured mesh files)
  ├─ *.log             (Console output / timing)
  └─ grainInfo_*.txt   (Microstructure data per rank)
```

### 6.2 **Object Lifetime and Ownership**

```
Simulation (stack)
  │
  └─→ Solver (heap, deleted in ~Simulation)
       │
       ├─→ CafeSolver
       │   │
       │   ├─→ FiniteElementManager (heap)
       │   │   ├─→ std::vector<Node> nodeVector_
       │   │   ├─→ std::vector<Element*> elementVector_
       │   │   └─→ VtuManager* vtuManager_
       │   │
       │   ├─→ CellularAutomataManager_3D (heap)
       │   │   ├─→ std::vector<Grain*> grainVector_
       │   │   │   └─→ Each Grain → Orientation* (shared, in global vector)
       │   │   ├─→ std::vector<Orientation*> orientationVector_
       │   │   ├─→ Octahedron* envelope_
       │   │   ├─→ ParallelCommManager_3D* commManager_
       │   │   └─→ VtrManager* vtrManager_
       │   │
       │   └─→ MapVoxelManager (heap)
       │
       └─→ Timer (heap)
```

### 6.3 **Critical Data Dependencies**

| Consumer | Producer | Data | Frequency |
|----------|----------|------|-----------|
| CA | FEM | `cellTemperature_[i]` | Each timestep via MapVoxelManager |
| CA phase | CA | `cellTemperature_[i]`, `T_S`, `T_L` | Each timestep |
| CA nucleation | CA | `cellTemperature_[i]`, `cellNucleationDT_[i]` | Each timestep |
| CA growth | CA | `cellTemperature_[i]`, grain state | Each substep |
| Output | CA | `cellGrainID_`, `cellOrientationID_`, `cellTemperature_` | Output frequency |
| Timing | CA+FEM | Accumulated wall clock time | At finalize |
| MPI comm | CA | Local `cellOrientationID_`, `grainVector_` | After growth step |

---

## 7. Key Algorithms and Methods

### 7.1 **Nucleation Algorithm**
```
create_nucleation_sites(numberDensity, μ_ΔT, σ_ΔT, surfacePopulation):
  Get list of candidate cells (surface or bulk)
  NumSites = numberDensity * volumePerCell * numCells
  
  for iSite = 1 to NumSites:
    Select random cell from candidates
    ΔT_c ~ N(μ_ΔT, σ_ΔT)  // Gaussian undercooling distribution
    if cellNucleationDT_[cell] < 0:  // First site here
      nucleationCellIDList_.push(cell)
      cellNucleationDT_[cell] = max(0, ΔT_c)
    elif ΔT_c < cellNucleationDT_[cell]:
      Update cellNucleationDT_[cell] = ΔT_c  // Lower threshold
```

### 7.2 **Rappaz-Gandin Growth Model**
```cpp
compute_growth_velocity(grain, time):
  ΔT = T_solidus - cellTemperature_[grain->cell_]
  v = a1 + a2 * ΔT + a3 * ΔT²  // Polynomial fit to experiments
  return max(0, v)  // Clip negative (no shrinking)
```
- **Physics**: Growth velocity ∝ undercooling
- **Tuning**: `a1, a2, a3` fit to solidification curves for specific material

### 7.3 **Cell Capture (Octahedral Envelope)**
```cpp
Octahedron::cell_is_captured(grain, cellCenter):
  // Transform to grain-centered, grain-oriented coords
  Δr = cellCenter - grain->center
  Δr_local = Rotation(-grain->orientation) * Δr
  
  // Check if inside octahedron
  sum = |Δr_local.x| + |Δr_local.y| + |Δr_local.z|
  
  if sum <= grain->length * (1 - truncationCoefficient_):
    return true  // Inside
  return false
```
- **Anisotropy**: Octahedral shape (not sphere) reflects crystal symmetry
- **Truncation**: Avoids excessive grain expansion artifact

### 7.4 **Adaptive Time Stepping**
```cpp
calculate_time_step():
  v_max = 0
  for grain g in activeGrainList_:
    v_g = compute_growth_velocity(g, time)
    v_max = max(v_max, v_g)
  
  if v_max > EPSILON:
    dT = timeStepFactor * cellSize / v_max
  else:
    dT = maxTimestep
  
  return min(dT, maxTimestep)
```
- **Rationale**: Restrict per-timestep grain growth to ~`timeStepFactor` cells
- **Typical factor**: 0.2 (grain advances 0.2 cell lengths per step)

---

## 8. Parallel Computing Strategy (MPI)

### 8.1 **Domain Decomposition**
- **Global mesh**: `(nx, ny, nz)` cells
- **Cartesian MPI topology**: 3D grid of processors
- **Each rank**: `(nxLocal, nyLocal, nzLocal)` cells + 1-layer ghost cells
- **Communication**: After each CA step, sync ghost cells with neighbors

### 8.2 **MPI Synchronization Points**
1. **Cell ID mapping**: `convert_id_*()` functions use `MPI_Comm_rank`, `MPI_Comm_size`
2. **Timestep selection**: `MPI_Allreduce(..., MPI_MIN)` global minimum
3. **Orientation sync**: `MPI_Allgather` all-to-all global orientation vector
4. **Timing stats**: `MPI_Allreduce(..., MPI_MAX/MPI_MIN)` max/min wall times

### 8.3 **Communication Pattern Example**
```
Processor 0    Processor 1    Processor 2
┌──────────┐   ┌──────────┐   ┌──────────┐
│ CA cells │   │ CA cells │   │ CA cells │
│ + ghosts │   │ + ghosts │   │ + ghosts │
└──────────┘   └──────────┘   └──────────┘
   │                 │               │
   ├─ Grow grains ───┼─ Grow grains ─┤
   │                 │               │
   └─ Exchange ghost cells ─→ ←──────┘
                     ↓
            MPI_Allreduce(dT_min)
                     ↓
         All ranks advance with same dT
```

---

## 9. Summary Table: Key Components

| Component | Purpose | Lifetime | Key Data |
|-----------|---------|----------|----------|
| **Simulation** | Top-level orchestrator | Stack | `solver_`, solver type flags |
| **CafeSolver** | Manages CA-FEM coupling | Heap | CA/FEM managers, timestep params |
| **CellularAutomataManager_3D** | CA kernel | Heap | Cells, grains, orientations |
| **Grain** | Crystal entity | Heap (in vector) | ID, center, size, orientation |
| **Orientation** | Crystal direction | Heap (global vector) | ID, Euler angles |
| **Octahedron** | Growth envelope shape | Heap | Transformation matrix, truncation |
| **FiniteElementManager** | Thermal solver | Heap | Nodes, elements, temperature |
| **MapVoxelManager** | Interpolation bridge | Heap | Interpolation weights |
| **ProblemPhysics** | Physics model | Heap | Material params, growth function |
| **Timer** | Performance tracking | Heap | Accumulated wall times |
| **VtuManager/VtrManager** | Output writing | Heap | Mesh + field data |

---

## 10. Key Input Parameters (From Example)

```
Domain size:           30 mm × 30 mm × 30 mm
Cell size (h):         0.5 mm
Number of cells:       800 × 720 × 252
Simulation time:       17.81 s
Time step (FEM):       0.001 s
Time step factor (CA): 0.2 (adaptive within FEM step)

Nucleation:
  Surface: site_density = 0 (no surface nucleation)
  Bulk:    site_density = 10e6 m⁻³
           ΔT_c ~ N(3, 1) K
  
Cooling rate:          dT/dt = -20 K/s
Material:              (Rappaz-Gandin coefficients)
                       a1 = -0.544e-4 mm/s
                       a2 = 2.03e-4 mm/(s·K)
```

---

## 11. Limitations and Design Assumptions

1. **One-way Coupling**: FEM → CA only; grain growth does not affect thermal properties
2. **No Recoil/Latent Heat**: Solid-liquid interface treatment simplified
3. **Isothermal Boundaries**: Phase change heat not explicitly modeled
4. **Cubic Cells**: CA resolution limited by cell size (no sub-cell features)
5. **Weak Coupling**: CA and FEM share same timestep; no subcycling
6. **Material-Specific**: Physics model (Rappaz-Gandin) requires fitting to each alloy
7. **Rectilinear Domain**: Assumes axis-aligned cubic cells (no unstructured CA mesh)

---

## 12. Extension Points

Future enhancements could include:
- **Two-way coupling**: Include latent heat release in FEM
- **Solute transport**: Alloying effects on growth velocity
- **Texture evolution**: Preferred grain orientation tracking
- **Remelting**: Multiple laser passes (CellularAutomataManager_3DRemelting partially implemented)
- **Mechanics**: Thermal stress effects (Node velocity member unused)
- **Adaptive mesh**: Dynamic CA cell refinement in mushy zone
