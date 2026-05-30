# UR3机械臂驱动

## 1.创建工作空间

```
mkdir -p ~/ros_ur3/src
cd ~/ros_ur3
```

## 2.下载并配置

本机ip需要和ur机械臂处于同一网段，ur机械臂为192.168.0.234，本机ip设置为192.168.0.236

```
# 下载下面三个链接的东西，解压放到src里面
#-----------------------------------------------------
https://github.com/UniversalRobots/Universal_Robots_ROS_Driver.git src/Universal_Robots_ROS_Driver

https://github.com/UniversalRobots/Universal_Robots_Client_Library

https://github.com/ros-industrial/universal_robot/tree/noetic-devel
#-----------------------------------------------------

sudo apt-get install ros-noetic-scaled-joint-trajectory-controller
sudo apt-get install ros-noetic-twist-controller
sudo apt-get install ros-noetic-industrial-robot-status-controller
sudo apt-get install ros-noetic-cartesian-trajectory-controller

rosdep install --from-paths src --ignore-src -y

catkin_make_isolated --install或者catkin build（推荐后者）

source ~/ros_ur3/install_isolated/setup.bash
# 永久添加到bashrc
# gedit ~/.bashrc
# source ~/ros_ur3/devel_isolated/setup.bash或者source ~/ros_ur3/develsetup.bash

# 测试仿真
方式一：
roslaunch ur3e_moveit_config demo.launch
方式二：
roslaunch ur_gazebo ur3e_bringup.launch
roslaunch ur3e_moveit_config moveit_planning_execution.launch sim:=true
roslaunch ur3e_moveit_config moveit_rviz.launch

# 连接真实机械臂
# 第一步，机械臂校准，生成my_ur3e_calibration.yaml校准文件
roslaunch ur_calibration calibration_correction.launch robot_ip:=192.168.0.234 
target_filename:="${HOME}/my_ur3e_calibration.yaml"
# 第二步，启动并加载校准文件
roslaunch ur_robot_driver ur3e_bringup.launch robot_ip:=192.168.0.234 kinematics_config:="/home/wjj/.ros/robot_calibration.yaml"
#（启动后，别忘了在示教器上点击运行external_control程序）
# 第三步，启动moveit规划
roslaunch ur3e_moveit_config moveit_planning_execution.launch
# 第四步，启动rviz
roslaunch ur3e_moveit_config moveit_rviz.launch
```

# 添加相机模型

```
cd ~/ros_ur3/src
git clone -b ros1-legacy https://github.com/IntelRealSense/realsense-ros.git
```

## _d435i.urdf.xacro

/realsense2_description/urdf/_d435i.urdf.xacro下就是相机模型文件

```xml
<?xml version="1.0"?>
<!--
License: Apache 2.0. See LICENSE file in root directory.
Copyright(c) 2020 Intel Corporation. All Rights Reserved

This is the URDF model for the Intel RealSense 435i camera, in it's
aluminum peripherial evaluation case.
-->

<robot name="sensor_d435i" xmlns:xacro="http://ros.org/wiki/xacro">
  <xacro:include filename="$(find realsense2_description)/urdf/_d435.urdf.xacro"/>
  <xacro:include filename="$(find realsense2_description)/urdf/_d435i_imu_modules.urdf.xacro"/>

  <xacro:macro name="sensor_d435i" params="parent *origin name:=camera use_nominal_extrinsics:=false">
    <xacro:sensor_d435 parent="${parent}" name="${name}" use_nominal_extrinsics="${use_nominal_extrinsics}">
      <xacro:insert_block name="origin" />
    </xacro:sensor_d435>
    <xacro:d435i_imu_modules name="${name}" use_nominal_extrinsics="${use_nominal_extrinsics}"/>
  </xacro:macro>
</robot>
```

## ur3e_with_d435i.xacro

/home/wjj/ros_ur3/src/universal_robot-noetic-devel/ur_gazebo/urdf下新建ur3e_with_d435i.xacro:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://wiki.ros.org/xacro" name="ur3e_with_d435i">

  <xacro:arg name="joint_limit_params" default=""/>
  <xacro:arg name="kinematics_params" default=""/>
  <xacro:arg name="physical_params" default=""/>
  <xacro:arg name="visual_params" default=""/>
  <xacro:arg name="transmission_hw_interface" default="hardware_interface/PositionJointInterface"/>
  <xacro:arg name="safety_limits" default="false"/>
  <xacro:arg name="safety_pos_margin" default="0.15"/>
  <xacro:arg name="safety_k_position" default="20"/>

  <xacro:include filename="$(find ur_gazebo)/urdf/ur_macro.xacro"/>
  <xacro:ur_robot_gazebo
    prefix=""
    joint_limits_parameters_file="$(arg joint_limit_params)"
    kinematics_parameters_file="$(arg kinematics_params)"
    physical_parameters_file="$(arg physical_params)"
    visual_parameters_file="$(arg visual_params)"
    transmission_hw_interface="$(arg transmission_hw_interface)"
    safety_limits="$(arg safety_limits)"
    safety_pos_margin="$(arg safety_pos_margin)"
    safety_k_position="$(arg safety_k_position)"/>

  <link name="world" />
  <joint name="world_joint" type="fixed">
    <parent link="world" />
    <child link="base_link" />
    <origin xyz="0 0 0" rpy="0 0 0" />
  </joint>
  <xacro:include filename="$(find realsense2_description)/urdf/_d435i.urdf.xacro"/>

  <xacro:sensor_d435i parent="tool0" name="camera" use_nominal_extrinsics="true">
    <origin xyz="0 0.04 0.01" rpy="0 -1.5708 -1.5708"/> 
  </xacro:sensor_d435i>

  <gazebo reference="camera_color_frame">
    <sensor type="depth" name="d435i_depth_sensor">
      <always_on>true</always_on>
      <update_rate>30.0</update_rate>
      <camera>
        <horizontal_fov>1.211258</horizontal_fov>
        <image>
          <format>B8G8R8</format>
          <width>640</width>
          <height>480</height>
        </image>
        <clip>
          <near>0.1</near>
          <far>10.0</far>
        </clip>
      </camera>
      <plugin name="camera_plugin" filename="libgazebo_ros_openni_kinect.so">
        <baseline>0.05</baseline>
        <alwaysOn>true</alwaysOn>
        <updateRate>30.0</updateRate>
        <cameraName>camera</cameraName>
        <imageTopicName>/camera/color/image_raw</imageTopicName>
        <cameraInfoTopicName>/camera/color/camera_info</cameraInfoTopicName>
        <depthImageTopicName>/camera/depth/image_raw</depthImageTopicName>
        <depthImageCameraInfoTopicName>/camera/depth/camera_info</depthImageCameraInfoTopicName>
        <pointCloudTopicName>/camera/depth/color/points</pointCloudTopicName> 
        <frameName>camera_color_optical_frame</frameName>
        <pointCloudCutoff>0.1</pointCloudCutoff>
        <pointCloudCutoffMax>10.0</pointCloudCutoffMax>
      </plugin>
    </sensor>
  </gazebo>
</robot>
```

## ur3e_d435i_gazebo.launch

/home/wjj/ros_ur3/src/universal_robot-noetic-devel/ur_gazebo/launch下新建ur3e_d435i_gazebo.launch：

```xml
<?xml version="1.0"?>
<launch>
  <arg name="paused" default="true" doc="Starts gazebo in paused mode" />
  <include file="$(find ur_gazebo)/launch/ur3e_bringup.launch">
    <arg name="robot_description_file" value="$(find ur_gazebo)/launch/inc/load_ur3e_d435i.launch.xml" />  
    <arg name="paused" value="$(arg paused)" />
  </include>
</launch>
```

## load_ur3e_d435i.launch.xml

/home/wjj/ros_ur3/src/universal_robot-noetic-devel/ur_gazebo/launch/inc下新建load_ur3e_d435i.launch.xml：

```xml
<?xml version="1.0"?>
<launch>
  <arg name="transmission_hw_interface" default="hardware_interface/PositionJointInterface"/>
  <arg name="safety_limits" default="false" />
  <arg name="safety_pos_margin" default="0.15" />
  <arg name="safety_k_position" default="20" />

  <arg name="joint_limit_params" default="$(find ur_description)/config/ur3e/joint_limits.yaml"/>
  <arg name="kinematics_params" default="$(find ur_description)/config/ur3e/default_kinematics.yaml"/>
  <arg name="physical_params" default="$(find ur_description)/config/ur3e/physical_parameters.yaml"/>
  <arg name="visual_params" default="$(find ur_description)/config/ur3e/visual_parameters.yaml"/>

  <param name="robot_description" command="$(find xacro)/xacro '$(find ur_gazebo)/urdf/ur3e_with_d435i.xacro'
    transmission_hw_interface:=$(arg transmission_hw_interface)
    safety_limits:=$(arg safety_limits)
    safety_pos_margin:=$(arg safety_pos_margin)
    safety_k_position:=$(arg safety_k_position)
    joint_limit_params:=$(arg joint_limit_params)
    kinematics_params:=$(arg kinematics_params)
    physical_params:=$(arg physical_params)
    visual_params:=$(arg visual_params)" />
</launch>

```

## ur_control.launch.xml

修改ur_control.launch.xml：

```xml
<?xml version="1.0"?>
<launch>
  <!--
    This file 'pretends' to load a driver for a UR robot, by accepting similar
    arguments and playing a similar role (ie: starting the driver node (in this
    case Gazebo) and loading the ros_control controllers).

    Some of the arguments to this .launch file will be familiar to those using
    the ur_robot_driver with their robot.

    Other parameters are specific to Gazebo.

    Note: we spawn and start the ros_control controllers here, as they are,
    together with gazebo_ros_control, essentially the replacement for the
    driver which would be used with a real robot.
  -->

  <!-- Parameters we share with ur_robot_driver -->
  <arg name="controller_config_file" doc="Config file used for defining the ROS-Control controllers."/>
  <arg name="controllers" default="joint_state_controller eff_joint_traj_controller"/>
  <arg name="stopped_controllers" default="joint_group_eff_controller"/>

  <!-- Gazebo parameters -->
  <arg name="gazebo_model_name" default="robot" doc="The name to give to the model in Gazebo (after spawning it)." />
  <arg name="gazebo_world" default="worlds/empty.world" doc="The '.world' file to load in Gazebo." />
  <arg name="gui" default="true" doc="If true, Gazebo UI is started. If false, only start Gazebo server." />
  <arg name="paused" default="false" doc="If true, start Gazebo in paused mode. If false, start simulation as soon as Gazebo has loaded." />
  <arg name="robot_description_param_name" default="robot_description" doc="Name of the parameter which contains the robot description (ie: URDF) which should be spawned into Gazebo." />
  <arg name="spawn_z" default="0.02" doc="At which height the model should be spawned. NOTE: lower values will cause the robot to collide with the ground plane." />
  <arg name="start_gazebo" default="true" doc="If true, Gazebo will be started. If false, Gazebo will be assumed to have been started elsewhere." />

  <!--Setting initial configuration -->
  <!-- <arg name="initial_joint_positions" default=" -J shoulder_pan_joint 0.0 -J shoulder_lift_joint -1.5708 -J elbow_joint 1.5708 -J wrist_1_joint -1.5708 -J wrist_2_joint -1.5708 -J wrist_3_joint 0.0" doc="Initial joint configuration of the robot"/> -->
  <arg name="initial_joint_positions" default=" -J shoulder_pan_joint 0.7854 -J shoulder_lift_joint -1.5708 -J elbow_joint 1.5708 -J wrist_1_joint -1.5708 -J wrist_2_joint -1.5708 -J wrist_3_joint 3.1416" doc="Initial joint configuration of the robot"/>
  
  <!-- Load controller settings -->
  <rosparam file="$(arg controller_config_file)" command="load"/>

  <!-- Start Gazebo and load the empty world if requested to do so -->
  <include file="$(find gazebo_ros)/launch/empty_world.launch" if="$(arg start_gazebo)">
    <arg name="world_name" value="$(arg gazebo_world)"/>
    <arg name="paused" value="$(arg paused)"/>
    <arg name="gui" value="$(arg gui)"/>
  </include>

  <!-- Spawn the model loaded earlier in the simulation just started -->
  <node name="spawn_gazebo_model" pkg="gazebo_ros" type="spawn_model"
    args="
      -urdf
      -param $(arg robot_description_param_name)
      -model $(arg gazebo_model_name)
      -z $(arg spawn_z)
      $(arg initial_joint_positions)
      "
    output="screen" respawn="false" />

  <!-- Load and start the controllers listed in the 'controllers' arg. -->
  <node name="ros_control_controller_spawner" pkg="controller_manager" type="spawner"
    args="$(arg controllers)" output="screen" respawn="false" />

  <!-- Load other controllers, but do not start them -->
  <node name="ros_control_stopped_spawner" pkg="controller_manager" type="spawner"
    args="--stopped $(arg stopped_controllers)" output="screen" respawn="false" />

