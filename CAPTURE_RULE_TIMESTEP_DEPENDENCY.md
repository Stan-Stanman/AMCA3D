# Capture Rule and Timestep Dependency in AMCA3D

This document explains the **Capture Rule** within the Cellular Automata (CA) grain growth model of AMCA3D and details exactly how it depends on the integration time step ($\Delta t$).

---

## 1. The Core Capture Mechanism

In cellular automata grain growth, a grain expands from its nucleus into neighboring liquid cells. The interface of each grain is represented by a 3D growth envelope (an octahedron matching the cubic crystal symmetry of the grain).

A cell is captured when its center falls inside the growing envelope of a neighboring grain:

$$\text{cell\_center} \in \text{Envelope}(L(t), \theta)$$

Where:
*   $L(t)$ is the current diagonal length of the octahedron.
*   $\theta$ is the grain's crystal orientation (Euler angles).

### Step-by-Step Execution:
1. **Grow Grain Envelope:** Update the length of all active grains based on growth velocity and the time step:
   $$L(t + \Delta t) = L(t) + v(\Delta T) \cdot \Delta t$$
   *(Implemented in `Grain::grow` in [`Grain.h`](file:///home/stan/AMCA3D/CA/include/Grain.h#L27))*

2. **Evaluate Capture Criterion:** In [`Octahedron::cell_is_captured`](file:///home/stan/AMCA3D/CA/src/Octahedron.cpp#L99-L128), check if neighbor cell center coordinates in the grain's local coordinate system $(c_x, c_y, c_z)$ satisfy:
   $$|c_x| + |c_y| + |c_z| \le L(t)$$

3. **Spawning the Captured Cell:** If a cell is captured, a new grain growth center is initialized at that cell using [`Octahedron::capture_cell`](file:///home/stan/AMCA3D/CA/src/Octahedron.cpp#L133-L275).

---

## 2. Dynamic Timestep Calculation (Stability & Accuracy)

The capture rule depends heavily on the size of the time step $\Delta t$ (variable `dT_` in code). If $\Delta t$ is too large, a grain could grow by multiple grid spacings in a single step, causing:
1. Grains capturing cells multiple layers away instantly, which bypasses intervening liquid cells.
2. Large numerical anisotropy and grid orientation bias.
3. Inaccurate competition/collision boundaries between adjacent grains.

To resolve this, AMCA3D implements a dynamic timestep limitation similar to the Courant-Friedrichs-Lewy (CFL) condition.

### The Equation:
$$\Delta t \le \text{timeStepFactor\_} \cdot \frac{h}{v_{\text{max}}}$$

Where:
*   $h$ is the cell size (`h_`).
*   $v_{\text{max}}$ is the maximum growth velocity among all active grains.
*   `timeStepFactor_` is a safety factor, defaulting to `0.2` (initialized in [`CellularAutomataManager.cpp`](file:///home/stan/AMCA3D/CA/src/CellularAutomataManager.cpp#L44)).

### Code Reference:
In [`CellularAutomataManager_3D::calculate_time_step()`](file:///home/stan/AMCA3D/CA/src/CellularAutomataManager_3D.cpp#L441-L511):
```cpp
double dtLoc = std::min(timeStepFactor_*h_ / maxVelocity, maxTimestep_);
MPI_Allreduce(&dtLoc, &dT_, 1, MPI_DOUBLE, MPI_MIN, MPI_COMM_WORLD);
return dT_;
```

This ensures that the maximum growth in a single step is limited to:
$$\Delta L_{\text{max}} = v_{\text{max}} \cdot \Delta t \le 0.2 \cdot h$$
 Grains grow at most **20% of a single cell width** per time step, ensuring smooth front propagation and correct capture sequencing.

---

## 3. Decentered Octahedron Spawning (Length and Center Dependency)

When a cell is captured, a new octahedron is created at the new cell center. To prevent grid-induced shape distortion, Rappaz and Gandin proposed the **de-centered growth algorithm**. 

The new grain's initial length ($L_{\text{new}}$) and center ($X_{\text{new}}$) are directly calculated based on the parent grain's current length $L(t)$ at that specific time step.

### Code Reference:
In [`Octahedron::capture_cell`](file:///home/stan/AMCA3D/CA/src/Octahedron.cpp#L133-L275):
1. **Local Interception calculation:**
   ```cpp
   double interception = fabs(cx) + fabs(cy) + fabs(cz); // <= parent length
   ```
2. **Perpendicular distance to parent face:**
   ```cpp
   double perpendicularDis = length - interception;
   ```
3. **Initial size of the new octahedron:**
   ```cpp
   double L1 = 0.5 * (std::min(Is1[0], truncatedValue) + std::min(Is1[1], truncatedValue));
   double L2 = 0.5 * (std::min(Js1[0], truncatedValue) + std::min(Js1[1], truncatedValue));
   double newGrainLength = sqrtf(2) * (std::max(L1, L2));
   ```
4. **New center location translation:**
   ```cpp
   double magnitude = (length - newGrainLength);
   // Coordinates are translated along the crystal orientation vector by `magnitude`
   ```

Because $L_{\text{new}}$ and $X_{\text{new}}$ are computed using the parent's `length` ($L(t)$), they inherit the direct history of the time steps $\Delta t$ used to grow the parent.
