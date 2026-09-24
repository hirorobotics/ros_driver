This guide calibrates a **fixed Ensenso camera relative to the base of an XYZ-only Cartesian robot**, using the existing ROS 2 driver and a printed Ensenso-compatible target.

The procedure requires a measured XYZ offset from the robot's tool frame to the target's 3D origin. Translation-only motion cannot distinguish this unknown mounting offset from camera translation. The action option used below is `robot_geometry: 4`, meaning `DOF3_FIX_PATTERN_POSE`; the number 4 is an enum value, not the number of robot axes. See the [action definition](../ensenso_camera_msgs/action/CalibrateHandEye.action) and [Ensenso's Cartesian-robot guidance](https://manual.ensenso.com/latest/commands/calibratehandeye.html#cartesian-robot).

**Start by preparing the target and measuring its mounting offset.** Generate a `SingleCustom` target with Ensenso's NxCalTab application, print at original scale, check its actual grid spacing, and mount it flat and rigidly on the robot tool. Enter the appropriate spacing and substrate thickness when generating the target. With this driver's automatic decoding, keep the complete custom target visible in both stereo images. ChArUco is not supported by this acquisition path. See [Ensenso's pattern instructions](https://manual.ensenso.com/latest/guides/calibration/patterns.html).

Measure the vector from the origin of `tool0` to the SDK-defined 3D origin of the mounted target, expressed along the **tool0 axes**. For the `SingleCustom` target used here, this is the center of the printed pattern on its printed surface. The pattern's X/Y axes lie in that surface, with orientation indicated by its marked corner or printed arrows; Z is normal to the surface. The corner-origin axes drawn in detection overlays describe a separate 2D coordinate system. See the [target frame diagram](https://manual.ensenso.com/latest/guides/calibration/patterns.html#coordinate-systems) and the SDK's distinction between [pattern top and pattern bottom](https://manual.ensenso.com/latest/commands/calibrateworkspace/parameters/offset.html).

Record the signed X, Y, Z components in meters. Measure to the printed pattern origin, rather than a paper corner, mounting-plate edge, or backing surface. Include the fixture and backing thickness when deriving this position from mechanical measurements. A measured fixture or a verified CAD mounting offset can establish this reference. An arbitrary zero offset would make the resulting camera translation incorrect. If these measurements are not yet available, obtain them before solving the calibration. With `robot_geometry: 4`, these components are expressed in `tool0`; they do not need to be re-expressed along the target's own axes.

The commands below use `base_link` for the robot base, `tool0` for the moving tool, `ensenso_optical_frame` for the camera, and `/ensenso` for the camera namespace. Replace the robot frame names with the names published by your robot. The driver must already be built for ROS 2; see [ROS2.md](ROS2.md). The examples use the Jazzy installation and workspace path on this machine.

**Open terminal A and start the camera for sample collection.** Enter the serial without a trailing exclamation mark; the launch command appends it so numeric serials remain strings.

```bash
source /opt/ros/jazzy/setup.bash
source /home/fafux/ros/ensenso_ws/install/setup.bash
read -r -p 'Camera serial: ' ENSENSO_CALIB_SERIAL

ros2 launch ensenso_camera stereo_node.launch.py \
  "serial:=${ENSENSO_CALIB_SERIAL:?Enter the camera serial}!" \
  namespace:=ensenso \
  fixed:=True \
  robot_frame:=base_link \
  wrist_frame:=tool0 \
  camera_frame:=ensenso_optical_frame \
  link_frame:=ensenso_optical_frame \
  target_frame:=ensenso_optical_frame
```

For collection, all three camera/link/target frame arguments deliberately match. In this checkout, the capture step estimates the target pose in `camera_frame`; setting `link_frame` to the robot base at this stage can require a camera-to-base TF that has not yet been calibrated. The operational launch later in this guide uses the base frame. Use this initial launch for calibration only: after solving, the SDK's internal link changes, but these frame parameters do not change with it.

**Open terminal B and check the robot pose source.** The robot driver must publish the current tool pose from actual robot state. These commands do not command robot motion.

```bash
source /opt/ros/jazzy/setup.bash
source /home/fafux/ros/ensenso_ws/install/setup.bash

ros2 run tf2_ros tf2_echo base_link tool0
```

Move the robot using its controller. Check that the reported translation follows the physical position, uses meters, and has the expected axis directions. For an XYZ-only machine, orientation should remain constant. Stop `tf2_echo` with Ctrl+C when finished.

**Reset the observation buffer once**, in terminal B:

```bash
ros2 action send_goal /ensenso/calibrate_hand_eye \
  ensenso_camera_msgs/action/CalibrateHandEye \
  '{command: 0}'
```

`command: 0` is `RESET`. It clears collected calibration observations; it does not erase the stored camera calibration.

**Collect at least eight successful observations**, distributed across X, Y, and Z in the working volume. Positions near the eight corners of a suitable box are one arrangement. Keep the target's orientation and mounting fixed, and choose positions where both camera images show the target clearly. The SDK recommends at least eight observations for a Cartesian robot; the driver itself only enforces a minimum of five.

At each position, stop the robot, allow it to settle, and run:

```bash
ros2 action send_goal /ensenso/calibrate_hand_eye \
  ensenso_camera_msgs/action/CalibrateHandEye \
  '{command: 1}'
```

`command: 1` is `CAPTURE_PATTERN`. Keep the robot still until the action finishes: this implementation looks up the latest robot TF after observing the target. Each accepted sample should have action status `SUCCEEDED`, `found_pattern: true`, `error.code: 0`, and an empty `error_message`. A successful action with `found_pattern: false` has not collected a sample. Multiple visible targets also prevent a valid capture.

Repeat the move, settle, capture sequence manually or through your own robot application. Do not reset the buffer between samples, and keep the camera node running throughout collection and solving.

**Enter the measured mounting offset and solve without storing to EEPROM.** Run this in terminal B, using decimal meters, for example `0.025` for 25 mm. The values must come from your mounting measurement.

```bash
read -r -p 'Target origin X in tool0, meters: ' ENSENSO_CALIB_X
read -r -p 'Target origin Y in tool0, meters: ' ENSENSO_CALIB_Y
read -r -p 'Target origin Z in tool0, meters: ' ENSENSO_CALIB_Z

ros2 action send_goal /ensenso/calibrate_hand_eye \
  ensenso_camera_msgs/action/CalibrateHandEye \
  "{
    command: 2,
    robot_geometry: 4,
    pattern_pose: {
      position: {
        x: ${ENSENSO_CALIB_X:?Enter the measured X offset},
        y: ${ENSENSO_CALIB_Y:?Enter the measured Y offset},
        z: ${ENSENSO_CALIB_Z:?Enter the measured Z offset}
      },
      orientation: {x: 0.0, y: 0.0, z: 0.0, w: 1.0}
    },
    write_calibration_to_eeprom: false
  }" --feedback
```

`command: 2` starts the calibration. With `robot_geometry: 4`, the three supplied pattern-position components are held fixed. The identity quaternion is a valid initial orientation guess; this mode does not fix the pattern orientation.

Check action status `SUCCEEDED`, `error.code: 0`, and an empty `error_message`. The result's `link` is the camera pose in the robot base frame, with translation in meters. `pattern_pose` is the target pose in the tool frame. Inspect the camera position/orientation, iteration count, and residual. The residual is an optimization diagnostic, not a direct guarantee of robot positioning accuracy. A wrong measured mounting offset can produce a plausible fit with the wrong absolute camera position.

Even with `write_calibration_to_eeprom: false`, the result is applied to the active SDK session. To improve the data, collect additional observations and solve again with the same geometry and measured offset. Avoid the supplied `calibrate_handeye` convenience script for this procedure: it does not set this Cartesian constraint or your measured offset.

**After reviewing the result, store the calibration.** In the same terminal B, with the camera node and observation buffer still alive, repeat the solve with EEPROM storage enabled:

```bash
ros2 action send_goal /ensenso/calibrate_hand_eye \
  ensenso_camera_msgs/action/CalibrateHandEye \
  "{
    command: 2,
    robot_geometry: 4,
    pattern_pose: {
      position: {
        x: ${ENSENSO_CALIB_X:?Enter the measured X offset},
        y: ${ENSENSO_CALIB_Y:?Enter the measured Y offset},
        z: ${ENSENSO_CALIB_Z:?Enter the measured Z offset}
      },
      orientation: {x: 0.0, y: 0.0, z: 0.0, w: 1.0}
    },
    write_calibration_to_eeprom: true
  }" --feedback
```

This recomputes the calibration from the retained observations and writes the resulting extrinsic link to the camera's EEPROM. Check the result again before stopping the node.

**Restart with the operational frame configuration.** Stop the camera launch in terminal A with Ctrl+C, then run the following in that same terminal, where the serial variable remains set:

```bash
ros2 launch ensenso_camera stereo_node.launch.py \
  "serial:=${ENSENSO_CALIB_SERIAL:?Enter the camera serial}!" \
  namespace:=ensenso \
  fixed:=True \
  robot_frame:=base_link \
  wrist_frame:=tool0 \
  camera_frame:=ensenso_optical_frame \
  link_frame:=base_link \
  target_frame:=base_link
```

The driver now uses the stored calibration to publish `base_link -> ensenso_optical_frame` and to express point clouds in `base_link`. Let this driver publish that camera transform; a second publisher for the same optical frame would create conflicting TF data.

**Validate at additional robot positions that were not used to fit the calibration.** In terminal B, inspect the camera transform:

```bash
ros2 run tf2_ros tf2_echo base_link ensenso_optical_frame
```

Stop the echo with Ctrl+C. Disable automatic link following so that detecting the target does not track it by changing the camera's link, then request its pose:

```bash
ros2 action send_goal /ensenso/set_parameter \
  ensenso_camera_msgs/action/SetParameter \
  '{parameters: [{key: FollowLink, string_value: Disabled}]}'

ros2 action send_goal /ensenso/locate_pattern \
  ensenso_camera_msgs/action/LocatePattern \
  '{number_of_shots: 5, target_frame: base_link}'
```

Check both actions for errors. The pattern result must have `found_pattern: true`, one pattern pose, and frame `base_link`. At each stationary validation position, compare the detected target origin with:

```text
expected_origin_in_base = tool_position_in_base
                        + tool_rotation_in_base * measured_target_offset_in_tool
```

If the tool axes are aligned with the base axes, this reduces to adding the measured X/Y/Z offsets to the tool coordinates. Include the reported tool rotation when they are not aligned. Check the discrepancy against your application's required tolerance across the working volume; investigate measurement, target scale/flatness, and robot TF errors if it is too large. Additional samples cannot resolve an incorrect mounting offset.

Finally, subscribe to `/ensenso/point_cloud` in your application or RViz and request a cloud:

```bash
ros2 action send_goal /ensenso/request_data \
  ensenso_camera_msgs/action/RequestData \
  '{request_point_cloud: true, publish_results: true}'
```

Its frame should be `base_link`. The frame parameters must remain configured on subsequent launches; storing the calibration does not store ROS frame names. If link following is enabled by your startup camera settings, keep it disabled for this fixed-camera calibration.

These commands were checked against this checkout's action definitions and ROS 2 launch arguments. They have not been run against your camera or robot. The physical target, measured mounting offset, and real robot TF are required to complete the calibration.