</launch>

```

# 生成物体脚本

## spawn_blocks.py

/home/wjj/ros_ur3/src/ur3_grasping/scripts下新建spawn_blocks.py：

```py
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import rospy
from gazebo_msgs.srv import SpawnModel
from geometry_msgs.msg import Pose, Quaternion
import random

# 定义简单的 XML 格式的方块模型 (SDF 格式)
# 这样我们就不用额外创建 .sdf 文件了
def create_cube_sdf(model_name, r, g, b, size=0.05):
    """
    生成一个指定颜色和大小的方块的 XML 字符串。
    size 的单位是米 (0.05 = 5厘米)。
    r, g, b 是 0-1 之间的浮点数。
    """
    return f"""<?xml version="1.0"?>
    <sdf version="1.6">
      <model name="{model_name}">
        <static>false</static>
        <link name="link">
          <inertial>
            <mass>0.1</mass>
            <inertia>
              <ixx>0.00001667</ixx><iyy>0.00001667</iyy><izz>0.00001667</izz>
            </inertia>
          </inertial>
          <collision name="collision">
            <geometry>
              <box>
                <size>{size} {size} {size}</size>
              </box>
            </geometry>
            <surface>
              <friction>
                <ode>
                  <mu>1.0</mu>
                  <mu2>1.0</mu2>
                </ode>
              </friction>
            </surface>
          </collision>
          <visual name="visual">
            <geometry>
              <box>
                <size>{size} {size} {size}</size>
              </box>
            </geometry>
            <material>
              <ambient>{r} {g} {b} 1</ambient>
              <diffuse>{r} {g} {b} 1</diffuse>
            </material>
          </visual>
        </link>
      </model>
    </sdf>"""

def spawn_model(model_name, model_xml, x, y, z):
    """调用 Gazebo 的服务来生成模型"""
    rospy.wait_for_service('/gazebo/spawn_sdf_model')
    try:
        spawn_sdf = rospy.ServiceProxy('/gazebo/spawn_sdf_model', SpawnModel)
        
        pose = Pose()
        pose.position.x = x
        pose.position.y = y
        # z 轴高度设置为方块边长的一半，让它正好躺在地面上
        pose.position.z = z 
        pose.orientation = Quaternion(0, 0, 0, 1) # 默认朝向

        # 调用服务
        status_msg = spawn_sdf(model_name, model_xml, "", pose, "world")
        if status_msg.success:
            rospy.loginfo(f"成功生成模型: {model_name}")
        else:
            rospy.logerr(f"生成模型 {model_name} 失败: {status_msg.status_message}")

    except rospy.ServiceException as e:
        rospy.logerr(f"服务调用失败: {e}")

if __name__ == '__main__':
    # 初始化 ROS 节点
    rospy.init_node('dataset_block_spawner')

    # --- 核心配置：方块的位置 ---
    # 假设你的 UR3e 底座在 (0,0,0)
    # 机械臂前伸，摄像头垂直向下拍摄的区域大约在 x = 0.3 到 0.5 之间
    
    # 我们把三个方块横向排列在 x = 0.4 的位置
    center_x = 0.4
    offset_y = 0.125
    block_size = 0.05 # 5厘米
    spawn_z = block_size / 2.0 # 中心点高度

    # 1. 生成红色方块 (在中间)
    red_xml = create_cube_sdf("red_block", 1, 0, 0, block_size)
    spawn_model("red_block", red_xml, x=center_x, y=0.0+ offset_y, z=spawn_z)

    # 2. 生成绿色方块 (偏左)
    green_xml = create_cube_sdf("green_block", 0, 1, 0, block_size)
    spawn_model("green_block", green_xml, x=center_x, y=0.1+ offset_y, z=spawn_z)

    # 3. 生成蓝色方块 (偏右)
    blue_xml = create_cube_sdf("blue_block", 0, 0, 1, block_size)
    spawn_model("blue_block", blue_xml, x=center_x, y=-0.1+ offset_y, z=spawn_z)

    print("\n")
    print("三个数据集方块已成功放置在机械臂前方的地面上！")
    print("现在你可以启动你的图像采集脚本了。")
```

# Aruco手眼标定

## spawn_aruco.py

在/home/wjj/ros_ur3/src/ur3_grasping/scripts下新建spawn_aruco.py：

```py
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import rospy
import os
import cv2
from gazebo_msgs.srv import SpawnModel
from geometry_msgs.msg import Pose, Quaternion

def setup_gazebo_model_dir():
    """在 ~/.gazebo/models 下自动生成 aruco_marker 模型文件和纹理"""
    home = os.path.expanduser("~")
    model_dir = os.path.join(home, ".gazebo/models/aruco_marker")
    mat_dir = os.path.join(model_dir, "materials/scripts")
    tex_dir = os.path.join(model_dir, "materials/textures")
    
    os.makedirs(mat_dir, exist_ok=True)
    os.makedirs(tex_dir, exist_ok=True)

    # 1. 使用 OpenCV 本地生成 ArUco 图像，彻底避开网络下载问题
    img_path = os.path.join(tex_dir, "aruco_marker.png")
    if not os.path.exists(img_path):
        print("正在使用 OpenCV 本地生成 ArUco 纹理图片...")
        try:
            # 兼容不同版本的 OpenCV API (Noetic 默认是 cv2.aruco.Dictionary_get)
            if hasattr(cv2.aruco, 'Dictionary_get'):
                aruco_dict = cv2.aruco.Dictionary_get(cv2.aruco.DICT_ARUCO_ORIGINAL)
                img = cv2.aruco.drawMarker(aruco_dict, 26, 400)
            else:
                aruco_dict = cv2.aruco.getPredefinedDictionary(cv2.aruco.DICT_ARUCO_ORIGINAL)
                img = cv2.aruco.generateImageMarker(aruco_dict, 26, 400)
            
            # 给 ArUco 码加上白色边框，这能大幅提高相机识别率
            img_with_border = cv2.copyMakeBorder(img, 50, 50, 50, 50, cv2.BORDER_CONSTANT, value=[255, 255, 255])
            cv2.imwrite(img_path, img_with_border)
            print("图片生成成功！")
        except Exception as e:
            print(f"OpenCV 生成图片失败，请检查 cv2 安装: {e}")

    # 2. 写入 model.config
    with open(os.path.join(model_dir, "model.config"), "w") as f:
        f.write("""<?xml version="1.0"?>
<model>
  <name>aruco_marker</name>
  <version>1.0</version>
  <sdf version="1.6">model.sdf</sdf>
  <author><name>Author</name></author>
  <description>Aruco Marker 26</description>
</model>""")

    # 3. 写入 material 脚本 (增加 lighting off，防止阴影导致识别失败)
    with open(os.path.join(mat_dir, "aruco.material"), "w") as f:
        f.write("""material ArucoMarker/Marker
{
  technique
  {
    pass
    {
      lighting off
      texture_unit
      {
        texture ../textures/aruco_marker.png
        filtering trilinear
      }
    }
  }
}""")

    # 4. 写入 model.sdf
    with open(os.path.join(model_dir, "model.sdf"), "w") as f:
        f.write("""<?xml version="1.0"?>
<sdf version="1.6">
  <model name="aruco_marker">
    <static>true</static>
    <link name="link">
      <visual name="visual">
        <geometry>
          <box><size>0.125 0.125 0.001</size></box> </geometry>
        <material>
          <script>
            <uri>model://aruco_marker/materials/scripts</uri>
            <uri>model://aruco_marker/materials/textures</uri>
            <name>ArucoMarker/Marker</name>
          </script>
        </material>
      </visual>
      <collision name="collision">
        <geometry><box><size>0.125 0.125 0.001</size></box></geometry>
      </collision>
    </link>
  </model>
</sdf>""")
    return model_dir

if __name__ == '__main__':
    rospy.init_node('aruco_spawner')
    setup_gazebo_model_dir()
    
    # 从 SDF 文件读取内容
    home = os.path.expanduser("~")
    with open(os.path.join(home, ".gazebo/models/aruco_marker/model.sdf"), "r") as f:
        model_xml = f.read()

    rospy.wait_for_service('/gazebo/spawn_sdf_model')
    try:
        spawn_sdf = rospy.ServiceProxy('/gazebo/spawn_sdf_model', SpawnModel)
        pose = Pose()
        pose.position.x = 0.2
        pose.position.y = 0.35
        pose.position.z = 0.001
        pose.orientation = Quaternion(0, 0, 0, 1)

        # 稍微改一下模型名字防止和刚才加载失败的模型在 Gazebo 缓存里冲突
        status_msg = spawn_sdf("aruco_marker_id26_fixed", model_xml, "", pose, "world")
        if status_msg.success:
            rospy.loginfo("成功生成 ArUco 标定板！")
    except rospy.ServiceException as e:
        rospy.logerr(f"生成失败: {e}")

```

## gazebo_handeye_calibration.launch

在/home/wjj/ros_ur3/src/ur3_grasping/launch下新建gazebo_handeye_calibration.launch用于启动仿真标定程序：

```xml
<?xml version="1.0"?>
<launch>
  <node pkg="aruco_ros" type="single" name="aruco_single">
    <remap from="/camera_info" to="/camera/color/camera_info" />
    <remap from="/image" to="/camera/color/image_raw" />
    
    <param name="image_is_rectified" value="True"/>
    <param name="marker_size"        value="0.1"/>
    <param name="marker_id"          value="26"/>
    <param name="reference_frame"    value="camera_color_optical_frame"/>   <param name="camera_frame"       value="camera_color_optical_frame"/>
    <param name="marker_frame"       value="aruco_marker_frame" />
  </node>

  <include file="$(find easy_handeye)/launch/calibrate.launch">
    <arg name="eye_on_hand"             value="true"/> <arg name="namespace_prefix"        value="ur3e_realsense_handeyecalibration"/>
    
    <arg name="robot_base_frame"        value="base_link"/> 
    <arg name="robot_effector_frame"    value="tool0"/> <arg name="tracking_base_frame"     value="camera_color_optical_frame"/> 
    <arg name="tracking_marker_frame"   value="aruco_marker_frame"/>
    
    <arg name="move_group"              value="manipulator"/> <arg name="freehand_robot_movement" value="false" /> </include>
</launch>

