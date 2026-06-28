> **SUPERSEDED** — this snapshot (2026-06-21) contains known errors. See `../overview.md` for the current, corrected version.
> Known errors: S2010 total inclusion volume stated as 10.77 cc (wrong — correct value is ~17.5 cc; calculation error in vol_mm³ formula); raysim described as having no mask→mesh pipeline (wrong — Stage 1 complete as of 2026-06-28); S3010 geometry assumed identical to S2010 (wrong — S3010 has 577 slices, 0.333984 mm spacing).

---

# STRUCTURE_REPORT — Repo Recon for raysim Export

**Date:** 2026-06-21 | **Branch:** feature/preprocessing | **Author:** Claude Sonnet 4.6 (read-only)

---

## 1. Inclusion Mask Inventory

### 1a. File Listing

#### S2010 (`output/preprocessing_data/paper_based/S2010/`)

| File | Size |
|---|---|
| filtered_norm_volume.npy | 572 MB |
| filtered_norm_volume.nrrd | 572 MB |
| filtered_volume_hu.npy | 572 MB |
| filtered_volume_hu.nrrd | 572 MB |
| filtered_volume.npy | 572 MB |
| filtered_volume.nrrd | 572 MB |
| hu_volume.npy | 572 MB |
| hu_volume.nrrd | 572 MB |
| normalized_volume.npy | 572 MB |
| normalized_volume.nrrd | 572 MB |
| windowed_volume.npy | 572 MB |
| windowed_volume.nrrd | 572 MB |
| mask_fat.nrrd | 2.1 MB |
| mask_glandular.nrrd | 1.1 MB |
| mask_inclusions.nrrd | 559 KB |
| spacing.npy | 152 B |

#### S3010 (`output/preprocessing_data/paper_based/S3010/`)

| File | Size |
|---|---|
| filtered_norm_volume.npy | 578 MB |
| filtered_norm_volume.nrrd | 578 MB |
| filtered_volume_hu.npy | 578 MB |
| filtered_volume_hu.nrrd | 578 MB |
| filtered_volume.npy | 578 MB |
| filtered_volume.nrrd | 578 MB |
| hu_volume.npy | 578 MB |
| hu_volume.nrrd | 578 MB |
| normalized_volume.npy | 578 MB |
| normalized_volume.nrrd | 578 MB |
| windowed_volume.npy | 578 MB |
| windowed_volume.nrrd | 578 MB |
| mask_fat.nrrd | 2.0 MB |
| mask_glandular.nrrd | 1.2 MB |
| mask_inclusions.nrrd | 646 KB |
| spacing.npy | 152 B |

### 1b. NRRD Metadata (SimpleITK, S2010)

| File | dtype | shape (z,y,x) | spacing (x,y,z) mm | origin LPS mm |
|---|---|---|---|---|
| mask_fat.nrrd | uint8 | (571, 512, 512) | (0.378906, 0.378906, 0.335000) | (−105.924, 41.474, −216.670) |
| mask_glandular.nrrd | uint8 | (571, 512, 512) | (0.378906, 0.378906, 0.335000) | (−105.924, 41.474, −216.670) |
| mask_inclusions.nrrd | uint8 | (571, 512, 512) | (0.378906, 0.378906, 0.335000) | (−105.924, 41.474, −216.670) |

> **S3010:** Automated SimpleITK read was in-progress at time of writing. From file sizes and code
> constants (NX=NY=512, NZ=571 for S3010 as well), geometry is expected to be identical in spacing
> but origin will differ. AMBIGUITY: S3010 origin not yet confirmed — run the analysis script to verify.

### 1c. Distinct Nonzero Values

- All three masks (fat, glandular, inclusions) for S2010: **binary** — only `[1]` present.
- None are pre-labeled multi-class volumes.

### 1d. Connected Components — mask_inclusions.nrrd (S2010)

> **Note:** The automated SimpleITK `ConnectedComponent` loop (full 571×512×512 array,
> ~150 M voxels, potentially thousands of tiny components) was still running at time of writing.
> The table below is derived from **`scipy.ndimage.label`** run in notebook
> `01_phantom_from_cirs_ct.ipynb` (Block 2.1) on the identical file with the same spatial
> interpretation. Threshold for listing: **>= 500 vox** (64 components found at that threshold;
> below that only noise/partial-volume fragments expected). For the >=30 vox count, see ambiguity
> note at the end of this section.

