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

# 仿真相关

## 添加相机模型

```
cd ~/ros_ur3/src
git clone -b ros1-legacy https://github.com/IntelRealSense/realsense-ros.git
```

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
    <origin xyz="0 -0.04 0.01" rpy="0 -1.5708 1.5708"/> 
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
  <arg name="initial_joint_positions" default=" -J shoulder_pan_joint 0.0 -J shoulder_lift_joint -1.5708 -J elbow_joint 1.5708 -J wrist_1_joint -1.5708 -J wrist_2_joint -1.5708 -J wrist_3_joint 0.0" doc="Initial joint configuration of the robot"/>

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

## 生成物体脚本

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

## 生成Aruco用于手眼标定

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
        pose.position.x = 0.4
        pose.position.y = 0.125
        pose.position.z = 0.001
        pose.orientation = Quaternion(0, 0, 0, 1)

        # 稍微改一下模型名字防止和刚才加载失败的模型在 Gazebo 缓存里冲突
        status_msg = spawn_sdf("aruco_marker_id26_fixed", model_xml, "", pose, "world")
        if status_msg.success:
            rospy.loginfo("成功生成 ArUco 标定板！")
    except rospy.ServiceException as e:
        rospy.logerr(f"生成失败: {e}")
```

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

## 标定流程

```
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

## 检验标定数据

可以看到误差非常小，注意camera_bottom_screw_frame和camera_color_optical_frame不一样！

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

## 添加机械夹爪

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
    <origin xyz="0 -0.04 0.01" rpy="0 -1.5708 1.5708"/> 
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

robotiq_2f_85_gripper_controllers.yaml

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



## 仿真抓取

使用抓取插件，方便仿真时模拟抓取

```
cd ~/ros_ur3/src
# 克隆 gazebo-pkgs 仓库，其中包含了 gazebo_grasp_fix 插件
git clone https://github.com/JenniferBuehler/gazebo-pkgs.git

git clone https://github.com/JenniferBuehler/general-message-pkgs.git

catkin build
```

test_grasp.py

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import rospy
import moveit_commander
from geometry_msgs.msg import Pose
import sys

class ManualGrasper:
    def __init__(self):
        moveit_commander.roscpp_initialize(sys.argv)
        rospy.init_node('manual_grasp_test')

        self.arm = moveit_commander.MoveGroupCommander("manipulator")
        self.gripper = moveit_commander.MoveGroupCommander("gripper")
        
        # 允许更小的运动容差
        self.arm.set_goal_position_tolerance(0.001)
        self.arm.set_max_velocity_scaling_factor(0.3)
        self.arm.set_max_acceleration_scaling_factor(0.3)
        
        print("--- 机械臂微调测试工具 ---")
        print("控制: w/s (x), a/d (y), q/e (z)")
        print("步进调节: z (变大), x (变小)")
        print("夹爪: g (闭合), o (打开)")
        print("退出: Esc")

    def move_rel(self, dx=0, dy=0, dz=0):
        current_pose = self.arm.get_current_pose().pose
        target_pose = current_pose
        target_pose.position.x += dx
        target_pose.position.y += dy
        target_pose.position.z += dz
        
        self.arm.set_pose_target(target_pose)
        # 使用 plan() + execute() 往往比 go() 在微调时更稳定
        success = self.arm.go(wait=True)
        return success

    def control_gripper(self, value):
        self.gripper.set_joint_value_target([value])
        self.gripper.go(wait=True)
        print(">> 夹爪指令已发送: %.2f" % value)

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

        step = 0.01 # 初始步进设为 1cm
        while not rospy.is_shutdown():
            print("\r当前步进: %.3f m | 准备就绪..." % step, end="")
            key = getch()
            
            if key == 'w': self.move_rel(dx=step)
            elif key == 's': self.move_rel(dx=-step)
            elif key == 'a': self.move_rel(dy=step)
            elif key == 'd': self.move_rel(dy=-step)
            elif key == 'q': self.move_rel(dz=step)
            elif key == 'e': self.move_rel(dz=-step)
            elif key == 'z': step = min(0.1, step + 0.005) # 调大步进
            elif key == 'x': step = max(0.001, step - 0.005) # 调小步进至 1mm
            elif key == 'g': self.control_gripper(0.5)
            elif key == 'o': self.control_gripper(0.0)
            elif key == '\x1b': break

if __name__ == '__main__':
    try:
        tester = ManualGrasper()
        tester.run()
    except rospy.ROSInterruptException:
        pass
