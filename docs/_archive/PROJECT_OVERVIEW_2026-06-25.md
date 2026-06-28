> **SUPERSEDED** — this snapshot (2026-06-25) contains known errors. See `../overview.md` for the current, corrected version.
> Known errors: Bug #4 marked Closed (wrong — patch was reverted); raysim marked Pending (wrong — Stage 1 done); `v0.1-sim-working` tag listed (wrong — removed during rollback).

---

# Project Overview — Robot-Assisted US-Guided Breast Biopsy

**Report date:** 2026-06-25  
**Branch inspected:** `feature/robot` (current checkout)  
**Author:** read-only automated analysis (Claude Sonnet 4.6)

> Note: `for_claude_code.md` is listed in `.gitignore` and is not present on disk. This report
> is based on direct inspection of all other tracked and local files.

---

## 1. Repository Tree (2 levels)

```
guided_breast_biopsy/
├── dataset/                  Raw input data (gitignored contents)
│   ├── CMB-BRCA/             TCIA CMB-BRCA patient collection — CT, MRI, PET-CT series
│   ├── FANTOCCIO/            CIRS Model 073 phantom DICOM (4 series: S1000, S2010, S3010, S5010)
│   ├── consider_cases/       Manually curated patient candidates (4 subject folders)
│   └── metadata.csv          TCIA download manifest (CSV)
├── docs/                     Project documentation
│   ├── kuka_run_in_gazebo_steps.md   Gazebo adaptation log: all bugs and fixes
│   └── SoA_analysis_paper_topic_guided_robot.html   State-of-the-art survey (robot biopsy)
├── external/                 Third-party code
│   ├── kuka_lbr_control/     KUKA control stack submodule (Morook97 fork)
│   └── MUSiK/                US k-Wave simulator (untracked on this branch)
├── notebooks/                Analysis and simulation notebooks
│   ├── musik/                MUSiK simulation workspace (entire directory untracked)
│   └── output/               Notebook output snapshots (preprocessing figures, gitignored)
├── output/                   Pipeline artifacts (gitignored)
│   ├── figures/              Preprocessing PNG visualisations
│   └── preprocessing_data/   NRRD/NPY volumes (paper_based/ and test/)
├── py_venv/                  Python virtual environment (gitignored)
├── inclusions_stats.py       Standalone connected-component script (SimpleITK)
├── README.md                 Single-line project title
└── STRUCTURE_REPORT.md       Repo-recon snapshot from 2026-06-21 (feature/preprocessing state)
```

**External workspaces referenced but not traversed:**

| Path | Role |
|---|---|
| `~/kuka_ws/` | colcon build workspace; symlinks `external/kuka_lbr_control` packages for ROS 2 build |
| `~/isaac_us_test/` | NVIDIA Isaac for Healthcare repo (GPU US raytracing simulator — not yet integrated) |

---

## 2. Git State

### 2a. Current branch

`feature/robot`

### 2b. Branch list

| Local | Remote |
|---|---|
| develop | origin/develop |
| feature/preprocessing | origin/feature/preprocessing |
| **feature/robot** (current) | origin/feature/robot |
| feature/simulation | origin/feature/simulation |
| main | origin/main |

### 2c. Last 15 commits (current branch `feature/robot`)

| Hash | Date | Subject |
|---|---|---|
| `67b00a2` | 2026-05-17 | chore: update file |
| `9646d57` | 2026-05-17 | fix(robot): revert rejected safety-threshold patches |
| `60341ef` | 2026-04-24 | docs(robot): add KUKA Gazebo adaptation log with bug fixes and test workflow |
| `74fbb67` | 2026-04-24 | feat(robot): simulation fully working — arm tracks Cartesian targets |
| `274eccb` | 2026-04-24 | feat(robot): advance kuka_lbr_control to fix effort controller shutdown in Gazebo |
| `3e45e1f` | 2026-04-24 | chore: update kuka_lbr_control fork with controllers submodule redirect |
| `8241970` | 2026-04-24 | refactor(robot): revert delta\_tau\_max YAML change (wrong hypothesis) |
| `614122c` | 2026-04-24 | refactor(robot): update kuka\_lbr\_control fork with delta\_tau\_max raised for Gazebo |
| `2526964` | 2026-04-24 | refactor(robot): update kuka\_lbr\_control fork with gazebo\_controllers.yaml fix |
| `39e3eea` | 2026-04-23 | refactor(robot): update kuka\_lbr\_control fork with Gazebo xacro patch |
| `8a54192` | 2026-04-22 | refactor(robot): switch kuka\_lbr\_control submodule to personal fork |
| `92e4094` | 2026-04-19 | feat(robot): add kuka\_lbr\_control as submodule under external/ |
| `c835390` | 2026-04-19 | docs(robot): import SoA analysis on guided robot techniques from legacy branch |
| `783889e` | 2026-04-19 | Update file |
| `04462b0` | 2026-02-21 | Delete file, moved to test branch |

