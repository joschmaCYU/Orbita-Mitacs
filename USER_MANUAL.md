# User Manual: Orbita 3-Axis Actuator & Ouster LiDAR System

This guide covers setup, calibration, deployment, and operation of the **Orbita** 3-DOF spherical orientation platform synchronized with the **Ouster 3D LiDAR** and elevation mapping pipeline.

---

## 1. System Architecture
```mermaid
flowchart TD
    subgraph Host["Host Machine / Docker (ROS Noetic)"]
        subgraph ROSNodes["ROS Nodes"]
            OusterDriver["ouster_ros / sensor.launch"]
            RelayNode["topic_tools / relay (/lidar_relay)"]
            MapperNode["elevation_mapper.py"]
            SerialNode["rosserial_python / serial_node.py"]
            UserCmd["User / Controller (/target_orientation)"]
            Viz["RViz / PlotJuggler"]
        end
        CSVFile[("real_lidar_50hz.csv")]
    end

    subgraph Hardware["Physical Hardware"]
        LiDAR["Ouster 3D LiDAR (Ethernet UDP)"]
        MCU["STM32 Board (FullReplacement.ino)"]
        Encoders["3x SPI Magnetic Encoders"]
        Motors["3x DC/BLDC Motors (PWM/DIR)"]
        OrbitaMech["Orbita 3-DOF Platform"]
    end

    %% Network & Serial
    LiDAR -->|Point Cloud Data| OusterDriver
    OusterDriver -->|/ouster/points| RelayNode
    RelayNode -.-> MapperNode
    MapperNode -->|Recorded 50Hz Grids| CSVFile
    
    UserCmd -->|geometry_msgs/Vector3 (Yaw, Pitch, Roll)| SerialNode
    SerialNode <-->|UART / USB Serial (/dev/ttyACM0)| MCU

    MCU -->|SPI Bus| Encoders
    MCU -->|PWM / Direction| Motors
    Motors --> OrbitaMech
    MCU -->|/motor Telemetry (50Hz)| SerialNode
    SerialNode --> Viz
    OusterDriver --> Viz
```

## 2. Hardware Requirements & Wiring

### Hardware Components
* **Orbita Actuator Platform**: 3-DOF spherical mechanism.
* **Microcontroller**: STM32 board programmed with the Arduino framework.
* **3x Magnetic Encoders**: Connected via 3 distinct SPI buses.
* **3x Motor Drivers**: Connected via PWM and Direction control pins.
* **3D LiDAR**: Ouster OS0 / OS1 LiDAR connected via Ethernet.
* **Host PC**: Running Linux with Docker support and USB access.

### STM32 Pin Assignment Summary

| Function | Motor 1 | Motor 2 | Motor 3 | Standby / Shared |
| :--- | :--- | :--- | :--- | :--- |
| **PWM Output** | `PA8` | `PA11` | `PA9` | - |
| **Direction IN1** | `PB5` | `PB12` | `PC0` | - |
| **Direction IN2** | `PB4` | `PB11` | `PC1` | - |
| **SPI Encoders** | `SPI_1` (PA7, PA6, PA5) | `SPI_2` (PB15, PB14, PB13) | `SPI_3` (PC12, PC11, PC10) | - |
| **Standby Pins** | - | - | - | `STYB1: PC6`, `STYB2: PB8` |

---

## 3. Environment Setup (Docker)
The project is pre-configured inside a Docker container running **ROS Noetic** 

### 3.1 Build the Docker Image
From the repository root:
```bash
docker build -t ros_ouster_sync .
```

### 3.2 Allow X11 GUI Forwarding (Host Terminal)
Run on your host machine before starting the container:
```bash
xhost +local:root
export LIBGL_ALWAYS_SOFTWARE=1
```

### 3.3 Run the Docker Container
```bash
docker run -it \
  --net=host \
  --ipc=host \
  --env="DISPLAY=$DISPLAY" \
  --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
  -v /dev:/dev \
  --privileged \
  -v $(pwd):/root/catkin_ws/src/orbita \
  --name ros_ouster_sync \
  ros_ouster_sync bash
```

### 3.4 Starting / Entering an Existing Container
* **Start container:** `docker start -i ros_ouster_sync`
* **Open another shell:** `docker exec -it ros_ouster_sync bash`
* **Stop container:** `docker stop ros_ouster_sync`

> [!WARNING]
> Always connect USB **before** starting the container
---

## 4. Flashing STM32 Firmware & Generating ROS Libraries

### 4.1 Generate Custom ROS Messages for Arduino
If custom messages (`mitacs/Motor.msg`) are modified, regenerate the Arduino `ros_lib` headers:
```bash
# Inside the container / ROS environment
rosrun rosserial_arduino make_libraries.py /tmp/ros_lib
```
Copy `/tmp/ros_lib/mitacs` into your Arduino libraries folder (`~/Arduino/libraries/ros_lib/`).

### 4.2 Flashing `FullReplacement.ino`
1. Open [arduino/FullReplacement/FullReplacement.ino](arduino/FullReplacement/FullReplacement.ino) in Arduino IDE or PlatformIO.
2. Select your STM32 board and configure the upload method (ST-Link or USB DFU/CDC).
3. Ensure serial baud rate is set to `115200`.
4. Compile and upload to the board.

> [!IMPORTANT]
> **Zero Calibration at Boot:**
> When the STM32 powers up, it pauses for 1 second and takes the current encoder readings as the **zero/initial reference position**. Ensure the Orbita head is physically in its neutral center position before powering or resetting the STM32 board.