Phantom total: **364,385 inclusion voxels** = **10.77 cc** | HU mean 88 [p5=61, p95=129]

| rank | scipy_label | voxels | vol_mm³ | equiv_diam_mm | SoS m/s | rho kg/m³ |
|---|---|---|---|---|---|---|
| 1 | 11014 | 32390 | 1560.1 | 14.4 | 1570 | 1090 |
| 2 | 10152 | 14446 | 695.8 | 11.0 | 1570 | 1090 |
| 3 | 9601 | 13615 | 655.8 | 10.8 | 1570 | 1090 |
| 4 | 10028 | 13283 | 640.0 | 10.7 | 1570 | 1090 |
| 5 | 10766 | 13005 | 626.6 | 10.6 | 1570 | 1090 |
| 6 | 9677 | 12086 | 582.4 | 10.4 | 1570 | 1090 |
| 7 | 9204 | 11528 | 555.5 | 10.2 | 1570 | 1090 |
| 8 | 10119 | 8295 | 399.7 | 9.1 | 1570 | 1090 |
| 9 | 10016 | 7519 | 362.2 | 8.8 | 1570 | 1090 |
| 10 | 10029 | 5165 | 248.9 | 7.8 | 1570 | 1090 |
| 11 | 10500 | 4539 | 218.7 | 7.5 | 1570 | 1090 |
| 12 | 10749 | 4447 | 214.3 | 7.4 | 1570 | 1090 |
| 13 | 8967 | 4044 | 194.9 | 7.2 | 1570 | 1090 |
| 14 | 9771 | 3907 | 188.2 | 7.1 | 1569 | 1089 |
| 15 | 9545 | 3905 | 188.2 | 7.1 | 1570 | 1090 |
| 16 | 10940 | 3780 | 182.2 | 7.0 | 1571 | 1090 |
| 17 | 9999 | 3766 | 181.5 | 7.0 | 1570 | 1090 |
| 18 | 9923 | 3349 | 161.4 | 6.8 | 1570 | 1090 |
| 19 | 9377 | 3087 | 148.8 | 6.6 | 1569 | 1089 |
| 20 | 9503 | 2791 | 134.5 | 6.4 | 1570 | 1090 |
| 21 | 9714 | 2670 | 128.7 | 6.3 | 1571 | 1091 |
| 22 | 9875 | 2457 | 118.4 | 6.1 | 1570 | 1090 |
| 23 | 9506 | 2248 | 108.3 | 5.9 | 1570 | 1090 |
| 24 | 9631 | 2242 | 108.0 | 5.9 | 1570 | 1090 |
| 25 | 9674 | 2199 | 106.0 | 5.9 | 1570 | 1090 |
| 26 | 9826 | 2125 | 102.4 | 5.8 | 1571 | 1091 |
| 27 | 9718 | 1982 | 95.5 | 5.7 | 1569 | 1090 |
| 28 | 9705 | 1949 | 93.9 | 5.6 | 1569 | 1089 |
| 29 | 9987 | 1844 | 88.8 | 5.5 | 1571 | 1090 |
| 30 | 9807 | 1779 | 85.7 | 5.5 | 1569 | 1089 |
| 31 | 9123 | 1755 | 84.6 | 5.4 | 1573 | 1092 |
| 32 | 608 | 1725 | 83.1 | 5.4 | 1571 | 1091 |
| 33 | 9927 | 1709 | 82.3 | 5.4 | 1570 | 1090 |
| 34 | 9988 | 1664 | 80.2 | 5.3 | 1570 | 1090 |
| 35 | 10157 | 1639 | 79.0 | 5.3 | 1570 | 1090 |
| 36 | 10698 | 1552 | 74.8 | 5.2 | 1571 | 1091 |
| 37 | 10095 | 1522 | 73.3 | 5.2 | 1571 | 1091 |
| 38 | 10699 | 1307 | 63.0 | 4.9 | 1571 | 1091 |
| 39 | 10481 | 1272 | 61.3 | 4.9 | 1570 | 1090 |
| 40 | 10596 | 1185 | 57.1 | 4.8 | 1571 | 1090 |
| 41 | 11222 | 1080 | 52.1 | 4.6 | 1568 | 1088 |
| 42 | 10670 | 1073 | 51.7 | 4.6 | 1571 | 1091 |
| 43 | 9909 | 1048 | 50.5 | 4.6 | 1571 | 1091 |
| 44 | 10793 | 1007 | 48.5 | 4.5 | 1569 | 1089 |
| 45 | 10488 | 933 | 44.9 | 4.4 | 1568 | 1089 |
| 46 | 10913 | 926 | 44.6 | 4.4 | 1569 | 1090 |
| 47 | 5637 | 846 | 40.8 | 4.3 | 1571 | 1091 |
| 48 | 7781 | 802 | 38.6 | 4.2 | 1574 | 1093 |
| 49 | 7117 | 781 | 37.6 | 4.2 | 1569 | 1089 |
| 50 | 6536 | 754 | 36.3 | 4.1 | 1571 | 1090 |
| 51 | 6794 | 747 | 36.0 | 4.1 | 1572 | 1091 |
| 52 | 10965 | 742 | 35.7 | 4.1 | 1567 | 1088 |
| 53 | 9642 | 732 | 35.3 | 4.1 | 1573 | 1092 |
| 54 | 9598 | 723 | 34.8 | 4.0 | 1570 | 1090 |
| 55 | 3762 | 699 | 33.7 | 4.0 | 1566 | 1088 |
| 56 | 9432 | 648 | 31.2 | 3.9 | 1574 | 1093 |
| 57 | 9682 | 608 | 29.3 | 3.8 | 1567 | 1088 |
| 58 | 10651 | 604 | 29.1 | 3.8 | 1567 | 1088 |
| 59 | 10728 | 601 | 29.0 | 3.8 | 1567 | 1088 |
| 60 | 9422 | 596 | 28.7 | 3.8 | 1568 | 1089 |
| 61 | 10926 | 571 | 27.5 | 3.7 | 1567 | 1088 |
| 62 | 9436 | 545 | 26.3 | 3.7 | 1570 | 1090 |
| 63 | 10574 | 523 | 25.2 | 3.6 | 1569 | 1089 |
| 64 | 11017 | 505 | 24.3 | 3.6 | 1569 | 1089 |

