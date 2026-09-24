You can test image acquisition, target detection, pose repeatability, and workspace calibration with a fixed camera and an Ensenso-compatible target, without a robot. No `tool0`, robot TF, or measured mounting offset is needed. The workspace will be defined by where you place the target. Robot-base calibration can be performed later using [CARTESIAN_CALIBRATION_ROS2.md](CARTESIAN_CALIBRATION_ROS2.md).

Use the printed `SingleCustom` Ensenso target discussed in that guide, mounted flat and printed at its original scale. Keep the complete custom pattern visible in both stereo images so this driver's automatic decoding can read it. Its built-in detection path does not detect ChArUco. See [Ensenso's target documentation](https://manual.ensenso.com/latest/guides/calibration/patterns.html).

The commands below assume a built ROS 2 driver, Jazzy, namespace `/ensenso`, and the workspace path on this machine. Source this environment in every terminal:

```bash
source /opt/ros/jazzy/setup.bash
source ros/ensenso_ws/install/setup.bash
ros2 pkg prefix ensenso_camera
```

If the last command reports that the package is missing, prepare and build the repository using [ROS2.md](ROS2.md). An `install/setup.bash` file alone does not mean the camera package has been built.

**Start the camera in terminal A.** Enter the serial without `!`; the command appends it to keep numeric serials as strings.

```bash
read -r -p 'Camera serial: ' ENSENSO_BENCH_SERIAL

ros2 launch ensenso_camera stereo_node.launch.py \
  "serial:=${ENSENSO_BENCH_SERIAL:?Enter the camera serial}!" \
  namespace:=ensenso \
  fixed:=True \
  camera_frame:=ensenso_optical_frame \
  link_frame:=workspace \
  target_frame:=workspace
```

Here `workspace` names the output reference frame. It requires no existing robot or external TF. Requests below use this configured target frame, which equals `link_frame`, so the driver does not need to look up an external transform. Before workspace calibration, reported poses reflect the camera's existing internal link; do not assume that they are in the optical frame. After calibration, the workspace is explicitly tied to the target's chosen pose.

**In terminal B, disable automatic link following in a parameter set for these tests.** Check `error.code: 0` and that the returned `FollowLink` value is `Disabled`.

```bash
ros2 action send_goal /ensenso/set_parameter \
  ensenso_camera_msgs/action/SetParameter \
  '{parameter_set: bench_test, parameters: [{key: FollowLink, string_value: Disabled}]}'
```

Use `parameter_set: bench_test` in subsequent requests. This keeps target detection from moving the workspace along with the target. For this new parameter set, leave projector control to the driver: it switches projection off for pattern/image acquisition and on when appropriate for 3D acquisition.

**Inspect images before calibrating.** In terminal C, start the image viewer:

```bash
ros2 run rqt_image_view rqt_image_view
```

Select `/ensenso/raw/left/image`. In terminal B, request a pair of images:

```bash
ros2 action send_goal /ensenso/request_data \
  ensenso_camera_msgs/action/RequestData \
  '{parameter_set: bench_test, request_raw_images: true, publish_results: true}'
```

Select `/ensenso/raw/right/image` and request again to check the second view. The viewer must subscribe before the request; repeat the request if you selected a topic after publishing. This is one acquisition per action, not a live stream. Check that the target is sharp, well lit, and completely visible in both images. These are raw images, without detection overlays. If the viewer is not installed, install `ros-jazzy-rqt-image-view` using your normal package-management workflow.

**Test detection and pose estimation.** Place the target on a stable surface, then run:

```bash
ros2 action send_goal /ensenso/locate_pattern \
  ensenso_camera_msgs/action/LocatePattern \
  '{parameter_set: bench_test, number_of_shots: 1}'
```

Look for action status `SUCCEEDED`, `error.code: 0`, `found_pattern: true`, and one entry in `pattern_poses`. The result includes the target position in meters, its orientation quaternion, and decoded pattern information. A successful action with `found_pattern: false` means no usable target was found. This action captures its own images; no preceding `request_data` action is necessary. It does not itself publish the image topics.

Repeat with the camera and target stationary to compare pose variation. Move or tilt the target between requests to explore detection range, then stop it before requesting again. Use `number_of_shots: 5` to average several stationary observations; keep the target still for the entire action.

**Try workspace calibration.** Keep the camera fixed and place the target where you want the new workspace origin and axes. Leave exactly one target visible and stationary. This goal defines the observed target pose as the identity in `workspace`:

```bash
ros2 action send_goal /ensenso/calibrate_workspace \
  ensenso_camera_msgs/action/CalibrateWorkspace \
  '{
    parameter_set: bench_test,
    number_of_shots: 10,
    defined_pattern_pose: {
      position: {x: 0.0, y: 0.0, z: 0.0},
      orientation: {x: 0.0, y: 0.0, z: 0.0, w: 1.0}
    },
    write_calibration_to_eeprom: false
  }'
```

Check `successful: true` and `error.code: 0`. This estimates the camera's extrinsic link to the target-defined workspace; it does not perform robot-base calibration or recalibrate the stereo intrinsics. The change applies immediately to the running SDK session, but the false EEPROM flag leaves the calibration stored on the camera unchanged.

**Reapply disabled link following after calibration, then verify the result without moving the target.** This matches the sequence used by the repository's [workspace-calibration test](../ensenso_camera_test/src/ensenso_camera_test/workspace_calibration.py).

```bash
ros2 action send_goal /ensenso/set_parameter \
  ensenso_camera_msgs/action/SetParameter \
  '{parameter_set: bench_test, parameters: [{key: FollowLink, string_value: Disabled}]}'

ros2 action send_goal /ensenso/locate_pattern \
  ensenso_camera_msgs/action/LocatePattern \
  '{parameter_set: bench_test, number_of_shots: 5}'
```

The target position should now be close to `(0, 0, 0)` in frame `workspace`, with rotation close to identity. Quaternions `(0, 0, 0, 1)` and `(0, 0, 0, -1)` represent the same orientation. This checks that the new reference frame has been applied, not absolute measurement accuracy.

Try these experiments while keeping the camera fixed:

| Experiment | Expected behavior |
|---|---|
| Repeat detection with the target stationary | Small pose changes from measurement noise. |
| Translate the target a measured distance along a workspace axis | The corresponding position component changes by approximately that distance; the workspace remains fixed. |
| Tilt the target | Its reported orientation changes. |
| Remove or obscure the target | Detection reports `found_pattern: false`, or an SDK error that explains the failure. |
| Replace the target and calibrate the workspace again | The new target placement becomes the new reference pose. |

A ruler-guided 50 mm motion along a workspace axis should produce about `0.050` m of change along that axis. Movement along a camera or table axis may affect several components if that axis is not aligned with the workspace. Printed scale, flatness, illumination, and the accuracy of the reference motion limit this check.

**Inspect the point cloud.** Start RViz in terminal C or another sourced terminal:

```bash
ros2 run rviz2 rviz2
```

Set Fixed Frame to `workspace`, add a `PointCloud2` display, and select `/ensenso/point_cloud`. Add a TF display if you want to see `ensenso_optical_frame`. Then request a cloud in terminal B:

```bash
ros2 action send_goal /ensenso/request_data \
  ensenso_camera_msgs/action/RequestData \
  '{parameter_set: bench_test, request_point_cloud: true, publish_results: true}'
```

The cloud is now expressed in the calibrated workspace. Valid points on the printed target surface should have approximately zero workspace Z. A table supporting a raised target need not have Z equal to zero. Very dark target dots can also produce missing depth points. Subscribe before requesting the cloud and repeat the action for each update.

**Finish the experiment by stopping and restarting the camera driver.** A fresh SDK session reopens the camera using its stored calibration. Reapply the test parameter set if you continue experimenting. Keep `write_calibration_to_eeprom: false` throughout these exercises; none of the commands above stores the temporary workspace calibration permanently.

If detection fails, check both images, target type, complete visibility of the custom pattern, blur, contrast, reflections, and excessive tilt. The driver requests projector-off capture for detection. The SDK documents `PatternNotFound` and `PatternNotDecodable` in [CollectPattern](https://manual.ensenso.com/latest/commands/collectpattern.html). If images look correct but the pattern cannot be decoded, verify the generated target and its printing scale.

The commands were checked against this checkout's interfaces and launch arguments, but have not been executed on your camera. Real hand-eye calibration will later require actual robot pose measurements.