```

## 标定流程Aruco 指令

```
# 启动不带夹爪，只带相机的
roslaunch ur_gazebo ur3e_d435i_gazebo.launch
roslaunch ur3e_moveit_config moveit_planning_execution.launch sim:=true
rosrun ur3_grasping spawn_aruco.py
roslaunch ur3_grasping gazebo_handeye_calibration.launch
```

在开始操作面板之前，必须确保在当前的初始姿态下，相机能清楚地看到我们刚才生成的 ArUco 码。 你可以新开一个终端输入：

```
rqt_image_view
```

在左上角的下拉菜单中选择 `/aruco_single/result` 话题。如果你能看到带有坐标轴的 ArUco 码，说明识别正常，可以开始标定了！

1. **检查初始位姿**：在 **自动运动面板 (右侧窗口)** 点击 `Check starting pose`。如果界面下方提示可以继续，说明 MoveIt 能够基于当前位置进行规划。
2. **生成并执行新姿态**：
   - 点击 `Next Pose`：系统会随机计算一个适合标定的新姿态。
   - 点击 `Plan`：MoveIt 开始规划路径。如果规划成功，右侧的 "No plan yet" 会变成绿色的提示。*(注意：如果规划失败显示红色，没关系，直接再点一次 `Next Pose` 换一个姿态即可)*。
   - 点击 `Execute`：机械臂会自动移动到规划好的位置。
3. **采集样本**：机械臂停稳后，**确保 `rqt_image_view` 中依然能成功识别到 ArUco 码**，然后在 **主面板 (左侧窗口)** 点击 `Take Sample`。此时左下角的 `Samples` 框里会新增一条数据。
4. **循环往复**：重复步骤 2 和步骤 3（`Next Pose` -> `Plan` -> `Execute` -> `Take Sample`）。为了保证精度，建议采集 **15 到 20 个**不同角度的样本。如果在某个姿态下相机丢失了视野（看不到二维码），直接放弃该点，进行下一个姿态。
5. **计算结果**：采集足够的样本后，在 **主面板** 点击 `Compute`。右下角的 `Result` 框中会立刻显示计算出的转换矩阵（Translation 和 Rotation）。
6. **保存数据**：点击 `Save`。标定结果（YAML 文件）会自动保存到 `~/.ros/easy_handeye/ur3e_realsense_handeyecalibration_eye_on_hand.yaml`。

## 检验标定数据Aruco

可以看到误差非常小，注意camera_bottom_screw_frame和camera_color_optical_frame不一样！

urdf中相机相对tool0的位置关系是tool0和camera_bottom_screw_frame的位置关系

```
wjj@wjj-pc:~$ rosrun tf tf_echo camera_bottom_screw_frame camera_color_optical_frame
At time 0.000
- Translation: [0.011, 0.033, 0.013]
- Rotation: in Quaternion [-0.500, 0.500, -0.500, 0.500]
            in RPY (radian) [-1.571, -0.000, -1.571]
            in RPY (degree) [-90.000, -0.000, -90.000]

wjj@wjj-pc:~$ rosrun tf tf_echo tool0 camera_color_optical_frame
At time 0.000
- Translation: [-0.032, -0.053, 0.021]
- Rotation: in Quaternion [0.000, 0.000, 0.000, 1.000]
            in RPY (radian) [0.000, -0.000, 0.000]
            in RPY (degree) [0.000, -0.000, 0.000]

wjj@wjj-pc:~$ cat ~/.ros/easy_handeye/ur3e_realsense_handeyecalibration_eye_on_hand.yaml
parameters:
  eye_on_hand: true
  freehand_robot_movement: false
  move_group: manipulator
  move_group_namespace: /
  namespace: /ur3e_realsense_handeyecalibration_eye_on_hand/
  robot_base_frame: base_link
  robot_effector_frame: tool0
  tracking_base_frame: camera_color_optical_frame
  tracking_marker_frame: aruco_marker_frame
transformation:
  qw: 0.9999891087036972
  qx: -0.0003923828160719017
  qy: -0.00017853548376711714
  qz: -0.004647217962613906
  x: -0.031856988782759096
  y: -0.051442619011138196
  z: 0.01865970873732761
```

# ChAruco手眼标定

## spawn_charuco.py

在 `/home/wjj/ros_ur3/src/ur3_grasping/scripts` 下新建 `spawn_charuco.py`

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import rospy
import os
import cv2
import numpy as np
import tf

from gazebo_msgs.srv import SpawnModel, DeleteModel
from geometry_msgs.msg import Pose, Quaternion
from tf.transformations import (
    quaternion_from_euler,
    euler_from_quaternion,
    quaternion_matrix
)


def setup_gazebo_model_dir():
    """
    在 ~/.gazebo/models 下自动生成 charuco_board 模型文件和纹理。
    严格保持：长边7(X轴) x 短边5(Y轴)，网格完美1:1且图案与实物完全一致。
    """

    home = os.path.expanduser("~")
    model_dir = os.path.join(home, ".gazebo/models/charuco_board")
    mat_dir = os.path.join(model_dir, "materials/scripts")
    tex_dir = os.path.join(model_dir, "materials/textures")

    os.makedirs(mat_dir, exist_ok=True)
    os.makedirs(tex_dir, exist_ok=True)

    # =========================================================
    # 1. ChArUco 核心参数
    # =========================================================
    squares_x = 7           # 严格长边 7 列
    squares_y = 5           # 严格短边 5 行
    square_length = 0.015   # 15 mm
    marker_length = 0.012   # 12 mm

    pixels_per_meter = 10000

    img_width = int(squares_x * square_length * pixels_per_meter)
    img_height = int(squares_y * square_length * pixels_per_meter)

    border_px = 50

    # 物理模型的真实长宽比例
    model_width = (img_width + 2 * border_px) / float(pixels_per_meter)
    model_height = (img_height + 2 * border_px) / float(pixels_per_meter)

    # =========================================================
    # 2. 生成原始纹理并进行【关键旋转处理】
    # =========================================================
    img_path = os.path.join(tex_dir, "charuco.png")
    rospy.loginfo("正在生成并对齐 ChArUco 纹理图片...")

    try:
        # ROS Noetic / OpenCV 4.2 常见 API
        aruco_dict = cv2.aruco.Dictionary_get(cv2.aruco.DICT_4X4_50)
        board = cv2.aruco.CharucoBoard_create(
            squares_x,
            squares_y,
            square_length,
            marker_length,
            aruco_dict
        )
        img = board.draw((img_width, img_height))

    except AttributeError:
        # 新版 OpenCV API
        aruco_dict = cv2.aruco.getPredefinedDictionary(cv2.aruco.DICT_4X4_50)
        board = cv2.aruco.CharucoBoard(
            (squares_x, squares_y),
            square_length,
            marker_length,
            aruco_dict
        )
        img = board.generateImage((img_width, img_height))

    # 加上白色边框
    img_with_border = cv2.copyMakeBorder(
        img,
        border_px,
        border_px,
        border_px,
        border_px,
        cv2.BORDER_CONSTANT,
        value=[255, 255, 255]
    )

    # 【核心修正】：顺时针旋转90度保存，用来对抗 Gazebo 贴图时自带的 90度反向映射
    img_rotated = cv2.rotate(img_with_border, cv2.ROTATE_90_CLOCKWISE)

    ok = cv2.imwrite(img_path, img_rotated)
    if not ok:
        raise RuntimeError("纹理图片写入失败: {}".format(img_path))

    # =========================================================
    # 3. 写 model.config
    # =========================================================
    with open(os.path.join(model_dir, "model.config"), "w") as f:
        f.write("""<?xml version="1.0"?>
<model>
  <name>charuco_board</name>
  <version>1.0</version>
  <sdf version="1.6">model.sdf</sdf>
</model>
""")

    # =========================================================
    # 4. 写 material 文件
    # =========================================================
    with open(os.path.join(mat_dir, "charuco.material"), "w") as f:
        f.write("""material CharucoBoard/Marker
{
  technique
  {
    pass
    {
      lighting off
      ambient 1 1 1 1
      diffuse 1 1 1 1
      specular 0 0 0 1

      texture_unit
      {
        texture charuco.png
        filtering trilinear
      }
    }
  }
}
""")

    # =========================================================
    # 5. 写 model.sdf
    # 【已完全修正】：
    # 物理尺寸严格定义为：X轴 = model_width(长边7)，Y轴 = model_height(短边5)
    # 配合上面旋转了90度的图片，渲染出来就是完美的正方形，且图案方向和你的照片一模一样！
    # =========================================================
    with open(os.path.join(model_dir, "model.sdf"), "w") as f:
        f.write("""<?xml version="1.0"?>
<sdf version="1.6">
  <model name="charuco_board">
    <static>true</static>

    <link name="link">
      <visual name="visual">
        <geometry>
          <box>
            <size>{model_width} {model_height} 0.001</size>
          </box>
        </geometry>

        <material>
          <script>
            <uri>model://charuco_board/materials/scripts</uri>
            <uri>model://charuco_board/materials/textures</uri>
            <name>CharucoBoard/Marker</name>
          </script>
        </material>
      </visual>

      <collision name="collision">
        <geometry>
          <box>
            <size>{model_width} {model_height} 0.001</size>
          </box>
        </geometry>
      </collision>
    </link>
  </model>
</sdf>
""".format(
            model_width=model_width,
            model_height=model_height
        ))

    rospy.loginfo("ChArUco 模型文件生成完成")
    rospy.loginfo("物理网格对齐尺寸: 仿真X轴(长边)=%.4f m, 仿真Y轴(短边)=%.4f m", model_width, model_height)

    return model_dir


def delete_model_if_exists(model_name):
    """如果 Gazebo 中已经存在同名模型，先删除。"""
    try:
        rospy.wait_for_service('/gazebo/delete_model', timeout=3.0)
        delete_model = rospy.ServiceProxy('/gazebo/delete_model', DeleteModel)
        resp = delete_model(model_name)
        if resp.success:
            rospy.loginfo("已删除旧模型: %s", model_name)
            rospy.sleep(0.5)
    except Exception:
        pass


def get_camera_optical_axis_ground_intersection(world_frame, camera_frame, board_z):
    """计算相机光轴与地面的交点"""
    listener = tf.TransformListener()
    rospy.sleep(1.0)

    listener.waitForTransform(world_frame, camera_frame, rospy.Time(0), rospy.Duration(5.0))
    trans, rot = listener.lookupTransform(world_frame, camera_frame, rospy.Time(0))

    cam_x, cam_y, cam_z = trans[0], trans[1], trans[2]
    roll, pitch, yaw = euler_from_quaternion(rot)

    T = quaternion_matrix(rot)
    optical_axis_world = T[0:3, 2]
    dx, dy, dz = optical_axis_world[0], optical_axis_world[1], optical_axis_world[2]

    if abs(dz) < 1e-6:
        raise RuntimeError("相机光轴几乎平行于地面，无法计算和地面的交点")

    scale = (board_z - cam_z) / dz
    board_x = cam_x + scale * dx
    board_y = cam_y + scale * dy

    return board_x, board_y, yaw


if __name__ == '__main__':
    rospy.init_node('charuco_spawner')
    entity_name = "charuco_board"

    # 1. 生成模型与正确映射的 SDF
    model_dir = setup_gazebo_model_dir()
    model_sdf_path = os.path.join(model_dir, "model.sdf")
    with open(model_sdf_path, "r") as f:
        model_xml = f.read()

    # 2. 获取当前相机位置对齐
    world_frame = "world"
    camera_frame = "camera_color_optical_frame"
    board_z = 0.001

    try:
        board_x, board_y, yaw = get_camera_optical_axis_ground_intersection(world_frame, camera_frame, board_z)
    except Exception as e:
        rospy.logerr("计算板子位置失败: %s", e)
        raise

    # 3. 刷新并生成
    delete_model_if_exists(entity_name)
    rospy.wait_for_service('/gazebo/spawn_sdf_model')

    try:
        spawn_sdf = rospy.ServiceProxy('/gazebo/spawn_sdf_model', SpawnModel)
        pose = Pose()
        pose.position.x = board_x
        pose.position.y = board_y
        pose.position.z = board_z

        # 保持板子水平，旋转角 yaw 与相机保持一致
        q = quaternion_from_euler(0.0, 0.0, yaw)
        pose.orientation = Quaternion(q[0], q[1], q[2], q[3])

        status_msg = spawn_sdf(entity_name, model_xml, "", pose, world_frame)
        if status_msg.success:
            rospy.loginfo(">> 成功！当前标定板已完全修正：长边为X轴(7)，短边为Y轴(5)，图案顺序完美对齐实物！")
        else:
            rospy.logerr("Gazebo 拒绝生成: %s", status_msg.status_message)
    except rospy.ServiceException as e:
        rospy.logerr("生成失败: %s", e)
```

## charuco_detector.py

在 `/home/wjj/ros_ur3/src/ur3_grasping/scripts` 下新建 `charuco_detector.py`

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import rospy
import cv2
import numpy as np
import tf2_ros
from cv_bridge import CvBridge
from sensor_msgs.msg import Image, CameraInfo
from geometry_msgs.msg import TransformStamped
from tf.transformations import quaternion_from_matrix

