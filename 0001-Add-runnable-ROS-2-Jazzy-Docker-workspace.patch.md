From a950494391ca905efe4884bfb00aeac2d77e6e07 Mon Sep 17 00:00:00 2001
From: Codex <codex@openai.com>
Date: Sat, 12 Sep 2026 11:29:12 +0200
Subject: [PATCH] Add runnable ROS 2 Jazzy Docker workspace

---
 .dockerignore |  4 ++++
 Dockerfile    | 32 ++++++++++++++++++++++++++++++
 README.md     | 55 +++++++++++++++++++++++++++++++++++++++++----------
 src/.gitkeep  |  1 +
 4 files changed, 82 insertions(+), 10 deletions(-)
 create mode 100644 .dockerignore
 create mode 100644 Dockerfile
 create mode 100644 src/.gitkeep

diff --git a/.dockerignore b/.dockerignore
new file mode 100644
index 0000000..5d4eed3
--- /dev/null
+++ b/.dockerignore
@@ -0,0 +1,4 @@
+.git
+build
+install
+log
diff --git a/Dockerfile b/Dockerfile
new file mode 100644
index 0000000..a9b55a5
--- /dev/null
+++ b/Dockerfile
@@ -0,0 +1,32 @@
+FROM osrf/ros:jazzy-desktop
+
+SHELL ["/bin/bash", "-c"]
+
+ARG DEBIAN_FRONTEND=noninteractive
+
+RUN apt-get update && apt-get install -y --no-install-recommends \
+    build-essential \
+    git \
+    python3-colcon-common-extensions \
+    python3-rosdep \
+    python3-opencv \
+    ros-jazzy-moveit \
+    ros-jazzy-ros2-control \
+    ros-jazzy-ros2-controllers \
+    ros-jazzy-ros-gz \
+    ros-jazzy-gz-ros2-control \
+    ros-jazzy-xacro \
+    ros-jazzy-robot-state-publisher \
+    ros-jazzy-joint-state-publisher-gui \
+    ros-jazzy-cv-bridge \
+    ros-jazzy-tf2-ros \
+    ros-jazzy-tf2-geometry-msgs \
+    ros-jazzy-rqt-image-view \
+    && rm -rf /var/lib/apt/lists/*
+
+RUN rosdep init 2>/dev/null || true
+RUN rosdep update
+
+WORKDIR /panda_ws
+
+CMD ["bash"]
diff --git a/README.md b/README.md
index 4a96172..0b780fc 100644
--- a/README.md
+++ b/README.md
@@ -103,26 +103,31 @@ Both commands should succeed before continuing.
 
 If Docker is not installed, follow the [official Ubuntu installation instructions](https://docs.docker.com/engine/install/ubuntu/).
 
-### 5.2 Create the project folder
+### 5.2 Clone this repository
 
-For a new local project:
+Run on the Ubuntu host or VM:
 
 ```bash
-mkdir -p ~/robot_projects/Panda-Sorting-Sim-Tutorial/src
-cd ~/robot_projects/Panda-Sorting-Sim-Tutorial
+mkdir -p ~/robot_projects
+cd ~/robot_projects
+git clone https://github.com/ytang19-glitch/Panda-Sorting-Simulation-Build-and-Learn.git
+cd Panda-Sorting-Simulation-Build-and-Learn
 ```
 
-If the repository is already cloned, enter its root folder instead.
+The repository already contains the `Dockerfile` and an initially empty `src/`
+directory. If it is already cloned, enter its root folder instead.
 
-### 5.3 Create a Dockerfile
+### 5.3 Inspect the Dockerfile
 
-Create a file named `Dockerfile` in the repository root:
+The repository root contains this `Dockerfile`:
 
 ```dockerfile
 FROM osrf/ros:jazzy-desktop
 
 SHELL ["/bin/bash", "-c"]
 
+ARG DEBIAN_FRONTEND=noninteractive
+
 RUN apt-get update && apt-get install -y --no-install-recommends \
     build-essential \
     git \
@@ -143,6 +148,9 @@ RUN apt-get update && apt-get install -y --no-install-recommends \
     ros-jazzy-rqt-image-view \
     && rm -rf /var/lib/apt/lists/*
 
+RUN rosdep init 2>/dev/null || true
+RUN rosdep update
+
 WORKDIR /panda_ws
 
 CMD ["bash"]
@@ -194,7 +202,32 @@ Use a ROS domain ID different from any real-robot system running nearby.
 
 > Graphical applications require additional display configuration. This command does not yet enable Gazebo or RViz windows.
 
-### 5.7 Check dependencies inside Docker
+### 5.7 Optionally import the reference Panda project
+
+To experiment with the complete reference packages before rebuilding them stage by
+stage, run these commands **on the host from this repository root**, not inside the
+container:
+
+```bash
+git clone https://github.com/heimizhou1314/Franka-Panda-Robot-Project.git \
+  ../Franka-Panda-Robot-Project
+
+cp -a ../Franka-Panda-Robot-Project/src/. ./src/
+```
+
+Using `./src/` avoids machine-specific absolute paths. Verify the import:
+
+```bash
+ls src
+```
+
+The expected package folders include `panda_description`, `panda_controller`,
+`panda_moveit`, `panda_vision`, `panda_commander`, and `panda_bringup`.
+
+The source remains on the host and is made visible inside Docker through the bind
+mount. Do not clone application code only inside a disposable container.
+
+### 5.8 Check dependencies inside Docker
 
 ```bash
 source /opt/ros/jazzy/setup.bash
@@ -213,7 +246,7 @@ Expected results:
 * Package commands return installation paths.
 * OpenCV prints its version.
 
-### 5.8 Re-enter the container
+### 5.9 Re-enter the container
 
 If the container has stopped:
 
@@ -233,7 +266,7 @@ In each new shell:
 source /opt/ros/jazzy/setup.bash
 ```
 
-### 5.9 Build application packages later
+### 5.10 Build application packages
 
 Once ROS packages have been added to `src`:
 
@@ -241,6 +274,8 @@ Once ROS packages have been added to `src`:
 cd /panda_ws
 source /opt/ros/jazzy/setup.bash
 
+rosdep update
+rosdep install --from-paths src --ignore-src -r -y
 colcon build --symlink-install
 source install/setup.bash
 ```
diff --git a/src/.gitkeep b/src/.gitkeep
new file mode 100644
index 0000000..8b13789
--- /dev/null
+++ b/src/.gitkeep
@@ -0,0 +1 @@
+
-- 
2.51.1