---

## 5. Step-by-Step Operation Guide

### Step 1: Verify Hardware Connections
1. Connect the STM32 USB cable to the host. Check detection:
   ```bash
   ls -l /dev/ttyACM* /dev/ttyUSB*
   ```
2. Connect the Ouster LiDAR Ethernet cable. Ping the sensor:
   ```bash
   # Using mDNS hostname:
   ping os-122220003768.local
   # Or using static IP:
   ping 169.254.185.245
   ```

### Step 2: Build the ROS Workspace (Inside Container)
```bash
cd ~/catkin_ws
catkin_make
source devel/setup.bash
```

### Step 3: Launch the Full Pipeline
Launch the LiDAR driver, topic relay, elevation logger, and serial communication in one command:

```bash
roslaunch orbita orbita_ouster_mitacs.launch sensor_hostname:=169.254.185.245 usb_port:=/dev/ttyACM0 viz:=true
```

Parameters:
* `sensor_hostname`: IP or hostname of the Ouster LiDAR (default `10.5.5.86`).
* `usb_port`: Serial port for the STM32 microcontroller (default `/dev/ttyACM0`).
* `baudrate`: Serial communication speed (default `115200`).
* `viz`: Open RViz automatically (`true` / `false`).

> [!TIP]
> To launch only the LiDAR and elevation logger without the motor controller, use:
> ```bash
> roslaunch orbita mitacs_ouster.launch sensor_hostname:=169.254.185.245 viz:=true
> ```

---

## 6. Controlling the Platform & Monitoring Data

### 6.1 Sending Orientation Targets
Command the 3-DOF platform by publishing a `geometry_msgs/Vector3` message to `/target_orientation`:

* `x`: Yaw angle in degrees
* `y`: Pitch angle in degrees
* `z`: Roll angle in degrees

```bash
# Example: 20 degrees Yaw, 0 Pitch, 0 Roll
rostopic pub -1 /target_orientation geometry_msgs/Vector3 "{x: 20.0, y: 0.0, z: 0.0}"

# Example: Return to home / zero
rostopic pub -1 /target_orientation geometry_msgs/Vector3 "{x: 0.0, y: 0.0, z: 0.0}"
```

### 6.2 Monitoring Motor Telemetry
The STM32 publishes telemetry for all 3 motors at 50 Hz on the `/motor` topic:
```bash
rostopic echo /motor
```
To visualize positions, speeds, and duty cycles in real time:
```bash
rosrun plotjuggler plotjuggler
```

### 6.3 Recording Elevation Grids
The node `scripts/elevation_mapper.py` listens to `/rl/elevation_grid`.
* Once data starts arriving, it records **500 frames** (10 seconds @ 50 Hz).
* Data is saved as `real_lidar_50hz.csv` in the execution directory.
* Each row contains 187 grid cell elevation values ($17 \times 11$).

To copy the recorded CSV out of the Docker container to your host home directory:
```bash
docker cp ros_ouster_sync:/root/catkin_ws/real_lidar_50hz.csv ~/
```

---

## 7. ROS Topics & Messages Reference

### Topic Interface

| Topic Name | Message Type | Direction | Description |
| :--- | :--- | :--- | :--- |
| `/target_orientation` | `geometry_msgs/Vector3` | **Subscribed by MCU** | Target orientation (degrees): `x=Yaw`, `y=Pitch`, `z=Roll`. |
| `/motor` | `orbita/Motor` (or `mitacs/Motor`) | **Published by MCU** | Motor state telemetry at 50 Hz (`name`, `position`, `speed`, `duty_cycle`). |
| `/ouster/points` | `sensor_msgs/PointCloud2` | **Published by LiDAR** | Raw 3D point cloud stream. |
| `/lidar_relay` | `sensor_msgs/PointCloud2` | **Relayed Topic** | Relayed point cloud topic via `topic_tools`. |
| `/rl/elevation_grid` | `std_msgs/Float32MultiArray` | **Subscribed by Logger** | 187-element 2.5D elevation grid for RL policy. |

### Message Definition: `Motor.msg`
```
string name         # Motor identifier ("MOT1", "MOT2", "MOT3")
float32 position    # Cumulative encoder step position
float32 speed       # Current speed (RPM)
int16 duty_cycle    # Applied PWM duty cycle (-255 to 255)
```

---

## 8. Troubleshooting & Common Issues

| Issue | Cause | Solution |
| :--- | :--- | :--- |
| **`Permission denied: '/dev/ttyACM0'`** | Linux user not in `dialout` group or missing device permissions. | Run `sudo chmod 666 /dev/ttyACM0` on the host, or add user to group: `sudo usermod -aG dialout $USER`. |
| **`Unable to sync with device; possible link problem...`** | `rosserial` baudrate mismatch or wrong USB port. | Verify baudrate is `115200` in both launch file and firmware. Check port with `ls /dev/ttyACM*`. |
| **LiDAR sensor not detected / ping timeout** | IP address subnet mismatch between host and sensor. | Set your network interface to the same subnet (e.g., `169.254.X.X` or `10.5.5.X`), or use link-local auto-config. |
| **Motors move in opposite directions or jitter** | Encoders not initialized in neutral position. | Power-cycle the STM32 board while holding the Orbita platform in its upright/center home position. |
| **RViz window fails to open inside Docker** | X11 display authorization missing. | Run `xhost +local:root` on host terminal before starting the container. |
