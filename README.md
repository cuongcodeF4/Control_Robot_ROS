# Control Robot ROS

> Control a differential-drive robot using **ROS 1 Noetic** by tilting an **MPU-6050 IMU sensor** connected to a **Raspberry Pi 4**. Supports Gazebo simulation on Ubuntu before deploying to real hardware.

---

## Table of Contents

- [System Overview](#system-overview)
- [Hardware Requirements](#hardware-requirements)
- [MPU-6050 Wiring](#mpu-6050-wiring)
- [Software Prerequisites](#software-prerequisites)
- [Network Configuration](#network-configuration)
- [Ubuntu PC Setup](#ubuntu-pc-setup)
- [Raspberry Pi 4 Setup](#raspberry-pi-4-setup)
- [Running — Simulation (Gazebo)](#running--simulation-gazebo)
- [Running — Real Hardware](#running--real-hardware)
- [Control Mapping](#control-mapping)
- [Project Structure](#project-structure)
- [Pictures](#pictures)

---

## System Overview

```
┌─────────────────────────────┐          LAN / Wi-Fi          ┌────────────────────────────────┐
│        Ubuntu PC            │ ◄────────────────────────────► │       Raspberry Pi 4           │
│                             │   ROS Topics over TCP/IP       │                                │
│  • ROS Master (roscore)     │                                │  • ROS Node (hardware)         │
│  • Gazebo Simulation        │                                │  • Reads MPU-6050 via I2C      │
│  • Visualization (RViz)     │                                │  • Publishes /cmd_vel          │
│  • Joystick teleop          │                                │  • Drives motors               │
└─────────────────────────────┘                                └─────────────┬──────────────────┘
                                                                             │ I2C (SDA/SCL)
                                                                     ┌───────▼────────┐
                                                                     │   MPU-6050     │
                                                                     │  Accelerometer │
                                                                     │  + Gyroscope   │
                                                                     └────────────────┘
```

- **Ubuntu PC** acts as the **ROS master**. It runs Gazebo simulation and can also send joystick commands.
- **Raspberry Pi 4** connects to the same local network, reads tilt data from the MPU-6050, and controls the robot motors.

---

## Hardware Requirements

| Component | Details |
|---|---|
| PC | Ubuntu 20.04 LTS, ROS Noetic |
| Raspberry Pi 4 | 2 GB RAM or higher, running Ubuntu Server 20.04 |
| MPU-6050 | 6-axis IMU (accelerometer + gyroscope), I2C interface |
| Robot chassis | Differential drive (2-wheel or 4-wheel) |
| Motor driver | e.g., L298N or L9110S |
| DC motors | Compatible with your chassis |
| Power supply | Battery pack for Raspberry Pi and motors |
| Network | Both devices on the same Wi-Fi or wired LAN |

---

## MPU-6050 Wiring

Connect the MPU-6050 to the **Raspberry Pi 4** GPIO header:

| MPU-6050 Pin | Raspberry Pi 4 Pin | GPIO |
|---|---|---|
| VCC | Pin 1 | 3.3V |
| GND | Pin 6 | GND |
| SDA | Pin 3 | GPIO 2 (SDA1) |
| SCL | Pin 5 | GPIO 3 (SCL1) |
| AD0 | GND | Set I2C address to 0x68 |
| INT | (optional) | — |

> **Note:** Do **not** connect VCC to 5V — the MPU-6050 logic is 3.3V.

Verify the sensor is detected after enabling I2C:

```bash
sudo i2cdetect -y 1
# Should show 0x68 in the output grid
```

---

## Software Prerequisites

### Ubuntu PC

1. **Install ROS Noetic:**

```bash
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu focal main" > /etc/apt/sources.list.d/ros-latest.list'
sudo apt install curl
curl -s https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | sudo apt-key add -
sudo apt update
sudo apt install ros-noetic-desktop-full
```

2. **Source ROS and add to `.bashrc`:**

```bash
echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

3. **Install catkin tools and build dependencies:**

```bash
sudo apt install python3-catkin-tools python3-rosdep python3-rosinstall
sudo rosdep init
rosdep update
```

4. **Install Gazebo (included with `desktop-full`) and verify:**

```bash
gazebo --version
```

5. **Install joystick support (optional):**

```bash
sudo apt install ros-noetic-joy ros-noetic-teleop-twist-joy
```

---

### Raspberry Pi 4

1. **Flash Ubuntu Server 20.04 ARM64** onto an SD card using [Raspberry Pi Imager](https://www.raspberrypi.com/software/).

2. **Install ROS Noetic (ARM):**

```bash
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu focal main" > /etc/apt/sources.list.d/ros-latest.list'
curl -s https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | sudo apt-key add -
sudo apt update
sudo apt install ros-noetic-ros-base
echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

3. **Enable I2C interface:**

```bash
sudo apt install i2c-tools
sudo usermod -aG i2c $USER
# Add to /boot/firmware/config.txt:
echo "dtparam=i2c_arm=on" | sudo tee -a /boot/firmware/config.txt
sudo reboot
```

4. **Install Python dependencies for MPU-6050:**

```bash
sudo apt install python3-pip
pip3 install smbus2 mpu6050-raspberrypi
```

5. **Install catkin tools:**

```bash
sudo apt install python3-catkin-tools python3-rosdep
sudo rosdep init
rosdep update
```

---

## Network Configuration

Both machines must be on the **same local network**. The Ubuntu PC runs `roscore` and acts as the ROS master.

### On Ubuntu PC

Find your IP address:

```bash
hostname -I
# Example: 192.168.1.100
```

Add to `~/.bashrc`:

```bash
export ROS_MASTER_URI=http://192.168.1.100:11311
export ROS_IP=192.168.1.100
```

### On Raspberry Pi 4

Find your IP address:

```bash
hostname -I
# Example: 192.168.1.101
```

Add to `~/.bashrc`:

```bash
export ROS_MASTER_URI=http://192.168.1.100:11311   # Ubuntu PC's IP
export ROS_IP=192.168.1.101                         # Raspberry Pi's own IP
```

Apply on both machines:

```bash
source ~/.bashrc
```

> **Tip:** Assign static IPs to both devices in your router settings to avoid having to update these values after each reboot.

---

## Ubuntu PC Setup

1. **Extract the Ubuntu ROS package:**

```bash
cd ~/
cp /path/to/Control_Robot_ROS/Package_ROS_on_Ubuntu/catkin_ws.zip ~/
unzip catkin_ws.zip
cd ~/catkin_ws
```

2. **Install package dependencies:**

```bash
rosdep install --from-paths src --ignore-src -r -y
```

3. **Build the workspace:**

```bash
catkin_make
# or using catkin tools:
catkin build
```

4. **Source the workspace:**

```bash
source ~/catkin_ws/devel/setup.bash
# Add permanently:
echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

---

## Raspberry Pi 4 Setup

1. **Transfer the Raspberry Pi package** to the Pi (via USB, SCP, or shared folder):

```bash
# From Ubuntu PC:
scp /path/to/Control_Robot_ROS/Package_control_raspberrypi/catkin_ws.zip pi@192.168.1.101:~/
```

2. **SSH into the Raspberry Pi:**

```bash
ssh ubuntu@192.168.1.101
```

3. **Extract and build:**

```bash
cd ~/
unzip catkin_ws.zip
cd ~/catkin_ws
rosdep install --from-paths src --ignore-src -r -y
catkin_make
```

4. **Source the workspace:**

```bash
source ~/catkin_ws/devel/setup.bash
echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

---

## Running — Simulation (Gazebo)

Run everything on the **Ubuntu PC** only. No Raspberry Pi or physical hardware needed.

**Terminal 1 — Start ROS master:**

```bash
roscore
```

**Terminal 2 — Launch Gazebo simulation:**

```bash
roslaunch <your_package_name> gazebo.launch
```

**Terminal 3 — (Optional) Control via joystick:**

```bash
roslaunch <your_package_name> teleop.launch
```

> Replace `<your_package_name>` with the actual ROS package name found inside `catkin_ws/src/`.

You should see the robot model appear in Gazebo. Use the joystick or publish to `/cmd_vel` to move it.

---

## Running — Real Hardware

Follow these steps **in order**. Do not start nodes on the Raspberry Pi before `roscore` is running on the Ubuntu PC.

### Step 1 — Start ROS Master (Ubuntu PC)

```bash
roscore
```

Leave this terminal open. Wait until you see:
```
started core service [/rosout]
```

### Step 2 — Launch Robot Nodes (Ubuntu PC)

Open a new terminal:

```bash
roslaunch <your_package_name> robot.launch
```

### Step 3 — Start Hardware Control Node (Raspberry Pi 4)

SSH into the Raspberry Pi:

```bash
ssh ubuntu@192.168.1.101
```

Then run the control node:

```bash
rosrun <your_raspberrypi_package_name> mpu6050_control.py
```

### Step 4 — Verify Topics

On the Ubuntu PC, confirm the MPU-6050 data and velocity commands are flowing:

```bash
rostopic list
rostopic echo /cmd_vel
```

### Step 5 — Stop Everything

Stop nodes in reverse order:
1. `Ctrl+C` on the Raspberry Pi node
2. `Ctrl+C` on the Ubuntu robot nodes
3. `Ctrl+C` on `roscore`

---

## Control Mapping

Tilt the MPU-6050 module to control robot movement:

| Tilt Direction | Robot Action |
|---|---|
| Tilt **Forward** (pitch +) | Move Forward |
| Tilt **Backward** (pitch −) | Move Backward |
| Tilt **Left** (roll −) | Turn Left |
| Tilt **Right** (roll +) | Turn Right |
| **Level / Flat** | Stop |

The accelerometer readings are mapped to linear and angular velocity values published on the `/cmd_vel` topic (`geometry_msgs/Twist`).

---

## Project Structure

```
Control_Robot_ROS/
├── Package_ROS_on_Ubuntu/
│   └── catkin_ws.zip          # ROS workspace for Ubuntu PC
│                              # Contains: Gazebo simulation, robot description (URDF),
│                              #           visualization (RViz), teleop nodes
├── Package_control_raspberrypi/
│   └── catkin_ws.zip          # ROS workspace for Raspberry Pi 4
│                              # Contains: MPU-6050 reader node,
│                              #           motor driver interface, /cmd_vel publisher
└── Picture/
    ├── Robot_in_Gazebo.jpg    # Robot simulated in Gazebo
    └── Tay_cam.jpg            # Physical MPU-6050 controller (hand-held)
```

---

## Pictures

### Robot in Gazebo Simulation

![Robot in Gazebo](Picture/Robot_in_Gazebo.jpg)

### MPU-6050 Hand Controller

![Hand Controller](Picture/Tay_cam.jpg)

---

## Author

**Thái Việt Cường** — [cuongcodeF4](https://github.com/cuongcodeF4)

---

## License

This project is open source. Feel free to use and modify it for educational purposes.
