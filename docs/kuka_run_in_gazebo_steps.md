# KUKA LBR iiwa14 Adaptation for Running on Ubuntu 24.04, ROS2 Jazzy in Gazebo

---

## Repository Architecture

```
Morook97/guided_breast_biopsy  @  feature/robot  @  74fbb67
└── external/kuka_lbr_control  →  Morook97/kuka_lbr_control  @  main  @  1d3fb13
    ├── lbr-stack/lbr_fri_ros2_stack  →  Morook97/lbr_fri_ros2_stack  @  thesis/gazebo-patch  @  c654981
    └── controllers  →  Morook97/ros2_effort_controller  @  thesis/configurable-shutdown-warmup  @  ae656f8
```

---

## Bug #0 — Effort-only command interface in Gazebo

**File modified:** `lbr-stack/lbr_fri_ros2_stack/lbr_description/ros2_control/lbr_system_interface.xacro`  
**Commit:** `b3001c3` on `Morook97/lbr_fri_ros2_stack @ thesis/gazebo-patch`

### Problem
`gz_ros2_control` does not support multi-command-interface loading (upstream issue `gz_ros2_control#182`). The lbr-stack xacro declared both `position` and `effort` command interfaces by default. Gazebo could not handle this and crashed at hardware initialization.

### Fix
Wrapped the `position` command interface declaration in `<xacro:unless value="${mode == 'gazebo'}">`, leaving only the `effort` interface active in Gazebo mode. Real hardware deployment keeps both interfaces unchanged.

### Why this approach
Additive and minimally invasive — does not change the interface for hardware, only conditionally excludes `position` when running in simulation.

---

## Bug #3 — Gazebo plugin loading wrong YAML config

**File modified:** `lbr-stack/lbr_fri_ros2_stack/lbr_description/gazebo/lbr_gazebo.xacro`  
**Commit:** `c654981` on `Morook97/lbr_fri_ros2_stack @ thesis/gazebo-patch`

### Problem
The `gz_ros2_control` plugin xacro originally pointed to a single YAML file:
```xml
<parameters>$(find lbr_description)/ros2_control/lbr_controllers.yaml</parameters>
```
That file contains only the base lbr-stack controllers (`joint_state_broadcaster`, `joint_trajectory_controller`, etc.). The professor's custom controllers (`cartesian_impedance_controller`, `gravity_compensation`) are declared in `kuka_control/config/gazebo_controllers.yaml`. As a result, when the launch requested `ctrl:=cartesian_impedance_controller`, the controller_manager answered "type not defined" because it had never seen that YAML.

### Fix
Added a second `<parameters>` tag pointing to the professor's YAML, keeping the first intact:
```xml
<parameters>$(find lbr_description)/ros2_control/lbr_controllers.yaml</parameters>
<parameters>$(find kuka_control)/config/gazebo_controllers.yaml</parameters>
```

### Why this approach
The official `gz_ros2_control` documentation confirms the `<parameters>` tag can appear multiple times. This is purely additive — the base lbr-stack controllers are preserved alongside the custom ones.

---

## Bug #4 — Hardcoded 10 Nm shutdown threshold crashes Gazebo at activation

This was the main bug requiring the most work. It involved changes across three repositories.

### Root Cause
In `effort_controller_base/src/effort_controller_base.cpp`, the function `computeJointEffortCmds` contained a hardcoded safety check:
```cpp
const double difference = tau[i] - m_efforts[i];
if (std::abs(difference) > 10.0) {   // hardcoded magic number
    std::terminate();                  // crashes the entire Gazebo process
}
```
`m_efforts[i]` is initialized to `0` in `on_configure`. On the first control tick after activation, the Cartesian Impedance Controller computes gravity-compensation torque for joint A4 starting at 90° — approximately **-17.16 Nm**. The difference `|0 - (-17.16)| = 17.16 > 10.0` triggered `std::terminate()`, crashing Gazebo with a generic Ubuntu dialog.

The 10.0 Nm threshold makes sense on real hardware, where the robot has its own internal gravity compensation and first-tick torques from the external controller are small. In Gazebo there is no internal compensation, so the first tick from a non-vertical initial pose legitimately requires large torque.

