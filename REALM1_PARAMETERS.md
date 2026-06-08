# Realm1 (Cellular Automata) Parameters Guide

This document describes all available parameters for the `cellular_automata` realm in AMCA3D input files.

## Table of Contents
1. [Domain Parameters](#domain-parameters)
2. [Discretization Parameters](#discretization-parameters)
3. [Nucleation Physics Parameters](#nucleation-physics-parameters)
4. [Growth Physics Parameters](#growth-physics-parameters)
5. [Solution Options](#solution-options)
6. [Output Parameters](#output-parameters)

---

## Domain Parameters

These parameters define the simulation domain geometry and location.

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `type` | string | `cubic` | Domain shape. Options: `cubic` (only supported currently) |
| `original_point` | array [x, y, z] | `[0, 0, 0]` | Coordinates of the domain origin point |
| `lateral_sizes` | array [x, y, z] | `[0.03, 0.03, 0.03]` | Physical dimensions of the domain in each direction (in physical units) |

### Example:
```yaml
domain:
  type: cubic
  original_point: [0, 0, 0]
  lateral_sizes: [0.03, 0.03, 0.03]
```

---

## Discretization Parameters

These parameters control how the domain is divided into discrete cells (mesh resolution).

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `cell_size` | float | `0.0005` | Physical size of each cubic cell. Controls mesh refinement. |
| `number_cells` | array [x, y, z] | `[800, 720, 252]` | Number of cells in each direction. Alternative to `cell_size` (computed automatically if only `cell_size` is given) |

**Note:** Smaller `cell_size` = finer mesh = more accurate but slower simulation.

**Relationship:** 
```
number_cells[i] = lateral_sizes[i] / cell_size
```

### Example:
```yaml
discretization:
  cell_size: 0.0005
  number_cells: [800, 720, 252]
```

---

## Nucleation Physics Parameters

These parameters control where and when grains nucleate (form new crystals).

### Structure:
```yaml
nucleation_rules:
  - surface:
      type: Gaussian
      site_density: 0.0
      mean: 2
      standard_deviation: 0.5
  
  - bulk:
      type: Gaussian
      site_density: 10e6
      mean: 3
      standard_deviation: 1
```

### Parameters:

| Parameter | Location | Type | Example | Description |
|-----------|----------|------|---------|-------------|
| `type` | surface/bulk | string | `Gaussian` | Distribution of critical undercooling. Currently supports `Gaussian` |
| `site_density` | surface/bulk | float | `10e6` | Number of nucleation sites per unit volume (sites/m³) |
| `mean` | surface/bulk | float | `3` | Mean critical undercooling ΔT_c (dimensionless units) |
| `standard_deviation` | surface/bulk | float | `1` | Standard deviation of the Gaussian distribution (dimensionless units) |

### Interpretation:

- **Surface**: Nucleation at the boundary/surface of the liquid
- **Bulk**: Nucleation inside the liquid volume
- Nucleation occurs when local undercooling exceeds the critical undercooling of that site
- Higher `site_density` = more nucleation sites = finer grains
- Higher `mean` = nucleation occurs at lower temperatures (later in solidification)

### Physical Meaning:
```
N_sites = site_density × Volume
ΔT_c ~ Gaussian(mean, std_dev)
Nucleation triggered when: ΔT_local > ΔT_c
```

---

## Growth Physics Parameters

These parameters control how grains grow once nucleated.

### Structure:
```yaml
problem_physics:
    type: RappazGandin
    initial_temperature: -0.2
    melting_temperature: 0.0
    t_dot: -20
    a1: -0.544e-4 
    a2: 2.03e-4
    a3: 0.0
```

### Parameters:

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `type` | string | `RappazGandin` | Growth kinetics model. Currently only `RappazGandin` supported |
| `initial_temperature` | float | `-0.2` | Starting temperature of the domain (dimensionless: T - T_melt) |
| `melting_temperature` | float | `0.0` | Reference melting temperature (dimensionless: ΔT = 0 at melting) |
| `t_dot` | float | `-20` | Cooling rate (K/s, or dimensionless time derivative) |
| `a1` | float | `-0.544e-4` | Linear coefficient of growth law: v = a1·ΔT + a2·ΔT² + a3·ΔT³ |
| `a2` | float | `2.03e-4` | Quadratic coefficient of growth law |
| `a3` | float | `0.0` | Cubic coefficient of growth law |

### Growth Law:

The interface velocity is computed from undercooling:
```
v(ΔT) = a1·ΔT + a2·ΔT² + a3·ΔT³
```

Where:
- `v` = grain growth velocity (cell/time step)
- `ΔT` = local undercooling (T_melt - T_local)
- `a1, a2, a3` = material-specific kinetic coefficients

### Physical Meaning:

- **Nucleation Example** (`t_dot: -20`): Simple idealized cooling → uniform grain structure
- **Real Materials** (e.g., Inconel): Realistic material coefficients → dendrite formation

---

## Solution Options

These parameters control simulation behavior and output.

### Structure:
```yaml
solution_options:
   name: my_options
   options:
     - random_type:
         rejection_sampling: no
     - cell_grain_length_cutoff_factor:
         factor_value: 4.0
     - output_microstructure_information:
         file_name: grainInfo.txt
         output_seeds: yes
     - load_microstructure_information:
         file_name: random
     - criterion_to_activate_grain:
         using_nucleation_seed: yes
     - parallel_specification:
         direction_with_all_processes: y
     - remove_melting:
         remove_melting: yes
     - phase_state:
         has_been_melted: no
```

### Parameters:

| Option | Sub-parameter | Type | Example | Description |
|--------|---------------|------|---------|-------------|
| `random_type` | `rejection_sampling` | bool | `no` | Random number generation method. `no` = standard, `yes` = rejection sampling |
| `cell_grain_length_cutoff_factor` | `factor_value` | float | `4.0` | Cutoff multiplier for grain size determination |
| `output_microstructure_information` | `file_name` | string | `grainInfo.txt` | Save grain structure to file for restart capability |
| `output_microstructure_information` | `output_seeds` | bool | `yes` | Include nucleation seed information in output |
| `load_microstructure_information` | `file_name` | string | `./PreRun/grainInfo.txt` | Load previous grain structure (restart capability) |
| `criterion_to_activate_grain` | `using_nucleation_seed` | bool | `yes` | Activate grains based on nucleation seeds |
| `parallel_specification` | `direction_with_all_processes` | string | `y` | Direction to parallelize across processes (`x`, `y`, or `z`) |
| `remove_melting` | `remove_melting` | bool | `yes` | Remove grains that have been remelted (for AM simulations) |
| `phase_state` | `has_been_melted` | bool | `no` | Initial state of cells (for restart simulations) |

---

## Output Parameters

These parameters control what and when results are saved.

### Structure:
```yaml
output:
  output_data_base_name: ./Results/Grains
  output_frequency: 100
  output_time_interval: 0.0002
  output_variables:
    - grain_ID
    - grain_orientation
    - temperature
```

### Parameters:

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `output_data_base_name` | string | `./Results/Grains` | Directory and prefix for output files |
| `output_frequency` | integer | `100` | Save results every N time steps (use `-1` to skip) |
| `output_time_interval` | float | `0.0002` | Time interval between outputs (in simulation time units) |
| `output_variables` | array | `[grain_ID, temperature]` | Variables to save: `grain_ID`, `grain_orientation`, `temperature`, etc. |

---

## Complete Example (Nucleation)

```yaml
realms:
  - name: realm1
    type: cellular_automata
    dimension: 3

    domain:
      type: cubic
      original_point: [0, 0, 0]
      lateral_sizes: [0.03, 0.03, 0.03]

    discretization:
      cell_size: 0.0005
      number_cells: [800, 720, 252]

    nucleation_rules:
      - surface:
          type: Gaussian
          site_density: 0.0
          mean: 2
          standard_deviation: 0.5
      - bulk:
          type: Gaussian
          site_density: 10e6
          mean: 3
          standard_deviation: 1

    problem_physics:
        type: RappazGandin
        initial_temperature: -0.2
        melting_temperature: 0.0
        t_dot: -20
        a1: -0.544e-4 
        a2: 2.03e-4
        a3: 0.0

    solution_options:
       name: my_options
       options:
         - random_type:
             rejection_sampling: no
         - cell_grain_length_cutoff_factor:
             factor_value: 4.0
         - output_microstructure_information:
             file_name: grainInfo.txt
         - parallel_specification:
             direction_with_all_processes: y

    output:
      output_data_base_name: ./Results/Grains
      output_frequency: 100
      output_variables:
        - grain_ID
```

---

## Output Variables

The `output_variables` list specifies which data fields to save to the VTK output files. Below is a comprehensive list of all supported output variables.

### Available Output Variables

| Variable Name | Type | Components | Description |
|---------------|------|-----------|-------------|
| `temperature` | Scalar | 1 | Local temperature at each cell |
| `temperature_rate` | Scalar | 1 | Time derivative of temperature (dT/dt) |
| `grain_ID` | Scalar | 1 | Unique identifier for each grain/crystal |
| `grain_orientation` | Vector | 3 | Crystal orientation (direction cosines) in 3D space |
| `grain_velocity` | Vector | 3 | Growth velocity vector of the grain |
| `element_birth` | Scalar | 1 | Time step when the cell was first activated/nucleated |
| `melted_flag` | Scalar | 1 | Boolean flag: 1 if cell has been melted, 0 otherwise |
| `phase_ID` | Scalar | 1 | Phase identifier (e.g., solid, liquid, void) |
| `alpha_phase` | Scalar | 1 | Phase fraction/composition (0 to 1) |
| `phase_orientation` | Vector | 3 | Phase-specific orientation |
| `velocity` | Vector | 3 | Local velocity magnitude and direction |
| `debug` | Vector | 3 | Debug information (for development only) |
| `debugMC` | Vector | 3 | Monte Carlo debug information (for development only) |

### Usage Examples

**Minimal Output (Grain Structure):**
```yaml
output:
  output_data_base_name: ./Results/Grains
  output_frequency: 100
  output_variables:
    - grain_ID
```

**Full Microstructure Analysis:**
```yaml
output:
  output_data_base_name: ./Results/Full
  output_frequency: 50
  output_variables:
    - temperature
    - grain_ID
    - grain_orientation
    - grain_velocity
    - melted_flag
    - element_birth
```

**For Additive Manufacturing (PBF) Simulations:**
```yaml
output:
  output_data_base_name: ./Results/AM
  output_frequency: 200
  output_variables:
    - temperature           # Track thermal field
    - grain_orientation     # Grain texture
    - melted_flag           # Remelting history
    - element_birth         # Solidification sequence
```

**For CNN Training (Microstructure Data):**
```yaml
output:
  output_data_base_name: ./Results/CNN_Data
  output_frequency: 1        # Save every step for diverse samples
  output_variables:
    - grain_ID              # Primary output for training
    - grain_orientation     # Crystal texture features
    - temperature           # Thermal gradients as input
```

### Output File Format

- Output is saved in **VTK format** (`.vtr`, `.vtu`, or `.pvd` files)
- Can be visualized with **ParaView** or other VTK viewers
- Each variable creates a separate data channel in the VTK file
- Files are generated according to `output_frequency` or `output_time_interval`

### Performance Considerations

- **More variables = Larger file size** — each variable adds ~(number_of_cells × 4-12 bytes)
- **Example:** 448×16×150 cells with 5 variables ≈ 100-150 MB per output file
- For CNN training, consider:
  - Using `grain_ID` (scalar) instead of unnecessary vector variables
  - Adjusting `output_frequency` to balance data diversity vs. storage
  - Reducing `output_time_interval` to capture more diverse microstructures

### Notes

- `grain_orientation` and `phase_orientation` are stored as direction cosines (3 components for 3D)
- `melted_flag` and `phase_ID` are useful for remelting/AM simulations
- `element_birth` helps track solidification chronology
- Debug variables are typically not used for regular simulations
- For older AMCA3D versions, some variables may not be available

---

## Tips for Parameter Selection

### For Coarse Grains:
- Lower `site_density` (fewer nucleation sites)
- Higher `mean` critical undercooling (nucleation delayed)

### For Fine Grains:
- Higher `site_density` (more nucleation sites)
- Lower `mean` critical undercooling (nucleation happens earlier)

### For Faster Simulation:
- Larger `cell_size` (coarser mesh)
- Smaller domain `lateral_sizes`
- Fewer `output_frequency` calls

### For More Accurate Results:
- Smaller `cell_size` (finer mesh, but slower)
- Material-specific `a1, a2, a3` coefficients
- Realistic `nucleation_rules` from experiments

---

## References

See the main README.md for more information on:
- Nucleation physics
- Dendrite growth kinetics
- Material-specific parameters for different alloys