class CharucoDetector:
    def __init__(self):
        rospy.init_node('charuco_detector')
        self.bridge = CvBridge()
        
        # 你的标定板参数
        self.squares_x = 7
        self.squares_y = 5
        self.square_length = 0.015
        self.marker_length = 0.012
        
        # 初始化 OpenCV 的 ChArUco 字典和板对象 (兼容 Noetic OpenCV 4.2)
        try:
            self.dictionary = cv2.aruco.Dictionary_get(cv2.aruco.DICT_4X4_50)
            self.board = cv2.aruco.CharucoBoard_create(
                self.squares_x, self.squares_y, self.square_length, self.marker_length, self.dictionary)
        except AttributeError:
            self.dictionary = cv2.aruco.getPredefinedDictionary(cv2.aruco.DICT_4X4_50)
            self.board = cv2.aruco.CharucoBoard(
                (self.squares_x, self.squares_y), self.square_length, self.marker_length, self.dictionary)

        self.camera_matrix = None
        self.dist_coeffs = None

        self.tf_broadcaster = tf2_ros.TransformBroadcaster()

        # 订阅话题
        rospy.Subscriber("/camera/color/camera_info", CameraInfo, self.info_callback)
        rospy.Subscriber("/camera/color/image_raw", Image, self.image_callback)
        
        # 可视化话题发布
        self.image_pub = rospy.Publisher("/charuco_detector/result", Image, queue_size=1)
        
        rospy.loginfo("ChArUco 检测节点已启动，等待相机数据...")

    def info_callback(self, msg):
        if self.camera_matrix is None:
            self.camera_matrix = np.array(msg.K).reshape(3, 3)
            self.dist_coeffs = np.array(msg.D)

    def image_callback(self, msg):
        if self.camera_matrix is None:
            return

        cv_img = self.bridge.imgmsg_to_cv2(msg, "bgr8")
        
        # 1. 检测 ArUco 码
        try:
            corners, ids, rejected = cv2.aruco.detectMarkers(cv_img, self.dictionary)
        except AttributeError:
            detectorParams = cv2.aruco.DetectorParameters()
            detector = cv2.aruco.ArucoDetector(self.dictionary, detectorParams)
            corners, ids, rejected = detector.detectMarkers(cv_img)

        if ids is not None and len(ids) > 0:
            # 2. 插值计算 ChArUco 棋盘格角点
            retval, charuco_corners, charuco_ids = cv2.aruco.interpolateCornersCharuco(
                corners, ids, cv_img, self.board)
            
            # 【核心修改】改为限速打印：每 0.5 秒最多打印一次，防止 30Hz 刷屏
            rospy.loginfo_throttle(0.5, "当前画面检测到 ChArUco 角点数量: %d / %d", retval, (self.squares_x - 1) * (self.squares_y - 1))
            
            # 至少需要 4 个角点来解算位姿
            if retval > 3:
                cv2.aruco.drawDetectedCornersCharuco(cv_img, charuco_corners, charuco_ids, (0, 255, 0))
                
                # 3. 估计标定板位姿
                valid, rvec, tvec = cv2.aruco.estimatePoseCharucoBoard(
                    charuco_corners, charuco_ids, self.board, self.camera_matrix, self.dist_coeffs, None, None)
                
                if valid:
                    cv2.drawFrameAxes(cv_img, self.camera_matrix, self.dist_coeffs, rvec, tvec, 0.05)
                    self.publish_tf(rvec, tvec, msg.header.stamp)
        else:
            # 未检测到任何码时，每 2.0 秒警告一次
            rospy.logwarn_throttle(2.0, "未检测到任何 ArUco 标记，无法插值 ChArUco 角点")

        self.image_pub.publish(self.bridge.cv2_to_imgmsg(cv_img, "bgr8"))

    def publish_tf(self, rvec, tvec, stamp):
        t = TransformStamped()
        t.header.stamp = stamp
        t.header.frame_id = "camera_color_optical_frame"
        t.child_frame_id = "charuco_board_frame"
        
        t.transform.translation.x = tvec[0][0]
        t.transform.translation.y = tvec[1][0]
        t.transform.translation.z = tvec[2][0]
        
        rmat, _ = cv2.Rodrigues(rvec)
        # 扩展为 4x4 齐次变换矩阵以计算四元数
        T = np.eye(4)
        T[:3, :3] = rmat
        q = quaternion_from_matrix(T)
        
        t.transform.rotation.x = q[0]
        t.transform.rotation.y = q[1]
        t.transform.rotation.z = q[2]
        t.transform.rotation.w = q[3]
        
        self.tf_broadcaster.sendTransform(t)

if __name__ == '__main__':
    try:
        CharucoDetector()
        rospy.spin()
    except rospy.ROSInterruptException:
        pass
```

在 `/home/wjj/ros_ur3/src/ur3_grasping/launch` 下新建 `gazebo_charuco_calibration.launch`

```
<?xml version="1.0"?>
<launch>
  <node pkg="ur3_grasping" type="charuco_detector.py" name="charuco_detector" output="screen" />

  <include file="$(find easy_handeye)/launch/calibrate.launch">
    <arg name="eye_on_hand"             value="true"/> 
    <arg name="namespace_prefix"        value="ur3e_realsense_charuco"/>
    
    <arg name="robot_base_frame"        value="base_link"/> 
    <arg name="robot_effector_frame"    value="tool0"/> 
    <arg name="tracking_base_frame"     value="camera_color_optical_frame"/> 
    
    <arg name="tracking_marker_frame"   value="charuco_board_frame"/>
    
    <arg name="move_group"              value="manipulator"/> 
    <arg name="freehand_robot_movement" value="false" /> 
  </include>
</launch>
```

## 标定流程指令

```
# 1. 启动仿真环境
roslaunch ur_gazebo ur3e_d435i_gazebo.launch

# 2. 启动 MoveIt
roslaunch ur3e_moveit_config moveit_planning_execution.launch sim:=true

# 3. 生成 ChArUco 标定板
rosrun ur3_grasping spawn_charuco.py

# 4. 启动识别与手眼标定 GUI
roslaunch ur3_grasping gazebo_charuco_calibration.launch

# 5. 查看视觉识别结果 (可选，确保相机能看到并识别标定板)
rqt_image_view
# 在下拉菜单选择 /charuco_detector/result
 
```

# 添加机械夹爪

```
cd ~/ros_ur3/src

git clone https://github.com/jr-robotics/robotiq.git

cd ~/ros_ur3/src/robotiq
# 删掉所有 3f 相关的包，眼不见心不烦
rm -rf robotiq_3f_gripper_articulated_gazebo_plugins
rm -rf robotiq_3f_gripper_articulated_gazebo
rm -rf robotiq_3f_gripper_articulated_msgs
rm -rf robotiq_3f_gripper_control
rm -rf robotiq_3f_gripper_joint_state_publisher
rm -rf robotiq_3f_gripper_visualization
rm -rf robotiq_3f_rviz

# 编译工作空间
cd ~/ros_ur3
catkin build
```

## ur3e_d435i_robotiq.xacro

在 `/home/wjj/ros_ur3/src/universal_robot-noetic-devel/ur_gazebo/urdf/` 目录下，新建 `ur3e_d435i_robotiq.xacro` 文件：

```
<?xml version="1.0"?>
<robot xmlns:xacro="http://ros.org/wiki/xacro" name="ur3e_d435i_robotiq">

  <xacro:arg name="joint_limit_params" default="$(find ur_description)/config/ur3e/joint_limits.yaml"/>
  <xacro:arg name="kinematics_params" default="$(find ur_description)/config/ur3e/default_kinematics.yaml"/>
  <xacro:arg name="physical_params" default="$(find ur_description)/config/ur3e/physical_parameters.yaml"/>
  <xacro:arg name="visual_params" default="$(find ur_description)/config/ur3e/visual_parameters.yaml"/>
  <xacro:arg name="transmission_hw_interface" default="hardware_interface/PositionJointInterface"/>
  <xacro:arg name="safety_limits" default="false"/>
  <xacro:arg name="safety_pos_margin" default="0.15"/>
  <xacro:arg name="safety_k_position" default="20"/>

  <xacro:include filename="$(find ur_gazebo)/urdf/ur_macro.xacro"/>
  <xacro:ur_robot_gazebo
    prefix=""
    joint_limits_parameters_file="$(arg joint_limit_params)"
    kinematics_parameters_file="$(arg kinematics_params)"
    physical_parameters_file="$(arg physical_params)"
    visual_parameters_file="$(arg visual_params)"
    transmission_hw_interface="$(arg transmission_hw_interface)"
    safety_limits="$(arg safety_limits)"
    safety_pos_margin="$(arg safety_pos_margin)"
    safety_k_position="$(arg safety_k_position)"/>

  <link name="world" />
  <joint name="world_joint" type="fixed">
    <parent link="world" />
    <child link="base_link" />
    <origin xyz="0 0 0" rpy="0 0 0" />
  </joint>

  <xacro:include filename="$(find realsense2_description)/urdf/_d435i.urdf.xacro"/>
  <xacro:sensor_d435i parent="tool0" name="camera" use_nominal_extrinsics="true">
    <origin xyz="0 0.04 0.01" rpy="0 -1.5708 -1.5708"/> 
  </xacro:sensor_d435i>

  <gazebo reference="camera_color_frame">
    <sensor type="depth" name="d435i_depth_sensor">
      <always_on>true</always_on>
      <update_rate>30.0</update_rate>
      <camera>
        <horizontal_fov>1.211258</horizontal_fov>
        <image>
          <format>B8G8R8</format>
          <width>640</width>
          <height>480</height>
        </image>
        <clip>
          <near>0.1</near>
          <far>10.0</far>
        </clip>
      </camera>
      <plugin name="camera_plugin" filename="libgazebo_ros_openni_kinect.so">
        <baseline>0.05</baseline>
        <alwaysOn>true</alwaysOn>
        <updateRate>30.0</updateRate>
        <cameraName>camera</cameraName>
        <imageTopicName>/camera/color/image_raw</imageTopicName>
        <cameraInfoTopicName>/camera/color/camera_info</cameraInfoTopicName>
        <depthImageTopicName>/camera/depth/image_raw</depthImageTopicName>
        <depthImageCameraInfoTopicName>/camera/depth/camera_info</depthImageCameraInfoTopicName>
        <pointCloudTopicName>/camera/depth/color/points</pointCloudTopicName> 
        <frameName>camera_color_optical_frame</frameName>
        <pointCloudCutoff>0.1</pointCloudCutoff>
        <pointCloudCutoffMax>10.0</pointCloudCutoffMax>
      </plugin>
    </sensor>
  </gazebo>
  
  <xacro:include filename="$(find robotiq_2f_85_gripper_gazebo)/urdf/robotiq_arg2f_85_macro.xacro" />
  <xacro:robotiq_arg2f_85_gazebo prefix="gripper_" transmission_hw_interface="hardware_interface/PositionJointInterface"/>

  <joint name="tool0_to_gripper" type="fixed">
    <parent link="tool0" />
    <child link="gripper_base_link" /> 
    <origin xyz="0 0 0" rpy="0 0 1.5708" />
  </joint>

  <gazebo>
  <plugin name="gazebo_grasp_fix" filename="libgazebo_grasp_fix.so">
    <arm>
      <arm_name>manipulator</arm_name>
      <palm_link>wrist_3_link</palm_link> 
      <gripper_link>gripper_left_inner_finger</gripper_link>
      <gripper_link>gripper_right_inner_finger</gripper_link>
    </arm>
    <forces_angle_tolerance>100</forces_angle_tolerance>
    <update_rate>10</update_rate>
    <grip_count_threshold>4</grip_count_threshold>
    <max_grip_count>8</max_grip_count>
    <release_tolerance>0.005</release_tolerance>
    <disable_collisions_on_attach>false</disable_collisions_on_attach>
    <contact_topic>__default_topic__</contact_topic>
  </plugin>
  </gazebo>

