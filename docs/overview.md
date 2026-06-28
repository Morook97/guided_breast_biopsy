# Project Overview — Robot-Assisted US-Guided Breast Biopsy

## Goal

Build a robot-assisted ultrasound-guided biopsy workflow using the CIRS Model 073 breast phantom and KUKA LBR iiwa14 arm combine to the NVIDIA Isaac raysim GPU to generetate sintetic real-time ecography images with sub-optimal resolution. The thesis has three blocks that will eventually form a closed loop: CT preprocessing → robot control → ultrasound simulation.

Clinical objective is the lesion localisation to guide the robotic arm and at the moment not diagnostic-grade imaging. The simulated B-mode must show where the bright inclusions are, fast enough for a control loop where the physical realism is approximate.

---

## Block 1 — CT Preprocessing

DICOM CT of the CIRS used in the blocks are S2010 and S3010 phantom series.

### Pipeline:

```
DICOM CT
  → HU conversion              (pixel × RescaleSlope + RescaleIntercept)
  → fat-based normalisation    (subtract mean HU of fat voxels, HU ∈ [−200, −50])
  → windowing                  (WW=350, WL=25 → [0,1])              [Prionas 2010]
  → 5-point median filter      (slice-by-slice, 2-D per axial slice) [Nelson 2008]
  → Gaussian-fit threshold     (t = μ + 3σ on glandular peak)       [Nelson 2008, Caballo 2018]
  → NRRD output (LPS spatial metadata via SimpleITK)
```

Threshold is data-driven: Gaussian fit on the glandular peak within phantom band [0.30, 0.85], left-side only to avoid inclusion-tail bias, k=3.

### Key numbers — S2010

| Metric | Value |
|---|---|
| Volume shape (z,y,x) | 571 × 512 × 512 |
| In-plane spacing | 0.378906 mm |
| Slice spacing | 0.335 mm |
| Voxel volume | 0.04809 mm³ |
| Total inclusion voxels | 364,385 |
| **Total inclusion volume** | **~17.5 cc** (17,525 mm³) |
| Raw connected components ≥ 30 vox | 761 |
| True inclusions after shape filter | **31** (730 membrane/artefact fragments discarded) |
| Manual 3D Slicer count | **~18** |

Shape filter criteria: min 150 vox, min bbox side ≥ 2.5 mm, fill ratio ≥ 0.15.

---

## Block 2 — Robot (KUKA iiwa14)

**Status: Gazebo running; Bug #4 open.**

KUKA LBR iiwa14 (7-DOF), ROS 2 Jazzy, Gazebo Harmonic (gz-sim8). The arm runs in Gazebo and tracks Cartesian targets.

### Three verified tests

| Test | Expected result | Status |
|---|---|---|
| Activation | Controllers switch; "Effort shutdown threshold: 500.00 Nm" logged | ✓ |
| Gravity hold | A4 effort ≈ −16.8 Nm, all joint velocities ≈ 0 | ✓ |
| Cartesian tracking | A2 +1.22 rad, A4 +0.52 rad after target (0.3, 0.0, 0.6) m | ✓ |

### Bug #4 — hardcoded 10 Nm shutdown

Fix path: (1) spawn pose — launch with vertical/zero-joints instead of A4=90°; (2) Gazebo gravity computation in the effort controller (KDL `JntToGravity`); (3) URDF inertia/mass sanity check.

---

## Block 3 — Ultrasound Simulation

Two simulators evaluated: MUSiK wave-based simulatore and NVIDIA Isaac ray-based simulator.

| Simulator | Family | Real-time | Verdict |
|---|---|---|---|
| MUSiK / k-Wave | wave-based | No (~220 s/frame floor) | Benchmarked, **ruled out** for the loop |
| NVIDIA Isaac raysim (standalone) | ray-based | Yes (~3.8 ms/frame) | **Adopted** |

### MUSiK summary

Runs 01–07 complete. Clinical pose locked: probe apex at CT voxel [256, 96, 296], beam along +Y, scan plane Z=296, depth 80 mm. The k-Wave physical floor (~220 s at grid_lambda=1.0, run 07) is a structural constraint — not a tuning problem: the acoustic round-trip over 80 mm imposes a minimum of ~2285 timesteps per scan line regardless of hardware. MUSiK retained for offline physical validation only.

### raysim summary — Stage 1 done

CT mask → OBJ mesh pipeline complete. 31 true inclusions meshed at their real CT positions (marching cubes, LPS mm, mandatory outward normals). Full anatomical scene rendered in raysim at the MUSiK apex probe pose; B-mode verified against CT axial slice Z=296.

| Metric | Value |
|---|---|
| Scene build (one-time) | ~1037 ms |
| Per-frame median (steady state) | **3.83 ms** |
| Effective FPS | **~250** |
| VRAM | **363 / 8188 MiB** (RTX 2000 Ada 8 GB) |