# Kinova arm ROS2 sim
Walkthrough of simulation Kinova gen3 robot arm in simulation with ROS2 Humble

## Resources
Followed this GitHub: https://github.com/Kinovarobotics/ros2_kortex <br />
This video at times (just enters the same commands listed in the GitHub above): https://www.youtube.com/watch?v=Vcb_A1MmC-g <br />
Resolved error from this issue post: https://github.com/ros-controls/ros2_control/issues/1889 <br />

Installing Rocker: https://github.com/osrf/rocker <br />
Installing Docker: https://docs.docker.com/engine/install/ubuntu/ <br />

## Prerequisites
If you're system is Ubuntu 22.04, you can skip all steps regarding Docker and Rocker. ROS2 Humble runs on Ubuntu 22.04, so if you're system is not that, then we will need to have a container with it. <br />
A docker container will handle running Ubuntu 22.04 to be able to use ROS2 Humble. However, we need an additional tool - rocker. Rocker is a tool to run docker images with customized local support injected for things like nvidia support. Without rocker, simulation software would not be able to run completely in a Docker container. <br />
Make sure you have these installed before beginning if your system is not Ubuntu 22.04:
1. Docker: https://docs.docker.com/engine/install/ubuntu/
2. Rocker: https://github.com/osrf/rocker

### Creating Docker container
Assuming we need a Docker container (system is not Ubuntu 22.04):
```
sudo rocker --x11 --name humble --nocleanup osrf/ros:humble-desktop
```
### Within Docker container
```
apt install git -y
```
```
apt install python3-colcon-common-extensions python3-vcstool
```
```
export COLCON_WS=~/workspace/ros2_kortex_ws
mkdir -p $COLCON_WS/src
```
```
cd $COLCON_WS
git clone https://github.com/Kinovarobotics/ros2_kortex.git src/ros2_kortex
vcs import src --skip-existing --input src/ros2_kortex/ros2_kortex.$ROS_DISTRO.repos
vcs import src --skip-existing --input src/ros2_kortex/ros2_kortex-not-released.$ROS_DISTRO.repos
```
```
vcs import src --skip-existing --input src/ros2_kortex/simulation.humble.repos
```
IF realtime_tools IS STILL **NOT** SYNCED TO NEWEST VERSION ON humble BRANCH. Last time I ran everything this was the case and had to do the following step.
```
cd $COLCON_WS
git clone https://github.com/ros-controls/realtime_tools.git ~/workspace/ros2_kortex_ws/src/realtime_tools
apt update
rosdep install --ignore-src --from-paths src -y -r
```
IF realtime_tools **IS** SYNCED TO NEWEST VERSION ON humble BRANCH. Only run this if you didn't already run the command in the previous step.
```
rosdep install --ignore-src --from-paths src -y -r
```
Colcon build may fail/freeze your system!!! In the ros2 kortex GitHub, they suggest using the flag `--parallel-workers 3` to mitigate your computer freezing. However, when you are using a Docker container, this colcon build is going to use up too much RAM, so I found the only way that works is to have everything be built sequentially. This will take a long time, up to 30 minutes. If you are not running this step through a Docker container, you may be able to get by with just the `parallel-wokers` flag and not using `export MAKEFLAGS="-j 1"` or the `--executor sequential` flag.
```
export MAKEFLAGS="-j 1"
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release --executor sequential
source ~/workspace/ros2_kortex_ws/install/setup.bash
echo "source ~/workspace/ros2_kortex_ws/install/setup.bash" >> ~/.bashrc
```
Everything should be set up! Test with 6 degrees of freedom and robotiq_2f_85 gripper. Other options are displayed on the ros2_kortex GitHub: https://github.com/Kinovarobotics/ros2_kortex.
```
ros2 launch kortex_description view_robot.launch.py gripper:=robotiq_2f_85 dof:=6
```
A simulation should be displayed! Yippee!

## Publishing joint trajectory
These steps will now outline how to publish specific destinations for the robot to reach.
In one terminal:
```
ros2 launch kortex_bringup kortex_sim_control.launch.py \
  robot_type:=gen3 \
  dof:=6 \
  gripper:=robotiq_2f_85 \
  use_sim_time:=true
```

In another terminal:
```
ros2 control load_controller joint_trajectory_controller
```
```
ros2 control set_controller_state joint_trajectory_controller inactive
```
```
ros2 control set_controller_state joint_trajectory_controller active
```
```
ros2 topic pub /joint_trajectory_controller/joint_trajectory trajectory_msgs/JointTrajectory "{
  joint_names: [joint_1, joint_2, joint_3, joint_4, joint_5, joint_6],
  points: [
    { positions: [0, 0, 0, 0, 0, 0], time_from_start: { sec: 10 } },
  ]
}" -1
```
To control gripper (in an additional terminal), `0.0=open`, `0.8=close`
```
ros2 control load_controller robotiq_gripper_controller
```
```
ros2 control set_controller_state robotiq_gripper_controller inactive
```
```
ros2 control set_controller_state robotiq_gripper_controller active
```
```
ros2 action send_goal /robotiq_gripper_controller/gripper_cmd control_msgs/action/GripperCommand "{command:{position: 0.0, max_effort: 100.0}}"
```
This lists the controllers (good for debugging)
```
ros2 control list_controllers
```
