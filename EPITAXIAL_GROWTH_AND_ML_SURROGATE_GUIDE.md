# Guide to Epitaxial Growth and U-Net Surrogate Modeling for AMCA3D

This document summarizes the physics of epitaxial growth within the Cellular Automata (CA) model, the intricacies of coupling with Finite Element (FE) solvers like Abaqus, and the architectural strategy for building a Probabilistic U-Net surrogate model.

## 1. Epitaxial Growth in the CA Model

Epitaxial growth occurs when grains from a previously solidified layer (the substrate) act as seeds for the new melt pool, growing directly into the liquid without a new nucleation event.

### How it Works Physically and Computationally
*   **Local Phenomenon:** Crystallographically, epitaxial growth is strictly local. The CA only looks at the immediate solid-liquid interface. It does not matter if the substrate grain extends 10 µm or 10 cm below the interface; the orientation inherited is identical.
*   **Implementation in AMCA3D:** The model handles this via the `CellularAutomataManager_3DRemelting` solver. Using the `activate_existing_grains` flag, the solver identifies liquid cells, checks their solid neighbors (the surviving substrate), and if the region is cooling, it *reactivates* those solid grains. These grains then resume growth into the melt pool.
*   **CA Domain Boundaries:** The CA only tracks voxels explicitly within its finite grid. The macroscopic thermal effects of a deep substrate (acting as a huge heat sink) are completely handled by the FE solver (Abaqus) and passed to the CA via the temperature field.

## 2. Building the Dataset for the U-Net Surrogate

To train a Probabilistic U-Net to act as an iterative, layer-by-layer surrogate, the dataset must teach the network how epitaxial grains compete with new equiaxed nucleation (the Columnar-to-Equiaxed Transition, or CET).

### Dataset Generation Strategy
*   **Mesoscale Track Models:** Do not simulate entire macroscopic parts in the FE solver for training data. Run thousands of small-block simulations (e.g., $64 \times 64 \times 64$ voxels) with a highly refined static mesh in Abaqus to simulate 1 or 2 laser tracks.
*   **Vary the Inputs:** 
    *   Vary the initial substrate (different pre-solidified grain structures in the bottom half of the block).
    *   Vary the thermal fields (different melt pool depths and cooling rates).
*   **Stochasticity:** Run the identical thermal/substrate setup multiple times with different CA random seeds. This teaches the probabilistic U-Net's latent space the natural randomness of new grain nucleation.

### Tensor Formulation
*   **Input Tensors ($X$):**
    1.  *Melt Boundary/Substrate:* The orientations of the substrate cells that survived the melt pool ($T_{max} < T_{liquidus}$). This defines exactly where epitaxial growth starts.
    2.  *Thermal Driving Forces:* Average Thermal Gradient ($G$) and Average Cooling Rate ($\dot{T}$) during the solidification window (the mushy zone).
*   **Output Tensors ($Y$):** The final 3D solidified microstructure. **Crucial:** Convert orientations from Euler angles to Quaternions (4 channels) or Continuous 6D representations to avoid neural network training singularities (like Gimbal lock).

## 3. Abaqus vs. CA: Thermal Gradients and Coupling

A critical aspect of multiscale modeling is understanding *who computes what* and *why*.

### Why Gradients Come From Abaqus, Not CA
*   The CA is a discrete state machine. It numerically solves ordinary differential equations for grain length ($dL/dt$), but it does not solve the continuous spatial partial differential heat equation ($\nabla \cdot (k \nabla T)$).
*   Abaqus (the FE solver) calculates the macroscopic heat transfer, including conduction into the deep substrate and the macroscopic release of Latent Heat of fusion. The CA simply reads the resulting temperature field $T(x,y,z,t)$ and derives the gradients from it.

### Overcoming Spatiotemporal Mismatches
1.  **Spatial Resolution:** A standard part-level Abaqus mesh is too coarse (e.g., 100 µm elements) and will smear out the extreme gradients of the melt pool. The CA (running at ~1 µm) will produce garbage if fed smeared data. **Solution:** Use a highly refined, static dense mesh in Abaqus for your small training blocks to capture true gradients perfectly.
2.  **Temporal Resolution (Micro-steps):** The CA takes micro-steps (e.g., $10^{-6}$ s) for numerical stability to prevent growth envelopes from overshooting cells. It simply linearly interpolates the Abaqus output frames. The CA does *not* discover new high-frequency thermal fluctuations.
3.  **U-Net Simplification:** Your U-Net does not need to learn these numerical micro-steps. The final physical microstructure is dictated by the *average* $G$ and $\dot{T}$ in the mushy zone. The U-Net can map directly from the initial thermal/substrate state to the final grain structure, skipping the micro-steps entirely.

### Two-Way Coupling and Latent Heat
While microscopic grain evolution releases latent heat that alters local gradients, attempting "Strong (Two-Way) Coupling" (where the CA constantly updates the FE solver based on individual voxel solidification) is computationally devastating and rarely done. The industry standard is **Weak Coupling**: Abaqus handles latent heat macroscopically (using an effective specific heat for the phase change), and passes the temperatures one-way to the CA. This captures the vast majority of the relevant physics for Additive Manufacturing.