> **Centroid LPS and bbox columns**: not yet available for S2010 from automated run (see note above).
> They can be reconstructed using `img.TransformContinuousIndexToPhysicalPoint()` on the result of
> `sitk.ConnectedComponent(mask_inclusions.nrrd)` when the background process completes.
> AMBIGUITY: exact centroids for the ~140+ components below 500 vox (which could still be >= 30 vox)
> are unknown — the SimpleITK CC will enumerate them when done.

**S2010 totals at >= 500 vox:** 64 components / all >= 30 vox
**S3010:** CC analysis not yet run in any existing notebook — automated run pending.

> **vol_mm³ formula:** voxels × (0.378906 × 0.378906 × 0.335) = voxels × 0.04818 mm³

---

## 2. Phantom Volume

### Confirmed geometry (S2010 `filtered_volume.nrrd`)

From mask file headers (all NRRD files share the same ITK grid) and hardcoded constants in all
simulation scripts:

| Parameter | Documented | Observed |
|---|---|---|
| shape (x,y,z vox) | 512 × 512 × 571 | **512 × 512 × 571** ✓ |
| spacing x mm | 0.379 | **0.37890625** ✓ |
| spacing y mm | 0.379 | **0.37890625** ✓ |
| spacing z mm | 0.335 | **0.335000** ✓ |
| origin LPS mm | — | (−105.924, 41.474, −216.670) |

No mismatch. Physical extent: 194.0 × 194.0 × 191.3 mm.

> Note: 0.37890625 = 512 × 0.37890625 = 193.9 mm, exactly matches 512 voxels at this spacing.
> The rounded value "0.379" in docs is accurate to 3 significant figures.

**S3010 `filtered_volume.nrrd`:** Slightly larger on disk (578 MB vs 572 MB). Header read was
in-progress — same NX=NY=512, NZ=571 expected from script constants, origin will differ.

---

## 3. Probe / Coordinate Geometry from MUSiK (raysim recipe)

**Source files:** `notebooks/musik/run_05_clinical_sim.py` (lines 40–148),
`run_06_fast_sim.py` (identical pose), `run_07_floor_test.py` (identical pose),
`01_phantom_from_cirs_ct.ipynb` (Blocks 0, 3, 3.1).

### 3a. MUSiK Coordinate System

MUSiK places the phantom **origin at the geometric center** of the volume:
- Phantom center voxel: `[NX/2, NY/2, NZ/2] = [256, 256, 285.5]`
- Units: **meters**
- Array read order: SimpleITK returns `(Z,Y,X)`; code transposes with `.T` → MUSiK array is `(X,Y,Z)`.

