# Robot Track — Key Results

## Gazebo stack

- **Robot:** KUKA LBR iiwa14, 7-DOF, torque-controlled
- **Middleware:** ROS 2 Jazzy
- **Simulator:** Gazebo Harmonic (gz-sim8), Ubuntu 24.04
- **Build workspace:** `~/kuka_ws/` (colcon; 23 packages, zero errors expected)

---

## Three verified tests

| Test | Expected result | Status |
|---|---|---|
| Activation | Controllers switch; log: "Effort shutdown threshold: 500.00 Nm" + "Successfully switched controllers!" | ✓ verified |
| Gravity hold | A4 effort ≈ −16.8 Nm; all joint velocities ≈ 0 | ✓ verified |
| Cartesian tracking | A2: +1.22 rad, A4: +0.52 rad after target (0.3, 0.0, 0.6) m | ✓ verified |

Full launch sequence in `docs/kuka_run_in_gazebo_steps.md` (main repo).

---

## Bug log

| Bug | Root cause | Fix | Status |
|---|---|---|---|
| #0 — effort-only interface | `gz_ros2_control` cannot load multi-command-interface; xacro declared `position` + `effort` | Wrapped `position` in `<xacro:unless value="${mode=='gazebo'}">` | **Closed** |
| #3 — wrong YAML loaded | Gazebo plugin pointed only at base-stack YAML, missing custom controllers | Added second `<parameters>` tag for `gazebo_controllers.yaml` | **Closed** |
| #4 — hardcoded 10 Nm shutdown | `effort_controller_base.cpp` terminates if `|tau − m_efforts| > 10` on first tick; gravity-comp at 90° elbow needs ~17 Nm | — | **OPEN** |

### Bug #4 — detailed status

The configurable-threshold + warmup patch (`effort_shutdown_threshold` YAML parameter + first-tick `m_efforts` seed) was reverted and archived on fork branch `thesis/configurable-shutdown-warmup` pending a supervisor decision.

Tag `v0.1-sim-working` (commit `74fbb67`) was removed during the rollback so it would not assert "fully working" over the reverted state.

**Safety rule (inviolable):** no safety parameter is ever raised to make the simulation pass. Effort/force/joint limits are not tunables; if the sim fails under realistic limits, the system is wrong, not the limits.

**Real fix path (ordered hypothesis list):**
1. Spawn pose — launch with vertical/zero-joints instead of A4=90°
2. Gazebo gravity computation in the effort controller (`KDL::JntToGravity` correctness)
3. URDF inertia/mass sanity check

Next milestone on successful fix: create tag `v0.2-sim-clean-fix`.

---

## Next steps

- Dedicated Bug #4 fix session (supervisor decision on path first)
- Wire Cartesian Impedance Controller to probe-contact task (compliant along surface normal, stiff in scan plane), per supervisor brief (23/02/2026)
- Connect robot pose to raysim to close the loop (kept decoupled until both sides are stable)
