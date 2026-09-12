# Panda Sorting Simulation — Build and Learn

A step-by-step learning project for rebuilding a vision-guided Panda pick-and-place simulation using ROS 2, Gazebo, MoveIt 2, OpenCV, and Docker.

Inspired by [heimizhou1314/Franka-Panda-Robot-Project](https://github.com/heimizhou1314/Franka-Panda-Robot-Project).

The goal is to understand how each component works, implement it incrementally, and document the development process.

> **Status:** Initial tutorial and development plan. The complete sorting implementation is not yet included or validated. The Docker instructions below prepare a development environment; they do not install a finished sorting application.

## 1. Project Scope

This project is simulation-only.

* Runs on a laptop using a Linux environment.
* Does not require a physical Panda, FR3, or RealSense camera.
* Uses a simulated camera and objects.
* Keeps development separate from any real-robot workspace.

### Intended task

The simulated robot will:

1. Observe colored cubes.
2. Detect a selected cube.
3. Estimate its position.
4. Transform that position into the robot-base frame.
5. Plan an approach and grasp.
6. Lift and transport the cube.
7. Release it at a designated location.
8. Report success or failure.

## 2. What I Want to Learn

| Topic             | Practical learning outcome                                             |
| ----------------- | ---------------------------------------------------------------------- |
| Docker            | Build an image, run containers, mount source code, manage dependencies |
| ROS 2             | Create packages, nodes, topics, parameters, actions, and launch files  |
| Robot description | Understand URDF/Xacro, links, joints, and coordinate frames            |
| Gazebo            | Create a world with a robot, objects, contact physics, and sensors     |
| ros2_control      | Connect simulated joints to controllers                                |
| MoveIt 2          | Configure planning groups, IK, collision geometry, and trajectories    |
| OpenCV            | Detect colored objects and extract image coordinates                   |
| Camera geometry   | Convert image measurements into spatial estimates                      |
| TF2               | Transform points between camera and robot frames                       |
| Task execution    | Coordinate approach, grasp, lift, transport, and release               |
| Testing           | Measure accuracy, execution failures, and sorting success              |

## 3. Target Software Stack

| Component                  | Target          |
| -------------------------- | --------------- |
| Container operating system | Ubuntu 24.04    |
| ROS                        | ROS 2 Jazzy     |
| Simulator                  | Gazebo Harmonic |
| Motion planning            | MoveIt 2        |
| Vision                     | OpenCV          |
| Programming                | Python and C++  |
| Environment management     | Docker          |

The first setup path assumes Docker is installed inside an Ubuntu laptop environment or Ubuntu virtual machine.

Windows PowerShell, WSL2, and a conventional Ubuntu VM have different graphical-display configurations. Commands for one environment should not be assumed to work unchanged in another.

## 4. Learning Roadmap

Complete and verify one stage before moving to the next.

| Stage | Build                              | Success condition                                                |
| ----- | ---------------------------------- | ---------------------------------------------------------------- |
| 01    | Docker development environment     | ROS commands and dependencies are available inside the container |
| 02    | Panda description and RViz display | Robot model loads with a connected TF tree                       |
| 03    | Gazebo world                       | Robot, table, cubes, and camera appear                           |
| 04    | Simulated controllers              | Arm and gripper respond to test commands                         |
| 05    | MoveIt configuration               | Robot executes a planned motion                                  |
| 06    | Fixed pick-and-place               | A cube is moved without using vision                             |
| 07    | Color detection                    | Detection follows the intended cube                              |
| 08    | Camera-to-base localization        | Estimated positions match known simulation positions             |
| 09    | Vision-guided sorting              | A detected cube is picked and placed                             |
| 10    | Reliability and testing            | Failures stop execution and results are recorded                 |

For each stage, document:

* What the component does.
* Which files were created or changed.
* How to run it.
* What successful output looks like.
* Common errors and their causes.
* A small experiment to test understanding.

## 5. Build the Docker Environment

### 5.1 Check Docker

Run in the Ubuntu host or VM terminal:

```bash
docker --version
docker info
```

Both commands should succeed before continuing.

If Docker is not installed, follow the [official Ubuntu installation instructions](https://docs.docker.com/engine/install/ubuntu/).

### 5.2 Clone this repository

Run on the Ubuntu host or VM:

```bash
mkdir -p ~/robot_projects
cd ~/robot_projects
git clone https://github.com/ytang19-glitch/Panda-Sorting-Simulation-Build-and-Learn.git
cd Panda-Sorting-Simulation-Build-and-Learn
```

The repository already contains the `Dockerfile` and an initially empty `src/`
directory. If it is already cloned, enter its root folder instead.

### 5.3 Inspect the Dockerfile

The repository root contains this `Dockerfile`:

```dockerfile
FROM osrf/ros:jazzy-desktop

SHELL ["/bin/bash", "-c"]

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    git \
    python3-colcon-common-extensions \
    python3-rosdep \
    python3-opencv \
    ros-jazzy-moveit \
    ros-jazzy-ros2-control \
    ros-jazzy-ros2-controllers \
    ros-jazzy-ros-gz \
    ros-jazzy-gz-ros2-control \
    ros-jazzy-xacro \
    ros-jazzy-robot-state-publisher \
    ros-jazzy-joint-state-publisher-gui \
    ros-jazzy-cv-bridge \
    ros-jazzy-tf2-ros \
    ros-jazzy-tf2-geometry-msgs \
    ros-jazzy-rqt-image-view \
    && rm -rf /var/lib/apt/lists/*

RUN rosdep init 2>/dev/null || true
RUN rosdep update

WORKDIR /panda_ws

CMD ["bash"]
```

This installs development dependencies only. It deliberately does not build a workspace before application packages exist.

### 5.4 Understand the Dockerfile

| Instruction | Meaning                                           |
| ----------- | ------------------------------------------------- |
| `FROM`      | Select the base ROS image                         |
| `SHELL`     | Use Bash for subsequent shell-form build commands |
| `RUN`       | Install software while building the image         |
| `WORKDIR`   | Set the default working directory                 |
| `CMD`       | Start Bash when the container runs                |

An **image** is the prepared environment. A **container** is an instance of that environment.

See the [Dockerfile overview](https://docs.docker.com/build/concepts/dockerfile/) for the underlying build concepts.

### 5.5 Build the image

Run from the folder containing `Dockerfile`:

```bash
docker build -t panda-tutorial:jazzy .
```

The final `.` selects the current directory as the build context.

If the build fails, investigate the first relevant error. Do not remove required dependencies simply to make the build finish.

### 5.6 Start a development container

This initial command starts a terminal-only container:

```bash
docker run -it \
  --name panda_tutorial_dev \
  --env ROS_DOMAIN_ID=42 \
  --mount type=bind,source="$(pwd)/src",target=/panda_ws/src \
  panda-tutorial:jazzy
```

The source mount makes the host’s `src` folder available inside Docker. Changes in that folder are shared in both directions.

Use a ROS domain ID different from any real-robot system running nearby.

> Graphical applications require additional display configuration. This command does not yet enable Gazebo or RViz windows.

### 5.7 Optionally import the reference Panda project

To experiment with the complete reference packages before rebuilding them stage by
stage, run these commands **on the host from this repository root**, not inside the
container:

```bash
git clone https://github.com/heimizhou1314/Franka-Panda-Robot-Project.git \
  ../Franka-Panda-Robot-Project

cp -a ../Franka-Panda-Robot-Project/src/. ./src/
```

Using `./src/` avoids machine-specific absolute paths. Verify the import:

```bash
ls src
```

The expected package folders include `panda_description`, `panda_controller`,
`panda_moveit`, `panda_vision`, `panda_commander`, and `panda_bringup`.

The source remains on the host and is made visible inside Docker through the bind
mount. Do not clone application code only inside a disposable container.

### 5.8 Check dependencies inside Docker

```bash
source /opt/ros/jazzy/setup.bash

printenv ROS_DISTRO
ros2 pkg prefix rviz2
ros2 pkg prefix moveit_ros_move_group
ros2 pkg prefix ros_gz_sim
ros2 pkg prefix gz_ros2_control
python3 -c "import cv2; print(cv2.__version__)"
```

Expected results:

* ROS distribution is `jazzy`.
* Package commands return installation paths.
* OpenCV prints its version.

### 5.9 Re-enter the container

If the container has stopped:

```bash
docker start -ai panda_tutorial_dev
```

If it is already running, open a second terminal:

```bash
docker exec -it panda_tutorial_dev bash
```

In each new shell:

```bash
source /opt/ros/jazzy/setup.bash
```

### 5.10 Build application packages

Once ROS packages have been added to `src`:

```bash
cd /panda_ws
source /opt/ros/jazzy/setup.bash

rosdep update
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

An empty workspace is not proof that the robot application has been built.

As packages are developed, declare their dependencies in `package.xml` and update the container environment accordingly.

### 5.11 Launch the Panda pick-and-place simulation

Build from the workspace root, **not** from `/panda_ws/src`:

```bash
cd /panda_ws
source /opt/ros/jazzy/setup.bash

rosdep update
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source /panda_ws/install/setup.bash
```

Verify that ROS 2 can find the package:

```bash
ros2 pkg list | grep panda_bringup
```

Then launch the application:

```bash
ros2 launch panda_bringup pick_and_place.launch.xml
```

If ROS reports `Package 'panda_bringup' not found`, check that the source
package is complete:

```bash
ls /panda_ws/src/panda_bringup
```

It should contain `package.xml`, `CMakeLists.txt`, and a `launch/`
directory. Then rebuild and source the workspace again:

```bash
cd /panda_ws
source /opt/ros/jazzy/setup.bash
colcon build --symlink-install
source /panda_ws/install/setup.bash
```

Every new Docker shell must source both ROS 2 and the built workspace:

```bash
source /opt/ros/jazzy/setup.bash
source /panda_ws/install/setup.bash
```

Optionally, configure this automatically for future shells:

```bash
echo 'source /opt/ros/jazzy/setup.bash' >> ~/.bashrc
echo 'source /panda_ws/install/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

## 6. Graphical Applications in Docker

Before Stage 02, configure graphical access for the actual laptop environment.

Verify these separately:

1. The Ubuntu environment can display graphical applications.
2. Docker can access the intended display.
3. RViz opens successfully.
4. Gazebo renders successfully.

For a virtual machine, also check its 3D acceleration and graphics drivers.

Do not assume `--gpus all` gives a conventional VM access to the laptop’s physical NVIDIA GPU. GPU availability depends on the host and virtualization setup.

## 7. Planned Package Organization

These packages will be created during development.

| Package                      | Responsibility                                   |
| ---------------------------- | ------------------------------------------------ |
| `panda_tutorial_description` | Robot description and model-display launch files |
| `panda_tutorial_sim`         | Gazebo world, sensors, and ROS–Gazebo bridges    |
| `panda_tutorial_control`     | ros2_control configuration and controller tests  |
| `panda_tutorial_moveit`      | Motion-planning configuration                    |
| `panda_tutorial_vision`      | Detection and object localization                |
| `panda_tutorial_tasks`       | Pick-and-place execution and failure handling    |
| `panda_tutorial_bringup`     | Integrated launch files                          |

## 8. Vision and Coordinate Transforms

An image location is not a robot-base position.

For a rectified pinhole image with valid optical-axis depth:

```text
Xc = (u - cx) * Zc / fx
Yc = (v - cy) * Zc / fy
```

Where:

* `(u, v)` is the detected pixel.
* `(cx, cy)` is the principal point.
* `(fx, fy)` is the focal length in pixel units.
* `Zc` is depth along the camera optical axis.

Then:

```text
p_base = R_base_camera * p_camera + t_base_camera
```

Planned implementation choices:

* Read intrinsics from `CameraInfo`.
* Use a simulated depth camera or an explicitly documented ray–plane intersection method.
* Use the correct optical frame.
* Publish stamped, structured messages.
* Validate estimates against simulation ground truth.

A known table plane can support localization without a depth camera, but the geometric assumption must be explicit and checked.

## 9. Reliability Goals

The integrated application should:

* Reject missing, stale, or invalid detections.
* Check target workspace limits.
* Include relevant obstacles in the MoveIt planning scene.
* Check planning and execution results.
* Stop the sequence after a failed operation.
* Verify grasping beyond simply checking that finger motion stopped.
* Avoid blocking callbacks that prevent feedback updates.
* Record task outcomes and failure reasons.

A successful animation alone does not establish reliable grasping.

## 10. Development Experiments

After the basic demonstration works:

* Move cubes to different positions.
* Change colors and illumination.
* Add distractor objects.
* Compare estimated and true object positions.
* Test unreachable targets.
* Test an empty grasp.
* Add repeated sorting trials.
* Measure success rate and cycle time.

Record the number of trials and test conditions with every reported result.

## 11. Progress Checklist

* [ ] Docker image builds
* [ ] Container dependency checks pass
* [ ] RViz opens inside Docker
* [ ] Panda model loads
* [ ] Gazebo world loads
* [ ] Controllers respond
* [ ] MoveIt plans and executes
* [ ] Fixed pick-and-place works
* [ ] Color detection works
* [ ] Localization is validated
* [ ] Vision-guided sorting works
* [ ] Failure handling is tested
* [ ] Repeated-trial results are documented

## 12. Attribution

Reference project:

[heimizhou1314/Franka-Panda-Robot-Project](https://github.com/heimizhou1314/Franka-Panda-Robot-Project)

This repository is intended as an independent educational rebuild, not a claim of authorship over the reference project.

Any reused source code, robot meshes, descriptions, or other assets must retain their applicable license and attribution notices. Original tutorial material and newly written code should be distinguished from third-party components.