### False Lead — delta_tau_max
Before understanding the root cause, `delta_tau_max` was raised from `1.0` to `50.0` in the YAML. This had no effect because `delta_tau_max` is a **per-cycle saturation** applied **after** the hardcoded `10.0` check. The check itself was the problem. This change was reverted in commit `e21a5df`.

---

### Fix Part A — C++ patch to effort_controller_base

**Repository:** `Morook97/ros2_effort_controller`  
**Branch:** `thesis/configurable-shutdown-warmup`  
**Commit:** `ae656f8`  
**Files modified:**
- `effort_controller_base/include/effort_controller_base/effort_controller_base.h`
- `effort_controller_base/src/effort_controller_base.cpp`

#### Changes

**Header (`effort_controller_base.h`)** — added two new members:
```cpp
double m_effort_shutdown_threshold;  // configurable shutdown threshold (Nm)
bool m_first_update;                 // true on first tick after activation
```

**`on_init()` in the .cpp** — declared the new ROS parameter with default 10.0 Nm (safe for real hardware):
```cpp
auto_declare<double>("effort_shutdown_threshold", 10.0);
```

**`on_configure()` in the .cpp** — read and validated the parameter:
```cpp
m_effort_shutdown_threshold = get_node()->get_parameter("effort_shutdown_threshold").as_double();
```

**`on_activate()` in the .cpp** — reset the warmup flag on every activation:
```cpp
m_first_update = true;
```

**`computeJointEffortCmds()` in the .cpp** — two modifications:
1. On the first tick (`m_first_update == true`), skip both the shutdown check and the delta_tau saturation, setting `m_efforts[i] = tau[i]` directly. This is the "warmup" — it initializes `m_efforts` to the actual gravity-compensation torque so that subsequent ticks see zero difference.
2. Replaced the hardcoded `10.0` with `m_effort_shutdown_threshold`, making the threshold configurable per-robot via YAML without recompiling.

#### Why both fixes (warmup + configurable threshold)
The warmup alone elegantly handles the activation transient with zero delay and is the primary fix. The configurable threshold provides a secondary safety net that can be tuned per-robot and also fixes the problem on any subsequent ticks where a large but legitimate torque step might occur during fast Cartesian motions.

---

### Fix Part B — YAML parameter for Gazebo

**Repository:** `Morook97/kuka_lbr_control`  
**Commit:** `556aec0` then `1d3fb13`  
**File modified:** `kuka_control/config/gazebo_controllers.yaml`

Added `effort_shutdown_threshold` to both controller sections:

```yaml
# cartesian_impedance_controller section
effort_shutdown_threshold: 500.0  # Nm (simulation-friendly)

# gravity_compensation section  
effort_shutdown_threshold: 500.0  # Nm (simulation-friendly)
```

The value was initially set to `100.0` Nm (sufficient to survive activation) and later raised to `500.0` Nm to allow large Cartesian target tracking. With stiffness 1000 N/m, moving the end-effector ~0.3 m requires transient joint torques up to ~280 Nm on A2.

For real hardware, the default in the C++ code (`10.0 Nm`) applies unless overridden in the hardware YAML.

---

### Fix Part C — Submodule and fork configuration

To apply patches to the professor's code without modifying his repository directly, a four-level Git fork cascade was set up:

| Repository | Branch | Role |
|---|---|---|
| `Morook97/ros2_effort_controller` | `thesis/configurable-shutdown-warmup` | C++ patches |
| `Morook97/kuka_lbr_control` | `main` | YAML patches + submodule pointer |
| `Morook97/lbr_fri_ros2_stack` | `thesis/gazebo-patch` | xacro patches (Bug #0 and #3) |
| `Morook97/guided_breast_biopsy` | `feature/robot` | Thesis root, tracks all submodule pointers |

The `.gitmodules` file in `Morook97/kuka_lbr_control` was updated to point `controllers` at the fork instead of the original `idra-lab/ros2_effort_controller`. The upstream remote is preserved locally to allow future `git fetch upstream` to pull updates from the professor.

---

## Final State — Verified Working

**Test 1 — Activation (startup):** Controller activates without crash. Log confirms:
```
Effort shutdown threshold: 500.00 Nm
Successfully switched controllers!
```

**Test 2 — Gravity hold:** With no target published, the arm holds its initial pose under gravity compensation. Joint positions stable at initial values, effort on A4 ~16.8 Nm (correct gravity compensation).

**Test 3 — Cartesian tracking:** After publishing target `(0.3, 0.0, 0.6)` with orientation `(0, 0.707, 0, 0.707)`, joint positions changed measurably (A2: +1.22 rad, A4: +0.52 rad) and Gazebo remained open.

---

## Git State at Completion

| Repository | Branch | Tag | HEAD |
|---|---|---|---|
| `Morook97/guided_breast_biopsy` | `feature/robot` | `v0.1-sim-working` | `74fbb67` |
| `Morook97/kuka_lbr_control` | `main` | — | `1d3fb13` |
| `Morook97/lbr_fri_ros2_stack` | `thesis/gazebo-patch` | — | `c654981` |
| `Morook97/ros2_effort_controller` | `thesis/configurable-shutdown-warmup` | — | `ae656f8` |

---

## Fork Strategy — Step by Step

The professor's code lives across multiple upstream repositories. All upstream dependencies were forked to guarantee reproducibility, keep a clean audit trail, and allow eventual PRs upstream. Below is the complete sequence of operations performed to set up the fork cascade.

### Step 1 — Fork lbr_fri_ros2_stack

The lbr-stack repository needed xacro patches (Bug #0 and Bug #3). Forked via GitHub web UI to `Morook97/lbr_fri_ros2_stack`, then:

```bash
cd ~/guided_breast_biopsy/external/kuka_lbr_control/lbr-stack/lbr_fri_ros2_stack
git remote rename origin upstream
git remote add origin https://github.com/Morook97/lbr_fri_ros2_stack.git
git fetch origin
git checkout -b thesis/gazebo-patch
# apply patches (Bug #0 and Bug #3)
git add <files>
git commit -m "..."
git push -u origin thesis/gazebo-patch
```

Then updated `.gitmodules` in `kuka_lbr_control`:
```bash
cd ~/guided_breast_biopsy/external/kuka_lbr_control
git config -f .gitmodules submodule.lbr_stack/lbr_fri_ros2_stack.url https://github.com/Morook97/lbr_fri_ros2_stack.git
git config -f .gitmodules submodule.lbr_stack/lbr_fri_ros2_stack.branch thesis/gazebo-patch
git submodule sync lbr-stack/lbr_fri_ros2_stack
```

### Step 2 — Fork kuka_lbr_control

The professor's main control repository needed YAML patches. Forked via GitHub web UI to `Morook97/kuka_lbr_control`, then:

```bash
cd ~/guided_breast_biopsy/external/kuka_lbr_control
git remote rename origin upstream
git remote add origin https://github.com/Morook97/kuka_lbr_control.git
git fetch origin
# apply YAML patches
git add kuka_control/config/gazebo_controllers.yaml
git commit -m "..."
git push origin main
```

Then updated the thesis root submodule pointer:
```bash
cd ~/guided_breast_biopsy
git add external/kuka_lbr_control
git commit -m "..."
git push
```

### Step 3 — Fork ros2_effort_controller (for Bug #4)

The effort controller source needed C++ patches. Forked via GitHub web UI to `Morook97/ros2_effort_controller`, then:

```bash
cd ~/guided_breast_biopsy/external/kuka_lbr_control/controllers
git remote rename origin upstream
git remote add origin https://github.com/Morook97/ros2_effort_controller.git
git fetch origin
git checkout -b thesis/configurable-shutdown-warmup
# apply C++ patches
git add effort_controller_base/include/effort_controller_base/effort_controller_base.h
git add effort_controller_base/src/effort_controller_base.cpp
git commit -m "feat(effort_controller_base): configurable shutdown threshold and first-tick warmup"
git push -u origin thesis/configurable-shutdown-warmup
```

Then updated `.gitmodules` in `kuka_lbr_control`:
```bash
cd ~/guided_breast_biopsy/external/kuka_lbr_control
git config -f .gitmodules submodule.controller.url https://github.com/Morook97/ros2_effort_controller.git
git config -f .gitmodules submodule.controller.branch thesis/configurable-shutdown-warmup
git submodule sync controllers
git config submodule.controller.branch thesis/configurable-shutdown-warmup
git add .gitmodules controllers
git commit -m "chore(submodule): point controllers to Morook97 fork on thesis branch"
git push origin main
```

Cascade to thesis root:
```bash
cd ~/guided_breast_biopsy
git add external/kuka_lbr_control
git commit -m "chore: update kuka_lbr_control fork with controllers submodule redirect"
git push
```

---

## Repositories Used

| Repository | Owner | Role | Branch used |
|---|---|---|---|
| `guided_breast_biopsy` | Morook97 | Thesis root | `feature/robot` |
| `kuka_lbr_control` | idra-lab (original) / Morook97 (fork) | Main control stack — launch files, YAML configs, build system | `main` |
| `lbr_fri_ros2_stack` | lbr-stack (original) / Morook97 (fork) | KUKA hardware description, xacro files, ros2_control interface | `thesis/gazebo-patch` |
| `ros2_effort_controller` | idra-lab (original) / Morook97 (fork) | Effort controller base + Cartesian Impedance + Gravity Compensation | `thesis/configurable-shutdown-warmup` |
| `lbr_fri_idl` | lbr-stack | FRI IDL definitions (not modified, used as-is) | pinned commit |
| `fri` | lbr-stack | KUKA FRI library (not modified, used as-is) | `fri-1.17` |

### Original upstream repositories

- `https://github.com/idra-lab/kuka_lbr_control`
- `https://github.com/lbr-stack/lbr_fri_ros2_stack`
- `https://github.com/idra-lab/ros2_effort_controller`

### Fork repositories (Morook97)

- `https://github.com/Morook97/kuka_lbr_control`
- `https://github.com/Morook97/lbr_fri_ros2_stack`
- `https://github.com/Morook97/ros2_effort_controller`
- `https://github.com/Morook97/guided_breast_biopsy`

---

## Workflow to Test That Everything Works

This is the minimal sequence to verify the simulation runs correctly from scratch after cloning or on a new machine.

### Prerequisites

- Ubuntu 24.04 LTS
- ROS 2 Jazzy installed and sourced
- Gazebo Harmonic (`gz-sim8`) installed
- colcon build tool installed

### Step 1 — Clone and initialize submodules

```bash
git clone https://github.com/Morook97/guided_breast_biopsy.git
cd guided_breast_biopsy
git checkout feature/robot
git submodule update --init --recursive
```

### Step 2 — Create and populate the colcon workspace

```bash
mkdir -p ~/kuka_ws/src
ln -s ~/guided_breast_biopsy/external/kuka_lbr_control/controllers ~/kuka_ws/src/controllers
ln -s ~/guided_breast_biopsy/external/kuka_lbr_control/kuka_control ~/kuka_ws/src/kuka_control
ln -s ~/guided_breast_biopsy/external/kuka_lbr_control/lbr-stack ~/kuka_ws/src/lbr-stack
```

### Step 3 — Build

```bash
cd ~/kuka_ws
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install
```

Expected: 23 packages built, zero errors. Deprecation warnings from the professor's code are normal and can be ignored.

### Step 4 — Launch Gazebo

```bash
source install/setup.bash
ros2 launch kuka_control gazebo.launch.py ctrl:=cartesian_impedance_controller
```

Expected log lines (in order):
```
Effort shutdown threshold: 500.00 Nm
Finished Base on_activate
Successfully switched controllers!
```

If Gazebo opens and the arm is visible holding its pose — activation test passed.

### Step 5 — Verify gravity hold

In a second terminal:
```bash
source ~/kuka_ws/install/setup.bash
ros2 topic echo /lbr/joint_states --once
```

Expected: A4 position ≈ 1.5708 rad, A4 effort ≈ -16.8 Nm (gravity compensation active), all velocities near zero.

### Step 6 — Verify Cartesian tracking

In a third terminal:
```bash
source ~/kuka_ws/install/setup.bash
ros2 topic pub -r 10 /lbr/cartesian_impedance_controller/target_frame geometry_msgs/msg/PoseStamped "{
  header: {frame_id: 'lbr_link_0'},
  pose: {
    position: {x: 0.3, y: 0.0, z: 0.6},
    orientation: {x: 0.0, y: 0.707, z: 0.0, w: 0.707}
  }
}" --times 100
```

Expected: the arm visibly moves in Gazebo. In the joint states terminal, A2 should move from ~0.436 rad to ~1.66 rad and A4 from ~1.571 rad to ~2.094 rad. Gazebo stays open with no crash.