### 2d. Submodules (`.gitmodules`)

| Path | Upstream URL | Pinned commit (current branch) |
|---|---|---|
| `external/kuka_lbr_control` | `https://github.com/Morook97/kuka_lbr_control.git` | `ac52581e` (heads/main) |

`external/MUSiK` is **not** a submodule on `feature/robot`. It is present on the local filesystem as an untracked git checkout. It is registered as a submodule on `origin/feature/simulation` only.

---

## 3. Preprocessing Pipeline

### 3a. Implementing notebook

`notebooks/03_preprocessing_phantom_paper_based.ipynb` on branch `origin/feature/preprocessing`  
(not present on `feature/robot`; inspected via `git show`).

A test variant exists at `notebooks/02_preprocessing_phantom_test.ipynb` (different normalization approach using CLAHE; its outputs are under `output/preprocessing_data/test/`). The paper-based pipeline is the canonical version.

### 3b. Pipeline steps (as coded)

| # | Step | Function / cell | Key parameters |
|---|---|---|---|
| 1 | DICOM load + HU conversion | `load_dicom_volume_hu()` | `HU = pixel × RescaleSlope + RescaleIntercept` (from DICOM tags) |
| 2 | Fat-based normalization | `normalize_by_fat()` | Fat ROI: `HU ∈ [−200, −50]`; subtract `mean(HU_fat)`; output still in HU |
| 3 | Windowing | `apply_windowing()` | WW = 350, WL = 25 → clip to [−150, +200] HU, rescale to [0, 1] |
| 4 | 5-point median filter | `apply_median_filter()` | `kernel_size = 5`, applied slice-by-slice (2-D per axial slice) |
| 4b | HU back-conversion | `filtered_to_hu()` | Invert step 3: `HU = v × 350 − 150`; output `filtered_volume_hu.nrrd` |
| 5a | Gaussian peak fit | `fit_glandular_peak()` | 200-bin histogram on phantom band [0.30, 0.85]; Gaussian smooth σ=3 for `find_peaks` only; Gaussian fit on left side of peak for σ_g |
| 5b | Threshold segmentation | `segment_inclusions()` | `t = μ_g + k × σ_g`, k = 3 (saved to disk); phantom band [0.30, 0.85] applied as body mask |

**Windowing reference [Prionas 2010]:** WW=350, WL=25 explicitly cited.  
**Median filter reference [Nelson 2008]:** 5-point slice-by-slice cited.  
**Gaussian fit [Nelson 2008, Caballo 2018]:** data-driven, left-side only to avoid inclusion tail bias.  

### 3c. Markdown vs code consistency check

The notebook header markdown states the pipeline as:  
"HU conversion → Fat-based normalization → Windowing → Median filter → Threshold segmentation"

The code implements this in the same order. There is one additional step (4b, HU back-conversion) that is not listed in the header markdown but is coded and produces `filtered_volume_hu.nrrd`. This is not a logical mismatch — the back-conversion is a derived output, not a distinct pipeline stage. **No substantive discrepancy found.**

---

## 4. Data Artifacts

### 4a. NRRD/NPY volumes (paper-based pipeline)

Both series share the same 16 files each: `hu_volume`, `normalized_volume`, `windowed_volume`, `filtered_volume`, `filtered_norm_volume`, `filtered_volume_hu` (× .npy and .nrrd), plus `mask_fat`, `mask_glandular`, `mask_inclusions` (.nrrd only), and `spacing.npy`.

#### S2010

