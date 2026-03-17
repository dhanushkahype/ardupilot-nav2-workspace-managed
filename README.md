# Indoor Autonomous Drone System — Managed Workspace
## ArduPilot SITL + ROS 2 + Cartographer SLAM + Nav2 MPPI Navigation

A GPS-denied indoor autonomous navigation stack for a quadrotor drone. This repository does **not** store source code directly — it tracks dependency versions via a `.repos` manifest and contains only configuration, launch files, and documentation.

---

## Architecture

```
Gazebo SITL
    │
    ├── 2D Lidar (/scan)
    │       └── Cartographer SLAM ──→ /tf (map → odom → base_link)
    │
    ├── /tf relay ──→ /ap/tf ──→ ArduPilot EKF3 (VisualOdometry)
    │
    └── Nav2
            ├── NavFn Global Planner
            ├── MPPI Controller (Omni motion model)
            └── /cmd_vel → twist_stamper → /ap/cmd_vel → ArduPilot
```

---

## Hardware (Production Target)

| Component | Model |
|---|---|
| Frame | KD50 Cinelifter (X4) |
| Flight Controller | CUAV X7+ |
| Companion Computer | Jetson Orin Nano |
| Lidar | Livox Mid-360 |
| Networking | Dedicated WiFi AP, Cyclone DDS unicast |

---

## Workspace Setup

This repo uses a **declarative workspace** structure managed by `vcs`. The `src/` directory is not committed — all dependencies are pinned in `my_working_setup.repos`.

### First-Time Setup

```bash
# Clone this repo
git clone <your-repo-url> ardu_dev_ws
cd ardu_dev_ws

# Hydrate src/ from the manifest
mkdir -p src
vcs import src < my_working_setup.repos

# Install system dependencies
sudo apt update
rosdep update
rosdep install --from-paths src --ignore-src -r -y
```

### Saving Workspace State

After switching branches or adding new packages, update the manifest:

```bash
vcs export src > my_working_setup.repos
git add my_working_setup.repos
git commit -m "Update workspace dependency versions"
```

### Useful VCS Commands

| Goal | Command |
|---|---|
| Check status of all repos | `vcs status src` |
| Pull latest from all remotes | `vcs pull src` |
| Bulk commit across all repos | `vcs custom src --args commit -m "message"` |

---

## Build Instructions

### Step 1 — ArduPilot SITL Binary

```bash
cd ~/ardu_dev_ws/src/ardupilot
./waf distclean
./waf configure --board sitl --enable-DDS --enable-VISUALODOM
./waf copter
```

> ⚠️ Always pass **both** `--enable-DDS` and `--enable-VISUALODOM` together. `distclean` wipes all flags.

Verify the build:
```bash
grep "HAL_VISUALODOM_ENABLED" ~/ardu_dev_ws/src/ardupilot/build/sitl/compile_commands.json | head -1
# Should show HAL_VISUALODOM_ENABLED=1
```

### Step 2 — ROS 2 Packages

```bash
cd ~/ardu_dev_ws
colcon build --packages-up-to ardupilot_ros ardupilot_gz_bringup ardupilot_cartographer
source install/setup.bash
```

### Rebuild After Config Changes

```bash
colcon build --packages-select ardupilot_cartographer --allow-overriding ardupilot_cartographer
source install/setup.bash
```

> ⚠️ Never use `--symlink-install` with `ardupilot_cartographer` — incompatible with `setup.py`.

---

## Environment

Add to `~/.bashrc`:

```bash
export ROS_DOMAIN_ID=1
source ~/ardu_dev_ws/install/setup.bash
```

---

## Launch Sequence

Open each in a separate terminal. Source the workspace first: `source install/setup.bash`