</robot>
```

## load_ur3e_d435i_robotiq.launch.xml

在 `/home/wjj/ros_ur3/src/universal_robot-noetic-devel/ur_gazebo/launch/inc/` 目录下新建 `load_ur3e_d435i_robotiq.launch.xml`：

```
<?xml version="1.0"?>
<launch>
  <arg name="transmission_hw_interface" default="hardware_interface/PositionJointInterface"/>
  <arg name="safety_limits" default="false" />
  <arg name="safety_pos_margin" default="0.15" />
  <arg name="safety_k_position" default="20" />

  <arg name="joint_limit_params" default="$(find ur_description)/config/ur3e/joint_limits.yaml"/>
  <arg name="kinematics_params" default="$(find ur_description)/config/ur3e/default_kinematics.yaml"/>
  <arg name="physical_params" default="$(find ur_description)/config/ur3e/physical_parameters.yaml"/>
  <arg name="visual_params" default="$(find ur_description)/config/ur3e/visual_parameters.yaml"/>

  <param name="robot_description" command="$(find xacro)/xacro '$(find ur_gazebo)/urdf/ur3e_d435i_robotiq.xacro'
    transmission_hw_interface:=$(arg transmission_hw_interface)
    safety_limits:=$(arg safety_limits)
    safety_pos_margin:=$(arg safety_pos_margin)
    safety_k_position:=$(arg safety_k_position)
    joint_limit_params:=$(arg joint_limit_params)
    kinematics_params:=$(arg kinematics_params)
    physical_params:=$(arg physical_params)
    visual_params:=$(arg visual_params)" /> [cite: 9]
</launch>

```

## ur3e_d435i_robotiq_gazebo.launch

在 `/home/wjj/ros_ur3/src/universal_robot-noetic-devel/ur_gazebo/launch/` 目录下新建 `ur3e_d435i_robotiq_gazebo.launch`：

```
<?xml version="1.0"?>
<launch>
  <arg name="paused" default="true" doc="Starts gazebo in paused mode" />
  
  <rosparam file="$(find robotiq_2f_85_gripper_gazebo)/config/robotiq_2f_85_gripper_controllers.yaml" command="load"/>

  <include file="$(find ur_gazebo)/launch/ur3e_bringup.launch">
    <arg name="robot_description_file" value="$(find ur_gazebo)/launch/inc/load_ur3e_d435i_robotiq.launch.xml" />  
    <arg name="paused" value="$(arg paused)" />
    <arg name="controllers" value="joint_state_controller eff_joint_traj_controller gripper_controller"/> [cite: 21]
  </include>
</launch>

```

## robotiq_2f_85_gripper_controllers.yaml

```
gripper_controller:
  # 关键修改：改为位置控制器，不再需要 PID gains
  type: position_controllers/GripperActionController
  joint: gripper_finger_joint
  action_monitor_rate: 20
  goal_tolerance: 0.01
  stall_velocity_threshold: 0.001
  stall_timeout: 1.0

```

# 仿真抓取测试指令

```
roslaunch ur_gazebo ur3e_d435i_robotiq_gazebo.launch
roslaunch ur3e_robotiq_moveit_config move_group.launch
roslaunch ur3e_robotiq_moveit_config moveit_rviz.launch
# 生成方块
rosrun ur3_grasping spawn_blocks.py 
# 测试抓取
rosrun ur3_grasping test_grasp.py 
# 颜色抓取
rosrun ur3_grasping color_grasp.py 
```

使用抓取插件，方便仿真时模拟抓取

```
cd ~/ros_ur3/src
# 克隆 gazebo-pkgs 仓库，其中包含了 gazebo_grasp_fix 插件
git clone https://github.com/JenniferBuehler/gazebo-pkgs.git

git clone https://github.com/JenniferBuehler/general-message-pkgs.git

catkin build
```

## test_grasp.py

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import rospy
import moveit_commander
from geometry_msgs.msg import Pose, Quaternion, PoseStamped
import sys

class ManualGrasper:
    def __init__(self):
        moveit_commander.roscpp_initialize(sys.argv)
        rospy.init_node('manual_grasp_test')

        self.scene = moveit_commander.PlanningSceneInterface()
        self.arm = moveit_commander.MoveGroupCommander("manipulator")
        self.gripper = moveit_commander.MoveGroupCommander("gripper")
        
        # 设置运动参数
        self.arm.set_goal_position_tolerance(0.001)
        self.arm.set_max_velocity_scaling_factor(0.1)
        self.arm.set_max_acceleration_scaling_factor(0.1)

        # 初始关节角度
        self.home_joints = [0.0, -1.5708, 1.5708, -1.5708, -1.5708, 0.0]
        
        # 添加虚拟桌面防止碰撞
        self.add_table_to_scene()
        
        print("--- 机械臂安全控制工具 (已修正姿态突跳) ---")
        print("控制: w/s (x), a/d (y), q/e (z)")
        print("步进: z (+), x (-) | 夹爪: g (闭合), o (打开)")
        print("回归: r (安全回到初始位姿)")
        print("退出: Esc")

    def add_table_to_scene(self):
        """在 MoveIt 中添加一个虚拟平面代表桌面"""
        rospy.sleep(1) # 等待场景初始化
        table_pose = PoseStamped()
        table_pose.header.frame_id = self.arm.get_planning_frame()
        table_pose.pose.position.z = -0.01 
        table_pose.pose.orientation.w = 1.0
        self.scene.add_box("table", table_pose, size=(2.0, 2.0, 0.01))
        rospy.loginfo("已将虚拟桌面加入规划场景")

    def go_home(self):
        """安全回到初始位姿"""
        print("\n>> 正在规划路径回归初始位姿...")
        self.arm.set_joint_value_target(self.home_joints)
        success = self.arm.go(wait=True)
        self.arm.stop()
        if success:
            print(">> 已成功回归。")
        else:
            print(">> 规划失败，请检查路径是否有障碍。")
        return success

    def move_rel(self, dx=0, dy=0, dz=0):
        """相对移动：保持当前姿态不变，仅改变位置"""
        # 1. 获取当前实时的位姿
        current_pose = self.arm.get_current_pose().pose
        
        target_pose = Pose()
        # 2. 在当前坐标基础上叠加位移
        target_pose.position.x = current_pose.position.x + dx
        target_pose.position.y = current_pose.position.y + dy
        target_pose.position.z = current_pose.position.z + dz
        
        # 3. 【核心修复】直接沿用当前的姿态四元数，防止旋转 90 度
        target_pose.orientation = current_pose.orientation
        
        # 执行规划
        self.arm.set_pose_target(target_pose)
        success = self.arm.go(wait=True)
        self.arm.stop()
        self.arm.clear_pose_targets()
        return success

    def control_gripper(self, value):
        """夹爪控制：带有阻挡检测的智能抓取"""
        self.gripper.set_joint_value_target([value])
        print("\n>> 正在发送夹爪指令: %.2f..." % value)
        
        # 1. 异步执行，不要阻塞等待 (wait=False)
        success = self.gripper.go(wait=False)
        if not success:
            print(">> 夹爪路径规划失败！")
            return

        # 2. 监控关节状态，判断是否夹到物体（堵转检测）
        prev_pos = self.gripper.get_current_joint_values()[0]
        stall_time = 0.0
        start_time = rospy.Time.now().to_sec()
        
        while not rospy.is_shutdown():
            rospy.sleep(0.05)  # 50ms 检查一次
            current_pos = self.gripper.get_current_joint_values()[0]
            
            # 判断 1：如果已经顺利到达目标位姿（例如空抓），退出循环
            if abs(current_pos - value) < 0.01:
                print(">> 夹爪已到达目标位置。")
                break
                
            # 判断 2：检测是否卡住（夹到物体触发吸附）
            # 如果关节位置变化极其微小，说明被挡住了
            if abs(current_pos - prev_pos) < 0.0005:
                stall_time += 0.05
                # 如果连续卡住超过 0.2 秒，判定为成功抓取
                if stall_time > 0.2:
                    print(">> 检测到夹爪受阻 (成功吸附)，主动停止动作！")
                    self.gripper.stop()
                    break
            else:
                stall_time = 0.0 # 如果还在动，重置计时器
                
            prev_pos = current_pos
            
            # 判断 3：安全超时保护（避免死循环，设置个5秒上限）
            if (rospy.Time.now().to_sec() - start_time) > 5.0:
                print(">> 夹爪动作超时退出。")
                self.gripper.stop()
                break

    def run(self):
        import tty, termios
        def getch():
            fd = sys.stdin.fileno()
            old_settings = termios.tcgetattr(fd)
            try:
                tty.setraw(sys.stdin.fileno())
                ch = sys.stdin.read(1)
            finally:
                termios.tcsetattr(fd, termios.TCSADRAIN, old_settings)
            return ch

        step = 0.01 
        while not rospy.is_shutdown():
            print("\r当前步进: %.3f m | 输入指令...", end="")
            key = getch()
            
            # 位置控制
            if key == 'w': self.move_rel(dx=step)
            elif key == 's': self.move_rel(dx=-step)
            elif key == 'a': self.move_rel(dy=step)
            elif key == 'd': self.move_rel(dy=-step)
            elif key == 'q': self.move_rel(dz=step)
            elif key == 'e': self.move_rel(dz=-step)
            
            # 功能控制
            elif key == 'r': self.go_home()
            elif key == 'z': step = min(0.1, step + 0.005)
            elif key == 'x': step = max(0.001, step - 0.005)
            elif key == 'g': self.control_gripper(0.5) # 闭合值根据实际修改
            elif key == 'o': self.control_gripper(0.0) # 打开值
            elif key == '\x1b': # Esc 退出
                print("\n程序退出")
                break

if __name__ == '__main__':
    try:
        tester = ManualGrasper()
        tester.run()
    except rospy.ROSInterruptException:
        pass
```

## block_manager.py

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import rospy
import random
from gazebo_msgs.srv import SpawnModel, DeleteModel
from geometry_msgs.msg import Pose, Quaternion
from std_msgs.msg import String # 用于接收抓取状态
from tf.transformations import quaternion_from_euler
import time

class GazeboBlockManager:
    def __init__(self):
        rospy.init_node('block_auto_manager')
        
        # 配置参数
        self.model_name = "dynamic_cube"
        self.target_x = 0.4
        self.target_y = 0.125
        self.last_spawn_time = 0
        
        # 订阅抓取状态话题
        rospy.Subscriber("/grasp_status", String, self.status_callback)
        
        rospy.loginfo(">> 自动方块管理器已启动。等待信号...")
        
        # 初始生成一个
        self.respawn_block()

    def create_cube_sdf(self, name, size, r=0.8, g=0.2, b=0.2):
        """生成随机尺寸的 SDF"""
        return f"""<?xml version="1.0"?>
        <sdf version="1.6">
          <model name="{name}">
            <link name="link">
              <inertial>
                <mass>0.05</mass>
                <inertia><ixx>0.00001</ixx><iyy>0.00001</iyy><izz>0.00001</izz></inertia>
              </inertial>
              <collision name="collision">
                <geometry><box><size>{size} {size} {size}</size></box></geometry>
                <surface>
                  <contact><ode><kp>100000.0</kp><kd>10.0</kd><max_vel>0.1</max_vel><min_depth>0.001</min_depth></ode></contact>
                  <friction><ode><mu>1.5</mu><mu2>1.5</mu2></ode></friction>
                </surface>
              </collision>
              <visual name="visual">
                <geometry><box><size>{size} {size} {size}</size></box></geometry>
                <material><ambient>{r} {g} {b} 1</ambient><diffuse>{r} {g} {b} 1</diffuse></material>
              </visual>
            </link>
          </model>
        </sdf>"""

    def respawn_block(self):
        """核心方法：清理并随机生成"""
        # 1. 删除旧的
        try:
            rospy.wait_for_service('/gazebo/delete_model', 1.0)
            del_srv = rospy.ServiceProxy('/gazebo/delete_model', DeleteModel)
            del_srv(self.model_name)
        except: pass
        
        time.sleep(0.5) # 等待物理引擎刷新

        # 2. 随机参数
        size = random.uniform(0.03, 0.05) # 3-5cm 随机
        yaw = random.uniform(0, 3.14159)  # 随机旋转
        q = quaternion_from_euler(0, 0, yaw)
        
        # 3. 生成新的
        rospy.wait_for_service('/gazebo/spawn_sdf_model')
        spawn_srv = rospy.ServiceProxy('/gazebo/spawn_sdf_model', SpawnModel)
        
        pose = Pose()
        pose.position.x = self.target_x
        pose.position.y = self.target_y
        pose.position.z = size / 2.0 + 0.01 # 稍微悬空防止卡进桌子
        pose.orientation = Quaternion(*q)
        
        xml = self.create_cube_sdf(self.model_name, size)
        spawn_srv(self.model_name, xml, "", pose, "world")
        rospy.loginfo(f">> 已生成新方块: 尺寸={size:.3f}m, 角度={yaw:.2f}rad")

    def status_callback(self, msg):
        """当收到 'dropped' 信号时触发刷新"""
        if msg.data == "dropped":
            rospy.loginfo(">> 检测到物体已丢弃，准备刷新...")
            # 增加一个小延迟，确保机械臂已经离开
            rospy.sleep(1.0)
            self.respawn_block()