| File | dtype | shape (z,y,x) | spacing (x,y,z) mm | origin LPS (x,y,z) mm | value range |
|---|---|---|---|---|---|
| `filtered_volume.nrrd` | float32 | (571, 512, 512) | (0.378906, 0.378906, 0.335000) | (−105.924, 41.474, −216.670) | [0.0, 1.0] |
| `filtered_volume_hu.nrrd` | float32 | (571, 512, 512) | (0.378906, 0.378906, 0.335000) | (−105.924, 41.474, −216.670) | [−150.0, 200.0] |
| `mask_inclusions.nrrd` | uint8 | (571, 512, 512) | (0.378906, 0.378906, 0.335000) | (−105.924, 41.474, −216.670) | [0, 1] |
| `mask_fat.nrrd` | uint8 | (571, 512, 512) | (0.378906, 0.378906, 0.335000) | (−105.924, 41.474, −216.670) | [0, 1] |
| `mask_glandular.nrrd` | uint8 | (571, 512, 512) | (0.378906, 0.378906, 0.335000) | (−105.924, 41.474, −216.670) | [0, 1] |

Physical extent: 193.9 × 193.9 × 191.3 mm. Voxel volume: 0.04809 mm³.

#### S3010

| File | dtype | shape (z,y,x) | spacing (x,y,z) mm | origin LPS (x,y,z) mm | value range |
|---|---|---|---|---|---|
| `filtered_volume.nrrd` | float32 | (577, 512, 512) | (0.333984, 0.333984, 0.335000) | (−91.485, 52.833, −217.690) | [0.0, 1.0] |
| `filtered_volume_hu.nrrd` | float32 | (577, 512, 512) | (0.333984, 0.333984, 0.335000) | (−91.485, 52.833, −217.690) | [−150.0, 200.0] |
| `mask_inclusions.nrrd` | uint8 | (577, 512, 512) | (0.333984, 0.333984, 0.335000) | (−91.485, 52.833, −217.690) | [0, 1] |
| `mask_fat.nrrd` | uint8 | (577, 512, 512) | (0.333984, 0.333984, 0.335000) | (−91.485, 52.833, −217.690) | [0, 1] |
| `mask_glandular.nrrd` | uint8 | (577, 512, 512) | (0.333984, 0.333984, 0.335000) | (−91.485, 52.833, −217.690) | [0, 1] |

Physical extent: 171.0 × 171.0 × 193.3 mm. Voxel volume: 0.037368 mm³.

> **Important:** S3010 has 577 axial slices (not 571) and in-plane spacing 0.333984 mm (not 0.378906 mm).
> The STRUCTURE_REPORT.md (written 2026-06-21 on `feature/preprocessing`) incorrectly assumed S3010
> geometry was identical to S2010. Values above are verified from the NRRD headers.

> The MUSiK simulation scripts (`run_05/06/07`) hardcode `DX_MM = DY_MM = 0.37890625`, `NX = NY = NZ`
> matching S2010. They cannot be applied to S3010 without modification.

#### Test pipeline outputs

Separate `output/preprocessing_data/test/S2010/` and `S3010/` directories exist with:
`clahe_volume.npy`, `hu_volume.npy`, `normalized_volume.npy`, `spacing.npy`, `windowed_volume.npy`.
These are from the CLAHE test variant (notebook `02_preprocessing_phantom_test.ipynb`) and are not used
downstream.

---

## 5. Segmentation

### 5a. Mask format

All three masks (`mask_inclusions`, `mask_fat`, `mask_glandular`) for both S2010 and S3010 are:  
- Format: NRRD, binary (uint8), values {0, 1} only.  
- Origin: notebook output (threshold applied in `03_preprocessing_phantom_paper_based.ipynb`, k=3).  
- Not a labeled Slicer export; Slicer was used only for visual review during patient candidate selection.

### 5b. Connected components — `mask_inclusions.nrrd`

Analysis run via `scipy.ndimage.label` using the project venv (SimpleITK also used for geometry).

#### S2010

| Metric | Value |
|---|---|
| Total inclusion voxels | 364,385 |
| Total inclusion volume | 17,525 mm³ (17.5 cc) |
| Total components (all sizes) | 19,423 |
| Components ≥ 500 vox | 64 |
| Voxel volume | 0.04809 mm³ |

> Note: `STRUCTURE_REPORT.md` reported 10.77 cc for this same mask. The correct value, computed
> from the actual NRRD header spacing (0.378906² × 0.335 = 0.04809 mm³/vox), is 17,525 mm³ = 17.5 cc.
> The STRUCTURE_REPORT figure appears to be a computation error. The 364,385 voxel count is consistent
> between both sources; only the volume conversion differed.

