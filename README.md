# Ginkgo_Odrive

Full workspace for Ginkgo CAN + ODrive control debugging.

This directory is meant to be a practical bench environment for:

- direct motor bring-up with the Ginkgo USB-CAN adapter
- ODrive CANSimple testing without ROS
- ROS 2 bridge testing with the `ginkgo_odrive_bridge` package
- documenting the motor and ODrive configuration in one place

## Visual References

![BLDC motor overview](https://commons.wikimedia.org/wiki/Special:FilePath/EC-Motor.svg)

Source: Wikimedia Commons, public domain. Useful as a quick reminder of the BLDC + electronic commutation model that the ODrive is controlling.

![Bus topology overview](https://commons.wikimedia.org/wiki/Special:FilePath/BusNetwork.svg)

Source: Wikimedia Commons, public domain. This is a simple way to visualize the shared CAN bus concept used between the host, adapter, and ODrive nodes.

![Trapezoidal velocity profile](https://upload.wikimedia.org/wikipedia/commons/thumb/6/6a/Jerk_loi_mouvement_vitesse_trapezoidale.svg/500px-Jerk_loi_mouvement_vitesse_trapezoidale.svg.png)

Source: Wikimedia Commons, CC BY-SA 3.0. This helps explain why the current position setup uses trap trajectory instead of a raw position step.

## Workspace Layout

```text
Ginkgo_Odrive/
  README.md
  requirements.txt
  ginkgo_tools/
    ginkgo_motor_tester.py
    odrive_ginkgo.py
    read_encoder_once.py
    telemetry_monitor.py
    position_step_test.py
    odrive_config_GIM8108-48_24V_clean.txt
  ros2_ws/
    src/
      ginkgo_odrive_bridge/
        ginkgo_odrive_bridge/
        launch/
        config/
        Python_USB_CAN_Test_64bits/
```

## What Each Part Does

- `ginkgo_tools/`
  Standalone Python tools and the PyQt6 GUI for direct CAN debugging.
- `ginkgo_tools/odrive_ginkgo.py`
  Shared abstraction layer for ODrive CANSimple messages over the Ginkgo adapter.
- `ginkgo_tools/odrive_config_GIM8108-48_24V_clean.txt`
  Clean working notes for the GIM8108-48 motor and the current ODrive setup.
- `ros2_ws/src/ginkgo_odrive_bridge/`
  The ROS 2 bridge package that maps `/joint_states` to ODrive CAN commands.

## Quick Start

### 1. Create and activate a virtual environment

```bash
cd /home/gerardo/Projects/Robotics/Ginkgo_Odrive
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### 2. Run the standalone GUI

```bash
python3 ginkgo_tools/ginkgo_motor_tester.py
```

### 3. Run the standalone CLI tools

```bash
python3 ginkgo_tools/read_encoder_once.py --node-id 0x10
python3 ginkgo_tools/telemetry_monitor.py --node-id 0x10 --interval 0.5
python3 ginkgo_tools/position_step_test.py --node-id 0x10 0.10 -0.10 0.00
```

## ROS 2 Bridge Workflow

From this workspace root:

```bash
source /opt/ros/humble/setup.bash
cd ros2_ws
colcon build --packages-select ginkgo_odrive_bridge --symlink-install
source install/setup.bash
ros2 launch ginkgo_odrive_bridge joint_state_bridge.launch.py
```

The default bridge parameters live in:

- `ros2_ws/src/ginkgo_odrive_bridge/config/joint_state_bridge.yaml`

The default node IDs there are `0x10`, `0x11`, and `0x12`, which matches the multi-axis pattern already used in the project.

## ODrive Configuration Notes

The cleaned motor configuration file is:

- `ginkgo_tools/odrive_config_GIM8108-48_24V_clean.txt`

The current confirmed CAN configuration in that file is:

- axis node ID: `0x10`
- CAN baud rate: `500000` bps
- protocol style: standard 11-bit ODrive CANSimple

## Why This Setup Uses Trapezoidal Trajectory

The current configuration uses:

```python
odrv0.axis0.controller.config.control_mode = CONTROL_MODE_POSITION_CONTROL
odrv0.axis0.controller.config.input_mode   = INPUT_MODE_TRAP_TRAJ
```

That is a good bring-up choice because:

- it gives bounded acceleration and deceleration instead of an abrupt position jump
- it reduces mechanical shock on the arm, gearbox, couplers, and mounts
- it makes bench tests more predictable and easier to repeat
- it is easier to reason about when your host is sending position targets in turns

In this mode, the host sends position setpoints and the ODrive internally generates a motion profile using:

- `trap_traj.config.vel_limit`
- `trap_traj.config.accel_limit`
- `trap_traj.config.decel_limit`

That is why the current standalone tools and ROS bridge are position-oriented: they all eventually send `Set_Input_Pos` CAN frames.

## What Would Change to Use Torque Control

If you switch to torque control, both the ODrive configuration and the host abstraction change.

On the ODrive side, the important changes are:

```python
odrv0.axis0.controller.config.control_mode = CONTROL_MODE_TORQUE_CONTROL
odrv0.axis0.controller.config.input_mode   = INPUT_MODE_PASSTHROUGH
```

You would usually stop thinking in terms of target turns and start thinking in terms of:

- commanded torque in Nm
- current limits
- thermal limits
- bus voltage behavior during acceleration and regen

On the host side, the abstraction changes from:

- "send position target in turns"

to:

- "send demanded torque in Nm"

In practical CAN terms, that means moving from `Set_Input_Pos (0x0C)` to `Set_Input_Torque (0x0E)`.

If you wanted this workspace to support torque control cleanly, the next software changes would be:

1. add torque-command helpers to `ginkgo_tools/odrive_ginkgo.py`
2. add a torque test CLI script
3. add torque widgets to the GUI
4. make the ROS bridge configurable so it can choose position or torque commands

## Ginkgo Adapter vs SN65HVD230 Transceiver

The Ginkgo adapter and the SN65HVD230 solve different layers of the stack.

### With the Ginkgo USB-CAN adapter

The current stack is:

```text
GUI / CLI / ROS bridge
  -> odrive_ginkgo.py
  -> ginkgo_can + ControlCAN.py
  -> Ginkgo native driver
  -> Ginkgo USB-CAN adapter
  -> CAN bus
  -> ODrive
```

The adapter already contains the host-facing transport layer, so your Python code can open a device, send frames, and receive frames through the vendor library.

### With an SN65HVD230

The SN65HVD230 is only a CAN transceiver. It is not a complete USB adapter and not a CAN controller by itself.

That means the stack becomes something like:

```text
GUI / CLI / ROS bridge
  -> ODrive CAN message encode/decode
  -> backend for a CAN controller
  -> CAN controller hardware
  -> SN65HVD230 transceiver
  -> CAN bus
  -> ODrive
```

So the abstraction changes like this:

- the ODrive CAN message layer stays mostly the same
- the Ginkgo-specific transport layer disappears
- `ControlCAN.py` and `ginkgo_can.bus` are replaced by a new backend

Examples of replacement backends:

- Linux SocketCAN
- a microcontroller with a CAN peripheral and a serial command bridge
- a different USB-CAN interface with its own Python or C library

In other words, `encode_set_input_pos()`, CAN IDs, and telemetry decoding can stay almost identical, while the low-level send/receive implementation is swapped out.

## Native Driver Note

The copied vendor bundle currently includes:

- Windows libraries
- macOS libraries

It does not currently include the Linux `libGinkgo_Driver.so` binary in the copied source tree.

If you want to use this workspace on Linux with the Ginkgo adapter, add the required vendor files under:

```text
ros2_ws/src/ginkgo_odrive_bridge/Python_USB_CAN_Test_64bits/lib/linux/64bit/
```

and, if needed:

```text
ros2_ws/src/ginkgo_odrive_bridge/Python_USB_CAN_Test_64bits/lib/linux/32bit/
```

The standalone tools already emit a clearer error if that Linux native library is missing.

## Key Files to Open First

- `ginkgo_tools/ginkgo_motor_tester.py`
- `ginkgo_tools/odrive_ginkgo.py`
- `ginkgo_tools/odrive_config_GIM8108-48_24V_clean.txt`
- `ros2_ws/src/ginkgo_odrive_bridge/ginkgo_odrive_bridge/joint_state_bridge.py`
- `ros2_ws/src/ginkgo_odrive_bridge/config/joint_state_bridge.yaml`

## References

- ODrive CAN protocol:
  https://docs.odriverobotics.com/v/latest/manual/can-protocol.html
- ODrive overview and controller inputs:
  https://docs.odriverobotics.com/v/latest/manual/overview.html
- ODrive CAN guide:
  https://docs.odriverobotics.com/v/latest/guides/arduino-can-guide.html
- TI SN65HVD23x family overview:
  https://www.ti.com/product/de-de/SN65HVD231/part-details/SN65HVD231DR

## Image Credits

- BLDC motor image:
  https://commons.wikimedia.org/wiki/File:EC-Motor.svg
- Bus topology image:
  https://commons.wikimedia.org/wiki/File:BusNetwork.svg
- Trapezoidal velocity profile image:
  https://commons.wikimedia.org/wiki/File:Jerk_loi_mouvement_vitesse_trapezoidale.svg
