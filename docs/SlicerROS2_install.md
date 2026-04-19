# Guide to Using and Testing SlicerROS2 on Ubuntu 24.04

This guide documents the steps to verify and test the integration between **3D Slicer** and **ROS 2 Jazzy** on **Ubuntu 24.04**, using the pre-compiled version of Slicer with the integrated SlicerROS2 module.

## Prerequisites

* **Operating System:** Ubuntu 24.04 LTS (Noble Numbat)
* **ROS 2:** Jazzy Jalisco (installed and configured system-wide)
* **Slicer:** Pre-compiled version `Slicer-5.6.2-2024-04-05-linux-amd64-SlicerROS2-1.0-Ubuntu-24.04-jazzy`

---
## STEP 0: Install prerequisites
## 0.1 Install ROS 2 Jazzy (Desktop Version)
```bash
sudo apt update
sudo apt install ros-jazzy-desktop
```

## 0.2 Environment Setup
Configure the ROS 2 environment and initialize rosdep to manage future dependencies:
```bash
source /opt/ros/jazzy/setup.bash

sudo rosdep init
rosdep update
```

To automatically source ROS 2 in every new terminal, add it to your .bashrc:
```bash
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
```

## 0.3 Install Required System Dependencies
Ubuntu 24.04 lacks several legacy libraries (Qt5, WebEngine, and Networking) required by 3D Slicer. Install them all::
```bash
sudo apt update
sudo apt install libqt5xmlpatterns5 libqt5x11extras5 libqt5multimedia5 libqt5opengl5 libqt5svg5 libqt5printsupport5 libqt5multimediawidgets5 libqt5webengine5 libqt5webenginewidgets5 libnsl2 libnss3 libasound2t64 libfontconfig1 libxcomposite1 libxcursor1 libxi6 libxtst6 libxrandr2 libpulse0 libdouble-conversion3
```
## 0.4 Download and Extract SlicerROS2
Download the pre-compile version of Slicer:
```bash
wget [https://github.com/rosmed/slicer_ros2_module/releases/download/v1.0/Slicer-5.6.2-2024-04-05-linux-amd64-SlicerROS2-1.0-Ubuntu-24.04-jazzy.tar.gz]
```
Extract the archive:
```bash
tar -xvf Slicer-5.6.2-2024-04-05-linux-amd64-SlicerROS2-1.0-Ubuntu-24.04-jazzy.tar.gz
```
Move it to your Home directory for easy access

## STEP 1: Environment Verification and Startup

### 1.1 Verify ROS 2
Open an **Ubuntu Terminal** and verify that ROS 2 is configured correctly:
```bash
ros2 doctor --report
```
The output should display distribution name: jazzy.

### 1.2 Start 3D Slicer
From the Ubuntu Terminal, navigate to the pre-compiled Slicer folder and launch it:
```bash
cd ~/Slicer-5.6.2-2024-04-05-linux-amd64-SlicerROS2-1.0-Ubuntu-24.04-jazzy/
./Slicer
```

### 1.3 Verify the ROS2Tests module
Once Slicer is open, open the Python Console and run the internal diagnostic tests:
```bash
tests = slicer.util.getModuleLogic('ROS2Tests')
tests.run()
```

---
## STEP 2: Communication Test
These tests demonstrate that the native C++ nodes of SlicerROS2 communicate correctly with the ROS 2 system layer.

### 2.1 From Slicer to ROS 2 (Publisher)
* **Step 1:** In Ubuntu Terminal 1, start listening to the topic:
```bash
ros2 topic echo /test_publisher
```

* **Step 2:** In the Slicer Python Console, initialize the node and send the message:
```bash
logic = slicer.util.getModuleLogic('ROS2')
node = logic.GetDefaultROS2Node()
pub = node.CreateAndAddPublisherNode('vtkMRMLROS2PublisherStringNode', '/test_publisher')

pub.Publish('SlicerROS2 works on Ubuntu 24.04!')
```

### 2.2 From ROS 2 to Slicer (Subscriber)
* **Step 1:** In the Slicer Python Console, create a listening node:
```bash
sub = node.CreateAndAddSubscriberNode('vtkMRMLROS2SubscriberStringNode', '/test_subscriber')
```

* **Step 2:** In Ubuntu Terminal 2, publish a message to the network:
```bash
ros2 topic pub --once /test_subscriber std_msgs/msg/String '{data: "Slicer receive message"}'
```

* **Step 3:** In the Slicer Python Console, read the last captured message:
```bash
print(sub.GetLastMessage())
```

## STEP 3: Turtlesim Control Test


**WARING !!!** 
Since Slicer v5.6.2 runs Python 3.9 internally and ROS 2 Jazzy requires Python 3.12 extensions, an isolated os.system call is used. This bypasses the environment conflict for geometric messages (Twist) that are not yet natively supported by the C++ nodes in SlicerROS2 v1.0.

* **Step 1:** In Ubuntu Terminal 3, launch the Turtlesim node:
```bash
ros2 run turtlesim turtlesim_node
```

* **Step 2:** In the Slicer Python Console, execute the following script:
```bash
import os

cmd = """bash -c "unset PYTHONPATH PYTHONHOME LD_LIBRARY_PATH; source /opt/ros/jazzy/setup.bash; ros2 topic pub --once /turtle1/cmd_vel geometry_msgs/msg/Twist '{linear: {x: 2.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 1.5}}'" """

os.system(cmd)
```
* **Result:** The turtle inside the turtlesim window will move in a curve, confirming the successful reception of the command.

## Usefull repository and web site
### Web site: ###
* https://slicer-ros2.readthedocs.io/en/v1.0/index.html
* SliceROS2 prerequisite: https://slicer-ros2.readthedocs.io/en/v1.0/pages/getting-started.html#pre-requisites

### Repository: ###
* SliceROS2: https://github.com/rosmed/slicer_ros2_module
* Download pre-compile verison of Slicer ROS2: https://github.com/rosmed/slicer_ros2_module/releases/download/v1.0/Slicer-5.6.2-2024-04-05-linux-amd64-SlicerROS2-1.0-Ubuntu-24.04-jazzy.tar.gz