Top 10 components by size (S2010, ≥ 500 vox):

| Rank | Voxels | Vol (mm³) | Equiv. diam. (mm) |
|---|---|---|---|
| 1 | 32,390 | 1,558.3 | 14.4 |
| 2 | 14,446 | 695.5 | 11.0 |
| 3 | 13,615 | 655.5 | 10.8 |
| 4 | 13,283 | 639.5 | 10.7 |
| 5 | 13,005 | 626.1 | 10.6 |
| 6 | 12,086 | 581.9 | 10.4 |
| 7 | 11,528 | 555.1 | 10.2 |
| 8 | 8,295 | 399.5 | 9.1 |
| 9 | 7,519 | 361.9 | 8.8 |
| 10 | 5,165 | 248.7 | 7.8 |

Centroids in LPS coordinates not yet generated (requires `inclusions_stats.py` to complete).

#### S3010

| Metric | Value |
|---|---|
| Total inclusion voxels | 517,256 |
| Total inclusion volume | 19,329 mm³ (19.3 cc) |
| Total components (all sizes) | 15,296 |
| Components ≥ 500 vox | 136 |
| Voxel volume | 0.037368 mm³ |

Top 10 components by size (S3010, ≥ 500 vox):

| Rank | Voxels | Vol (mm³) | Equiv. diam. (mm) |
|---|---|---|---|
| 1 | 31,456 | 1,175.4 | 13.1 |
| 2 | 19,880 | 742.9 | 11.2 |
| 3 | 18,348 | 685.6 | 10.9 |
| 4 | 18,109 | 676.7 | 10.9 |
| 5 | 17,233 | 644.0 | 10.7 |
| 6 | 16,626 | 621.3 | 10.6 |
| 7 | 14,546 | 543.6 | 10.1 |
| 8 | 10,939 | 408.8 | 9.2 |
| 9 | 9,464 | 353.6 | 8.8 |
| 10 | 8,412 | 314.3 | 8.4 |

> S3010 has notably more large components (136 vs 64 ≥ 500 vox) and higher total inclusion volume
> (19.3 cc vs 17.5 cc), despite smaller voxels, suggesting this phantom series contains more or
> denser inclusions.

---

## 6. Simulation

### 6a. MUSiK (k-Wave GPU, CIRS phantom)

**Framework:** MUSiK (Chan et al., IEEE TBME 2026), GPU k-Wave full-wave solver.  
**Location:** `external/MUSiK/` (local checkout, untracked on this branch).  
**Simulation workspace:** `notebooks/musik/` — entirely untracked by git (all runs, logs, figures are filesystem-only).

#### Completed runs

| Run | Directory | Purpose | Config | Wall-clock |
|---|---|---|---|---|
| 01 | `runs/01_phantom_cirs_ct/` | Initial phantom setup | — (notebook-driven, no JSON) | — |
| 02 | `runs/02_phantom_cirs_ct_z0/` | Phantom at Z=0 | — | — |
| 03 | `runs/03_synthetic_acquisition/` | Synthetic scan test | voxel 0.25 mm | — |
| 04 | `runs/04_high_contrast/` | High-contrast variant | voxel 0.25 mm | — |
| 05 | `runs/05_clinical_apex_z296/` | **Clinical scan, 32 rays** | voxel 0.25 mm, 32 rays | 462.1 min |
| 06 | `runs/06_fast_calib/` | Fast calibration, 4 rays | voxel 0.40 mm, 4 rays | 60.9 min |
| 07 | `runs/07_floor_test/` | k-Wave floor benchmark | grid_lambda=1.0 (0.257 mm vox), 4 rays | 15.6 min |

Runs 01–04 exist as directories with `phantom/` and `results/` subdirectories but no full JSON configs;
they were run from the notebook directly. Runs 05–07 have full config JSON and were run from standalone scripts.

#### Locked clinical probe pose (runs 05/06/07)

```python
CLINICAL_POSE = geometry.Transform(
    rotation=(np.pi/2, 0, 0),        # Euler XYZ: π/2 about X
    translation=(0.0, -0.060625, 0.003685))  # metres, MUSiK frame
```

Runtime sanity gate (9/9 checks passed in run 05):
- Probe centroid (voxel): [256, 95, 296] (expected [256, 96, 296])
- Beam direction: [0, 1, 0]
- Dense inclusions in FOV: 2.5%

