# CNN Surrogate Model for Additive Manufacturing Turbine Blade
## Complete Analysis & Timeline

**Project Goal:** Create a neural network surrogate model for AMCA3D cellular automata to predict grain microstructure in turbocharged turbine blades manufactured via additive manufacturing.

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Voxel & CA Fundamentals](#voxel--ca-fundamentals)
3. [CNN Architecture Strategy](#cnn-architecture-strategy)
4. [Thermal Input Analysis](#thermal-input-analysis)
5. [Implementation Approaches](#implementation-approaches)
6. [Timeline Comparison](#timeline-comparison)
7. [Final Recommendations](#final-recommendations)

---

## Executive Summary

| Aspect | Value |
|--------|-------|
| **CNN Input Block Size** | 128 × 128 × 128 voxels |
| **Physical Domain** | 256 µm × 256 µm × 256 µm |
| **Voxel Resolution** | 2 µm per voxel |
| **Grains per Block** | 20-50 grains |
| **Training Patches Needed** | 50-100 patches |
| **Total Project Timeline** | **1-1.5 months** |
| **Key Speedup vs Full AMCA3D** | **10,000-100,000×** |

---

## Voxel & CA Fundamentals

### What is a Voxel in AMCA3D?

**Definition:** In AMCA3D, voxels are called "cells" in the cellular automata grid.

**Key Parameters:**

```yaml
domain:
  type: cubic
  original_point: [0, 0, 0]
  lateral_sizes: [Lx, Ly, Lz]          # Physical domain size

discretization:
  cell_size: h                          # Voxel size (USER-DEFINED)
  number_cells: [nx, ny, nz]            # Computed as: L/h
```

**Relationship:** `number_cells = lateral_sizes / cell_size`

### Voxel Size is NOT Fixed

- **Fixed for a given simulation:** Once set in input file, remains constant
- **Configurable:** Can change between simulations
- **Application-dependent:** Typical range 0.1-10 µm (for AM)

### Maximum Voxel Count

| Scenario | Voxels | Memory | Feasibility |
|----------|--------|--------|-------------|
| Small simulations | 100K-500K | 5-25 MB | ✓ Easy |
| Medium simulations | 1M-10M | 50-500 MB | ✓ Standard |
| Large simulations | 10M-100M | 500MB-5GB | ✓ HPC required |
| Very large | ~1 billion | ~50GB | ⚠ Extreme HPC only |

**Memory Rule of Thumb:** ~50 bytes per voxel

### Example: Actual AMCA3D Cases

**Nucleation Example:**
- Domain: 0.03 × 0.03 × 0.03 units
- Cell size: 0.0005 units
- Voxels: 60 × 60 × 60 = **216,000 voxels**

**PBF_x Example:**
- Domain: 0.448 × 0.016 × 0.15 units
- Cell size: 0.001 units
- Voxels: 448 × 16 × 150 = **1,075,200 voxels**

---

## CNN Architecture Strategy

### Block Size Decision

**Three Options Evaluated:**

#### **Option 1: Small Blocks (64³ voxels)**
- Physical size: 64 × 64 × 64 µm
- Voxel count: 262,144
- Pros: Fast, many blocks from one blade
- Cons: Loses grain interaction patterns
- **Not recommended**

#### **Option 2: Medium Blocks (128³ voxels) ⭐ RECOMMENDED**
- Physical size: 128 × 128 × 128 µm (at 1 µm/voxel) or **256 µm cube (at 2 µm/voxel)**
- Voxel count: 2.1 million
- Grains captured: 20-50 per block
- GPU memory: 100-200 MB per block (inference), 200-300 MB (training)
- Pros:
  - Captures multiple grains for interaction effects
  - Manageable GPU memory
  - Good compromise between accuracy and speed
  - Powers of 2 are ideal for CNN architectures
- Cons: Moderate computational cost
- **Recommended for turbine blade application**

#### **Option 3: Large Blocks (256³ voxels)**
- Physical size: 256 × 256 × 256 µm
- Voxel count: 17 million
- Pros: Full cross-section context
- Cons: Very slow, high memory needs (800MB-1.5GB)
- **Overkill for turbine blade**

### Recommended CNN Architecture

**Input:** 128 × 128 × 128 voxel blocks with features:
- Temperature field T(x,y,z)
- Cooling rate dT/dt
- Phase state (liquid/mushy/solid)

**Output:** Final grain microstructure OR time-evolution predictions

**Network Type:** 3D U-Net or SegNet
- Parameters: ~5-10 million
- Training time: 5-20 hours on single GPU
- Inference time: <1 second per block

### Dataset Size

From one turbine blade simulation:
- Blade volume: ~50 mm³
- Block volume (256 µm)³: 2.1 × 10⁻¹¹ m³
- Non-overlapping blocks: ~23,840
- With 50% overlap: **~190,000 possible training samples**

But for diversity: **Generate 50-100 patches with varied parameters**

---

## Thermal Input Analysis

### Key Discovery: Temperature is SMOOTH!

**Physics Principle:** Temperature governed by diffusion equation:
```
∂T/∂t = α ∇²T + source
```

**Consequence:** Gradients smooth out over physical length scale:
```
δ_thermal = √(α × Δt)
```

For Inconel 625 @ 0.5 seconds:
- α ≈ 3-4 × 10⁻⁶ m²/s
- **δ_thermal ≈ 1.4 mm**

→ Features smaller than 1 mm are effectively smoothed
→ 1-2 µm CA voxels sample a very smooth field!

### Abaqus Mesh Resolution: Proper Context

**Corrected Mesh Classification:**

| Application | Coarse | Medium | Fine |
|-------------|--------|--------|------|
| Structural mechanics | 10-50 mm | 1-10 mm | 0.1-1 mm |
| **AM Thermal** | **100-500 µm** | **50-100 µm** ✓ | **1-10 µm** |

**Your 50-100 µm mesh is APPROPRIATE for AM thermal!** (Not "coarse")

### Interpolation Error Analysis

**Question:** When interpolating from coarse Abaqus to fine CA voxels, is accuracy lost?

**Answer:** Small but acceptable error for ML training

**Quantitative Analysis:**

Melt pool thermal gradient: dT/dx ≈ 10⁶-10⁷ K/m

Over 2 µm voxel distance:
```
ΔT = (dT/dx) × 2×10⁻⁶ m ≈ 20 K
```

Grain growth model sensitivity (Rappaz-Gandin):
```
v = a₁ + a₂ × ΔT
Δv = a₂ × ΔT_error ≈ 0.001 m/(s·K) × 20 K ≈ 0.02 mm/s
Relative error in growth: ~1-2%
```

**Verdict:** ✓ **Acceptable for ML training!**

### Why Coarse Abaqus Works for CNN

1. **Temperature is physically smooth** (parabolic PDE smooths gradients)
2. **Interpolation error is small** (~1-2% in growth velocity)
3. **CNNs are robust learners** — learn patterns, not pixel-perfect data
4. **Real deployment also uses coarse data** — training on it improves robustness
5. **Data augmentation adds robustness** — noise, cooling rate variations

### Validation Strategy

To verify Abaqus coarseness is acceptable:

1. **Run 3-5 reference simulations:**
   - Generate HIGH-RESOLUTION Abaqus (fine mesh 1-2 µm in critical regions)
   - Run full CA with this data → record ground truth microstructures

2. **Generate COARSE Abaqus simulations** (your standard setup)
   - Run CA with interpolated coarse data
   - Compare to ground truth

3. **Quantify differences:**
   - Grain size distribution: Should be <10% different ✓
   - Grain count: Should be <20% different ✓
   - Spatial grain map: Should have 80%+ correlation ✓

4. **Decision:**
   - If acceptable → Proceed with coarse mesh
   - If not acceptable → Use adaptive mesh refinement (Option 2 below)

---

## Implementation Approaches

### Critical Insight: CA-Only (No FEM) Approach

**KEY DECISION:** You will import thermal data from Abaqus, NOT run FEM in AMCA3D!

**Impact on Computation:**
- FEM solver accounts for ~50-80% of coupled simulation time
- Removing FEM: **2-5× FASTER!**
- AMCA3D supports this via `LoadDataFromFile` class

**AMCA3D Configuration:**
```yaml
solvers:
   - cellular_automata              # ← Only CA, no FEM!

realms:
   - name: realm1
     type: cellular_automata
     dimension: 3
     
     # Temperature loaded from external file
     load_data_from_file:
         file_path: ./temperature_history.txt
```

### Three Implementation Options

#### **Option 1: Full Blade Thermal (Expensive)**

**Workflow:**
1. Run Abaqus on entire blade (100 µm mesh)
2. Extract temperature history for all regions
3. Extract 50-100 patches from result
4. Run CA on each patch

**Timeline:**
- Abaqus (full blade, 100 µm): 5-10 days on 8 cores
- CA generation (50 patches): 2-4 weeks parallel
- CNN training: 1 day
- **Total: 4-8 weeks**

**Pros:** Thermally consistent across blade, complete picture
**Cons:** Slow Abaqus simulation, not parallelizable

---

#### **Option 2: Small Patch Thermal (RECOMMENDED) ⭐**

**Workflow:**
1. Define 50-100 small Abaqus domains (256-512 µm cubes)
2. Run Abaqus on each patch IN PARALLEL
3. Run CA on each patch
4. Train CNN on collected data

**Timeline:**
- Abaqus (50 patches, 100-200 µm, parallel): **<1 day**
- CA generation (50 patches, 1-2 days each): **2-4 weeks**
- CNN training: **1 day**
- **Total: 2-3 weeks** ✓✓✓

**Pros:**
- Fastest Abaqus (fully parallelizable)
- Thermal variety (different positions in blade)
- Sufficient for CNN training
- Realistic (patches are independent)

**Cons:** Requires parallel infrastructure

---

#### **Option 3: Hybrid/Adaptive Mesh (Balanced)**

**Workflow:**
1. Full blade Abaqus with adaptive mesh:
   - Coarse (100-200 µm) away from melt pool
   - Fine (10-20 µm) in melt pool region
2. Extract patches
3. Run CA on each

**Timeline:**
- Abaqus (full blade, adaptive): 1-2 weeks
- CA generation: 2-4 weeks
- CNN training: 1 day
- **Total: 3-5 weeks**

**Pros:** High quality thermal data, manageable computation
**Cons:** Requires Abaqus expertise, moderate time investment

---

### Recommended Approach: **Option 2 + Data Augmentation**

**Why:**
- Fastest overall timeline (2-3 weeks)
- Parallelizable Abaqus phase
- Sufficient data diversity for CNN
- Practical for research setting

**Enhancement - Data Augmentation:**
1. Generate base 50-100 patches with standard parameters
2. Augment by:
   - Adding ±5% synthetic thermal noise
   - Varying cooling rates ±10%
   - Multiple nucleation densities
   - Different material parameter sets

**Result:** CNN trained on ~500-1000 augmented samples → highly robust!

---

## Timeline Comparison

### Detailed Abaqus Runtime Estimates

**Turbocharger Turbine Blade Geometry:**
- Blade length (radial): 20-30 mm
- Blade chord: 10-15 mm
- Blade thickness: 2-5 mm (average)
- Volume: ~50-150 mm³

**Element Count & Runtime:**

| Mesh | Domain | Elements | 1 CPU | 8 CPUs | 32 CPUs | 64 CPUs |
|------|--------|----------|-------|--------|---------|---------|
| 100 µm | Patch (256 µm) | 0.03M | 30 min | 5 min | 2 min | 1 min |
| 100 µm | Full blade | 0.75M | 12 h | 1.5 h | 30 min | 20 min |
| **50 µm** | **Patch (256 µm)** | **2.1M** | **1 h** | **10 min** | **3 min** | **2 min** |
| **50 µm** | **Full blade** | **6M** | **24 h** | **3 h** | **1 h** | **45 min** |
| 10 µm | Patch | 17M | 8 h | 1 h | 20 min | 12 min |
| 10 µm | Full blade | 450M | 1 year | 6 weeks | 2 weeks | 1 week |

*Estimates based on implicit thermal solver with phase change, ~2000 timesteps, adaptive timestepping*

### Project Timeline Scenarios

#### **Scenario A: Fast Track (Patch-based, recommended)**

```
Week 1-2:
  └─ Abaqus thermal patches (100-200 µm, 50 patches parallel)
     └─ Time: <1 day on cluster, setup ~1 week

Week 2-4:
  └─ CA-only simulations (50-100 patches, 1-2 days each)
     └─ With 5-10 node parallelization: 2-3 weeks calendar time
     └─ Run with data augmentation variations

Week 4-5:
  └─ CNN training (5-20 GPU hours)
  └─ Model validation & tuning

Week 5-6:
  └─ Deploy on full blade (minutes to run)
  └─ Final report & analysis

**TOTAL: 6 weeks** ✓
**RECOMMENDED FOR RESEARCH PROJECT**
```

#### **Scenario B: Full Blade (Conservative)**

```
Week 1-3:
  └─ Abaqus full blade thermal (100 µm mesh)
     └─ 5-10 days on 8-16 cores

Week 3-5:
  └─ Extract patches from Abaqus result (1-2 days)

Week 5-8:
  └─ CA-only simulations (50-100 patches)
     └─ 1-2 days per patch × 50-100 patches = 2-4 weeks parallel

Week 8-9:
  └─ CNN training & validation

Week 9-10:
  └─ Deployment & final analysis

**TOTAL: 10 weeks**
**MORE THOROUGH BUT SLOWER**
```

#### **Scenario C: High Quality (Adaptive mesh)**

```
Week 1-3:
  └─ Abaqus full blade (adaptive mesh: 100 µm coarse + 20 µm refined)
     └─ 1-2 weeks computation

Week 3-4:
  └─ Extract & verify patches

Week 4-7:
  └─ CA-only simulations (50-100 patches)

Week 7-9:
  └─ CNN training & validation

**TOTAL: 9 weeks**
**BALANCED: GOOD QUALITY + REASONABLE TIME**
```

---

## Final Recommendations

### For Your Specific Case (Turbine Blade CNN Surrogate)

#### ✓ **Recommended Configuration**

| Parameter | Value |
|-----------|-------|
| CNN Block Size | **128 × 128 × 128 voxels** |
| Physical Domain | **256 × 256 × 256 µm @ 2 µm/voxel** |
| Abaqus Mesh | **100 µm (full blade) OR 100-200 µm (patches)** |
| Approach | **CA-only (no FEM)** |
| Training Strategy | **Patch-based + data augmentation** |
| Training Patches | **50-100 patches** |
| Total Timeline | **1-1.5 months** |
| Final Speedup | **10,000-100,000× vs full AMCA3D** |

#### ✓ **Implementation Roadmap**

**Phase 1: Prepare Thermal Input (1-2 weeks)**
- [ ] Export Abaqus thermal history (or run if new)
- [ ] Format as AMCA3D-compatible .txt files
- [ ] Validate mesh resolution (50-100 µm appropriate)
- [ ] Test interpolation accuracy on 1-2 reference cases

**Phase 2: Generate CA Training Data (2-4 weeks)**
- [ ] Set up 50-100 small Abaqus domains OR extract patches from full blade
- [ ] Run Abaqus in parallel: <1 day (patch approach) or 5-10 days (full blade)
- [ ] Generate CA simulations for each patch (1-2 days each with parallelization)
- [ ] Apply data augmentation (noise, parameter variations)
- [ ] Collect output microstructures

**Phase 3: Train & Validate CNN (1-2 weeks)**
- [ ] Prepare training data (~1000 augmented samples)
- [ ] Design 3D U-Net architecture (~5-10M parameters)
- [ ] Train on GPU cluster (5-20 hours)
- [ ] Validate against held-out test set

**Phase 4: Deploy (1 week)**
- [ ] Run CNN on full blade temperature field (minutes)
- [ ] Assemble predictions from overlapping patches
- [ ] Generate final grain microstructure map
- [ ] Compare to any available experimental data
- [ ] Document results & analysis

#### ⚠️ **Critical Decision Points**

1. **Abaqus Mesh Quality:**
   - If existing mesh is 50-100 µm: ✓ Use as-is
   - If coarser (200-500 µm): ⚠ Consider refinement
   - If finer (10-20 µm): ✓ Excellent but may be slower

2. **Full Blade vs Patches:**
   - Available cluster with 5-10 nodes? → Use patches (faster)
   - Limited parallelization? → Use full blade (simpler)
   - Need maximum accuracy? → Use adaptive mesh

3. **Validation:**
   - Always run 1-2 reference cases comparing coarse vs fine Abaqus
   - Verify grain structure differences < 15%
   - If larger, refine mesh or use adaptive approach

### ✓ **Why This Approach Works**

1. **Thermally appropriate:** 50-100 µm Abaqus mesh is standard for AM
2. **Interpolation acceptable:** ~1-2% error in growth velocity
3. **CA-only efficient:** 2-5× faster than coupled FEM-CA
4. **CNN robust:** Learns patterns even with coarse input
5. **Practical timeline:** Achievable in 1-1.5 months
6. **Huge speedup:** 10,000× faster than full AMCA3D at deployment

### ✓ **What Makes This Feasible**

| Advantage | Impact |
|-----------|--------|
| CA-only approach | Removes FEM solver (50% time savings) |
| Patch-based strategy | Fully parallelizable Abaqus phase |
| 128³ block size | Manageable GPU memory, good physics context |
| Coarse mesh OK | Physics is smooth at voxel scale |
| Data augmentation | Robust CNN despite slight thermal coarseness |

---

## References & Resources

### Key AMCA3D Components Used
- `LoadDataFromFile` (CA/src/LoadDataFromFile.cpp) - Temperature import
- `MapVoxelManager` (CA/include/MapVoxelManager.h) - FEM-to-CA interpolation
- `CellularAutomataManager_3D` (CA/include/CellularAutomataManager_3D.h) - CA engine
- `ProblemPhysics_RappazGandin1993` - Growth velocity model

### Recommended CNN Architectures
- **3D U-Net:** Best for voxel-level predictions
- **SegNet variant:** Alternative with good memory efficiency
- **Graph Neural Networks:** For grain-level predictions (advanced)

### Expected Performance
- **Training time:** 5-20 GPU hours (single GPU)
- **Inference time:** <1 second per 128³ block
- **Memory:** ~100-200 MB per block (inference)
- **Accuracy:** 80-95% structural agreement with full AMCA3D

---

## Questions & Clarifications

### Q: Why not use only Abaqus predictions?
**A:** Abaqus doesn't predict microstructure (grains, orientations). It only provides temperature. The CNN learns the temperature→microstructure mapping from AMCA3D training data.

### Q: Why CA-only instead of coupled CA-FEM?
**A:** Coupled approach is 2-5× slower due to FEM solver overhead. Since CA doesn't affect temperature (one-way coupling), importing external T-field is faster and more accurate if using high-fidelity FEM.

### Q: What if my Abaqus mesh is coarser than 100 µm?
**A:** 
- 100-200 µm: Still acceptable (small interpolation error ~1-2%)
- 200-500 µm: Acceptable for machine learning (CNN learns robustness)
- >500 µm: Not recommended (misses melt pool details)

### Q: Can I use experimental temperature measurements?
**A:** **YES!** This would actually improve accuracy. If you have thermography or other T-field data, it's perfect for this approach.

### Q: How do I handle multi-pass manufacturing?
**A:** Run separate CA simulations for each pass with remelting physics enabled. Include previous grain structure as initialization.

---

## Document Version

**Created:** June 2026
**Status:** Final Recommendations
**Next Review:** After Phase 1 thermal validation

---

**For questions or clarifications, refer to the detailed analysis sections above or conduct quick validation tests (Phase 1) before committing to full timeline.**