```

测试指令

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



block_manager.py

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

grconvnet_vision_test.py

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import rospy
import sys
import os
import cv2
import numpy as np
import threading
import time
from std_msgs.msg import String

# 兼容性补丁
np.float = float
np.int = int
np.bool = bool
np.object = object

import torch
import message_filters
from sensor_msgs.msg import Image
import moveit_commander
from geometry_msgs.msg import Pose, Quaternion
from tf.transformations import quaternion_matrix, quaternion_from_euler

# 导入 GR-ConvNet
GRCONVNET_PATH = '/home/wjj/GR-ConvNet/robotic-grasping' 
sys.path.append(GRCONVNET_PATH)
from inference.post_process import post_process_output
from utils.dataset_processing.grasp import detect_grasps

class DeepLearningGrasper:
    def __init__(self):
        moveit_commander.roscpp_initialize(sys.argv)
        rospy.init_node('dl_auto_grasp_node', anonymous=True)

        self.arm = moveit_commander.MoveGroupCommander("manipulator")
        self.gripper = moveit_commander.MoveGroupCommander("gripper")
        self.status_pub = rospy.Publisher("/grasp_status", String, queue_size=1)

        self.arm.set_max_velocity_scaling_factor(0.5)
        self.home_joints = [0.0, -1.5708, 1.5708, -1.5708, -1.5708, 0.0]
        self.drop_joints = [1.5708, -1.5708, 1.5708, -1.5708, -1.5708, 0.0]

        # 模型加载
        self.device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
        self.model = torch.load(os.path.join(GRCONVNET_PATH, 'trained-models/jacquard-rgbd-grconvnet3-drop0-ch32/epoch_42_iou_0.93'), map_location=self.device)
        self.model.eval()

        self.img_lock = threading.Lock()
        self.raw_rgb = None
        self.raw_depth = None
        self.latest_display = None
        self.best_grasp = None
        self.is_moving = False 

        # 订阅
        color_sub = message_filters.Subscriber("/camera/color/image_raw", Image)
        depth_sub = message_filters.Subscriber("/camera/depth/image_raw", Image)
        self.ts = message_filters.ApproximateTimeSynchronizer([color_sub, depth_sub], 10, 0.1)
        self.ts.registerCallback(self.image_callback)

        # 启动后台线程
        threading.Thread(target=self.ai_loop, daemon=True).start()
        threading.Thread(target=self.robot_loop, daemon=True).start()

    def image_callback(self, color_msg, depth_msg):
        try:
            rgb = np.frombuffer(color_msg.data, dtype=np.uint8).reshape(color_msg.height, color_msg.width, 3).copy()
            if color_msg.encoding != "bgr8": rgb = rgb[:, :, ::-1]
            
            if depth_msg.encoding in ["32FC1", "32fc1"]:
                depth = np.frombuffer(depth_msg.data, dtype=np.float32).reshape(depth_msg.height, depth_msg.width).copy()
            else:
                depth = np.frombuffer(depth_msg.data, dtype=np.uint16).reshape(depth_msg.height, depth_msg.width).copy()

            with self.img_lock:
                self.raw_rgb = rgb
                self.raw_depth = depth
        except: pass

    def ai_loop(self):
        """三窗显示逻辑"""
        # 预存中间和右侧窗口的缓存，防止移动时画面变黑
        last_predict_view = None
        last_heatmap_view = None

        while not rospy.is_shutdown():
            with self.img_lock:
                if self.raw_rgb is None: 
                    time.sleep(0.1)
                    continue
                img_c = self.raw_rgb.copy()
                img_d = self.raw_depth.copy()

            # 1. 裁剪原始画面（最左侧窗）
            h, w = img_c.shape[:2]
            crop_size = min(h, w)
            left, top = (w - crop_size) // 2, (h - crop_size) // 2
            raw_live = img_c[top:top+crop_size, left:left+crop_size].copy()
            cv2.putText(raw_live, "LIVE CAMERA", (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 0.8, (255, 255, 255), 2)

            # 2. 如果机械臂在移动，跳过推理，使用缓存
            if self.is_moving:
                cv2.putText(raw_live, "STATUS: MOVING", (10, 60), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 0, 255), 2)
                mid_view = last_predict_view if last_predict_view is not None else np.zeros_like(raw_live)
                right_view = last_heatmap_view if last_heatmap_view is not None else np.zeros_like(raw_live)
            else:
                # 3. 正常执行推理
                depth_crop = img_d[top:top+crop_size, left:left+crop_size]
                rgb_224 = cv2.resize(raw_live, (224, 224))
                depth_224 = cv2.resize(depth_crop, (224, 224))
                
                d_norm = np.clip(depth_224 - np.mean(depth_224), -1, 1)
                r_norm = (cv2.cvtColor(rgb_224, cv2.COLOR_BGR2RGB).astype(np.float32)/255.0 - 0.5) / 0.5
                input_tensor = torch.from_numpy(np.concatenate((np.expand_dims(d_norm,0), r_norm.transpose(2,0,1)), axis=0)).unsqueeze(0).float().to(self.device)

                with torch.no_grad():
                    p, c, s, w_p = self.model(input_tensor)
                
                q_img, ang_img, width_img = post_process_output(p, c, s, w_p)
                grasps = detect_grasps(q_img, ang_img, width_img=width_img, no_grasps=1)

                # 渲染中间窗（预测位姿）
                mid_view = raw_live.copy()
                cv2.putText(mid_view, "PREDICTION", (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 0.8, (0, 255, 0), 2)
                
                # 渲染右侧窗（热图）
                heatmap = cv2.applyColorMap(cv2.normalize(q_img, None, 0, 255, cv2.NORM_MINMAX, cv2.CV_8U), cv2.COLORMAP_JET)
                right_view = cv2.resize(heatmap, (crop_size, crop_size))
                cv2.putText(right_view, "GRASP QUALITY (Q)", (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 0.8, (255, 255, 255), 2)

                if len(grasps) > 0:
                    g = grasps[0]
                    prob = q_img[int(g.center[0]), int(g.center[1])]
                    scale = crop_size / 224.0
                    self.best_grasp = {
                        'u': int(g.center[1] * scale + left),
                        'v': int(g.center[0] * scale + top),
                        'angle': g.angle + 1.5708, 'prob': prob
                    }
                    pts = (g.as_gr.points * scale).astype(np.int32)
                    cv2.drawContours(mid_view, [pts[:, [1, 0]]], 0, (0, 0, 255), 3)
                
                # 更新缓存
                last_predict_view = mid_view.copy()
                last_heatmap_view = right_view.copy()

            # 4. 三窗横向大拼接
            with self.img_lock:
                self.latest_display = np.hstack((raw_live, mid_view, right_view))
            
            time.sleep(0.01)

    def robot_loop(self):
        self.is_moving = True
        self.arm.set_joint_value_target(self.home_joints)
        self.arm.go(wait=True)
        self.is_moving = False

        while not rospy.is_shutdown():
            if self.best_grasp and self.best_grasp['prob'] > 0.5 and not self.is_moving:
                target = self.best_grasp.copy()
                self.is_moving = True 
                self.execute_grasp(target)
                self.best_grasp = None
                self.is_moving = False 
            time.sleep(0.1)

    def execute_grasp(self, target):
        tx, ty, tz = self.image_to_world(target['u'], target['v'])
        q = quaternion_from_euler(3.14159, 0, target['angle'])
        p = Pose()
        
        # 1. 移动到物体正上方的悬停位置 (目前是基于图像推算高度 + 0.3米)
        p.position.x, p.position.y, p.position.z = tx, ty, tz + 0.3
        p.orientation = Quaternion(*q)

        self.arm.set_pose_target(p)
        if self.arm.go(wait=True):
            # ---------------- 修改的核心部分 ----------------
            # 2. 改为固定向下一定距离 (这里以向下移动 0.25 米为例)
            # 你可以根据实际的夹爪长度和摄像机安装高度修改 down_distance 的值
            down_distance = 0.17
            p.position.z -= down_distance
            # ------------------------------------------------
            
            self.arm.set_pose_target(p)
            self.arm.go(wait=True)
            
            # 3. 闭合夹爪
            self.gripper.set_joint_value_target([0.5])
            self.gripper.go(wait=True)
            
            # 4. 抓取后向上抬起 (相对抬起 0.15 米)
            p.position.z += 0.15
            self.arm.set_pose_target(p)
            self.arm.go(wait=True)
            
            # 5. 移动到放置区
            self.arm.set_joint_value_target(self.drop_joints)
            self.arm.go(wait=True)
            
            # 6. 张开夹爪，放下物体
            self.gripper.set_joint_value_target([0.0])
            self.gripper.go(wait=True)
            
            # 7. 发送信号给方块管理器并复位
            rospy.loginfo(">> 发送刷新信号...")
            self.status_pub.publish("dropped")
            self.arm.set_joint_value_target(self.home_joints)
            self.arm.go(wait=True)

    def image_to_world(self, u, v):
        fx, fy, cx, cy = 462.137, 462.137, 320.5, 240.5
        trans, quat = [0.350, 0.163, 0.278], [-0.706, 0.708, 0.006, -0.006]
        ray_cam = np.array([(u - cx) / fx, (v - cy) / fy, 1.0, 0.0])
        ray_world = np.dot(quaternion_matrix(quat), ray_cam)
        target_z = 0.03
        t = (target_z - trans[2]) / ray_world[2]
        return trans[0] + t * ray_world[0], trans[1] + t * ray_world[1], target_z

    def run_ui(self):
        cv2.namedWindow("GR-ConvNet Integrated Vision", cv2.WINDOW_NORMAL)
        while not rospy.is_shutdown():
            if self.latest_display is not None:
                with self.img_lock:
                    show_img = self.latest_display.copy()
                cv2.imshow("GR-ConvNet Integrated Vision", show_img)
            if cv2.waitKey(1) & 0xFF == ord('q'):
                break

if __name__ == '__main__':
    try:
        grasper = DeepLearningGrasper()
        grasper.run_ui()
    except rospy.ROSInterruptException:
        pass
    finally:
        cv2.destroyAllWindows()
```

使用grconvnet进行抓取

```
roslaunch ur_gazebo ur3e_d435i_robotiq_gazebo.launch
roslaunch ur3e_robotiq_moveit_config move_group.launch
rosrun ur3_grasping block_manager.py 

(GraspSAM) wjj@wjj-pc:~$ LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libffi.so.7 rosrun ur3_grasping grconvnet_vision_test.py

```

