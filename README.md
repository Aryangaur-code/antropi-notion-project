# Antropi-notion-project
ROS 2 Humble/Jazzy-based TurtleBot3 Waffle simulation using Gazebo Sim and RViz2, exploring Nav2 autonomous navigation, SLAM/map visualization, LiDAR sensing, odometry, TF transforms and velocity control. Includes simulation setup, debugging and navigation testing.
# TurtleBot3 ROS 2 Simulation & Autonomous Navigation

A robotics simulation project using **ROS 2 Jazzy, Gazebo Sim, RViz2, and Nav2** to explore differential-drive robot control, sensor integration, odometry, coordinate transforms, map visualization, and autonomous navigation.

**Project status:** Simulation and ROS–Gazebo communication were observed working. Nav2 autonomous navigation was tested but encountered TF transform and component-container failures; successful goal completion remains to be verified.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Objectives](#objectives)
- [Technology Stack](#technology-stack)
- [System Architecture](#system-architecture)
- [Implementation and Work Completed](#implementation-and-work-completed)
- [Environment Setup](#environment-setup)
- [Running the Simulation](#running-the-simulation)
- [ROS 2 Topic and Node Inspection](#ros-2-topic-and-node-inspection)
- [Gazebo and RViz Visualization](#gazebo-and-rviz-visualization)
- [Velocity Control](#velocity-control)
- [Nav2 Navigation Workflow](#nav2-navigation-workflow)
- [Debugging and Troubleshooting](#debugging-and-troubleshooting)
- [Challenges and Lessons Learned](#challenges-and-lessons-learned)
- [Current Limitations](#current-limitations)
- [Future Improvements](#future-improvements)
- [References](#references)

## Project Overview

This project investigates the setup and debugging of a simulated mobile robot using ROS 2 and the Gazebo simulation environment.

The robot used is the **TurtleBot3 Waffle**, a differential-drive mobile robot model. Gazebo Sim provides the simulated environment and robot, while ROS 2 handles communication between the simulation, robot state publisher, visualization tools, and navigation components.

RViz2 is used to visualize the map, laser scan data, coordinate frames, robot-related displays, and navigation interface. Nav2 is used to explore path planning, controller execution, localization, and goal-based navigation.

The work focused not only on launching the simulation but also on inspecting ROS 2 topics, examining the ROS–Gazebo bridge, monitoring robot motion, and diagnosing failures in the navigation stack.

## Objectives

1. Launch the TurtleBot3 Waffle simulation in Gazebo Sim.
2. Visualize the robot and environment using RViz2.
3. Inspect ROS 2 topics, nodes, services, and message types.
4. Examine communication between Gazebo and ROS 2 through `ros_gz_bridge`.
5. Verify the availability of odometry, joint-state, laser scan, and velocity topics.
6. Explore Nav2 components, including the planner, controller, behavior server, and BT Navigator.
7. Investigate navigation failures involving TF transforms and lifecycle-managed nodes.
8. Develop a reproducible workflow for testing and debugging a simulated mobile robot.

## Technology Stack

| Technology | Purpose |
|---|---|
| Ubuntu | Linux development environment |
| ROS 2 Jazzy | Robot middleware and communication |
| Gazebo Sim | Robot and environment simulation |
| TurtleBot3 Waffle | Simulated differential-drive robot |
| RViz2 | Visualization and navigation interface |
| Nav2 | Navigation, planning, control, and localization |
| `ros_gz_bridge` | Communication between ROS 2 and Gazebo Transport |
| `tf2` | Coordinate-frame transformations |
| `geometry_msgs` | Velocity command messages |
| `sensor_msgs` | Sensor and joint-state messages |
| `nav_msgs` | Odometry and map-related messages |
| Linux terminal | Launching, inspecting, and debugging components |

## System Architecture

The project uses the following conceptual architecture:

```text
                 Gazebo Sim
                     |
          TurtleBot3 Waffle
                     |
          Gazebo Transport Topics
                     |
                ros_gz_bridge
                     |
                  ROS 2
                     |
       +-------------+-------------+
       |             |             |
    /scan          /odom       /joint_states
       |             |             |
       +-------------+-------------+
                     |
                   Nav2
       +-------------+-------------+
       |             |             |
    Planner       Controller    BT Navigator
       |             |             |
       +-------------+-------------+
                     |
                  /cmd_vel
                     |
                ros_gz_bridge
                     |
                Gazebo Sim
```

*Note: This diagram represents the intended data flow. The exact active connections depend on the bridge configuration and the runtime state of the navigation components.*

### Key communication topics

| Topic | Message type | Purpose |
|---|---|---|
| `/clock` | `rosgraph_msgs/msg/Clock` | Simulation time |
| `/cmd_vel` | `geometry_msgs/msg/Twist` | Robot velocity commands |
| `/odom` | `nav_msgs/msg/Odometry` | Estimated robot pose and velocity |
| `/joint_states` | `sensor_msgs/msg/JointState` | Wheel-joint position and velocity |
| `/scan` | `sensor_msgs/msg/LaserScan` | LiDAR measurements |
| `/imu` | `sensor_msgs/msg/Imu` | Inertial measurements |
| `/tf` | `tf2_msgs/msg/TFMessage` | Coordinate-frame transforms |

These topics were observed during the project at different stages. Their presence in the graph does not, by itself, guarantee that every navigation component is receiving valid data.

## Implementation and Work Completed

### 1. Gazebo simulation setup

The TurtleBot3 Waffle model was launched in Gazebo Sim. The environment displayed the robot, ground plane, and simulated obstacles.

The simulation was inspected to confirm that the robot model was present and that Gazebo continued running while ROS 2 commands were executed in separate terminals.

### 2. ROS–Gazebo bridge inspection

The ROS–Gazebo bridge was inspected using:

```bash
ros2 node info /ros_gz_bridge
```

The output showed that the bridge subscribed to `/cmd_vel` and published topics including `/clock`, `/imu`, `/joint_states`, `/odom`, `/scan`, and `/tf`.

The bridge configuration was inspected at:

```text
/opt/ros/jazzy/share/nav2_minimal_tb3_sim/configs/turtlebot3_waffle_bridge.yaml
```

The configuration contained mappings for clock, joint states, odometry, TF, IMU, laser scan, and velocity commands.

The relevant direction settings included:

- `GZ_TO_ROS`: data flows from Gazebo to ROS 2.
- `ROS_TO_GZ`: data flows from ROS 2 to Gazebo.

The bridge was also inspected for its active configuration:

```bash
ros2 param get /ros_gz_bridge config_file
```

This helped establish which configuration file the running bridge was using.

### 3. ROS 2 topic inspection

The ROS 2 topic graph was examined to identify available communication channels.

```bash
ros2 topic list
```

Filtered inspection was used to find important topics:

```bash
ros2 topic list | grep -E 'clock|odom|joint_states|scan|cmd_vel'
```

Topic information was inspected using:

```bash
ros2 topic info /cmd_vel --verbose
ros2 topic info /cmd_vel_smoothed --verbose
```

The `/cmd_vel` inspection showed publishers associated with `collision_monitor` and `docking_server`, and a subscriber associated with `ros_gz_bridge`.

The `/cmd_vel_smoothed` inspection showed a publisher associated with `velocity_smoother` and a subscriber associated with `collision_monitor`.

These observations helped trace the intended velocity-command pipeline.

### 4. Robot odometry and joint-state monitoring

The robot's odometry was examined with:

```bash
ros2 topic echo /odom --once
```

The output included the robot's estimated pose, orientation, and velocity.

An example observed odometry message contained:

- Reference frame: `odom`
- Child frame: `base_footprint`
- Forward linear velocity: approximately `0.1 m/s` during one observation
- Angular velocity: approximately zero during that observation

Wheel-joint feedback was inspected using:

```bash
ros2 topic echo /joint_states --once
```

The output included the wheel joints:

```text
wheel_left_joint
wheel_right_joint
```

Position and velocity fields were present. This provided evidence of joint-state data being published during the simulation.

### 5. Velocity-command testing

A velocity publisher was used to investigate whether ROS 2 commands could reach the simulated robot.

Example command:

```bash
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist \
"{linear: {x: 0.1}, angular: {z: 0.0}}"
```

This sends one forward velocity command with a linear velocity of `0.1 m/s` and zero commanded angular velocity.

The command was used as a communication test. Successful autonomous movement should be confirmed separately by observing the robot respond in Gazebo.

**Safety note:** Run velocity tests only when the robot has sufficient clear space. A single message does not guarantee that the robot will move or stop as expected, particularly when command arbitration or collision monitoring is involved.

### 6. RViz2 visualization

RViz2 was used to inspect the robot's navigation state and environment representation.

The configured visualization included:

- Grid
- RobotModel display
- TF frames
- LaserScan
- Bumper Hit
- Map
- AMCL Particle Swarm
- Global Planner
- Controller
- MarkerArray

The navigation interface displayed localization status, navigation status, feedback, estimated time, remaining distance, and recovery information.

During testing, the map and laser-related visualizations were visible. However, the RViz2 global status later showed an error associated with missing transforms, and the navigation interface reported aborted goals.

### 7. Nav2 navigation launch and node inspection

Nav2 was launched through the `nav2_bringup` simulation launch workflow.

The running launch process was inspected with:

```bash
ps -ef | grep -E 'nav2|bringup' | grep -v grep
```

The launch process was identified as running `tb3_simulation_launch.py` with `headless:=False`.

At an earlier stage, the navigation stack loaded components including:

- `planner_server`
- `controller_server`
- `behavior_server`
- `bt_navigator`

The controller logs showed that an MPPI controller and its critics were instantiated. The planner logs showed the `GridBased` planner being configured.

Later, the component container terminated unexpectedly, and the navigation nodes were no longer visible in the ROS 2 node graph.

This distinction is important: successfully loading a planner or controller does not establish that autonomous navigation completed successfully.

## Environment Setup

The project was tested with ROS 2 Jazzy and the installed TurtleBot3/Nav2 simulation packages.

### Prerequisites

- Ubuntu with ROS 2 Jazzy installed
- Gazebo Sim and the ROS–Gazebo integration packages
- TurtleBot3 simulation assets
- Nav2 and RViz2 packages
- A terminal with access to the ROS 2 environment

Source ROS 2 before running commands:

```bash
source /opt/ros/jazzy/setup.bash
```

Verify the ROS distribution:

```bash
echo $ROS_DISTRO
```

Expected output:

```text
jazzy
```

Check that the simulation launch file is available:

```bash
ros2 pkg prefix nav2_bringup
```

The exact package availability depends on the local installation.

## Running the Simulation

### Step 1: Start the TurtleBot3 simulation

Open a terminal and run:

```bash
source /opt/ros/jazzy/setup.bash
export TURTLEBOT3_MODEL=waffle
ros2 launch nav2_bringup tb3_simulation_launch.py headless:=False
```

This is the standard launch command used for the TurtleBot3/Nav2 simulation workflow. If your installation uses a modified launch file or a different package layout, use the command that matches your installed setup.

Keep this terminal open.

### Step 2: Inspect the running ROS 2 graph

Open a second terminal:

```bash
source /opt/ros/jazzy/setup.bash
ros2 node list
ros2 topic list
```

Check the bridge:

```bash
ros2 node info /ros_gz_bridge
```

### Step 3: Check sensor and odometry data

```bash
ros2 topic echo /odom --once
ros2 topic echo /joint_states --once
```

Check whether the laser scan is publishing:

```bash
ros2 topic hz /scan
```

Allow the command to run briefly, then stop it using `Ctrl + C`.

### Step 4: Open RViz2

If RViz2 was not started by the launch process, use a separate terminal:

```bash
source /opt/ros/jazzy/setup.bash
ros2 run rviz2 rviz2
```

Select the correct fixed frame and configure the necessary displays. If a Nav2 RViz configuration is available in your installation, use that configuration instead of building the display layout from scratch.

### Step 5: Check navigation availability

```bash
ros2 node list | grep -E 'planner_server|controller_server|bt_navigator|amcl'
```

If the relevant nodes are missing, navigation is not ready for goal testing.

## ROS 2 Topic and Node Inspection

The following commands were useful throughout the debugging process.

| Task | Command |
|---|---|
| List all topics | `ros2 topic list` |
| List nodes | `ros2 node list` |
| Inspect a node | `ros2 node info /ros_gz_bridge` |
| Inspect velocity topic | `ros2 topic info /cmd_vel --verbose` |
| Read odometry | `ros2 topic echo /odom --once` |
| Read joint states | `ros2 topic echo /joint_states --once` |
| Monitor LiDAR frequency | `ros2 topic hz /scan` |
| Inspect bridge configuration | `ros2 param get /ros_gz_bridge config_file` |
| Inspect launch process | `ps -ef \| grep -E 'nav2\|bringup' \| grep -v grep` |
| Inspect recent ROS logs | `ls -lt ~/.ros/log \| head` |

These commands help distinguish between a running simulation, a functioning ROS topic graph, and an operational navigation stack.

## Nav2 Navigation Workflow

The intended navigation workflow is:

1. Launch Gazebo and the TurtleBot3 robot.
2. Verify ROS–Gazebo communication.
3. Confirm that odometry, laser scan, and coordinate transforms are available.
4. Initialize localization using a valid map and an initial robot pose.
5. Verify that the global and local costmaps are receiving valid data.
6. Set a reachable goal in RViz2.
7. Observe the planner generating a path and the controller issuing velocity commands.
8. Confirm that the robot reaches the goal without an aborted action.

The workflow was investigated, but the final navigation steps were blocked by the transform and component-container failures described below.

## Debugging and Troubleshooting

Debugging was a major part of this project.

### Issue 1: Nav2 nodes disappeared

Initially, the navigation-related nodes were visible. Later, running:

```bash
ros2 node list | grep -E 'planner_server|controller_server|bt_navigator|amcl'
```

returned no matching nodes.

The process inspection showed that the top-level launch process was still running, but the navigation components were unavailable.

**Finding:** A running launch process does not guarantee that every child process or component remains healthy.

### Issue 2: Missing `map` transform

The logs contained messages similar to:

```text
Timed out waiting for transform from base_link to map
Invalid frame ID "map"
```

This indicates that the required transform between the robot and the map frame was unavailable to the affected navigation component at that time.

Potential areas to inspect include:

- Whether localization is active and has received an initial pose.
- Whether the map frame is being published correctly.
- Whether the TF tree contains the expected `map`, `odom`, and robot base frames.
- Whether simulation time and timestamps are consistent.
- Whether the relevant transforms are arriving before the navigation timeout.

Useful commands:

```bash
ros2 topic echo /tf --once
ros2 topic echo /tf_static --once
ros2 topic echo /odom --once
```

A complete TF diagnosis requires checking the actual frame tree, not just confirming that `/tf` exists.

### Issue 3: Global planner failed to create a path

The planner logs reported failures similar to:

```text
GridBased plugin failed to plan
Failed to create plan with tolerance of: 0.500000
```

The planner repeatedly failed to create a path from the reported starting pose to the requested goal.

Possible causes include an unreachable goal, occupied or unknown costmap cells, incorrect localization, or invalid map-to-robot transforms. The logs alone do not establish which cause was responsible.

### Issue 4: Component-container crash

The component-container log included an error reporting that the process exited with code `-6`.

Afterward, the navigation nodes were missing from the ROS 2 node list.

This suggests that the container hosting the navigation components terminated. The exit code alone does not identify the root cause; additional crash information and logs around the termination would be needed.

The following commands were used to inspect logs:

```bash
ls -lt ~/.ros/log | head
```

```bash
grep -inE 'error|exception|failed|fatal|abort' \
~/.ros/log/component_container_isolated_486_1792924709069.log | tail -20
```

The log filename above is specific to the original session. A new launch may generate a different filename.

### Issue 5: Repeated velocity publishing

A continuous velocity-publishing command produced repeated messages in the terminal. This is normal when using a repeating publisher.

For a one-time command test, use `--once`:

```bash
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist \
"{linear: {x: 0.1}, angular: {z: 0.0}}"
```

This makes the test easier to observe and avoids unnecessarily flooding the terminal with repeated messages.

## Challenges and Lessons Learned

### 1. Understanding the ROS 2 graph

Inspecting topics, nodes, services, and message types helped identify which parts of the system were communicating and which navigation components were missing.

### 2. Separating simulation health from navigation health

Gazebo and RViz2 could remain open even when the Nav2 component container had terminated. This demonstrated the importance of monitoring individual nodes rather than relying solely on the graphical interfaces.

### 3. Diagnosing coordinate transforms

The missing `map` transform prevented the navigation stack from consistently obtaining the required robot pose in the map frame. This highlighted the dependency of localization, costmaps, planning, and goal execution on a valid TF tree.

### 4. Tracing velocity commands

Inspecting `/cmd_vel`, `/cmd_vel_smoothed` and the bridge configuration helped clarify the intended command flow from ROS 2 to Gazebo.

### 5. Using logs to narrow down failures

Instead of repeatedly restarting the system without evidence, process inspection and log filtering were used to identify planner failures, missing navigation nodes and a component-container termination.

## Current Limitations

The following items remain to be verified or resolved:

- Stable execution of the Nav2 component container.
- Availability of all required navigation nodes throughout the session.
- A valid and continuous TF tree, especially the `map` to `odom` and robot base transforms.
- Successful localization initialization.
- Reliable global and local costmap updates.
- Successful path planning for a reachable goal.
- Completion of an autonomous navigation goal.
- A recorded demonstration of successful end-to-end navigation.

The map visualization and topic outputs provide evidence of parts of the simulation pipeline working, but they should not be interpreted as proof of completed autonomous navigation.

## Future Improvements

- Resolve the component-container crash and verify stable Nav2 lifecycle transitions.
- Validate the full TF tree and simulation timestamps.
- Confirm localization and costmap initialization before sending goals.
- Test navigation using reachable goals in clear regions of the map.
- Record successful goal execution in both Gazebo and RViz2.
- Document the exact ROS 2 package versions and launch configuration.
- Add reproducible test scripts and diagnostic output to the repository.
- Capture a short demo video showing the verified functionality.


## References

1. M. J. Athul Krishna et al., "ROS2-Powered Autonomous Navigation for TurtleBot3: Integrating Nav2 Stack in Gazebo, RViz and Real-World Environments," in *2024 IEEE International Conference on Signal Processing, Informatics, Communication and Energy Systems (SPICES)*, 2024. DOI: [10.1109/SPICES62143.2024.10779642](https://doi.org/10.1109/SPICES62143.2024.10779642).

2. A. K. Kashyap and K. Konathalapalli, "Autonomous navigation of ROS2 based Turtlebot3 in static and dynamic environments using intelligent approach," *International Journal of Information Technology*, 2025. DOI: [10.1007/s41870-025-02500-5](https://doi.org/10.1007/s41870-025-02500-5).

3. "Implementation of a ROS 2 Framework for Mobile Robot Localization and Navigation Using AMCL and Nav2," in *2025 IEEE 2nd International Conference on Computing, Applications and Systems (COMPAS)*, 2025. DOI: [10.1109/COMPAS67506.2025.11381826](https://doi.org/10.1109/COMPAS67506.2025.11381826).

4. P. Gyanani, M. Agarwal, R. Osari, and G. Chandore, "Autonomous Mobile Vehicle Using ROS2 and 2D-Lidar and SLAM Navigation," *Research Square*, preprint, 2024. [Read paper](https://assets-eu.researchsquare.com/files/rs-4323431/v1_covered_82bd3b7d-4f4c-4916-b061-bba5a6d57501.pdf).

5. "Implementing an Autonomous Navigation Stack in ROS2: Simulation on the Bumperbot Platform," *IFAC-PapersOnLine*, vol. 59, no. 30, pp. 1018–1023, 2025. DOI: [10.1016/j.ifacol.2025.12.373](https://doi.org/10.1016/j.ifacol.2025.12.373).

### Technical Documentation

- [ROS 2 Jazzy Documentation](https://docs.ros.org/en/jazzy/)
- [Nav2 Documentation](https://docs.nav2.org/)
- [Gazebo Documentation](https://gazebosim.org/docs/)
- [TurtleBot3 Documentation](https://emanual.robotis.com/docs/en/platform/turtlebot3/)

---

## Author

**Aryan Gaur**

This project explores ROS 2-based mobile robotics simulation, communication between simulation and robotics middleware, visualization, navigation-stack inspection and systematic debugging.