| # | Component | Command |
|---|---|---|
| 1 | Gazebo | `ros2 launch ardupilot_gz_bringup iris_maze.launch.py lidar_dim:=2` |
| 2 | micro-ROS Agent | `ros2 run micro_ros_agent micro_ros_agent udp4 --port 2019 --domain-id 1` |
| 3 | Cartographer SLAM | `ros2 launch ardupilot_cartographer cartographer.launch.py` |
| 4 | TF Relay | `ros2 run topic_tools relay /tf /ap/tf` |
| 5 | Nav2 Navigation | `ros2 launch ardupilot_cartographer navigation.launch.py` |
| 6 | MAVProxy GCS | `mavproxy.py --master=:14550 --map --console` |

**Timing:**
- After Terminal 1 — wait ~60s for Gazebo to fully load
- After Terminal 3 — wait until map appears in RViz before launching Nav2
- After arming — wait ~15s on the ground for EKF to converge before takeoff

---

## Flight Sequence

```bash
# In MAVProxy
mode guided
arm throttle
takeoff 2
```

Once airborne, use the **Nav2 Goal** button in RViz to send navigation goals.

---

## ArduPilot Parameters

Set once via MAVProxy, then they persist in `mav.parm`:

```
param set AHRS_EKF_TYPE 3
param set EK2_ENABLE 0
param set EK3_ENABLE 1
param set EK3_SRC1_POSXY 6
param set EK3_SRC1_POSZ 1
param set EK3_SRC1_VELXY 6
param set EK3_SRC1_VELZ 0
param set EK3_SRC1_YAW 6
param set VISO_TYPE 1
param set GPS1_TYPE 0
param set ARMING_SKIPCHK 1
param set DDS_DOMAIN_ID 1
param set DDS_ENABLE 1
param set VISO_POS_M_NSE 0.5
param set VISO_VEL_M_NSE 0.3
```

---

## Key Configuration Files

| File | Path |
|---|---|
| Cartographer config | `src/ardupilot_ros/ardupilot_cartographer/config/cartographer.lua` |
| Nav2 config | `src/ardupilot_ros/ardupilot_cartographer/config/navigation.yaml` |
| Cartographer launch | `src/ardupilot_ros/ardupilot_cartographer/launch/cartographer.launch.py` |
| Navigation launch | `src/ardupilot_ros/ardupilot_cartographer/launch/navigation.launch.py` |

> ⚠️ Always edit files under `src/` and rebuild. Never edit `install/` directly — it gets overwritten on every build.

---

## Troubleshooting

**VisOdom not healthy**
```bash
ros2 topic hz /ap/tf                        # should be ~150Hz
ros2 topic echo /ap/tf --once | grep frame_id   # must contain odom → base_link
```

**Map scrambling on launch**
```bash
pkill -9 -f relay                           # kill all relay instances
ros2 topic info /tf -v | grep "Node name"   # should show only 2 publishers
ros2 run topic_tools relay /tf /ap/tf       # restart single relay
```

**Multiple Gazebo instances / port conflict**
```bash
pkill -9 -f gz
pkill -9 -f arducopter
lsof -i :9002                               # should return nothing before relaunch
```

**Nav2 controller_server fails to start**
- Check `navigation.yaml` — all float values must have decimal points (`0.3` not `0`)
- Ensure `inflation_radius` ≥ `robot_radius` in both costmaps

**EKF variance failsafe mid-flight**
- Increase noise tolerance: `param set VISO_POS_M_NSE 0.5`
- Reduce loop closure rate in `cartographer.lua`: `POSE_GRAPH.optimize_every_n_nodes = 90`

---

## Roadmap

- [x] ArduPilot SITL + ROS 2 DDS bridge
- [x] Cartographer 2D SLAM
- [x] Nav2 MPPI autonomous navigation
- [ ] FAST-LIO2 3D SLAM with Livox Mid-360
- [ ] Full 3D navigation (x, y, z)
- [ ] Hardware-in-the-loop testing (CUAV X7+ + Jetson Orin Nano)
- [ ] Real-world indoor flight
