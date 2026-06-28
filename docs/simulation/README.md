# `feature/simulation` — Ultrasound image simulation

Generation of a B-mode ultrasound image from the phantom CT, given a probe pose,
to close the control loop with the robot. This is **track 2** of the thesis.
The supervisor-confirmed goal is **lesion localisation**, not diagnostic-grade
imaging: the simulated image must reveal *where* the bright inclusions are, fast
enough for the loop (seconds, ideally real-time).

The track went through a deliberate funnel: a full state-of-the-art review, then
a benchmarked candidate (MUSiK / k-Wave) that was **ruled out on physical
grounds**, then the adopted simulator (NVIDIA Isaac standalone raytracer,
"raysim").

---

## Decision summary

| Simulator | Family | Real-time | Verdict |
|---|---|---|---|
| **MUSiK / k-Wave** | wave-based | No (~220 s/acq floor) | Implemented, benchmarked, **ruled out** for the loop; kept for offline physical validation |
| Isaac full workflow | ray-based | Yes | **Blocked by hardware** (needs 24 GB VRAM / 64 GB RAM) |
| **Isaac standalone (raysim)** | ray-based | Yes (~3.8 ms/frame) | **Adopted** — Stage 1 done |
| Field II / virtual-breast-project | wave-based | No | Offline ground-truth reference only |

Full reasoning, 32 curated papers and the feasibility table are in the SoA
deliverable (`../soa/SoA_simulators.pdf`).

---

## Why MUSiK was ruled out (the physical floor)

MUSiK orchestrates **k-Wave**, a full-wave acoustic solver: it integrates the
acoustic wave equation on a 3-D grid, timestep by timestep, for every scan line
("ray"). This is physically accurate but the cost is bounded by physics, not by
the GPU:

- The clinical reference run (run 05: voxel 0.25 mm, 32 rays) took **462 minutes**
  for a single B-mode — against a target of seconds-to-5-minutes.
- The `voxel_size` parameter passed to `SimProperties` is **ignored**:
  `optimize_simulation_parameters` regenerates the grid internally from
  `voxel_size = SoS / (2 * max_frequency * grid_lambda)`. The real cost driver is
  `grid_lambda` (hardcoded to 2); total cost scales as `grid_lambda^4`.
- Even at the absolute Nyquist floor (run 07: `grid_lambda = 1.0`), the cost stays
  **~220 s per acquisition** (~72 s GPU + ~150 s CPU prep). The acoustic
  round-trip over 80 mm at ~1540 m/s imposes a minimum of ~2285 timesteps
  **regardless of hardware** — a 100× faster GPU would reach minutes, never
  sub-second.

Conclusion: real-time is **structurally impossible** with k-Wave; this is not a
tuning problem. All timing claims come from real run logs (`timing_*.json`), not
estimates.

---

## What raysim is (the adopted simulator)

`i4h-sensor-simulation/ultrasound-raytracing` (NVIDIA Isaac for Healthcare),
the standalone module isolated from the unusable full workflow. A C++/CUDA OptiX
raytracer with Python bindings whose only job is: take a triangle mesh of the
anatomy plus a 6-DoF probe pose, return a B-mode image.

Stage 0 closed: verified native on the RTX 2000 Ada at **~5 ms/frame** initial,
**~3.8 ms/frame median** in steady-state, **363 MiB VRAM peak**, with the
mesh→B-mode pipeline validated across three demos (sphere → sphere-in-oval →
anatomical liver).

Stage 1 done: real phantom inclusions (31 true inclusions from S2010 CT mask)
meshed at their actual CT positions, anatomical scene rendered, probe swept at 60
poses, real-time confirmed. See `stage1_handoff.md` for the full pipeline and
coordinate transform recipe.

**raysim data contract** (learned during Stage 0): one mesh per OBJ file,
triangles with mandatory normals, units in mm; only 6 predefined materials
(contrast matters, not absolute values); the phantom inclusions are
hyperechoic, so materials are inverted relative to the rehearsal; un-insonified
pixels are `-FLT_MAX` and must be masked.

---

## Reusable asset: probe geometry from MUSiK

The probe/coordinate geometry from the MUSiK scripts is **portable to raysim** and
saves re-deriving the LPS→simulator transform. Clinical pose, verified at runtime
(9/9 checks in the run05 log): probe apex at CT voxel `[256, 96, 296]`, beam along
`+Y` (apex→chest wall), scan plane `Z=296` (≈ `Z_LPS −117.5`), depth 80 mm,
sweep 15°, focus 40 mm.

---

## Evolution / steps taken

1. **SoA review of US simulators** (Apr 2026). Systematic Scopus search, 4 queries
   executed, 3069 rows → 2723 deduplicated → 32 curated papers in 6 thematic
   blocks. Classification (wave / ray / convolution / learning) follows
   Solano-Cordero et al., MDPI 2025. Output: the HTML/PDF SoA deliverables.
