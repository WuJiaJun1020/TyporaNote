# 1.easy_handeye

手眼标定

# 2.realsense-ros

相机包

# 3.robotiq（夹爪模型包）

## robotiq_2f_85_gripper_gazebo

### config

#### robotiq_2f_85_gripper_controllers.yaml

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

### launch

#### robotiq_2f_85_bringup.launch

```
<?xml version="1.0"?>
<launch>
  <!--
    Main entry point for loading a single Robotiq 2F 85 Gripper into Gazebo, in isolation, in the
    empty world.
  -->

  <!--Robot description and related parameter files -->
  <arg name="robot_description_file" default="$(dirname)/inc/load_robotiq_2f_85_gripper.launch" doc="Launch file which populates the 'robot_description' parameter."/>

  <!-- Controller configuration -->
  <arg name="controller_config_file" default="$(find robotiq_2f_85_gripper_gazebo)/config/robotiq_2f_85_gripper_controllers.yaml" doc="Config file used for defining the ROS-Control controllers."/>
  <arg name="controllers" default="joint_state_controller gripper_controller" doc="Controllers that are activated by default."/>

  <!-- robot_state_publisher configuration -->
  <arg name="tf_prefix" default="" doc="tf_prefix used for the robot."/>
  <arg name="tf_pub_rate" default="125" doc="Rate at which robot_state_publisher should publish transforms."/>

  <!-- Gazebo parameters -->
  <arg name="paused" default="false" doc="Starts Gazebo in paused mode" />
  <arg name="gui" default="true" doc="Starts Gazebo gui" />
  <arg name="spawn_z" default="0.1" doc="At which height the model should be spawned. NOTE: lower values will cause the robot to collide with the ground plane." />

  <!-- Load urdf on the parameter server -->
  <include file="$(arg robot_description_file)">
  </include>

  <!-- Robot state publisher -->
  <node pkg="robot_state_publisher" type="robot_state_publisher" name="robot_state_publisher">
    <param name="publish_frequency" type="double" value="$(arg tf_pub_rate)" />
    <param name="tf_prefix" value="$(arg tf_prefix)" />
  </node>

  <!-- Start the 'driver' (ie: Gazebo in this case) -->
  <include file="$(dirname)/inc/robotiq_2f_85_gripper_control.launch">
    <arg name="controller_config_file" value="$(arg controller_config_file)"/>
    <arg name="controllers" value="$(arg controllers)"/>
    <arg name="gui" value="$(arg gui)"/>
    <arg name="paused" value="$(arg paused)"/>
    <arg name="spawn_z" value="$(arg spawn_z)"/>
  </include>
</launch>

```

#### inc

load_robotiq_2f_85_gripper.launch

```
<?xml version="1.0"?>
<launch>
  <arg name="transmission_hw_interface" default="hardware_interface/EffortJointInterface" />
  
  <param name="robot_description" command="$(find xacro)/xacro '$(find robotiq_2f_85_gripper_gazebo)/urdf/robotiq_arg2f_85.xacro'
  transmission_hw_interface:=$(arg transmission_hw_interface)" 
  />  
</launch>
```

robotiq_2f_85_gripper_control.launch

```
<?xml version="1.0"?>
<launch>
  <!--
    This file 'pretends' to load a driver for a Robotiq 2F 85 Gripper, by accepting similar
    arguments and playing a similar role (ie: starting the driver node (in this
    case Gazebo) and loading the ros_control controllers).

    Some of the arguments to this .launch file will be familiar to those using
    the ur_robot_driver with their robot.

    Other parameters are specific to Gazebo.

    Note: we spawn and start the ros_control controllers here, as they are,
    together with gazebo_ros_control, essentially the replacement for the
    driver which would be used with a real robot.
  -->

  <!-- Parameters  -->
  <arg name="controller_config_file" doc="Config file used for defining the ROS-Control controllers."/>
  <arg name="controllers" default="joint_state_controller gripper_controller"/>

  <!-- Gazebo parameters -->
  <arg name="gazebo_model_name" default="robot" doc="The name to give to the model in Gazebo (after spawning it)." />
  <arg name="gazebo_world" default="worlds/empty.world" doc="The '.world' file to load in Gazebo." />
  <arg name="gui" default="true" doc="If true, Gazebo UI is started. If false, only start Gazebo server." />
  <arg name="paused" default="false" doc="If true, start Gazebo in paused mode. If false, start simulation as soon as Gazebo has loaded." />
  <arg name="robot_description_param_name" default="robot_description" doc="Name of the parameter which contains the robot description (ie: URDF) which should be spawned into Gazebo." />
  <arg name="spawn_z" default="0.1" doc="At which height the model should be spawned. NOTE: lower values will cause the robot to collide with the ground plane." />
  <arg name="start_gazebo" default="true" doc="If true, Gazebo will be started. If false, Gazebo will be assumed to have been started elsewhere." />

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
      -z $(arg spawn_z)"
    output="screen" respawn="false" />

  <!-- Load and start the controllers listed in the 'controllers' arg. -->
  <node name="ros_control_controller_spawner" pkg="controller_manager" type="spawner"
    args="$(arg controllers)" output="screen" respawn="false" />

</launch>
```



