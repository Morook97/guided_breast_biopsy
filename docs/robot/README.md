# `feature/robot` — KUKA LBR iiwa14 simulation & control

Simulation and control of the 7-DOF KUKA LBR iiwa14 arm that carries the
ultrasound probe. This is **track 3** of the thesis. The arm moves to inclusion
positions and (in the full loop) receives ultrasound feedback; the control
strategy follows the supervisor brief (23/02/2026): Cartesian Impedance Control,
compliant along the surface normal and stiff in the scan plane, with a minimal
image-based visual servoing on the lesion centroid. Needle insertion (the biopsy
phase) is a later, safety-critical, quasi-static task.

The ultrasound simulator and the robot stack are intentionally **decoupled**
throughout development.

---

## Stack

- **Robot:** KUKA LBR iiwa14, 7-DOF, torque-controlled.
- **Middleware:** ROS 2 Jazzy.
- **Simulator:** Gazebo Harmonic (gz-sim8) on Ubuntu 24.04.
- **Build workspace:** `~/kuka_ws/` (colcon; symlinks the packages from
  `external/kuka_lbr_control`).

### Fork cascade

All forks under `Morook97/`, layered as submodules:

```
guided_breast_biopsy (feature/robot)
└── external/kuka_lbr_control          (fork of idra-lab/kuka_lbr_control)
    ├── lbr-stack/lbr_fri_ros2_stack   (fork; Gazebo bring-up patch)
    └── controllers/                   (effort controllers, incl. cartesian_impedance_controller)
        └── ros2_effort_controller     (fork of idra-lab/ros2_effort_controller)
```

The Cartesian controller used for probe-tissue contact lives in
`external/kuka_lbr_control/controllers/cartesian_impedance_controller`.

---

## Working state

The arm runs in Gazebo and tracks Cartesian targets. Three tests are documented
in `docs/kuka_run_in_gazebo_steps.md`:

| Test | Expected result |
|---|---|
| Activation | controllers switch successfully, threshold logged |
| Gravity hold | A4 effort ≈ −16.8 Nm, all joint velocities ≈ 0 |
| Cartesian tracking | A2 +1.22 rad, A4 +0.52 rad after target (0.3, 0.0, 0.6) m |

---

## Bug log (Gazebo adaptation)

The KUKA stack was written for the real FRI hardware; bringing it up in Gazebo
required adapting the command interfaces and the effort-controller shutdown
logic. **No safety parameter was ever raised to "make it work"** — this is an
inviolable supervisor rule. Effort/force/joint limits are not tunables; when the
sim failed under realistic limits, the real cause was diagnosed instead.

| Bug | Root cause | Fix | Status |
|---|---|---|---|
| #0 — effort-only interface | `gz_ros2_control` cannot load a multi-command-interface; the xacro declared both `position` and `effort` | wrapped `position` in `<xacro:unless value="${mode=='gazebo'}">` | Closed |
| #3 — wrong YAML loaded | the Gazebo plugin pointed only at the base-stack YAML, missing the custom controllers | added a second `<parameters>` tag for `gazebo_controllers.yaml` | Closed |
| #4 — hardcoded 10 Nm shutdown | `effort_controller_base.cpp` terminates if `|tau − m_efforts| > 10` on the first tick; gravity-comp at the 90° elbow needs ~17 Nm | configurable `effort_shutdown_threshold` + first-tick warmup that seeds `m_efforts` | see note |

> **Note on Bug #4.** An earlier attempt raised `delta_tau_max` (commit
> `614122c`) — this was the **wrong hypothesis** and was reverted (`8241970`,
> recorded in `9646d57 "revert rejected safety-threshold patches"`). The rejected
> C++ patches are archived on the fork branch
> `thesis/configurable-shutdown-warmup` pending a supervisor decision. The
> **real** Bug #4 fix is to be pursued in a dedicated session, investigating in
> order: (1) spawn pose (vertical / zero-joints instead of A4=90°),
> (2) Gazebo gravity computation in the effort controller (KDL `JntToGravity`),
> (3) URDF inertia/mass sanity check.

---

## Evolution / steps taken

1. **SoA review of guided-robot techniques** (Mar 2026). 15 papers read in full,
   split into US-tracking (impedance control, visual servoing, planning) and
   biopsy (complete systems, force/position, deformation). Anchor reference:
   **Ferrari 2023** (UR5e, same CIRS Model 073 phantom, full autonomous pipeline).
   Output: the HTML/PDF SoA deliverable (`SoA_robot_thesis_Moro.pdf`).
2. **kuka_lbr_control added** as a submodule, then switched to a personal fork
   to carry the Gazebo adaptations.
3. **Gazebo xacro patch** (Bug #0): single effort command interface for Gazebo.
4. **Controllers YAML fix** (Bug #3): load the custom controllers config.
5. **Bug #4 investigation**: `delta_tau_max` change tried and reverted as a wrong
   lead; configurable shutdown threshold + first-tick warmup applied; the
   safety-threshold patches were rolled back in full and archived.
6. **Working simulation** reached: the arm tracks Cartesian targets in Gazebo.
   Tagged `v0.1-sim-working` at the time, later removed during the rollback so the
   tag would not assert "fully working" over the reverted state.

---

## How to run

> Launching Gazebo needs the GUI. The exact, current launch command is in
> `docs/kuka_run_in_gazebo_steps.md`; verify the active branch first.

```
cd ~/kuka_ws
source /opt/ros/jazzy/setup.bash
source install/setup.bash
```

Then follow the launch sequence in `docs/kuka_run_in_gazebo_steps.md` to bring up
the KUKA in Gazebo with the controller active, and publish a Cartesian target (or
the joint motion) in a second terminal to make the arm move. Expected build: all
packages, zero errors.

---

## Repository layout (this branch)

```
external/kuka_lbr_control/
  controllers/
    cartesian_impedance_controller/     probe-tissue contact control
    effort_controller_base/             effort controller (Bug #4 lives here)
    gravity_compensation/
    joint_impedance_controller/
  kuka_control/{config,launch}/         bring-up launch + controller YAMLs
  lbr-stack/lbr_fri_ros2_stack/         FRI stack (Gazebo patch)
docs/
  kuka_run_in_gazebo_steps.md           launch steps + the three verified tests
  SoA_analysis_paper_topic_guided_robot.html   robot SoA
~/kuka_ws/                              colcon build workspace (outside the repo)
```

---

## Conventions

- All code, comments, commit messages in English; conventional-commit style;
  terse messages, no AI/supervisor references.
- **Safety parameters are never tuned to make the simulation pass.** If the sim
  fails under realistic limits, the system is wrong, not the limits.
- Claude Code never runs git commands and never edits safety parameters; commands
  are prepared and validated in the supervision chat first.
- Before any robot session: verify branch, recent tags, and existing solutions —
  a past Bug #4 session wasted effort by working on the wrong branch on an
  already-addressed problem.

---

## Next

- Dedicated Bug #4 session (real fix, in the ordered hypotheses above); on success
  create tag `v0.2-sim-clean-fix`.
- Wire the Cartesian Impedance Controller to the probe-contact task (compliant
  along normal, stiff in scan plane), per the supervisor brief.
- Eventually connect the robot pose to the ultrasound simulator to close the loop
  (kept decoupled until both sides are stable).
