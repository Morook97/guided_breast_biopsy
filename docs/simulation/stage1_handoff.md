# Simulation Track — Stage 1 Handoff / README

Robot-assisted ultrasound-guided breast biopsy — synthetic ultrasound from CT phantom data.
This document summarises what the simulation track achieved and hands off to **Stage 33
(Gazebo <-> Isaac communication)**.

---

## 1. One-line summary

From the segmented CT of the CIRS S2010 phantom we build triangle meshes of the inclusions,
load them into the NVIDIA Isaac **raysim** GPU ray tracer, and generate B-mode ultrasound
images in real time (~3.8 ms/frame, ~250 FPS). Inclusion positions are verified against the CT.
Goal is lesion **localisation**, not diagnostic imaging.

---

## 2. Status — what is done

- CT mask -> OBJ mesh pipeline (marching cubes, physical mm, normals) — DONE.
- 31 true inclusions identified from the binary mask (membrane/artefacts filtered out).
- raysim Python API recovered and driven from our own scripts.
- Anatomical scene: inclusions at their REAL positions, probe at the MUSiK apex pose,
  B-mode correlated to CT axial slice Z=296 (the raysim equivalent of MUSiK run07).
- Real-time confirmed: scene built once, probe pose swept, every frame timed.

What is NOT done (next stages): driving the probe pose from the real KUKA arm (Stage 33);
patient-specific tissue background; quantitative acoustic calibration.

---

## 3. Repositories, environments, key paths

- Thesis repo: `~/guided_breast_biopsy`  (branch `feature/simulation`, local commits only, no push)
  - Simulation work lives under `~/guided_breast_biopsy/simulation/`
    (`scripts/`, `meshes/`, `figures/`, `recon/`, `PROCESS_LOG.md`)
  - Data analysis venv (SimpleITK, scipy, scikit-image):
    `~/guided_breast_biopsy/py_venv/bin/python3`
- raysim engine repo (separate, NOT the thesis repo): `~/isaac_us_test/ultrasound-raytracing`
  - Compiled engine: `build-release/ray_sim_python*.so` (C++/CUDA, OptiX) — never edited by us
  - Python env that can `import raysim.cuda`: `~/isaac_us_test/ultrasound-raytracing/.venv/bin/python3.10`
  - Note: `build_and_run.sh` has one intentional local edit
    (`NVIDIA_DRIVER_CAPABILITIES=compute,graphics,utility`, needed for OptiX/GL). Commit it in
    the raysim repo separately when convenient.

Input data:
- `output/preprocessing_data/paper_based/S2010/mask_inclusions.nrrd` — binary k=3 inclusion mask,
  uint8, shape (z,y,x) = 571x512x512, spacing 0.378906 x 0.378906 x 0.335 mm, LPS.
- `output/preprocessing_data/paper_based/S2010/filtered_volume.nrrd` — CT volume, [0,1] float, same geometry.

---

## 4. The CT -> mesh -> B-mode pipeline

1. **Inclusion inventory.** Connected components on the binary mask -> 761 components >=30 vox.
   The k=3 mask is contaminated by the phantom membrane/base (thin 1-voxel laminae). A shape
   filter (min 150 vox, min bbox side >= 2.5 mm, fill ratio >= 0.15) keeps 31 compact true
   inclusions; the ~18 largest match the manual 3D Slicer count.
2. **Mesh export.** Per inclusion: crop + light Gaussian smoothing, marching cubes
   (scikit-image), vertices from voxel index to physical LPS mm via the image affine (verified
   against `TransformContinuousIndexToPhysicalPoint`), outward per-vertex normals, single-object
   OBJ in millimetres. This matches the raysim data contract (one mesh per file, triangles only,
   mandatory normals, mm).
3. **Render.** Load meshes into a raysim `World`, assign materials, set the probe pose,
   call `simulate()` -> dB B-mode image with a `-FLT_MAX` sentinel for non-insonified pixels.

---

## 5. The LPS -> raysim coordinate transform (VERIFIED — reuse this in Stage 33)

MUSiK probe geometry: apex at CT voxel **[256, 96, 296]**, beam along **+Y** (apex -> chest),
scan plane Z=296. The apex in LPS is computed with
`mask.TransformContinuousIndexToPhysicalPoint([256,96,296])` (~ (-8.92, 77.85, -117.51) mm).

raysim probe frame: depth = +Z, lateral = X, elevational = Y. Mapping (per vertex P in LPS):

```
depth     Z_raysim = P_y_LPS - apex_y      # MUSiK beam direction (+Y)
lateral   X_raysim = P_x_LPS - apex_x
elevation Y_raysim = P_z_LPS - apex_z
```

i.e. swap the LPS Y and Z axes and translate the apex to the origin. Verified: inclusion 9
(LPS centroid (-16.8, 113.9, -116.1)) lands at depth 36.0 mm, lateral -7.9 mm, elevation 1.5 mm
— matching its CT depth. Six inclusions fall within +-5 mm of the scan plane Z=296:

| label | depth (mm) | lateral (mm) |
|-------|-----------:|-------------:|
| 36    | 30.1       | 24.0         |
| 9     | 36.0       | -7.9         |
| 5     | 45.7       | 34.1         |
| 17    | 48.6       | -9.2         |
| 135   | 70.1       | -0.8         |
| 128   | 70.6       | -18.6        |

Verification was visual: the B-mode bright spots align with the red inclusions on the CT slice.

---

## 6. The raysim control API (what our scripts call)

