# Consolidated Key Results — All Three Tracks

> Last updated 2026-06-28. All numbers are measured values from actual runs, not estimates.

---

## Track 1 — CT Preprocessing

| Metric | S2010 | S3010 |
|---|---|---|
| CT shape (z,y,x vox) | 571 × 512 × 512 | 577 × 512 × 512 |
| In-plane spacing (mm) | 0.378906 | 0.333984 |
| Slice spacing (mm) | 0.335 | 0.335 |
| Total inclusion voxels | 364,385 | 517,256 |
| **Total inclusion volume** | **~17.5 cc** | ~19.3 cc |
| Raw CC components ≥ 30 vox | 761 | — |
| **True inclusions (shape filter)** | **31** | — |
| Filtered out (membrane/artefact) | 730 | — |
| Manual 3D Slicer count | **~18** | — |

Shape filter: min 150 vox, min bbox side ≥ 2.5 mm, fill ratio ≥ 0.15.

> **Open — 31 vs ~18:** the shape-filtered Python count and the manual Slicer count differ. The ~18 largest of the 31 match the Slicer set visually. Present both, assert neither pending supervisor decision.
>
> The 17.5 cc figure supersedes the 10.77 cc calculation error in `_archive/STRUCTURE_REPORT_2026-06-21.md`.

---

## Track 2 — Ultrasound Simulation

### MUSiK / k-Wave (ruled out for the loop)

| Run | Config | Wall-clock |
|---|---|---|
| 05 — clinical | 0.25 mm voxel, 32 rays | 462.1 min |
| 06 — fast calibration | 0.40 mm voxel, 4 rays | 60.9 min |
| 07 — k-Wave floor | grid_lambda=1.0, 4 rays | ~220 s (15.6 min) |

**Physical floor: ~220 s per acquisition — structurally not real-time (hardware-independent).**

### raysim — Stage 1 done (adopted)

| Metric | Value |
|---|---|
| Scene build (one-time) | ~1037 ms |
| Per-frame median (steady state) | **3.83 ms** |
| Effective FPS | **~250** |
| VRAM peak | **363 / 8188 MiB** |
| Hardware | RTX 2000 Ada 8 GB |

31 real phantom inclusions rendered at CT positions. Real-time confirmed.

---

## Track 3 — Robot (Gazebo)

| Test | Result |
|---|---|
| Activation | ✓ verified |
| Gravity hold (A4 ≈ −16.8 Nm) | ✓ verified |
| Cartesian tracking (target 0.3, 0.0, 0.6 m) | ✓ verified |
| **Bug #4** | **OPEN** — patch reverted, pending supervisor decision |

Tag `v0.1-sim-working` removed during rollback.

---

## Open questions — all tracks

| # | Question | Track |
|---|---|---|
| 1 | Inclusion count: 31 (Python shape filter) vs ~18 (manual Slicer) — which is ground truth for the full anatomical scene? | Preprocessing / Simulation |
| 2 | Bug #4 real fix path — supervisor decision needed before proceeding | Robot |
| 3 | raysim fidelity acceptance criterion: define "the mass is localised" visually | Simulation |
| 4 | Patient pipeline: body-mask + breast-region extraction strategy (phantom band [0.30, 0.85] does not transfer) | Preprocessing |
| 5 | Stage 3 transport: Gazebo ↔ Isaac — ROS 2 node, shared pose topic, rate, frame chain | Simulation / Robot |
