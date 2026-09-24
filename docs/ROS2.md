# ROS2
The ROS2 packages in this repo are ignored by default and do not yet contain the C++ and Python source files. Only files
that cannot be maintained in the ROS1 packages, because their structure is too different (e.g. CMakeLists) or there is
no way to provide a compatibility layer between ROS1 and ROS2 (e.g. ensenso_camera_msgs2/action/GetParameter.action,
here the `Time` interface changed), are already present in the ROS2 packages.

## Prepare the repo
We provide a script that takes care of ignoring all ROS1 packages and recognizing all ROS2 packages within this repo as
well as copying the C++ and Python and all other relevant files into the ROS2 packages.

Execute the script as follows in order to prepare your ROS2 build:
```
# Assuming you have an ament workspace in your home directory
# and the rosdriver has been cloned or linked in the `src` directory
cd ~/ament_workspace/src/ros_driver

# If ROS2 is sourced
./.github/scripts/prepare_ros2_build.sh

# Or if ROS2 is not sourced
export ROS_VERSION=2 && ./.github/scripts/prepare_ros2_build.sh
```

## Install external dependencies
For a ROS 2 build without PCL, export `ENSENSO_WITH_PCL=OFF` before installing dependencies.
Both the external dependency script and the ROS 2 package manifest honor this setting;
see [Building without PCL](#building-without-pcl) below.

Before we can build the ament workspace we have to install an Ensenso SDK and some external dependencies. If you do not
have an Ensenso SDK installed, you can run:
```
export ENSENSO_INSTALL=/opt/ensenso
export ENSENSO_SDK_VERSION=3.3.1385
# Omit the next lines if you have ROS2 sourced, otherwise replace <your-ros-distro> (e.g. with "humble").
export ROS_VERSION=2
export ROS_DISTRO=<your-ros-distro>
# Install the dependencies.
./.github/scripts/install_external_dependencies.sh
```

If you already have an Ensenso SDK installed, then you can simply run the following commands:
```
sudo apt-get -y install libopencv-dev python3-opencv
sudo apt-get -y install ros-${ROS_DISTRO}-tf-transformations
sudo pip3 install transforms3d
```

If you wish to use our xacro based launch files, you have to install these additional dependencies:
```
sudo apt install ros-${ROS_DISTRO}-joint-state-publisher-gui
sudo apt install ros-${ROS_DISTRO}-xacro
```

## Build the package
Now you should be able to run `colcon build`.

### Building without PCL

ROS 2 has an optional `ENSENSO_WITH_PCL` CMake switch, which defaults to `ON`.
Set the environment variable to `OFF` before running `rosdep` or the external dependency
installer, and pass the matching CMake option when building. The environment variable
controls conditional package dependencies; the CMake option controls compilation and
also overrides any cached setting from a previous build. Use uppercase `ON` or `OFF`.
The Ensenso SDK, OpenCV, Boost headers, and the remaining ROS 2 dependencies are still required.

After preparing the repository for ROS 2, these commands can be used from the workspace
root. They install dependencies for, and build, only the driver and its interfaces.
Separate build/install directories prevent leftover point-cloud executables from an
older installation from appearing in this variant.

```bash
export ENSENSO_WITH_PCL=OFF
rosdep install --from-paths src/ros_driver/ensenso_camera2 src/ros_driver/ensenso_camera_msgs2 \
  --ignore-src --rosdistro "$ROS_DISTRO" -y
colcon build --packages-up-to ensenso_camera \
  --build-base build_no_pcl --install-base install_no_pcl \
  --cmake-args -DENSENSO_WITH_PCL=OFF
source install_no_pcl/setup.bash
```

Use the regular `stereo_node.launch.py` or `mono_node.launch.py` launch files.
The `ensenso_description` package can be built separately if camera-model launch files
are needed. The existing point-cloud test suite assumes a PCL-enabled driver and is
not included in the build selection above.

In this variant:

- Raw/rectified images, disparity images, and depth images remain available, subject to
  the camera's normal capabilities. Depth images use NxLib's internal point map without
  converting it into a PCL cloud or returning a ROS point cloud.
- Pattern detection, hand-eye calibration, workspace calibration, and TF publishing
  retain their existing implementations.
- Point-cloud topics are not advertised. The `texture_point_cloud` executable, its launch
  file, and the `color_point_cloud` script are not built/installed.
- `RequestData` requests for point clouds or normals abort with `error.code = 102` and an
  explanatory message. An empty goal also aborts because it normally requests a cloud;
  explicitly select at least one image type. A mixed image/cloud request aborts as a whole.
- `TexturedPointCloud` requests abort with the same error code. `TelecentricProjection`
  supports explicit depth-image-only requests; its cloud requests/default goals abort.
- Action/message definitions remain compatible, including their `PointCloud2` result
  fields. Defining these ROS messages does not require PCL.

For example, to acquire raw images (replace `/camera` with the camera's namespace):

```bash
ros2 action send_goal /camera/request_data ensenso_camera_msgs/action/RequestData \
  '{request_raw_images: true, publish_results: false, include_results_in_response: true}'
```

The `request_data` Python helper requests clouds and normals by default. When using it
with this build, set its ROS parameters `point_cloud:=false` and `normals:=false`.

To restore full point-cloud support, export `ENSENSO_WITH_PCL=ON`, install the enabled
PCL dependencies, and rebuild with `-DENSENSO_WITH_PCL=ON`. The ROS 1 build is unchanged.

## Run the nodes and scripts
After the build you should be able to launch the ensenso camera nodes and run the provided Python scripts:

```
source ~/ament_workspace/install/setup.bash

# Type the following to see the installed scripts and nodes
ros2 run ensenso_camera <tab><tab>

# Type the following to see the installed launch files
ros2 launch ensenso_camera <tab><tab>

```

## Documentation
The usage is basically the same as for ROS1 except of course for the different ROS2 CLI as shown above.

There is no documentation platform for third-party packages for ROS2 yet, but since we neither changed the API nor the
message and action definitions for ROS2, you can simply refer to our [ROS1 wiki](http://wiki.ros.org/ensenso_driver).

We still provide the same nodes as for ROS1, but one thing that has changed from ROS1 to ROS2 is that there are no
nodelets in ROS2 (the concept has been replaced by that of a `Component`), which is why the launch file names have
changed and some launch files have even been removed. Please keep this in mind when doing the
[tutorials](http://wiki.ros.org/ensenso_driver/Tutorials) on nodelets.

## Limitations
Since ROS2 does not support type masquerading (yet and probably never), point clouds in ROS2 are published as
`sensor_msgs::msg::PointCloud2` and not directly as `pcl::PointCloud<T>` messages as in ROS1. This requires the user to
convert the received point cloud in case `pcl` format is desired.

For more information see:
* https://twitter.com/therealfergs/status/1300128818137649154 (see the comments by Sean Kelly)
* https://github.com/mikeferguson/ros2_cookbook/blob/main/rclcpp/pcl.md
* https://discourse.ros.org/t/optimized-ros-pcl-conversion/25833