#### Transducer (all runs)

| Parameter | Value |
|---|---|
| Type | `transducer.Focused` |
| Frequency | 3 MHz |
| Elements | 128 |
| Width (lateral) | 35 mm |
| Height (elevational) | 5 mm |
| Sector sweep | π/12 (15°) |
| Azimuthal focus | 40 mm |
| Elevational focus | 20 mm |
| Baseline SoS | 1540 m/s |

#### Simulation grid (run 05, canonical)

| Parameter | Value |
|---|---|
| Grid size | 80 × 40 × 10 mm |
| Voxel size | 0.25 × 0.25 × 0.25 mm |
| PML size | (16, 6, 6) |
| PML alpha | 2 |
| Attenuation (bona) | 6 |
| alpha\_coeff | 0.5 |
| alpha\_power | 1.5 |
| T\_END | 1.14286 × 10⁻⁴ s |
| MAX\_DEPTH (reconstructed) | 88.0 mm |

B-mode output (run 05): matrix (2436 × 635) pixels, peak 5.775 × 10⁶.

#### Acoustic tissue properties

| Tissue | c (m/s) | ρ (kg/m³) | σ | scale |
|---|---|---|---|---|
| Breast parenchyma (background) | 1510 | 1020 | 20 | 1 × 10⁻⁴ |
| Dense lesion (inclusions) | 1570 | 1090 | 150 | 5 × 10⁻⁵ |
| Water (outside phantom) | 1540 | 1000 | — | — |

#### Output figures (on-disk)

`notebooks/musik/figures/`: 16 PNG files including B-mode and CT overlay for runs 05, 06, 07, plus
probe geometry diagnostics from notebook development.

### 6b. Mesh / OBJ / raysim export

No mesh files (`.obj`, `.stl`, `.ply`) exist anywhere in the repo (confirmed by `find`).  
No `raysim`, `marching_cubes`, `trimesh`, or `.obj` keywords appear in any `.py` or `.ipynb` file
outside `external/`.  
**The voxel-mask-to-mesh pipeline does not exist.** The simulation pipeline terminates at the
voxel-mask + MUSiK B-mode stage.

### 6c. Isaac US simulator

`~/isaac_us_test/` exists and contains the NVIDIA Isaac for Healthcare repository (GPU raytracing
US simulator). It is not referenced from any notebook or script in this repo and is not a git
submodule. Its presence on the filesystem indicates investigative work; no integration with the
thesis pipeline has been implemented.

---

## 7. Robot / ROS 2 Stack

### 7a. Architecture

Four-level fork cascade, all personal forks under `Morook97/`:

```
guided_breast_biopsy (feature/robot)
└── external/kuka_lbr_control  →  Morook97/kuka_lbr_control @ main @ ac52581e
    ├── lbr-stack/lbr_fri_ros2_stack  →  Morook97/lbr_fri_ros2_stack @ thesis/gazebo-patch @ c654981
    └── controllers  →  Morook97/ros2_effort_controller @ thesis/configurable-shutdown-warmup @ ae656f8
```

Upstream originals (not modified): `idra-lab/kuka_lbr_control`, `lbr-stack/lbr_fri_ros2_stack`,
`idra-lab/ros2_effort_controller`.

The build workspace is `~/kuka_ws/` (colcon, symlinks to `external/kuka_lbr_control` packages).
Expected build: 23 packages, zero errors (per workflow documentation).

### 7b. Target robot

KUKA LBR iiwa14 (7-DOF torque-controlled arm). ROS 2 Jazzy, Gazebo Harmonic (gz-sim8), Ubuntu 24.04.

### 7c. Documented bugs and status

| Bug | Root cause | Fix applied | Status |
|---|---|---|---|
| #0 — Effort-only Gazebo interface | `gz_ros2_control` cannot load multi-command-interface; xacro declared `position` + `effort` | Wrapped `position` in `<xacro:unless value="${mode == 'gazebo'}">` | Closed |
| #3 — Wrong YAML loaded | Gazebo plugin pointed only at base stack YAML, missing custom controllers | Added second `<parameters>` tag for `gazebo_controllers.yaml` | Closed |
| #4 — Hardcoded 10 Nm shutdown threshold | `effort_controller_base.cpp` terminates if `|tau - m_efforts| > 10` on first tick; gravity-comp at 90° elbow requires ~17 Nm | C++ patch: configurable `effort_shutdown_threshold` (YAML: 500 Nm for Gazebo) + first-tick warmup that seeds `m_efforts` directly | Closed |