2. **MUSiK installed** as a submodule (`external/MUSiK`, fork `Morook97/MUSiK`),
   k-Wave GPU backend, dedicated venv `venv_musik`. Acoustic phantom built from
   `filtered_volume.nrrd` + `mask_inclusions.nrrd`.
3. **MUSiK runs 01–07** (`notebooks/musik/runs/`): from initial phantom setup to
   the clinical run (05), fast calibration (06) and the k-Wave floor benchmark
   (07). Runs 05–07 have full JSON configs and timing logs.
4. **MUSiK ruling**: timing analysis proved the ~220 s physical floor. Documented
   in `MUSiK_simulator.md`. MUSiK reclassified as offline validation tool.
5. **raysim adoption** (`~/isaac_us_test/`). Stage 0 closed: bare-metal + Docker
   builds verified, ~3.8 ms/frame median on the Ada, mesh→B-mode validated on
   three demos. Install fixes recorded: CUDA arch `sm_89` (not `sm_80`),
   `NVIDIA_DRIVER_CAPABILITIES=compute,graphics,utility`, GCC 13 / CUDA 12.6
   resolved via `CMAKE_CUDA_HOST_COMPILER=/usr/bin/g++-12`.
6. **Stage 1 (done)**: CT mask → OBJ mesh pipeline complete. Connected-component
   analysis of `mask_inclusions.nrrd` (S2010): 761 components ≥30 voxels; shape
   filter (min 150 vox, min bbox side ≥2.5 mm, fill ratio ≥0.15) selects **31
   true inclusions** (730 membrane/artefact fragments discarded). The ~18 largest
   match the manual 3D Slicer count — the 31-vs-18 reconciliation is an open
   question for supervisors. Per-inclusion: crop + Gaussian smooth + marching
   cubes → OBJ in LPS mm with outward normals. Full anatomical scene rendered with
   inclusions at real CT positions and the MUSiK apex probe pose. Probe-sweep test
   (60 poses): **~3.8 ms/frame median, ~250 FPS, 363 MiB VRAM**. See
   `stage1_handoff.md` for the coordinate transform recipe and API details.

---

## Status

- **MUSiK:** complete and benchmarked. Ruled out for the loop, retained for
  offline physical validation.
- **raysim Stage 0:** closed (pipeline validated on demo scenes).
- **raysim Stage 1:** **done** — CT mask → OBJ mesh → anatomical scene → real-time
  B-mode of real phantom inclusions confirmed.
- **Stage 3 (next):** drive probe pose from the simulated KUKA arm (Gazebo ↔
  Isaac communication). The simulation bottleneck is solved; the open piece is
  the ROS 2 / pose transport between Gazebo and the raysim process.

---

## Repository layout (simulation work)

> `notebooks/musik/` is **not tracked** by git (runs, logs, figures are local
> only). Back it up: if the machine is re-imaged, this work is lost.

```
external/MUSiK/                          MUSiK + k-wave-python (submodule)
notebooks/musik/
  01_phantom_from_cirs_ct.ipynb          CT -> acoustic phantom
  run_05_clinical_sim.py                 clinical run (462 min)
  run_06_fast_sim.py                     fast calibration (61 min)
  run_07_floor_test.py                   k-Wave floor benchmark (16 min)
  runs/                                  per-run configs + phantom + results
  logs/                                  sim logs + timing_*.json
  figures/                               B-mode + CT overlays
~/isaac_us_test/ultrasound-raytracing/   raysim (OptiX) — standalone module
simulation/
  scripts/                               mesh export, raysim control, sweep scripts
  meshes/                                per-inclusion OBJ files
  figures/                               B-mode outputs, CT overlays, sweep frames
  recon/                                 inclusion inventory data
  PROCESS_LOG.md                         session-by-session work log
```

---

## Conventions

- Two virtualenvs: `py_venv` (preprocessing/analysis, meshing) and raysim `.venv`
  (GPU render, Python 3.10). Keep them separate; pass data as OBJ files + numpy
  .npz.
- All code/comments/commits in English; conventional-commit style.
- Mesh conversion done in Python (reproducible, deterministic coordinates), not
  in Slicer.
- Verify transforms on the image/overlay, not on faith (the k-Wave lesson).

---

## Open questions (for supervisors)

- **Reference inclusion set:** 31 compact inclusions (Python shape filter) vs ~18
  (manual 3D Slicer count). The 31-vs-18 reconciliation is open — present both,
  assert neither as ground truth.
- **raysim fidelity bar:** localisation confirmed as the goal — define the visual
  acceptance criterion for "the mass is localised" against the CT overlay.
- **Stage 3 transport:** ROS 2 node wrapping raysim, shared pose topic, or other
  mechanism for Gazebo ↔ Isaac pose communication (rate, frame chain).