Mapping to CT LPS frame (S2010):
```
MUSiK_x_m = (CT_vox_X - 256) * 0.000378906   # "left/right" across phantom
MUSiK_y_m = (CT_vox_Y - 256) * 0.000378906   # "anterior/posterior" into body
MUSiK_z_m = (CT_vox_Z - 285.5) * 0.000335    # "superior/inferior" (axial)
```

### 3b. Clinical Probe Pose (runs 05 / 06 / 07, locked)

```python
CLINICAL_POSE = geometry.Transform(
    rotation=(np.pi/2, 0, 0),           # Euler XYZ: π/2 around X
    translation=(0.0, -0.060625, 0.003685))  # meters, MUSiK frame
```

**Verified at runtime (sanity gate, 9/9 checks ok in run 05 log):**

| Check | Expected | Observed |
|---|---|---|
| Probe centroid (voxel) | [256, 96, 296] | **[256, 95, 296]** ✓ |
| Probe centroid (mm) | X=97, Y=36, Z=99 | X=97.0, Y=36.0, Z=99.2 |
| Beam direction | [0, 1, 0] | **[0, 1, 0]** ✓ |

**Translation in CT voxels:**
- `x_vox = 0/0.000379 + 256 = 256` (midline, X-centered)
- `y_vox = -0.060625/0.000379 + 256 ≈ 96` → **apex (skin surface) at Y_vox = 96**
- `z_vox = 0.003685/0.000335 + 285.5 ≈ 296.5` → **scan plane Z = 296** (densest inclusion slice)

**Rotation:** `(π/2, 0, 0)` rotates the probe's default beam axis (+X) to the +Y direction.
Result: beam fires from apex (Y=96) inward toward the thorax along **+Y direction**.

### 3c. Transducer Parameters

```python
transducer.Focused(
    max_frequency = 3e6,          # 3 MHz
    elements      = 128,
    width         = 35e-3,        # 35 mm array width (lateral, X)
    height        = 5e-3,         # 5 mm elevational aperture (Z)
    sweep         = np.pi/12,     # 15° total sector angle
    ray_num       = 32,           # (run 05) / 4 (runs 06/07)
    focus_azimuth = 40e-3,        # 40 mm focal depth
    focus_elevation = 20e-3)
SOS_BL = 1540.0  # m/s (baseline speed of sound)
```

### 3d. Imaging Depth and Axial Range

| Parameter | Value | Source |
|---|---|---|
| Beam depth (Y) | 80 mm | `DEPTH_MM = 80.0` (run_05, line 48) |
| Apex Y-voxel | 96 | `Y_SURF_IDX = 96` |
| Apex Y-mm (from center) | −60.6 mm | `translation_y = -0.060625 m` |
| Deepest Y-voxel | 96 + int(80/0.379) ≈ 307 | `FOV_Y1 = Y_SURF_IDX + int(DEPTH_MM/DY_MM)` |
| Scan plane Z-voxel | 296 | `PZ_SCAN = 296` (run_05, line 47) |
| Elevational FOV | 10 mm → Z ∈ [282, 310] | `FOV_Z0/Z1` (run_05, lines 145–146) |
| Lateral FOV | 40 mm → X ∈ [203, 309] | `FOV_X0/X1` (run_05, lines 143–144) |
| Dense in FOV | 2.5 % | run 05 log (sanity gate) |

**T_END from completed run 05:** `1.14286e-04 s` → `MAX_DEPTH = T_END × SOS/2 = 88.0 mm`
(k-Wave adds a PML margin, so actual simulated depth > 80 mm physical imaging depth.)

**B-mode output geometry (run 05 log):**
- Matrix: `(2436, 635)` pixels
- Lateral half-width at max depth: `88.0 × sin(7.5°) = 11.5 mm`
- Depth axis: 0 → 88 mm (physical), 0 → 80 mm useful

### 3e. LPS ↔ Probe Frame Mapping

```
MUSiK_X  → raysim LATERAL   (along array face, left-right in B-mode)
MUSiK_Y  → raysim DEPTH     (Z_depth into body, apex=0, thorax=+)
MUSiK_Z  → raysim ELEVATIONAL (out-of-plane)
```