if __name__ == '__main__':
    manager = GazeboBlockManager()
    rospy.spin()
```



# 定位测试

## visual_centering.py

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import rospy
import moveit_commander
import tf2_ros
import tf.transformations as tf_trans
import numpy as np
import cv2
from cv_bridge import CvBridge
from sensor_msgs.msg import Image, CameraInfo
from geometry_msgs.msg import Pose
import sys
import threading

# [新增] 导入所需的库
import serial
import random

class VisualCentering:
    def __init__(self):
        moveit_commander.roscpp_initialize(sys.argv)
        rospy.init_node('visual_centering_node')

        # 1. 初始化机械臂
        self.arm = moveit_commander.MoveGroupCommander("manipulator")
        self.arm.set_goal_position_tolerance(0.001)
        self.arm.set_max_velocity_scaling_factor(0.1)
        self.arm.set_max_acceleration_scaling_factor(0.1)

        self.tf_buffer = tf2_ros.Buffer()
        self.tf_listener = tf2_ros.TransformListener(self.tf_buffer)
        
        self.camera_frame = "camera_color_optical_frame"
        self.base_frame = "base_link"

        # 2. 视觉与映射参数
        self.bridge = CvBridge()
        self.color_image = None
        
        self.cx = None 
        self.cy = None 
        self.fx = None 
        self.fy = None 
        self.Zc = None
        
        # [新增] 蓝牙串口初始化参数设置
        self.serial_port = '/dev/rfcomm0'
        self.baud_rate = 9600
        self.ble_serial = None
        self.init_bluetooth()
        
        # 订阅相机内参话题
        rospy.Subscriber("/camera/color/camera_info", CameraInfo, self.info_callback)
        rospy.loginfo("=============================================")
        rospy.loginfo("正在等待读取相机的真实内参 (fx, fy, cx, cy)...")
        rospy.loginfo("=============================================")
        
        # 3. 初始化姿态并移至拍照点
        self.home_joints = [0.7854, -1.5708, 1.5708, -1.5708, -1.5708, 3.1416]
        self.safe_orientation = None
        
        rospy.loginfo("正在移动到固定拍照位姿...")
        self.go_home()

        rospy.sleep(0.5) 
        try:
            trans = self.tf_buffer.lookup_transform(self.base_frame, self.camera_frame, rospy.Time(0), rospy.Duration(1.0))
            self.Zc = trans.transform.translation.z 
            rospy.loginfo("=============================================")
            rospy.loginfo(">> 成功自动计算启动拍摄深度 Zc (到地板距离): %.4f 米", self.Zc)
            rospy.loginfo("=============================================")
        except Exception as e:
            rospy.logwarn("启动时自动获取 Zc 失败 (TF未就绪)，将在双击定位时尝试动态获取: %s", e)

        # 4. 开启交互窗口
        cv2.namedWindow("Visual_Centering")
        cv2.setMouseCallback("Visual_Centering", self.mouse_click)
        
        rospy.Subscriber("/camera/color/image_raw", Image, self.image_callback, queue_size=1, buff_size=2**24)
        
        rospy.loginfo("--- 视觉定位节点已启动 ---")
        rospy.loginfo("请在弹出的画面中【双击左键】目标，机械臂将自动对准！")
        rospy.loginfo("按【R】键回归拍照位，按【G】键发送蓝牙随机数，按【Q】键退出。")

    # [新增] 蓝牙串口初始化函数
    def init_bluetooth(self):
        try:
            # 参考 ble_test.py 串口打开方式 [cite: 6, 7]
            self.ble_serial = serial.Serial(
                port=self.serial_port,
                baudrate=self.baud_rate,
                bytesize=serial.EIGHTBITS,
                parity=serial.PARITY_NONE,
                stopbits=serial.STOPBITS_ONE,
                timeout=0.5,
                write_timeout=2,
            )
            rospy.loginfo(f"[OK] 蓝牙串口 {self.serial_port} 初始化成功！")
        except serial.SerialException as e:
            rospy.logwarn(f"[WARN] 无法打开蓝牙串口 {self.serial_port}，请确认蓝牙已连接且绑定了 rfcomm0。异常: {e}")

    # [新增] 生成随机数并通过蓝牙发送
    def send_ble_random(self):
        if self.ble_serial and self.ble_serial.is_open:
            # 生成 0 - 100 的随机数
            rand_val = random.randint(0, 100)
            # 参考 ble_test.py 追加换行符 [cite: 9]
            msg = f"{rand_val}\r\n"
            try:
                # 参考 ble_test.py 发送数据逻辑 [cite: 9]
                self.ble_serial.write(msg.encode("utf-8"))
                self.ble_serial.flush()
                rospy.loginfo(f"[TX] 成功发送蓝牙数据: {rand_val}")
            except serial.SerialException as e:
                rospy.logerr(f"[WARN] 蓝牙发送失败 (可能已断开): {e}")
        else:
            rospy.logwarn("[WARN] 蓝牙串口未连接，无法发送数据。请先执行蓝牙绑定操作。")

    def info_callback(self, msg):
        if self.fx is None:
            self.fx = msg.K[0]  
            self.cx = msg.K[2]  
            self.fy = msg.K[4]  
            self.cy = msg.K[5]  
            
            rospy.loginfo(">> 成功动态获取相机内参:")
            rospy.loginfo("   fx: %.2f, fy: %.2f", self.fx, self.fy)
            rospy.loginfo("   cx: %.2f, cy: %.2f", self.cx, self.cy)

    def go_home(self):
        self.arm.set_joint_value_target(self.home_joints)
        success = self.arm.go(wait=True)
        self.arm.stop()
        
        rospy.sleep(0.2) 
        current_pose = self.arm.get_current_pose().pose
        self.safe_orientation = current_pose.orientation
        
        if not success:
            rospy.logwarn("MoveIt 返回未完全成功 (通常是因为已在目标位置或容差超时)。已强制获取当前姿态。")
            
        return success

    def image_callback(self, msg):
        try:
            self.color_image = self.bridge.imgmsg_to_cv2(msg, "bgr8")
        except Exception as e:
            rospy.logerr("图像转换失败: %s", e)

    def mouse_click(self, event, x, y, flags, param):
        if event == cv2.EVENT_LBUTTONDBLCLK:
            rospy.loginfo("检测到点击坐标: u=%d, v=%d", x, y)
            
            move_thread = threading.Thread(target=self.execute_centering, args=(x, y))
            move_thread.daemon = True
            move_thread.start()

    def execute_centering(self, target_u, target_v):
        if self.safe_orientation is None:
            rospy.logwarn("未获取到安全姿态，拒绝移动。")
            return

        if self.fx is None or self.fy is None:
            rospy.logwarn("尚未获取到相机内参，正在等待，请稍后再试！")
            return

        try:
            trans = self.tf_buffer.lookup_transform(self.base_frame, self.camera_frame, rospy.Time(0), rospy.Duration(1.0))
            self.Zc = trans.transform.translation.z
        except Exception as e:
            rospy.logerr("动态获取相机高度 Zc 失败: %s", e)
            if self.Zc is None:
                return
            rospy.logwarn("无法获取实时高度，将复用历史高度 Zc: %.4f 米", self.Zc)

        delta_u = target_u - self.cx
        delta_v = target_v - self.cy
        
        move_x_cam = (delta_u * self.Zc) / self.fx
        move_y_cam = (delta_v * self.Zc) / self.fy
        
        rospy.loginfo("=============================================")
        rospy.loginfo(">> 当前定位使用的 Zc (到地板距离): %.4f 米", self.Zc)
        rospy.loginfo("像素偏差: du=%.1f, dv=%.1f -> 物理移动: dx=%.3fm, dy=%.3fm", 
                      delta_u, delta_v, move_x_cam, move_y_cam)
        rospy.loginfo("=============================================")

        try:
            trans = self.tf_buffer.lookup_transform(self.base_frame, self.camera_frame, rospy.Time(0), rospy.Duration(1.0))
        except Exception as e:
            rospy.logerr("TF 失败: %s", e)
            return

        q = [trans.transform.rotation.x, trans.transform.rotation.y, trans.transform.rotation.z, trans.transform.rotation.w]
        rot_matrix = tf_trans.quaternion_matrix(q)
        
        v_cam = np.array([move_x_cam, move_y_cam, 0.0, 0.0])
        v_base = np.dot(rot_matrix, v_cam)

        current_pose = self.arm.get_current_pose().pose
        target_pose = Pose()
        target_pose.position.x = current_pose.position.x + v_base[0]
        target_pose.position.y = current_pose.position.y + v_base[1]
        target_pose.position.z = current_pose.position.z 
        target_pose.orientation = self.safe_orientation  

        rospy.loginfo("正在移动机械臂对准目标...")
        self.arm.set_pose_target(target_pose)
        success = self.arm.go(wait=True)
        self.arm.stop()
        self.arm.clear_pose_targets()
        
        if success:
            rospy.loginfo("到达目标上方！你可以继续微调，或按 R 键重置。")

    def run(self):
        rate = rospy.Rate(30)
        while not rospy.is_shutdown():
            if self.color_image is not None:
                display_img = self.color_image.copy()
                
                if self.cx is not None and self.cy is not None:
                    cv2.line(display_img, (int(self.cx), 0), (int(self.cx), 480), (0, 255, 0), 1)
                    cv2.line(display_img, (0, int(self.cy)), (640, int(self.cy)), (0, 255, 0), 1)
                
                cv2.imshow("Visual_Centering", display_img)
            
            key = cv2.waitKey(1) & 0xFF
            if key == ord('q') or key == 27: 
                break
            elif key == ord('r'):
                rospy.loginfo("回归固定拍照位...")
                home_thread = threading.Thread(target=self.go_home)
                home_thread.daemon = True
                home_thread.start()
            # [新增] 监听小写 g 键
            elif key == ord('g'):
                self.send_ble_random()
                
            rate.sleep()
            
        # [新增] 退出前关闭串口释放资源，参考 ble_test.py [cite: 20, 21]
        if self.ble_serial and self.ble_serial.is_open:
            self.ble_serial.close()
            rospy.loginfo("[INFO] 蓝牙串口已关闭")
            
        cv2.destroyAllWindows()
        moveit_commander.roscpp_shutdown()

if __name__ == '__main__':
    try:
        node = VisualCentering()
        node.run()
    except rospy.ROSInterruptException:
        pass
```

## visual_centering_charuco.py

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import rospy
import moveit_commander
import tf2_ros
import tf.transformations as tf_trans
import numpy as np
import cv2
import yaml
import os
import math
from cv_bridge import CvBridge
from sensor_msgs.msg import Image, CameraInfo
from geometry_msgs.msg import Pose
import sys
import threading

