# crazyflie_consensus_single_drone

A ROS 2 workspace for orchestrating a **Crazyflie 2.1+ nano-drones** through a fully automated consensus-based circular trajectory mission. The drone takes off in formation, executes a synchronized circular orbit with **staggered positioning**, completes exactly one full revolution, and lands autonomously — all while logging complete telemetry to CSV.

---

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Hardware Requirements](#hardware-requirements)
- [Software Prerequisites](#software-prerequisites)
- [Installation](#installation)
- [Drone Configuration](#drone-configuration)
- [Running the System](#running-the-system)
- [Flight Sequence](#flight-sequence)
- [Data Logging](#data-logging)
- [Customizing Parameters](#customizing-parameters)
- [Project Structure](#project-structure)
- [Troubleshooting](#troubleshooting)

---

## Overview

This project implements a **centralized controller** using ROS 2 and the [Crazyswarm2](https://github.com/IMRCLab/crazyswarm2) framework. Three Crazyflie 2.1 drones (`cf1) are tracked via a motion capture system and commanded to fly in a synchronized circular pattern at 1 metre altitude with a 0.3 m orbit radius.

The system is composed of three ROS 2 nodes working in a publisher-subscriber pipeline:

```
Motion Capture System
        |
        v
[swarm_navigation]  -->  /swarm/state  -->  [central_swarm_control]
                                                      ^
[swarm_guidance]    -->  /swarm/targets  ------------|
```

---

## System Architecture

### Nodes

| Node | Package | Role |
|---|---|---|
| `swarm_telemetry_node` | `swarm_navigation` | Subscribes to individual drone poses from the motion capture system and aggregates them into a single `/swarm/state` topic |
| `swarm_guidance_node` | `swarm_guidance` | Computes real-time circular trajectory targets for all 3 drones with 120° phase offsets and publishes them to `/swarm/targets` |
| `swarm_trajectory_controller` | `central_swarm_control` | Fuses state and targets; commands each drone through `TAKEOFF → TRACK → LAND` phases and writes the flight log |

### ROS 2 Topics

| Topic | Message Type | Published By | Subscribed By |
|---|---|---|---|
| `/{drone_id}/pose` | `geometry_msgs/PoseStamped` | Crazyswarm2 / Motion capture | `swarm_navigation` |
| `/swarm/state` | `geometry_msgs/PoseArray` | `swarm_navigation` | `central_swarm_control` |
| `/swarm/targets` | `geometry_msgs/PoseArray` | `swarm_guidance` | `central_swarm_control` |
| `/{drone_id}/cmd_position` | `crazyflie_interfaces/Position` | `central_swarm_control` | Crazyflie firmware |

---

## Hardware Requirements

- **1× Bitcraze Crazyflie 2.1** nano-drones
- **1× Crazyradio** (2.4 GHz USB radio dongle)
- **Motion capture system** compatible with `librigidbodytracker` (e.g. Qualisys, Vicon, or OptiTrack)
- A Linux PC running Ubuntu 22.04 (the workspace was built and tested on this)
- Sufficient flight space: at least **2 m × 2 m × 2 m** clear area

> **Battery note:** The firmware is configured to warn at 3.8 V and cut off at 3.5 V. Always fly on a full charge.

---

## Software Prerequisites

### 1. ROS 2 Humble

Follow the [official ROS 2 Humble installation guide](https://docs.ros.org/en/humble/Installation.html).

```bash
# After installation, source ROS 2
source /opt/ros/humble/setup.bash
```

### 2. Crazyswarm2

This workspace depends on the Crazyswarm2 framework. Install it from source into your ROS 2 workspace or follow the [Crazyswarm2 documentation](https://imrclab.github.io/crazyswarm2/installation.html).

```bash
# Core dependencies
sudo apt install -y python3-rosdep python3-colcon-common-extensions

# Crazyswarm2-specific Python deps
pip3 install cflib rowan
```

### 3. Python Dependencies

```bash
pip3 install numpy
```

---

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/Gowtham0517/crazyflie_consensus_single_drone.git
cd crazyflie_consensus_single_drone
```

### 2. Install ROS 2 Dependencies

```bash
rosdep update
rosdep install --from-paths src --ignore-src -r -y
```

### 3. Build the Workspace

```bash
colcon build --symlink-install
```

### 4. Source the Workspace

```bash
source install/setup.bash
```

> **Tip:** Add this to your `~/.bashrc` so it sources automatically:
> ```bash
> echo "source ~/crazyflie_consensus_single_drone/install/setup.bash" >> ~/.bashrc
> ```

---

## Drone Configuration

Before flying, update `src/central_swarm_control/config/crazyflies.yaml` to match your physical drone radio addresses and initial floor positions.

```yaml
robots:
  cf1:
    enabled: true
    uri: radio://0/80/2M/E7E7E7E731   # <-- change to your cf1 address
    initial_position: [0.0, 0.0, 0.0]
    type: cf21
```

### Finding Your Drone's Radio Address

Use the [Bitcraze CFclient](https://github.com/bitcraze/crazyflie-clients-python) to scan for connected drones and identify their URIs. The format is:

```
radio://<radio_index>/<channel>/<datarate>/<address>
```

### Initial Placement

Place the three drones on the floor matching the `initial_position` values in the YAML:

| Drone | X (m) | Y (m) | Z (m) |
|---|---|---|---|
| cf1 | 0.0 | 0.0 | 0.0 |

The drone should be in the centre of arena facing the X - axis direction.

### Controller & Estimator

The YAML is pre-configured to use the **Mellinger controller** (`controller: 2`) and the **Kalman filter estimator** (`estimator: 2`) — both recommended for trajectory tracking. Do not change these unless you have specific reasons.

---

## Running the System

Open **four separate terminal windows** and source the workspace in each:

```bash
source install/setup.bash
```

### Terminal 1 — Crazyflie Server (Hardware Backend)

This launches the Crazyswarm2 server that connects to the drones via Crazyradio and handles the motion capture interface.

```bash
ros2 launch central_swarm_control physical_swarm.launch.py
```

Wait until you see all three drones connect and the motion capture positions appear in the console output before proceeding.

### Terminal 2 — Swarm Navigation (Telemetry Aggregator)

```bash
ros2 run swarm_navigation navigation_node
```

Expected output:
```
[swarm_telemetry_node]: 👀 SWARM Telemetry: Tracking cf1...
```

### Terminal 3 — Swarm Guidance (Trajectory Generator)

```bash
ros2 run swarm_guidance guidance_node
```

Expected output:
```
[swarm_guidance_node]: 🎯 SWARM Guidance: Generating paths...
```

### Terminal 4 — Central Swarm Controller (Flight Execution)

```bash
ros2 run central_swarm_control control_node
```

Expected output sequence:
```
[swarm_trajectory_controller]: 💾 Logging Swarm to swarm_flight_log_YYYYMMDD_HHMMSS.csv | Waiting for telemetry...
[swarm_trajectory_controller]: ✅ Swarm floor frames locked!
[swarm_trajectory_controller]: 🚀 Swarm Launching: 3D Spiral Takeoff active...
[swarm_trajectory_controller]: 🔄 Orbit intercept confirmed. Orbiting exactly 1 full circle...
[swarm_trajectory_controller]: 🛬 Circle complete! Initiating automated structural vertical landing...
[swarm_trajectory_controller]: 🛑 Swarm landed safely. Terminating flight script.
```

> **Emergency stop:** Press `Ctrl+C` in Terminal 4 at any time. This triggers the `emergency_land()` function which immediately sends zero-altitude commands to all drones.

---

## Flight Sequence

The controller runs a single-phase autonomous flight:

### Phase 1: TAKEOFF (5 seconds)

Each drone linearly interpolates from its floor position to its corresponding orbit entry point at `z = 1.0 m`. The X and Y positions ramp smoothly toward the first circular target while altitude climbs from 0 to 1 m.

### Phase 2: TRACK (12.57 seconds — exactly 1 full orbit)

All three drones hold their positions on the circular orbit. The trajectory is:

- **Radius:** 0.3 m
- **Angular velocity (ω):** 0.5 rad/s  
- **Altitude:** 1.0 m (constant)
- **Phase offsets:** cf1 = 0°.

The orbit period is `T = 2π / ω ≈ 12.57 s`. The controller tracks for exactly one period before initiating landing.

### Phase 3: LAND (4 seconds)

The X and Y positions are frozen at the last recorded positions. Altitude ramps down from 1.0 m to 0.0 m over 4 seconds. The script exits cleanly once landing is complete.

---

## Data Logging

Every flight automatically writes a timestamped CSV log file to the directory from which `control_node` is run:

```
swarm_flight_log_YYYYMMDD_HHMMSS.csv
```

### Log Format

| Column | Description |
|---|---|
| `timestamp_sec` | Elapsed time since flight start (seconds) |
| `flight_state` | Current phase: `TAKEOFF`, `TRACK`, or `LAND` |
| `cf1_act_x/y/z` | cf1 actual position from motion capture (m) |
| `cf1_targ_x/y/z` | cf1 commanded target position (m) |

The log is written at **50 Hz** (every 20 ms control loop cycle). A typical 1-orbit flight (~22 seconds total) produces approximately 1,100 rows.

---

## Customizing Parameters

All key trajectory parameters are in `src/swarm_guidance/swarm_guidance/guidance_node.py`:

```python
self.radius = 0.3      # Orbit radius in metres
self.omega  = 0.5      # Angular velocity in rad/s (period = 2π/omega ≈ 12.57 s)
self.z_height = 1.0    # Cruise altitude in metres
```

And in `src/central_swarm_control/central_swarm_control/control_node.py`:

```python
# Phase durations
TAKEOFF_DURATION = 5.0    # seconds (line: if t > 5.0)
TRACK_DURATION   = 12.57  # seconds, one full circle (line: if t_track >= 12.57)
LAND_DURATION    = 4.0    # seconds (line: if t_land > 4.0)
```

To add more drones, extend the `self.drones` list in both `control_node.py` and `navigation_node.py`, and add corresponding entries to `crazyflies.yaml`.

---

## Project Structure

```
crazyflie_consensus_single_drone/
├── src/
│   ├── central_swarm_control/          # Main flight controller package
│   │   ├── central_swarm_control/
│   │   │   └── control_node.py         # TAKEOFF / TRACK / LAND state machine
│   │   ├── config/
│   │   │   └── crazyflies.yaml         # Drone URIs, positions, firmware params
│   │   ├── launch/
│   │   │   └── physical_swarm.launch.py  # Launch file for Crazyflie server
│   │   ├── package.xml
│   │   └── setup.py
│   │
│   ├── swarm_guidance/                 # Circular trajectory generator
│   │   ├── swarm_guidance/
│   │   │   └── guidance_node.py        # Publishes /swarm/targets at 50 Hz
│   │   ├── package.xml
│   │   └── setup.py
│   │
│   ├── swarm_navigation/               # Telemetry aggregator
│   │   ├── swarm_navigation/
│   │   │   └── navigation_node.py      # Aggregates drone poses → /swarm/state
│   │   ├── package.xml
│   │   └── setup.py
│   │
│   └── crazyswarm2/                    # Crazyswarm2 framework (dependency)
│
├── build/                              # Colcon build artifacts
├── install/                            # Colcon install artifacts
├── log/                                # Colcon build logs
├── swarm_flight_log_*.csv              # Auto-generated flight telemetry logs
└── README.md
```

---

## Troubleshooting

**Drone not connecting in Terminal 1**
- Check that the Crazyradio dongle is plugged in: `lsusb | grep -i bitcraze`
- Verify the drone URIs in `crazyflies.yaml` match your actual drone addresses using CFclient
- Ensure you have USB permissions: `sudo usermod -aG plugdev $USER` (then log out and back in)

**`swarm_telemetry_node` not receiving poses**
- Confirm the motion capture system is publishing poses on `/{cf_id}/pose` topics
- Check topic names: `ros2 topic list | grep pose`
- Verify the motion capture rigid body names match `cf1`, `cf2`, `cf3`

**Control node stays in `WAITING` state**
- It needs both `/swarm/state` AND `/swarm/targets` to start. Confirm both the navigation and guidance nodes are running: `ros2 topic echo /swarm/state` and `ros2 topic echo /swarm/targets`

**Drones drift during orbit**
- Verify the Kalman filter is properly initialized (let drones sit still for a few seconds before flight)
- Check `extPosStdDev: 1e-3` in `crazyflies.yaml` — adjust based on your motion capture accuracy
- Ensure the Mellinger controller is active: `stabilizer.controller: 2`

**Build fails with missing `crazyflie_interfaces`**
- The Crazyswarm2 packages must be built first. Ensure Crazyswarm2 is cloned into `src/crazyswarm2` and included in the workspace build.

---

## Maintainer

**Gowtham** — [venkatagowtham517@gmail.com](mailto:venkatagowtham517@gmail.com)

---

## License

Apache-2.0
