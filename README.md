# Orbita
This project is a ROS Noetic package designed to control a 3-axis robotic actuator (a spherical mechanism/head, named Orbita) synchronized with a 3D LiDAR (Ouster OS0) and an elevation mapping system (intended to be fead to a RL policy).

**[User manual](https://github.com/joschmaCYU/Orbita-Mitacs/blob/main/USER_MANUAL.md)**

 ### 1. Low-Level & Embedded Control
  - FullReplacement.ino: 
    - Inverse Kinematics: Computes motor disk angles from a 3D orientation target (Yaw/Pitch/Roll using quaternions).
    - Sensor Acquisition: Reads 3 magnetic encoders via SPI with multi-turn tracking.
    - Motor Control: Implements a dual-loop PID controller (position and speed) for 3 motors at ~1 kHz.
    - ROS Interface: Communicates via rosserial (receives targets on /target_orientation and publishes motor telemetry to /motor).

 ### 2. Data Processing & ROS Scripts
  - elevation_mapper.py: Python ROS node subscribing to /rl/elevation_grid (17×11 = 187 grid values) and recording data at 50 Hz to a CSV file (real_lidar_50hz.csv).
  - Motor.msg: Custom ROS message defining motor state (name, position, speed, duty_cycle).

### 3. Launch Files
  - orbita_ouster_mitacs.launch: Starts the complete pipeline (Ouster LiDAR driver + point cloud relay + elevation mapper + rosserial bridge).
  - mitacs_ouster.launch: Starts only the LiDAR and mapping pipeline without motor control.

 ### 4. Environment & Deployment
  - Dockerfile: Containerized environment (ROS Noetic Desktop Full, ouster-ros, ANYbotics elevation_mapping packages, ONNX Runtime, etc.).
  - cheatsheet.md: Quick reference cheat sheet for Docker commands, LiDAR network setup, Arduino message library generation, and visualization tools (RViz, PlotJuggler).
  - package.xml & CMakeLists.txt: Catkin build setup and package dependencies configuration.
