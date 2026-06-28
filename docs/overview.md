# Project Overview — Robot-Assisted US-Guided Breast Biopsy

**Date:** 2026-06-28 | **Author:** Andrea Moro | **Status:** active

> This document supersedes the dated snapshots in `_archive/`. Those files are kept for reference but contain known errors listed inline below.

---

## Goal

Build a robot-assisted ultrasound-guided biopsy workflow for the CIRS Model 073 breast phantom, using a KUKA LBR iiwa14 arm and the NVIDIA Isaac raysim GPU ray tracer. The thesis has three loosely coupled tracks that will eventually form a closed loop: CT preprocessing → ultrasound simulation → robot control.

**Clinical objective: lesion *localisation*, not diagnostic-grade imaging.** The simulated B-mode must show *where* the bright inclusions are, fast enough for a control loop. Physical realism is approximate by design — homogeneous tissue presets, uncalibrated speckle, six fixed material presets. Relative contrast is meaningful; absolute intensity is not.

---

## Track 1 — CT Preprocessing

**Status: done (phantom); patient pipeline not yet run.**

DICOM CT of the CIRS S2010 and S3010 phantom series → normalised floating-point volume → binary inclusion mask.

Pipeline (all steps reference-grounded, non-learning constraint is a supervisor decision, Tomassini 23/02/2026):

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

> **Open question for supervisors:** the shape-filtered Python count (31) and the manual Slicer count (~18) differ. The ~18 largest of the 31 match the Slicer set visually. Both numbers are presented; neither is asserted as ground truth pending supervisor decision.

> **Correction vs `_archive/STRUCTURE_REPORT_2026-06-21.md`:** that report stated 10.77 cc — this was a calculation error. The correct voxel volume from the NRRD header is 0.378906² × 0.335 = 0.04809 mm³/vox → 17,525 mm³ = **17.5 cc**.

### Key numbers — S3010

| Metric | Value |
|---|---|
| Volume shape (z,y,x) | 577 × 512 × 512 |
| In-plane spacing | 0.333984 mm |
| Total inclusion volume | ~19.3 cc |

S2010 and S3010 do **not** share geometry. S3010 has never been used in any simulation script — all MUSiK runs and raysim work use S2010.

---

## Track 2 — Ultrasound Simulation

**Status: Stage 1 done; Stage 3 (Gazebo ↔ Isaac) next.**

Two simulators evaluated through a deliberate funnel: full SoA review → benchmarked candidate (MUSiK) ruled out on physical grounds → adopted simulator (raysim).

| Simulator | Family | Real-time | Verdict |
|---|---|---|---|
| MUSiK / k-Wave | wave-based | No (~220 s/frame floor) | Benchmarked, **ruled out** for the loop |
| NVIDIA Isaac raysim (standalone) | ray-based | Yes (~3.8 ms/frame) | **Adopted** — Stage 1 done |

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

> **Correction vs `_archive/PROJECT_OVERVIEW_2026-06-25.md`:** that report stated raysim was "Pending" and that "the mask→mesh pipeline does not exist." Both are obsolete as of Stage 1 completion.

### LPS → raysim coordinate transform (verified — reuse in Stage 3)

Apex in LPS (from `mask.TransformContinuousIndexToPhysicalPoint([256, 96, 296])`): ≈ (−8.92, 77.85, −117.51) mm.

```
depth    Z_raysim = P_y_LPS − apex_y      # MUSiK beam direction (+Y)
lateral  X_raysim = P_x_LPS − apex_x
elev     Y_raysim = P_z_LPS − apex_z
```

Verified: inclusion 9 (LPS centroid (−16.8, 113.9, −116.1)) lands at depth 36.0 mm, lateral −7.9 mm, elevation 1.5 mm — matching its CT depth. Six inclusions fall within ±5 mm of scan plane Z=296.

---

## Track 3 — Robot (KUKA iiwa14)

**Status: Gazebo running; Bug #4 open.**

KUKA LBR iiwa14 (7-DOF), ROS 2 Jazzy, Gazebo Harmonic (gz-sim8). The arm runs in Gazebo and tracks Cartesian targets.

### Three verified tests

| Test | Expected result | Status |
|---|---|---|
| Activation | Controllers switch; "Effort shutdown threshold: 500.00 Nm" logged | ✓ |
| Gravity hold | A4 effort ≈ −16.8 Nm, all joint velocities ≈ 0 | ✓ |
| Cartesian tracking | A2 +1.22 rad, A4 +0.52 rad after target (0.3, 0.0, 0.6) m | ✓ |

### Bug #4 — hardcoded 10 Nm shutdown (OPEN)

The configurable-threshold + warmup patch was reverted and archived on fork branch `thesis/configurable-shutdown-warmup` pending a supervisor decision. Tag `v0.1-sim-working` was removed during the rollback.

Real fix path (ordered): (1) spawn pose — launch with vertical/zero-joints instead of A4=90°; (2) Gazebo gravity computation in the effort controller (KDL `JntToGravity`); (3) URDF inertia/mass sanity check.

> **Correction vs `_archive/PROJECT_OVERVIEW_2026-06-25.md`:** that report marked Bug #4 as "Closed" and listed tag `v0.1-sim-working`. Both are wrong — the patch was reverted, the tag removed.

**Safety rule (inviolable):** no safety threshold is ever raised to make the simulation pass. If the sim fails under realistic limits, the system is wrong, not the limits.

---

## Open questions for supervisor meeting

| # | Question | Track |
|---|---|---|
| 1 | Inclusion count: 31 (Python shape filter) vs ~18 (manual Slicer) — which is ground truth for the full scene? | Preprocessing / Simulation |
| 2 | Bug #4 fix path — supervisor decision needed before proceeding | Robot |
| 3 | raysim fidelity bar — define the visual acceptance criterion for "the mass is localised" | Simulation |
| 4 | Patient pipeline — body-mask + breast-region extraction strategy (phantom band [0.30, 0.85] does not transfer to patients) | Preprocessing |
| 5 | Stage 3 transport — ROS 2 node wrapping raysim, shared pose topic, or other mechanism for Gazebo ↔ Isaac | Simulation / Robot |