class VisualCenteringCharuco:
    def __init__(self):
        moveit_commander.roscpp_initialize(sys.argv)
        rospy.init_node('visual_centering_charuco_node')

        # =====================================================================
        # 【手动修正 / 标定参数配置区】
        # =====================================================================
        self.use_manual_calibration = True
        
        self.calib_x  =  0.03485299792195698
        self.calib_y  =  0.05047829145349603
        self.calib_z  =  0.014236611933224954
        self.calib_qx =  0.0014603744589137904
        self.calib_qy = -0.003567052766137113
        self.calib_qz =  0.9980282724813554
        self.calib_qw =  0.06264751207258762

        self.offset_x = 0.000  
        self.offset_y = 0.000
        self.offset_z = 0.000

        self.offset_roll_deg  = 0.0  
        self.offset_pitch_deg = 0.0  
        self.offset_yaw_deg   = 0.0  
        # =====================================================================

        # 获取外参矩阵
        self.T_tool0_cam = self.get_calibration_matrix()

        # 初始化机械臂
        self.arm = moveit_commander.MoveGroupCommander("manipulator")
        self.arm.set_goal_position_tolerance(0.001)
        self.arm.set_max_velocity_scaling_factor(0.1)
        self.arm.set_max_acceleration_scaling_factor(0.1)

        self.tf_buffer = tf2_ros.Buffer()
        self.tf_listener = tf2_ros.TransformListener(self.tf_buffer)
        
        self.base_frame = "base_link"
        self.tool_frame = "tool0"

        # 视觉与映射参数
        self.bridge = CvBridge()
        self.color_image = None
        
        self.cx = None 
        self.cy = None 
        self.fx = None 
        self.fy = None 
        self.Zc = None 
        
        rospy.Subscriber("/camera/color/camera_info", CameraInfo, self.info_callback)
        rospy.loginfo("=============================================")
        rospy.loginfo("正在等待读取相机的真实内参 (fx, fy, cx, cy)...")
        rospy.loginfo("=============================================")
        
        # 初始化姿态并移至拍照点
        self.home_joints = [0.7854, -1.5708, 1.5708, -1.5708, -1.5708, 3.1416]
        self.safe_orientation = None
        
        rospy.loginfo("正在移动到固定拍照位姿...")
        self.go_home()

        rospy.sleep(0.5) 
        self.update_camera_pose()

        # 开启交互窗口
        cv2.namedWindow("Visual_Centering_Charuco")
        cv2.setMouseCallback("Visual_Centering_Charuco", self.mouse_click)
        rospy.Subscriber("/camera/color/image_raw", Image, self.image_callback, queue_size=1, buff_size=2**24)
        
        rospy.loginfo("--- 基于 ChArUco 标定的视觉定位节点已启动 ---")
        rospy.loginfo("请在弹出的画面中【双击左键】目标，机械臂将自动对准！")
        rospy.loginfo("按【R】键回归拍照位，按【Q】键退出。")

    def get_calibration_matrix(self):
        if self.use_manual_calibration:
            rospy.loginfo(">> [配置] 正在使用手动输入的基准标定参数。")
            t_base = [self.calib_x, self.calib_y, self.calib_z]
            q_base = [self.calib_qx, self.calib_qy, self.calib_qz, self.calib_qw]
        else:
            yaml_path = "~/.ros/easy_handeye/ur3e_realsense_charuco_eye_on_hand.yaml"
            full_path = os.path.expanduser(yaml_path)
            if not os.path.exists(full_path):
                rospy.logerr(f"找不到标定文件: {full_path}")
                sys.exit(1)
                
            with open(full_path, 'r') as f:
                calib_data = yaml.safe_load(f)
                
            trans = calib_data['transformation']
            t_base = [trans['x'], trans['y'], trans['z']]
            q_base = [trans['qx'], trans['qy'], trans['qz'], trans['qw']]
            rospy.loginfo(">> [配置] 正在读取 YAML 标定文件。")

        T_base = tf_trans.quaternion_matrix(q_base)
        T_base[0:3, 3] = t_base

        roll_rad  = math.radians(self.offset_roll_deg)
        pitch_rad = math.radians(self.offset_pitch_deg)
        yaw_rad   = math.radians(self.offset_yaw_deg)
        
        T_offset = tf_trans.euler_matrix(roll_rad, pitch_rad, yaw_rad, axes='sxyz')
        T_offset[0:3, 3] = [self.offset_x, self.offset_y, self.offset_z]

        T_final = np.dot(T_base, T_offset)
        
        final_q = tf_trans.quaternion_from_matrix(T_final)
        final_euler_rad = tf_trans.euler_from_matrix(T_final, axes='sxyz')
        final_euler_deg = [math.degrees(angle) for angle in final_euler_rad]

        rospy.loginfo("==========================================================")
        rospy.loginfo(">> [配置] 最终使用的手眼外参 (叠加补偿后):")
        rospy.loginfo("   平移 (X, Y, Z)   : [%.5f, %.5f, %.5f] 米", T_final[0,3], T_final[1,3], T_final[2,3])
        rospy.loginfo("   四元数 (qx..qw)  : [%.5f, %.5f, %.5f, %.5f]", final_q[0], final_q[1], final_q[2], final_q[3])
        rospy.loginfo("   欧拉角 (R, P, Y) : [%.3f°, %.3f°, %.3f°]", final_euler_deg[0], final_euler_deg[1], final_euler_deg[2])
        rospy.loginfo("==========================================================")

        return T_final

    def info_callback(self, msg):
        if self.fx is None:
            self.fx = msg.K[0]
            self.cx = msg.K[2]
            self.fy = msg.K[4]
            self.cy = msg.K[5]
            
            rospy.loginfo(">> 成功动态获取相机内参:")
            rospy.loginfo("   fx: %.2f, fy: %.2f", self.fx, self.fy)
            rospy.loginfo("   cx: %.2f, cy: %.2f", self.cx, self.cy)

    def go_home(self):
        self.arm.set_joint_value_target(self.home_joints)
        success = self.arm.go(wait=True)
        self.arm.stop()
        rospy.sleep(0.2) 
        
        current_pose = self.arm.get_current_pose().pose
        self.safe_orientation = current_pose.orientation
        
        if not success:
            rospy.logwarn("MoveIt 返回未完全成功 (通常是因为已在目标位置或容差超时)。已强制获取当前姿态。")
            
        return success

    def image_callback(self, msg):
        try:
            self.color_image = self.bridge.imgmsg_to_cv2(msg, "bgr8")
        except Exception as e:
            rospy.logerr("图像转换失败: %s", e)

    def update_camera_pose(self):
        try:
            trans = self.tf_buffer.lookup_transform(self.base_frame, self.tool_frame, rospy.Time(0), rospy.Duration(1.0))
            
            t = [trans.transform.translation.x, trans.transform.translation.y, trans.transform.translation.z]
            q = [trans.transform.rotation.x, trans.transform.rotation.y, trans.transform.rotation.z, trans.transform.rotation.w]
            
            T_base_tool0 = tf_trans.quaternion_matrix(q)
            T_base_tool0[0:3, 3] = t
            
            self.T_base_cam = np.dot(T_base_tool0, self.T_tool0_cam)
            self.Zc = self.T_base_cam[2, 3]
            return True
        except Exception as e:
            rospy.logwarn("无法获取 tool0 TF: %s", e)
            return False

    def mouse_click(self, event, x, y, flags, param):
        if event == cv2.EVENT_LBUTTONDBLCLK:
            rospy.loginfo("检测到点击坐标: u=%d, v=%d", x, y)
            move_thread = threading.Thread(target=self.execute_centering, args=(x, y))
            move_thread.daemon = True
            move_thread.start()

    def execute_centering(self, target_u, target_v):
        if self.safe_orientation is None:
            rospy.logwarn("未获取到安全姿态，拒绝移动。")
            return
            
        if self.fx is None or self.fy is None:
            rospy.logwarn("尚未获取到相机内参，正在等待，请稍后再试！")
            return

        if not self.update_camera_pose():
            rospy.logwarn("未能更新相机在基座下的真实位姿。")
            return

        delta_u = target_u - self.cx
        delta_v = target_v - self.cy
        
        move_x_cam = (delta_u * self.Zc) / self.fx
        move_y_cam = (delta_v * self.Zc) / self.fy
        
        rospy.loginfo("=============================================")
        rospy.loginfo(">> 当前定位使用的真实标定深度 Zc: %.4f 米", self.Zc)
        rospy.loginfo("像素偏差: du=%.1f, dv=%.1f -> 物理移动: dx=%.3fm, dy=%.3fm", 
                      delta_u, delta_v, move_x_cam, move_y_cam)
        rospy.loginfo("=============================================")

        v_cam = np.array([move_x_cam, move_y_cam, 0.0, 0.0])
        v_base = np.dot(self.T_base_cam, v_cam)

        current_pose = self.arm.get_current_pose().pose
        target_pose = Pose()
        target_pose.position.x = current_pose.position.x + v_base[0]
        target_pose.position.y = current_pose.position.y + v_base[1]
        target_pose.position.z = current_pose.position.z 
        target_pose.orientation = self.safe_orientation 

        rospy.loginfo("正在移动机械臂对准目标...")
        self.arm.set_pose_target(target_pose)
        success = self.arm.go(wait=True)
        self.arm.stop()
        self.arm.clear_pose_targets()
        
        if success:
            rospy.loginfo("到达目标上方！你可以继续微调，或按 R 键重置。")

    def run(self):
        rate = rospy.Rate(30)
        while not rospy.is_shutdown():
            if self.color_image is not None:
                display_img = self.color_image.copy()
                if self.cx is not None and self.cy is not None:
                    cv2.line(display_img, (int(self.cx), 0), (int(self.cx), 480), (0, 255, 0), 1)
                    cv2.line(display_img, (0, int(self.cy)), (640, int(self.cy)), (0, 255, 0), 1)
                
                cv2.imshow("Visual_Centering_Charuco", display_img)
            
            key = cv2.waitKey(1) & 0xFF
            if key == ord('q') or key == 27:
                break
            elif key == ord('r'):
                rospy.loginfo("回归固定拍照位...")
                threading.Thread(target=self.go_home, daemon=True).start()
                
            rate.sleep()
            
        cv2.destroyAllWindows()
        moveit_commander.roscpp_shutdown()

if __name__ == '__main__':
    try:
        node = VisualCenteringCharuco()
        node.run()
    except rospy.ROSInterruptException:
        pass
```

## spawn_practice_plane.py

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import rospy
from gazebo_msgs.srv import SpawnModel, DeleteModel
from geometry_msgs.msg import Pose

def create_grid_plane_sdf(size_x=0.75, size_y=0.75, grid_step=0.1, line_width=0.002):
    """
    动态生成一个带有网格线的平面 SDF 字符串。
    采用在一个 Link 中放置一个白色底板 + 多个黑色线条 Visual 的方式，防止 Z-fighting 闪烁。
    """
    # 基础白色平面的 SDF 头部
    # z轴厚度设定为 0.001 (1毫米) 来模拟“没有厚度”
    sdf = f"""<?xml version="1.0"?>
    <sdf version="1.6">
      <model name="practice_plane">
        <static>true</static>
        <link name="link">
          <collision name="collision">
            <geometry>
              <box><size>{size_x} {size_y} 0.001</size></box>
            </geometry>
            <surface>
              <friction>
                <ode>
                  <mu>1.0</mu>
                  <mu2>1.0</mu2>
                </ode>
              </friction>
            </surface>
          </collision>
          <visual name="base_white_plane">
            <pose>0 0 0 0 0 0</pose>
            <geometry>
              <box><size>{size_x} {size_y} 0.001</size></box>
            </geometry>
            <material>
              <ambient>1 1 1 1</ambient>
              <diffuse>1 1 1 1</diffuse>
            </material>
          </visual>
    """
    
    # 动态生成网格线 (黑色)
    # Z 轴微调到 0.0006，确保线条刚好浮在白色平面之上一点点，避免渲染闪烁
    line_z = 0.0006 
    
    # 1. 生成平行于 Y 轴的线 (沿 X 轴分布)
    num_x = int(size_x / grid_step) + 1
    start_x = -size_x / 2.0
    for i in range(num_x):
        curr_x = start_x + i * grid_step
        if curr_x > size_x / 2.0: curr_x = size_x / 2.0 # 防止浮点数溢出
        sdf += f"""
          <visual name="grid_line_x_{i}">
            <pose>{curr_x} 0 {line_z} 0 0 0</pose>
            <geometry>
              <box><size>{line_width} {size_y} 0.001</size></box>
            </geometry>
            <material>
              <ambient>0 0 0 1</ambient>
              <diffuse>0 0 0 1</diffuse>
            </material>
          </visual>"""

    # 2. 生成平行于 X 轴的线 (沿 Y 轴分布)
    num_y = int(size_y / grid_step) + 1
    start_y = -size_y / 2.0
    for i in range(num_y):
        curr_y = start_y + i * grid_step
        if curr_y > size_y / 2.0: curr_y = size_y / 2.0
        sdf += f"""
          <visual name="grid_line_y_{i}">
            <pose>0 {curr_y} {line_z} 0 0 0</pose>
            <geometry>
              <box><size>{size_x} {line_width} 0.001</size></box>
            </geometry>
            <material>
              <ambient>0 0 0 1</ambient>
              <diffuse>0 0 0 1</diffuse>
            </material>
          </visual>"""

    # 闭合标签
    sdf += """
        </link>
      </model>
    </sdf>"""
    return sdf

def spawn_plane():
    rospy.init_node('spawn_practice_plane')
    model_name = "practice_plane"
    
    # ================= 核心配置区 =================
    # 根据你的需求：x在 0~0.75，y在 0~0.75 范围内
    size_x = 0.75
    size_y = 0.75
    
    # 【控制网格密度】
    # 0.10 代表 10cm 一个网格，0.05 代表 5cm 一个网格
    grid_spacing = 0.05 
    # ==============================================

    # 1. 尝试删除已有的旧平面（方便你重复运行脚本调密度）
    try:
        rospy.wait_for_service('/gazebo/delete_model', timeout=1.0)
        delete_model = rospy.ServiceProxy('/gazebo/delete_model', DeleteModel)
        delete_model(model_name)
    except:
        pass # 如果模型不存在就跳过

    # 2. 生成新平面
    rospy.wait_for_service('/gazebo/spawn_sdf_model')
    try:
        spawn_sdf = rospy.ServiceProxy('/gazebo/spawn_sdf_model', SpawnModel)
        
        # 计算生成位置
        # 由于你的要求是覆盖 0 到 0.75，而 Gazebo 生成时是以模型中心为准的
        # 所以我们需要把它的中心放在一半的位置 (0.375, 0.375)
        pose = Pose()
        pose.position.x = size_x / 2.0  
        pose.position.y = size_y / 2.0  
        pose.position.z = 0.001         # 紧贴地面
        pose.orientation.w = 1.0        # 默认方向

        # 获取生成的 SDF 字符串
        sdf_xml = create_grid_plane_sdf(size_x, size_y, grid_spacing)
        
        status_msg = spawn_sdf(model_name, sdf_xml, "", pose, "world")
        if status_msg.success:
            rospy.loginfo(f"网格平面生成成功! 范围: x(0~{size_x}), y(0~{size_y}), 网格间距: {grid_spacing}m")
            rospy.loginfo("如果想要调整网格密度，可以直接修改代码中的 grid_spacing 变量并重新运行该脚本。")
        else:
            rospy.logerr(f"生成失败: {status_msg.status_message}")

    except rospy.ServiceException as e:
        rospy.logerr(f"Gazebo 服务调用失败: {e}")

if __name__ == '__main__':
    try:
        spawn_plane()
    except rospy.ROSInterruptException:
        pass
```