The `delta_tau_max` YAML change (commit `614122c`) was a false lead and was reverted (`8241970`).
Commit `9646d57` ("revert rejected safety-threshold patches") is the committed record of this revert.

### 7d. Verified working state

Three tests documented in `docs/kuka_run_in_gazebo_steps.md`:

| Test | Expected result | Verified |
|---|---|---|
| Activation | Log: "Effort shutdown threshold: 500.00 Nm" + "Successfully switched controllers!" | Yes |
| Gravity hold | A4 effort ≈ −16.8 Nm, all velocities ≈ 0 | Yes |
| Cartesian tracking | A2: +1.22 rad, A4: +0.52 rad after target (0.3, 0.0, 0.6) m | Yes |

Git tag `v0.1-sim-working` at `74fbb67`.

---

## 8. Completeness Matrix

| Track | Status | Evidence |
|---|---|---|
| CT preprocessing — phantom (S2010, S3010) | **Done** | All NRRD/NPY artifacts present; masks generated and verified |
| CT preprocessing — patient | **Pending** | `01_extraction_CT_patient.ipynb` exists; manual Slicer review done; no patient volume in output/ |
| Segmentation — phantom inclusions | **Done** | `mask_inclusions.nrrd` present for S2010 and S3010; CC analysis run |
| Segmentation — patient | **Pending** | No patient output files exist anywhere in repo |
| US simulation — MUSiK / phantom | **Done** | Runs 01–07 complete; clinical pose locked; B-mode images generated |
| US simulation — raysim or Isaac | **Pending** | No mesh export; no raysim code; Isaac present locally but unconnected |
| Robot simulation — Gazebo | **Done** | Full Gazebo stack working; three tests verified; tag v0.1-sim-working |
| Robot–US integration | **Pending** | No code connecting robot pose to US simulator in any branch |
| Patient pipeline end-to-end | **Pending** | Depends on patient preprocessing and segmentation (both pending) |

---

## 9. Open Questions / Not Verifiable from Repo

1. **S2010 inclusion centroids in LPS:** `inclusions_stats.py` (SimpleITK CC) was not run during
   this analysis to avoid a long blocking compute. Centroid and bounding-box data per component are
   therefore not available in this report. Run `python3 inclusions_stats.py` from the project root
   with the venv active.

2. **S3010 never used in any simulation script:** All MUSiK run scripts (`run_05/06/07`) hardcode
   S2010 geometry (`DX_MM=0.37890625`, `NZ=571`). Adapting to S3010 requires updating these constants.

3. **MUSiK runs untracked:** The entire `notebooks/musik/` directory is absent from git (confirmed
   in `.gitignore` entries and `git status`). All run artifacts, logs, and B-mode images exist only
   on the local filesystem. If the machine is wiped or re-imaged, this work is lost.

4. **`feature/simulation` branch content:** That branch registers `external/MUSiK` as a submodule
   and contains an older `notebooks/musik/README.md`. Its relationship to the current untracked
   `notebooks/musik/` workspace is not clear — the branches have diverged since the simulation
   branch was created.

5. **raysim input format unknown:** No raysim API or documentation is present in the repo.
   The format required for exporting inclusion geometry (OBJ, STL, numpy, custom) must be confirmed
   with the raysim provider before the mask-to-mesh pipeline can be scoped.

6. **Patient dataset completeness:** `dataset/consider_cases/` contains four patient subjects
   (MSB-00727, MSB-01799, MSB-02599, MSB-08876). Whether these have been reviewed in Slicer and
   cleared for segmentation is not recorded in any tracked file.

7. **MUSiK external dependency version:** `external/MUSiK/` is a local checkout (untracked).
   The exact commit pinned is not recorded in `.gitmodules` on this branch. Reproducibility requires
   recording the commit hash or adding it as a proper submodule.

8. **STRUCTURE_REPORT.md volume discrepancy:** That document (2026-06-21) reported S2010 total
   inclusion volume as 10.77 cc. The value recomputed from the current NRRD header is 17.5 cc
   (364,385 vox × 0.04809 mm³/vox). The discrepancy source was not identified; 10.77 cc appears
   to be a calculation error in that report.