```python
import raysim.cuda as rs
materials = rs.Materials()
world = rs.World("water")                                   # ambient medium
world.add(rs.Mesh(box_obj,  materials.get_index("liver")))  # background tissue
world.add(rs.Mesh(incl_obj, materials.get_index("fat")))    # hyperechoic inclusion
sim = rs.RaytracingUltrasoundSimulator(world, materials)    # build scene ONCE
sp = rs.SimParams(); sp.conv_psf = True; sp.buffer_size = 4096
sp.t_far = 80.0; sp.b_mode_size = (500, 500)
probe = rs.CurvilinearProbe(rs.Pose(position=pos_mm, rotation=rot_rad))  # pose drives the view
img = sim.simulate(probe, sp)                               # ~4 ms -> dB image (np array)
```

Six fixed materials: water, blood, fat, liver, muscle, bone. Only relative contrast matters
(absolute echo intensity is not meaningful). Hyperechoic inclusion on tissue: background=liver,
inclusion=fat (chosen by a 4-material comparison). Multiple meshes per scene are allowed;
mesh and sphere primitives cannot be mixed.

---

## 7. Results and timing (RTX 2000 Ada 8 GB)

| metric | value |
|--------|-------|
| scene build (one-time) | ~1037 ms |
| per-frame min | 3.20 ms |
| per-frame mean | 4.00 ms |
| per-frame median (steady state) | 3.83 ms |
| per-frame max | 6.66 ms (first frame, GPU warm-up) |
| effective FPS | ~250 |
| VRAM | 363 / 8188 MiB |

Probe-sweep test: scene built once, 60 probe poses (lateral slide + gentle tilt), every frame
timed; the masses move through the fan as the probe slides (only the pose changes, no rebuild).
For reference, MUSiK (full-wave) needs ~220 s for one comparable frame; raysim and MUSiK are
complementary (raysim = fast localisation loop, MUSiK = high-fidelity reference).

---

## 8. Validity and limitations (state these to supervisors)

- Timing is honest wall-clock, scene built once; slightly pessimistic vs CUDA-event timing.
  Steady-state cost ~3.8 ms/frame.
- Geometry is verified against the CT; the probe reproduces the MUSiK apex pose.
- Physical realism is approximate BY DESIGN (goal = localisation): homogeneous liver-preset
  background (not patient-specific tissue), six fixed material presets (not measured CIRS
  acoustic properties), plausible-but-uncalibrated speckle. Relative contrast is meaningful,
  absolute intensity is not.

---

## 9. File map (simulation/scripts/)

Data prep (run with `py_venv`):
- `find_inclusions.py` / `inclusions_stats.py` — inclusion inventory + shape filter
- `verify_inclusion.py` — overlay a chosen inclusion on the CT (3 planes)
- `mesh_export.py` — single inclusion mask -> OBJ (LPS mm)
- `anat_A_mesh.py` — select in-slice inclusions, mesh them transformed to raysim coords,
  export the CT slice for the overlay

raysim control (run with raysim `.venv/bin/python3.10`, on GPU):
- `inclusion_test.py` — first pipeline/timing test (single inclusion, water)
- `inclusion_mat.py` — material comparison (water/muscle/fat/bone)
- `anat_B_render.py` — anatomical scene, water background (run08)
- `anat_B2_render.py` — anatomical scene, tissue background (run09)
- `anat_sweep.py` — scene built once, probe swept, per-frame timing (run10)  <-- closest to the robot loop

---

## 10. Next stage (33) — Gazebo <-> Isaac

Goal: drive the probe pose from the simulated KUKA arm instead of a scripted sweep.

Plan (the simulation bottleneck is already solved):
1. Build the raysim scene ONCE (tissue + inclusion meshes), as in `anat_sweep.py`.
2. Each iteration: read the KUKA end-effector / probe pose from Gazebo (robot track).
3. Convert that pose into the raysim probe frame using the SAME LPS->raysim transform
   in Section 5 (this is the integration glue to write).
4. `rs.CurvilinearProbe(rs.Pose(position, rotation))` -> `sim.simulate()` (~4 ms) -> display.
5. Loop.

Open questions for Stage 33:
- Transport between Gazebo (ROS2) and the raysim process (a ROS2 node wrapping raysim? a shared
  pose topic? rate?).
- Exact frame chain: world/base -> end-effector -> probe tip -> raysim probe frame
  (compose the robot TF with the LPS->raysim transform; verify on a known pose).
- Probe model match: raysim CurvilinearProbe defaults vs the real clinical probe (sector, depth).
- Whether to keep the homogeneous tissue block or move to a patient-specific tissue mesh.

---

## 11. Workflow conventions (for the new chat to respect)

- Chat in Italian; all code, commits, comments, READMEs in English (Conventional Commits;
  no supervisor names, no bug numbers, not AI-sounding).
- Two coordinate-frame venvs: `py_venv` (SimpleITK/skimage, meshing) and raysim `.venv`
  (GPU render). Keep them separate; pass data as OBJ files + numpy .npz.
- One thesis track per Claude Code session. Claude Code never runs git; read-only except for
  explicitly named output targets; never writes to `.claude/` or memory files. Commands one per
  line, no `&&` chaining.
- Verify before trusting (the k-Wave lesson): check transforms on the image/overlay, not on faith.
- No time estimates unless measured in the current session.
- raysim repo (`~/isaac_us_test`) is third-party: keep it clean, do not modify beyond the one
  documented `build_and_run.sh` edit.