### urdf

#### robotiq_arg2f_85.xacro

```
<?xml version="1.0"?>
<robot name="robotiq_arg2f_85_gazebo" xmlns:xacro="http://ros.org/wiki/xacro">
  <!--
    This is a top-level xacro instantiating the Gazebo-specific version of the
    'robotiq_arg2f_85_gripper' macro (ie: 'robotiq_arg2f_85_gripper_gazebo') and passing it values for all its
    required arguments.

    This file should be considered the Gazebo-specific variant of the file
    with the same name in the robotiq_2f_85_gripper_visualization package. It accepts the same
    arguments, but instead of configuring everything for a real robot, will
    generate a Gazebo-compatible URDF with a ros_control hardware_interface
    attached to it.

    Only use this top-level xacro if you plan on spawning the gripper in Gazebo
    'by itself', without any gripper or any other geometry attached.

    If you need to attach it to a robot and integrate it into a larger workcell and want to spawn that as a single entity in
    Gazebo, DO NOT EDIT THIS FILE.

    Instead: create a new top-level xacro, give it a proper name, include the
    required '.xacro' files, instantiate the models (ie: call the macros) and
    connect everything by adding the appropriate joints.
  -->

  <!--
    Import main macro.

    NOTE: this imports the Gazebo-wrapper main macro, NOT the regular
          xacro macro (which is hosted by robotiq_2f_85_gripper_visualization).
  -->
   <!-- import main macro -->
  <xacro:include filename="$(find robotiq_2f_85_gripper_gazebo)/urdf/robotiq_arg2f_85_macro.xacro" />

  <!-- parameters -->
  <xacro:arg name="transmission_hw_interface" default="hardware_interface/EffortJointInterface"/>
  <!--
    legal values:
      - hardware_interface/PositionJointInterface
      - hardware_interface/EffortJointInterface

    NOTE: this value must correspond to the controller configured in the
          controller .yaml files in the 'config' directory.
  -->

  <!-- gripper -->
  <xacro:robotiq_arg2f_85_gazebo 
    prefix=""
    transmission_hw_interface="$(arg transmission_hw_interface)"/>

  <!--
    Attach the Gazebo model to Gazebo's world frame.
  -->
  <link name="world"/>
  <joint name="world_joint" type="fixed">
    <parent link="world"/>
    <child link="base_link"/>
    <origin xyz="0 0 0" rpy="0 0 0"/>
  </joint>
</robot>

```

#### robotiq_arg2f_85_macro.xacro

