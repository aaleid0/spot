# Spot ROS1 Package

ROS Noetic driver and control stack for the Boston Dynamics Spot robot.

---

## Table of Contents
- [Environment Setup](#environment-setup)
- [Connect to Spot](#connect-to-spot)
- [Manual Operation Workflow](#manual-operation-workflow)
- [Control Modes and Priority](#control-modes-and-priority)
- [Running Nodes](#running-nodes)
- [Intel RealSense D455](#intel-realsense-d455)
- [Keyboard Teleop Controls](#keyboard-teleop-controls)
- [Published / Subscribed Topics](#published--subscribed-topics)

---

## Environment Setup

Real-hardware operation is intended for Ubuntu 20.04 with ROS Noetic; the macOS Conda environment alone does not provide ROS or hardware operation.

Add to `~/.bashrc`:
```bash
export BOSDYN_CLIENT_USERNAME=user
export BOSDYN_CLIENT_PASSWORD="<YOUR_SPOT_PASSWORD>"
```

Never commit real Spot credentials to this repository.

Install dependencies:
```bash
mkdir -p spot_ws/src
cd spot_ws/src
git clone https://github.com/Shawn207/spot.git
cd spot/src/spot
pip install -r requirements.txt
pip install pynput          # required for keyboard teleop
cd ~/spot_ws
catkin_make
```

---

## Connect to Spot

| Method | IP Address |
|--------|-----------|
| Ethernet via Spot Dock | `192.168.50.3` |
| Ethernet via rear connector | `10.0.0.3` |
| Spot Wi-Fi | `192.168.80.3` |

The current code does not read `SPOT_HOSTNAME`. Before running, set the same robot IP in both:

- `src/spot/spot_spot.py` (`hostname`)
- `src/spot/estop_nogui.py` (`argv.append(...)`)

For example:

```bash
hostname = "<SPOT_IP_ADDRESS>"
argv.append("<SPOT_IP_ADDRESS>")
```

Keep Spot's physical E-Stop accessible. The software endpoint below is required by this code, but it is not a replacement for the physical safety control.

---

## Manual Operation Workflow

Use a clear test area and begin at low speed. Source ROS and the Catkin workspace in every terminal:

```bash
source /opt/ros/noetic/setup.bash
source ~/spot_ws/devel/setup.bash
```

Start the processes in this order:

```bash
# Terminal 1 — ROS master
roscore

# Terminal 2 — software E-Stop endpoint (start before the driver)
cd ~/spot_ws/src/spot/src/spot
python estop_nogui.py

# Terminal 3 — Spot driver, keyboard inputs only, no detection models
rosrun spot spot_base.py _detection_model:=none _control_mode:=teleop_only

# Terminal 4 — keyboard control
rosrun spot spot_teleop_keyboard.py
```

Press `X` to stand, drive with the keys below, and press `Z` to sit when finished. Keep the E-Stop endpoint running until Spot is safely stopped. `Space` only commands zero velocity; it is not a hardware E-Stop.

`spot_base.py` already publishes the Spot camera, pose, odometry, and TF. Do **not** also run `img_pub.py` or `publisher.py`; they are standalone/legacy alternatives that duplicate those publishers.

Mode parameters:

| Parameter | Safe manual value | Other values |
|-----------|-------------------|--------------|
| `~detection_model` | `none` | `yolov8`, `yolov7-sam2`, `v7+v8` |
| `~control_mode` | `teleop_only` | `all` (also enables RViz and planner goals) |

---

## Control Modes and Priority

Three control modes are supported. **Priority is fixed** — higher modes override lower ones while active:

| Priority | Mode | Trigger | Description |
|----------|------|---------|-------------|
| **1 (Highest)** | Keyboard Teleop | `spot_teleop_keyboard.py` running | Direct velocity commands; overrides all other inputs while the node is alive |
| **2** | RViz Click | `2D Nav Goal` in RViz → `/move_base_simple/goal` | Single-goal trajectory commands |
| **3 (Lowest)** | Autonomous Planner | `autonomous_robot` package → `/spot/waypoint` | Exploration and navigation goals |

**How the override works:** `spot_teleop_keyboard.py` publishes a 1 Hz heartbeat on `/spot/teleop_active`. While the heartbeat is fresh (< 2 s old), `spot_base.py` silently drops all incoming `/spot/waypoint` and `/move_base_simple/goal` messages. When the teleop node exits, autonomous control resumes automatically after ~2 s.

**Use teleop for:**
- Immediate zero-velocity overrides during autonomous runs
- Manual positioning for LiDAR-camera calibration or SlideSLAM initialization
- Operations where autonomous planning is not desired

---

## Running Nodes

### Core

```bash
# Terminal 1 — ROS master
roscore

# Terminal 2 — E-Stop (before the driver)
cd ~/spot_ws/src/spot/src/spot && python estop_nogui.py

# Terminal 3 — Main driver with the original defaults
rosrun spot spot_base.py
```

The default driver enables the configured detector plus RViz/planner goal inputs. For manual control without models, use the command in [Manual Operation Workflow](#manual-operation-workflow).

### Keyboard Teleop (highest priority)

```bash
# In a new terminal — teleop immediately overrides planner and RViz
rosrun spot spot_teleop_keyboard.py
```

Close the teleop terminal (or press ESC) to hand control back to the planner/RViz when `control_mode:=all`. Those inputs remain disabled in `teleop_only` mode.

---

## Intel RealSense D455

The D455 is independent of `spot_base.py`. Connect it by USB 3 to the Ubuntu/ROS computer and run Intel's existing ROS1 driver; no custom camera-control script is required. For ROS Noetic, use the `realsense2_camera` ROS1 package (`ros1-legacy` upstream branch).

```bash
# Optional installation when the binary package is available
sudo apt install ros-noetic-realsense2-camera

# Start synchronized RGB and depth, with depth registered to RGB
roslaunch realsense2_camera rs_camera.launch \
  align_depth:=true enable_sync:=true \
  color_width:=640 color_height:=480 color_fps:=30 \
  depth_width:=640 depth_height:=480 depth_fps:=30
```

Primary topics:

| Data | Topic |
|------|-------|
| RGB | `/camera/color/image_raw` |
| RGB calibration | `/camera/color/camera_info` |
| Depth aligned to RGB | `/camera/aligned_depth_to_color/image_raw` |

Keep `/spot_image` and the RealSense topics separate. `spot_base.py` does not need to subscribe to the D455. A viewer, recorder, or processing node subscribes to the topics it needs.

Three normal data workflows:

1. **View live:** use `rqt_image_view` or RViz.
2. **Record and process later:** preserve the raw messages and timestamps with `rosbag`.
3. **Process live:** create a separate ROS node that subscribes to RGB, aligned depth, camera info, and Spot TF/odometry.

```bash
rosbag record \
  /camera/color/image_raw \
  /camera/color/camera_info \
  /camera/aligned_depth_to_color/image_raw \
  /spot/odom /tf /tf_static
```

To place D455 measurements in Spot/world coordinates, measure or calibrate the rigid mount transform and publish `spot_base -> camera_link`. The RealSense driver supplies the camera's internal frames. `enable_sync` pairs D455 color/depth frames; it does not synchronize them with Spot. For slow manual experiments, associate Spot TF/odometry using each image's ROS timestamp. Accurate fast-motion work requires improved timestamp handling and pose interpolation.

---

## Keyboard Teleop Controls

The teleop node uses `termios`/stdin so it works over **SSH without X11**.

| Key | Action |
|-----|--------|
| `W` / `S` | Forward / Backward |
| `A` / `D` | Strafe Left / Right |
| `Q` / `E` | Rotate Left / Right |
| `R` / `F` | Increase / Decrease body height (limit ±10 cm) |
| `↑` / `↓` | Pitch forward / backward |
| `←` / `→` | Roll left / right |
| `,` / `.` | Body yaw left / right (limit ±25°) |
| `H` | Reset body pose to default |
| `Z` | Sit |
| `X` | Stand |
| `Space` | Command zero velocity (not an E-Stop) |
| `+` / `-` | Increase / Decrease speed |
| `ESC` or `Ctrl+C` | Exit teleop (returns control to planner/RViz) |

**Multi-key movement:** W+Q, A+E, etc. are combinable — each movement key tracks its own hold timestamp (250 ms expiry). Terminal key-repeat keeps it active while held; releasing a key causes it to expire within ~250 ms.

Body pose keys are incremental per press and are not affected by the hold timeout.

Default speed: **0.3 m/s linear**, **0.2 rad/s angular**. Range: 0.1 – 1.5 m/s.

---

## Published / Subscribed Topics

### spot_base.py

| Direction | Topic | Type | Description |
|-----------|-------|------|-------------|
| Sub | `/spot/waypoint` | `PoseStamped` | Goal from autonomous planner; `all` mode only (priority 3) |
| Sub | `/move_base_simple/goal` | `PoseStamped` | Goal from RViz; `all` mode only (priority 2) |
| Sub | `/spot/cmd_vel` | `Twist` | Velocity from teleop (priority 1) |
| Sub | `/spot/teleop_active` | `Bool` | Teleop heartbeat — suppresses lower-priority inputs |
| Sub | `/spot/sit` | `Bool` | Sit command from teleop |
| Sub | `/spot/stand` | `Bool` | Stand command from teleop |
| Sub | `/spot/body_pose` | `Twist` | Body pose from teleop (`linear.z`=height, `angular.xyz`=roll/pitch/yaw) |
| Pub | `/spot/odom` | `Odometry` | Robot odometry at 33 Hz |
| Pub | `/spot/pose` | `PoseStamped` | Robot pose at 33 Hz |
| Pub | `/spot_image` | `Image` | Camera feed at 10 Hz |
| Pub | `/spot/teleop_feedback` | `String` | Feedback messages to teleop node |

### spot_teleop_keyboard.py

| Direction | Topic | Type | Description |
|-----------|-------|------|-------------|
| Pub | `/spot/cmd_vel` | `Twist` | Velocity commands at 10 Hz |
| Pub | `/spot/body_pose` | `Twist` | Body pose on key press |
| Pub | `/spot/teleop_active` | `Bool` | Heartbeat at 1 Hz |
| Pub | `/spot/sit` | `Bool` | Sit on `Z` |
| Pub | `/spot/stand` | `Bool` | Stand on `X` |
| Sub | `/spot/odom` | `Odometry` | Position display |
| Sub | `/spot/teleop_feedback` | `String` | Feedback display |