## 纯定位测试指令

生成网格地图，然后点击指定位置后，视觉中心移动到指定位置

```
roslaunch ur_gazebo ur3e_d435i_robotiq_gazebo.launch

roslaunch ur3e_robotiq_moveit_config move_group.launch

rosrun ur3_grasping visual_centering.py
或者
# visual_centering_charuco.py加了手眼标定信息和手动纠正
rosrun ur3_grasping visual_centering_charuco.py

rosrun ur3_grasping spawn_practice_plane.py
```

## 抓取定位测试指令

生成红色小方块，利用grconvnet定位并抓取

```
roslaunch ur_gazebo ur3e_d435i_robotiq_gazebo.launch

roslaunch ur3e_robotiq_moveit_config move_group.launch

rosrun ur3_grasping spawn_blocks.py

LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libffi.so.7 rosrun ur3_grasping grconvnet_grasp_centering.py

```

# 真实机械臂

## real_realsense_720p.launch

```
<?xml version="1.0"?>
<launch>
  <include file="$(find realsense2_camera)/launch/rs_camera.launch">
    <arg name="enable_color" value="true"/>
    <arg name="enable_depth" value="true"/>
    <arg name="align_depth" value="true"/>

    <arg name="color_width" value="1280"/>
    <arg name="color_height" value="720"/>
    <arg name="color_fps" value="30"/>
    <arg name="depth_width" value="1280"/>
    <arg name="depth_height" value="720"/>
    <arg name="depth_fps" value="30"/>
  </include>
</launch>
```

## real_charuco_detector.py

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import rospy
import cv2
import numpy as np
import tf2_ros
from cv_bridge import CvBridge
from sensor_msgs.msg import Image, CameraInfo
from geometry_msgs.msg import TransformStamped
from tf.transformations import quaternion_from_matrix

class RealCharucoDetector:
    def __init__(self):
        rospy.init_node('real_charuco_detector')
        self.bridge = CvBridge()
        
        # ======= 严格匹配你已经打印好的物理标定板参数 =======
        self.squares_x = 7
        self.squares_y = 5
        self.square_length = 0.015  # 15 mm -> 0.015 米
        self.marker_length = 0.012  # 12 mm -> 0.012 米
        
        # 初始化 OpenCV 的 ChArUco 字典 (DICT_4X4_50)
        try:
            self.dictionary = cv2.aruco.Dictionary_get(cv2.aruco.DICT_4X4_50)
            self.board = cv2.aruco.CharucoBoard_create(
                self.squares_x, self.squares_y, self.square_length, self.marker_length, self.dictionary)
        except AttributeError:
            self.dictionary = cv2.aruco.getPredefinedDictionary(cv2.aruco.DICT_4X4_50)
            self.board = cv2.aruco.CharucoBoard(
                (self.squares_x, self.squares_y), self.square_length, self.marker_length, self.dictionary)

        self.camera_matrix = None
        self.dist_coeffs = None
        self.tf_broadcaster = tf2_ros.TransformBroadcaster()

        # 订阅真实相机的内置参数和原始彩色图像
        rospy.Subscriber("/camera/color/camera_info", CameraInfo, self.info_callback)
        rospy.Subscriber("/camera/color/image_raw", Image, self.image_callback)
        
        # 可视化结果发布
        self.image_pub = rospy.Publisher("/real_charuco_detector/result", Image, queue_size=1)
        rospy.loginfo("真实 ChArUco 检测节点已启动，等待 720P 相机流...")

    def info_callback(self, msg):
        # 实时动态读取真实相机的出厂/自带内参，绝不写死
        if self.camera_matrix is None:
            self.camera_matrix = np.array(msg.K).reshape(3, 3)
            self.dist_coeffs = np.array(msg.D)
            rospy.loginfo("成功加载真实相机内参矩阵！")

    def image_callback(self, msg):
        if self.camera_matrix is None:
            return

        cv_img = self.bridge.imgmsg_to_cv2(msg, "bgr8")
        
        # 1. 检测基础 ArUco 标记
        try:
            corners, ids, rejected = cv2.aruco.detectMarkers(cv_img, self.dictionary)
        except AttributeError:
            detectorParams = cv2.aruco.DetectorParameters()
            detector = cv2.aruco.ArucoDetector(self.dictionary, detectorParams)
            corners, ids, rejected = detector.detectMarkers(cv_img)

        if ids is not None and len(ids) > 0:
            # 2. 插值出高精度的棋盘格黑白交叉内角点
            retval, charuco_corners, charuco_ids = cv2.aruco.interpolateCornersCharuco(
                corners, ids, cv_img, self.board)
            
            # 限速打印，防止 30Hz 刷屏终端
            rospy.loginfo_throttle(0.5, "当前成功提取高质量角点数量: %d / %d", retval, (self.squares_x - 1) * (self.squares_y - 1))
            
            # 至少需要 4 个内角点才能解算 PnP 位姿
            if retval > 3:
                # 绘制绿色角点
                cv2.aruco.drawDetectedCornersCharuco(cv_img, charuco_corners, charuco_ids, (0, 255, 0))
                
                # 3. 结合真实内参解算标定板在相机系下的 3D 位姿
                valid, rvec, tvec = cv2.aruco.estimatePoseCharucoBoard(
                    charuco_corners, charuco_ids, self.board, self.camera_matrix, self.dist_coeffs, None, None)
                
                if valid:
                    # 画出 5 厘米长的 3D 坐标轴方便肉眼评估稳定性
                    cv2.drawFrameAxes(cv_img, self.camera_matrix, self.dist_coeffs, rvec, tvec, 0.05)
                    self.publish_tf(rvec, tvec, msg.header.stamp)
        else:
            rospy.logwarn_throttle(2.0, "视野内未发现标定板，请调整相机视野。")

        self.image_pub.publish(self.bridge.cv2_to_imgmsg(cv_img, "bgr8"))

    def publish_tf(self, rvec, tvec, stamp):
        t = TransformStamped()
        t.header.stamp = stamp
        t.header.frame_id = "camera_color_optical_frame"
        t.child_frame_id = "charuco_board_frame" # 必须与 easy_handeye 配置严格一致
        
        t.transform.translation.x = tvec[0][0]
        t.transform.translation.y = tvec[1][0]
        t.transform.translation.z = tvec[2][0]
        
        rmat, _ = cv2.Rodrigues(rvec)
        T = np.eye(4)
        T[:3, :3] = rmat
        q = quaternion_from_matrix(T)
        
        t.transform.rotation.x = q[0]
        t.transform.rotation.y = q[1]
        t.transform.rotation.z = q[2]
        t.transform.rotation.w = q[3]
        
        self.tf_broadcaster.sendTransform(t)

if __name__ == '__main__':
    try:
        RealCharucoDetector()
        rospy.spin()
    except rospy.ROSInterruptException:
        pass
```

## real_charuco_calibration.launch

```
<?xml version="1.0"?>
<launch>
  <node pkg="ur3_grasping" type="real_charuco_detector.py" name="real_charuco_detector" output="screen" />

  <include file="$(find easy_handeye)/launch/calibrate.launch">
    <arg name="eye_on_hand"             value="true"/> 
    <arg name="namespace_prefix"        value="ur3e_realsense_real_charuco"/>
    
    <arg name="robot_base_frame"        value="base_link"/> 
    <arg name="robot_effector_frame"    value="tool0"/> 
    <arg name="tracking_base_frame"     value="camera_color_optical_frame"/> 
    <arg name="tracking_marker_frame"   value="charuco_board_frame"/>
    
    <arg name="freehand_robot_movement" value="true" /> 
    <arg name="move_group"              value="manipulator"/> 
  </include>
</launch>
```

## 真实机械臂标定指令

```
# 终端 1：启动真实 UR 机械臂驱动
roslaunch ur_robot_driver ur3e_bringup.launch robot_ip:=192.168.0.234 kinematics_config:=/home/wjj/.ros/robot_calibration.yaml
# （此时在示教器上运行 external_control 程序）

# 终端 2：启动真实相机的 720P 数据流
roslaunch ur3_grasping real_realsense_720p.launch

# 终端 3：【新增】启动 MoveIt 规划执行
roslaunch ur3e_moveit_config moveit_planning_execution.launch

# 终端 4：启动手眼标定与 ChArUco 识别算法
roslaunch ur3_grasping real_charuco_calibration.launch

# 终端 5：打开视觉监控，检查角点提取质量
rqt_image_view
```

标定结果1：

```
translation: 

  x: -0.011770460741427898

  y: 0.05682209510448768

  z: 0.03434709411625774

rotation: 

  x: 0.002670524858811799

  y: 0.010239985867424806

  z: -0.9999343858179239

  w: 0.004385777621453845
  
```

```
        self.calib_x  = -0.015023628963959668
        self.calib_y  =  0.05162170975609621
        self.calib_z  =  0.03516081375972209
        self.calib_qx = 0.002670524858811799
        self.calib_qy =  0.010239985867424806
        self.calib_qz = -0.9999343858179239
        self.calib_qw =  0.004385777621453845

```



## 真实机械臂纯定位测试指令

```
# 1. 启动真实 UR 机械臂驱动与 MoveIt (需在示教器运行 external_control)
roslaunch ur_robot_driver ur3e_bringup.launch robot_ip:=192.168.0.234 kinematics_config:=/home/wjj/.ros/robot_calibration.yaml

# 2
roslaunch ur3e_moveit_config moveit_planning_execution.launch

# 3. 启动真实相机
roslaunch ur3_grasping real_realsense_720p.launch

# 4
rosrun ur3_grasping real_visual_centering.py
```