Probe-frame origin = apex contact point: `(x=0, y_MUSiK=-0.060625, z=0.003685)` m  
In CT voxels: `[256, 96, 296]`  
In CT LPS mm: `(−105.924 + 256×0.379, 41.474 + 96×0.379, −216.670 + 296×0.335)`
             = `(-8.9, 78.0, -117.5)` mm

**Recipe for raysim (units meters):**
1. Read `mask_inclusions.nrrd` with SimpleITK, transpose `.T` to get `(X,Y,Z)` array.
2. Shift to MUSiK frame: `x_m = (i_x - 256)*0.000379`, `y_m = (i_y - 256)*0.000379`, `z_m = (i_z - 285.5)*0.000335`
3. Map to raysim: `raysim_X = x_m`, `raysim_Z_depth = y_m - (-0.060625)` (depth from apex), `raysim_Y_elev = z_m - 0.003685`
4. Imaging cone: X ∈ [−20, +20] mm, Z_depth ∈ [0, 80] mm, Y_elev ∈ [−5, +5] mm (centered on Z=296).

---

## 4. Existing Mesh / Simulator-Export Work

### 4a. Mesh Files

```
find . -not -path "*/external/*" \( -name "*.obj" -o -name "*.stl" -o -name "*.ply" \)
```
**Result: no files found.** No OBJ, STL, or PLY files exist in the repo (excluding `external/`).

### 4b. Code Grep

```
grep -r --include="*.py" --include="*.ipynb" -l \
    "raysim|isaac|marching_cubes|trimesh|\.obj" . --exclude-dir=external
```
**Exit code 1 — no matches.** The keywords `raysim`, `isaac`, `marching_cubes`, `trimesh`, `.obj` do
not appear in any `.py` or `.ipynb` file outside `external/`.

### 4c. Verdict

**NO** mask-to-mesh / OBJ-export code exists anywhere in the project.
The gap is real and confirmed. The simulation pipeline stops at the voxel-mask stage;
no mesh generation step exists yet.

---

## 5. Git Snapshot (read-only)

**Current branch:** `feature/preprocessing`

**`git status -s` summary:**
- 1 modified (unstaged): `external/kuka_lbr_control` (submodule pointer)
- 2 untracked directories: `external/MUSiK/`, `notebooks/musik/`

**Branches:**

| Local | Remote |
|---|---|
| develop | origin/develop |
| **feature/preprocessing** (current) | origin/feature/preprocessing |
| feature/robot | origin/feature/robot |
| feature/simulation | origin/feature/simulation |
| main | origin/main |

**Last 5 commits:**

| Hash | Message |
|---|---|
| 6c1c5a3 | Add data-driven bright-inclusion threshold (Gaussian fit, k=3) |
| 1a6632a | feat(preprocessing): bright inclusion threshold via Gaussian fit on glandular peak |
| e7a3fde | feat(preprocessing): import and organize notebooks from legacy branches |
| 783889e | Update file |
| 04462b0 | Delete file, moved to test branch |

> The `notebooks/musik/` directory is **untracked** — all MUSiK work (runs 01–07, logs, figures)
> lives outside git and is not committed to any branch. It would be lost if the filesystem were
> wiped. Consider committing or at minimum tarring the runs directory.

---

## OPEN QUESTIONS / AMBIGUITIES

1. **CC centroid/bbox for S2010:** The automated SimpleITK `ConnectedComponent` loop was still
   running at report time (slow on 571×512×512 with many tiny components). Re-run or filter with
   `>= 30 vox` before the `argwhere` loop to avoid O(n_components × volume) complexity.

2. **S3010 CC and origin:** S3010 has never been used in any notebook or simulation script —
   its mask_inclusions geometry, number of masses, and LPS origin are unconfirmed.

3. **MUSiK coordinate handedness vs LPS:** The exact sign conventions of `geometry.Transform`
   (Euler XYZ vs intrinsic/extrinsic) are not confirmed from source — only sanity-gate checks
   verify the net result. For raysim, derive from the verified voxel-centroid check, not the
   rotation parameters alone.

4. **`feature/simulation` branch:** All MUSiK notebooks and run scripts are on the local filesystem
   only (untracked). The `feature/simulation` branch content was not checked — it may contain
   older versions of the phantom/probe setup.

5. **raysim input format unknown:** No raysim documentation or API is present in this repo.
   The mesh export format (OBJ, STL, numpy, custom) must be confirmed with the raysim team before
   implementing the mask→mesh pipeline.