```
<?xml version="1.0"?>
<robot xmlns:xacro="http://ros.org/wiki/xacro">

  <!--
    Main xacro macro definition of the "Gazebo robot" model.

    This wraps the model of the real robot and adds all elements and parameters
    required by Gazebo.

    It also adds the gazebo_ros_control plugin.

    NOTE: this is NOT a URDF. It cannot directly be loaded by consumers
    expecting a flattened '.urdf' file. See the top-level '.xacro' for that
    (but note: that .xacro must still be processed by the xacro command).

  -->


  <!-- Definition of the main macro -->
  <xacro:macro name="robotiq_arg2f_85_gazebo" params="
    prefix 
    transmission_hw_interface:=hardware_interface/EffortJointInterface"
  >

  <!--
    Import the xacro macro for the REAL gripper (which we'll augment with Gazebo
    specific elements in the wrapper macro below).

    NOTE: this imports the '_macro.xacro' from robotiq_2f_85_gripper_visualization, as that contains
    the definitions for the real gripper.
  -->
  <xacro:include filename="$(find robotiq_2f_85_gripper_visualization)/urdf/robotiq_arg2f_85_macro.xacro"/>


  <!-- Instantiate model for the REAL robot. -->
  <xacro:robotiq_arg2f_85 
  prefix="${prefix}"
  transmission_hw_interface="${transmission_hw_interface}"/>

  <!-- Mimic joints plugin-->
    <gazebo>
      <plugin filename="libroboticsgroup_upatras_gazebo_mimic_joint_plugin.so" name="${prefix}mimic_robotiq_85_1">
        <joint>${prefix}finger_joint</joint>
        <mimicJoint>${prefix}left_inner_knuckle_joint</mimicJoint>
        <multiplier>1.0</multiplier>
      </plugin>
      <plugin filename="libroboticsgroup_upatras_gazebo_mimic_joint_plugin.so" name="${prefix}mimic_robotiq_85_2">
        <joint>${prefix}finger_joint</joint>
        <mimicJoint>${prefix}left_inner_finger_joint</mimicJoint>
        <multiplier>-1.0</multiplier>
      </plugin>
      <plugin filename="libroboticsgroup_upatras_gazebo_mimic_joint_plugin.so" name="${prefix}mimic_robotiq_85_3">
        <joint>${prefix}finger_joint</joint>
        <mimicJoint>${prefix}right_inner_knuckle_joint</mimicJoint>
        <multiplier>1.0</multiplier>
      </plugin>
      <plugin filename="libroboticsgroup_upatras_gazebo_mimic_joint_plugin.so" name="${prefix}mimic_robotiq_85_4">
        <joint>${prefix}finger_joint</joint>
        <mimicJoint>${prefix}right_inner_finger_joint</mimicJoint>
        <multiplier>-1.0</multiplier>
      </plugin>
      <plugin filename="libroboticsgroup_upatras_gazebo_mimic_joint_plugin.so" name="${prefix}mimic_robotiq_85_5">
        <joint>${prefix}finger_joint</joint>
        <mimicJoint>${prefix}right_finger_joint</mimicJoint>
        <multiplier>1.0</multiplier>
      </plugin>
    </gazebo>

    <!-- Links colors  -->
    <gazebo reference="${prefix}robotiq_arg2f_base_link">
        <material>Gazebo/Black</material>
    </gazebo>
    <gazebo reference="${prefix}left_outer_knuckle">
        <material>Gazebo/Grey</material>
    </gazebo>
    <gazebo reference="${prefix}left_outer_finger">
        <material>Gazebo/Black</material>
    </gazebo>
    <gazebo reference="${prefix}left_inner_finger">
        <material>Gazebo/Grey</material>
    </gazebo>
    <gazebo reference="${prefix}left_inner_knuckle">
        <material>Gazebo/Black</material>
    </gazebo>
    <gazebo reference="${prefix}right_outer_knuckle">
        <material>Gazebo/Grey</material>
    </gazebo>
    <gazebo reference="${prefix}right_outer_finger">
        <material>Gazebo/Black</material>
    </gazebo>
    <gazebo reference="${prefix}right_inner_finger">
        <material>Gazebo/Grey</material>
    </gazebo>
    <gazebo reference="${prefix}right_inner_knuckle">
        <material>Gazebo/Black</material>
    </gazebo>

    <!--
      Inject Gazebo ROS Control plugin, which allows us to use ros_control
      controllers to control the virtual robot hw.
    -->
    <!-- <gazebo>
      <plugin name="ros_control" filename="libgazebo_ros_control.so">
      </plugin>
    </gazebo> -->
    
  </xacro:macro>
</robot>

```

# 4.universal_robot-noetic-devel

# 5.Universal_Robots_Client_Library-master

# 6.Universal_Robots_ROS_Driver

4,5,6为ur机械臂的三个官方包

# 7.ur3e_robotiq_moveit_config（我创建的带机械臂和夹爪的moveit包）



# 8.ur3_grasping（仿真、真实机械臂抓取相关代码）

## scripts

### 