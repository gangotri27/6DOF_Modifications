# 6DOF_Modifications

---

## open_manipulator_bringup

### /config/open_manipulator_x/hardware_controller_manager.yaml

**Change:** Added joint5_roll to joints
```yaml
arm_controller:
    ros__parameters:
      joints:
        - joint1
        - joint2
        - joint3
        - joint4
        - joint5_roll
```
**Explanation:** Included the new joint5_roll in the list of joints that the hardware controller manages. This makes sure the wrist roll joint is now recognized and controlled by the hardware controller.

### /config/open_manipulator_x/initial_positions.yaml	

**Change:** Added joint5_roll to joint_names and modified home position
```yaml
  ros__parameters:
    joint_names:
      - joint1
      - joint2
      - joint3
      - joint4
      - joint5_roll

    step_names: [home]  # Names of the steps

    home: [0.0, -1.0, 1.0, 0.0, 0.0]  # Home position
```
**Explanation:** Appended joint5_roll to the joint_names array and adjusted the home position values accordingly. This ensures the default (home) pose includes the new wrist roll joint in the initial configuration.

### /launch/open_manipulator_x.launch.py	

**Change:** Added parameterValue by importing it
```py
from launch_ros.parameter_descriptions import ParameterValue

# control_node
parameters=[
            {'robot_description': ParameterValue(urdf_file, value_type=str)},
            controller_manager_config
        ],

# robot_controller_spawner
parameters=[{'robot_description': ParameterValue(urdf_file, value_type=str)}],

# robot_state_publisher_node
parameters=[
            {'robot_description': ParameterValue(urdf_file, value_type=str)},
            {'use_sim_time': use_sim}
        ],
```
**Explanation:** Imported ParameterValue into the launch file so that parameters (like the robot description) are passed as explicit typed strings. In practice, this forces the Xacro-generated URDF to be fully evaluated into a string before sending it to nodes (e.g., robot_state_publisher). This avoids silent failures or type ambiguities when loading the robot model at runtime.

---

## open_manipulator_description

### /gazebo/open_manipulator_x.gazebo.xacro	

**Change:** Added link reference and trans for link 5 with joint 5 (to actuator 5)
```xacro
<xacro:Link reference="${prefix}wrist_roll_link"/>

<xacro:SimpleTransmission trans="${prefix}trans_roll" joint="${prefix}joint5_roll" actuator="${prefix}actuator_roll" />
```
**Explanation:** Added a <link_reference> and a <transmission> for the new wrist joint (link 5) in the Gazebo configuration. The link reference provides Gazebo with the physical properties (mass, inertia, etc.) of the new link for stable simulation, and the transmission block tells ROS how to actuate this joint. (Whenever a new joint is added, both a LinkReference and a SimpleTransmission are needed so that Gazebo can simulate physics correctly and ROS can control the joint.)

### /meshes/open_manipulator_x	

**Change:** Added .stl files of gripper_link_with_bracket and wrist_link

### /ros2_control/open_manipulator_x_position.ros2_control.xacro	
**1) Change:** Changed the number of joints and transmissions
```xacro
<param name="number_of_joints">6</param>
<param name="number_of_transmissions">6</param>
```
**Explanation:** Updated the ROS 2 control configuration to account for six joints instead of five (and similarly six transmissions). Since a sixth joint (joint5_roll) was added, these counts were incremented from 5 to 6 so that the control system knows how many joints and transmissions to expect.

**2) Change:** Added joint to transmission and transmission to joint matrices
```xacro
<param name="transmission_to_joint_matrix">
              1, 0, 0, 0, 0, 0,
              0, 1, 0, 0, 0, 0,
              0, 0, 1, 0, 0, 0,
              0, 0, 0, 1, 0, 0,
              0, 0, 0, 0, 1, 0,
              0, 0, 0, 0, 0, 1
</param>
<param name="joint_to_transmission_matrix">
              1, 0, 0, 0, 0, 0,
              0, 1, 0, 0, 0, 0,
              0, 0, 1, 0, 0, 0,
              0, 0, 0, 1, 0, 0,
              0, 0, 0, 0, 1, 0,
              0, 0, 0, 0, 0, 1
</param>
```
**Explanation:** Expanded the 5×5 mapping matrices to 6×6 identity matrices to include the new joint. This effectively maps each of the six transmissions to the correct joint and vice versa. In other words, the new matrices ensure that after adding joint5_roll, each transmission corresponds one-to-one with each joint in the correct order.

**3) Change:** Added a few params to dxl6 for prismatic motion
```xacro
<param name="use_revolute_to_prismatic_gripper">1</param>
<param name="revolute_to_prismatic_dxl">dxl6</param>
<param name="revolute_max">-1.52</param>
<param name="revolute_min">0.92</param>
<param name="revolute_to_prismatic_joint">${prefix}gripper_left_joint</param>
<param name="prismatic_max">0.019</param>
<param name="prismatic_min">-0.01</param>
```
**Explanation:** Configured parameters for the new motor (dxl6) to enable prismatic (linear) motion of the gripper. These parameters define how a revolute input (rotation) is converted to prismatic motion: which motor drives it (dxl6), which joint it affects (gripper_left_joint), and the corresponding limits (revolute_max/min and prismatic_max/min). This allows the sixth actuator to properly control the linear extension of the gripper.

**4) Change:** Added an extra joint block for joint5_roll
```xacro
<joint name="${prefix}joint5_roll">
        <command_interface name="position"/>
        <state_interface name="position"/>
        <state_interface name="velocity"/>
        <state_interface name="effort"/>
</joint>
```
**Explanation:** Added a new <joint> element named ${prefix}joint5_roll in the ROS 2 control configuration. This defines the control interfaces (command_interface name="position" and various state interfaces) for the wrist roll joint. In other words, it tells the ROS 2 control system how to command and read the position (and velocity/effort) of the new joint.

**5) Change:** Replaced dxl5 with dxl5 and dxl6 related gpio’s
```xacro
<gpio name="dxl5">
        <param name="type">dxl</param>
        <param name="ID">15</param>
        <command_interface name="Goal Position"/>
        <state_interface name="Present Position"/>
        <state_interface name="Present Velocity"/>
        <state_interface name="Present Current"/>
        <param name="Operating Mode">3</param>
        <param name="Position P Gain">800</param>
        <param name="Position I Gain">100</param>
        <param name="Position D Gain">100</param>
        <param name="Profile Velocity">20</param>
        <param name="Profile Acceleration">10</param>
        <param name="Drive Mode">0</param>
        <param name="Return Delay Time">0</param>
</gpio>
<gpio name="dxl6">
        <param name="type">dxl</param>
        <param name="ID">16</param>
        <command_interface name="Goal Position"/>
        <state_interface name="Present Position"/>
        <state_interface name="Present Velocity"/>
        <state_interface name="Present Current"/>
        <param name="Operating Mode">5</param>
        <param name="Profile Velocity">20</param>
        <param name="Profile Acceleration">10</param>
        <param name="Drive Mode">4</param>
        <param name="Return Delay Time">0</param>
        <param name="Goal Current">200</param>
</gpio>
```
**Explanation:** Updated the GPIO (motor) configuration blocks to handle both motors 5 and 6. The existing <gpio name="dxl5"> section (ID 15) was kept (with its parameters), and a new <gpio name="dxl6"> section (ID 16) was added for the extra motor. Each block specifies the same control interfaces and gains for its motor. In short, this duplicates the configuration for motor 5 and adds a corresponding one for motor 6, enabling control of the new actuator.

### /urdf/open_manipulator_x_arm.urdf.xacro	

**Change:** Modifications are made to be compatible with the extra joint by adding the required lines

---

## open_manipulator_teleop

### /open_manipulator_teleop/open_manipulator_x_teleop.py	

**Change:** Modified in such a way that joint 5 also moves using teleop

---

## open_manipulator_moveit_config

### /config/open_manipulator_x/open_manipulator_x.srdf	
**1) Change:** Added joint5_roll in the group name and group states    
```srdf
    <group name="arm">
        <joint name="joint1"/>
        <joint name="joint2"/>
        <joint name="joint3"/>
        <joint name="joint4"/>
        <joint name="joint5_roll"/>
        <joint name="end_effector_joint"/>
    </group>
    <group_state name="init" group="arm">
        <joint name="joint1" value="0"/>
        <joint name="joint2" value="0"/>
        <joint name="joint3" value="0"/>
        <joint name="joint4" value="0"/>
        <joint name="joint5_roll" value="0"/>
    </group_state>
    <group_state name="home" group="arm">
        <joint name="joint1" value="0"/>
        <joint name="joint2" value="-1"/>
        <joint name="joint3" value="0.7"/>
        <joint name="joint4" value="0.3"/>
        <joint name="joint5_roll" value="0"/>
    </group_state>
```
**Explanation:** Included joint5_roll in the MoveIt SRDF definitions. The arm group now lists joint5_roll among its joints, and both the init and home group states have been updated to include joint5_roll with its value. This ensures MoveIt’s planning group and predefined poses account for the new wrist roll joint.

**2) Change:** Added and modified disable collisions between wrist_roll_link, gripper left and right links in correspondence to other links
```srdf
    <disable_collisions link1="gripper_left_link" link2="link5" reason="Never"/>
    <disable_collisions link1="gripper_left_link" link2="wrist_roll_link" reason="Adjacent"/>
    <disable_collisions link1="gripper_right_link" link2="link5" reason="Never"/>
    <disable_collisions link1="gripper_right_link" link2="wrist_roll_link" reason="Adjacent"/>
    <disable_collisions link1="link4" link2="wrist_roll_link" reason="Never"/>
    <disable_collisions link1="link5" link2="wrist_roll_link" reason="Adjacent"/>
    <disable_collisions link1="wrist_roll_link" link2="end_effector_link" reason="Adjacent"/>
```

### /config/open_manipulator_x/joint_limits.yaml	

**Change:** Added joint limits for joint5_roll
```yaml
joint5_roll:
    has_position_limits: true
    min_position: -3.141592654   # -180°
    max_position:  3.141592654   # +180°
    has_velocity_limits: true
    max_velocity: 5.0
    has_acceleration_limits: true
    max_acceleration: 5.0
```
**Explanation:** Added configuration entries defining the limits (e.g., position range, velocity, effort) for joint5_roll. This establishes the safety and motion constraints for the wrist roll joint within MoveIt or other planning tools

### /config/open_manipulator_x/moveit_controllers.yaml	

**Change:** Added joint5_roll in the joints list
```yaml
joints:
      - joint1
      - joint2
      - joint3
      - joint4
      - joint5_roll
```

---

## ros2_controller

### /om_gravity_compensation_controller/src/CMakeLists.txt	

**Change:** Modified orocos_kdl  to orocos_kdl_vendor
<br>
**Explanation:** The dependency was changed from orocos_kdl to orocos_kdl_vendor to improve build portability and reliability in ROS 2. While orocos_kdl refers to the raw Kinematics and Dynamics Library that must be installed manually on the system, orocos_kdl_vendor is a ROS 2 vendor package that automatically provides this library if it is not already available. This ensures consistent builds across different platforms and environments without changing any functionality of the code that uses KDL.

---

## open_manipulator_gui

### /include/open_manipulator_x_gui/main_window.hpp	

**Change:** Added the below lines in public Q_SLOTS :
```hpp
  void on_btn_wrist_ccw_clicked(void);
  void on_btn_wrist_cw_clicked(void);
```
**Explanation:** The functions void on_btn_wrist_cw_clicked(void) and void on_btn_wrist_ccw_clicked(void) were added to enable manual wrist rotation control through the graphical user interface. These functions are Qt slot handlers that are triggered when the user clicks the corresponding Wrist Clockwise and Wrist Counter-Clockwise buttons in the GUI. Their purpose is to connect user interactions in the interface to the underlying ROS control logic, allowing the wrist joint to be commanded directly from the GUI; without these slots, the wrist buttons would not perform any action.


### /src/open_manipulator_x/main_window.cpp

**1)Change:** Modified the setColumnCount from 6 to 7
```cpp
tableWidget->setColumnCount(7);
```
**Explanation:** A new joint (Joint 5 roll) was added, so the table needed one extra column to display its value. Increasing the column count ensures every joint and the status column have proper space in the GUI.

**2)Change:** Added new joint to QStringList headers
```cpp
headers << "Joint 1" << "Joint 2" << "Joint 3" << "Joint 4" << "Joint 5 roll" << "Gripper" << "Status";
```
**Explanation:** The table headers were updated to reflect the newly added wrist-roll joint. Without this change, the GUI column labels would not match the actual joint data being displayed.

**3)Change:** Wrist buttons disabled initially (Safe Start)
```cpp
ui.btn_wrist_ccw->setEnabled(false);
ui.btn_wrist_cw->setEnabled(false);
```
**Explanation:** The wrist control buttons are disabled at startup to prevent accidental motion before the ROS node and communication are fully initialized. This ensures safe startup behavior.

**4)Change:** Modified timerCallback to display joint 5 and gripper
```cpp
ui.txt_j5->setText(QString::number(joint_angle.at(4), 'f', 3));
ui.txt_grip->setText(QString::number(joint_angle.at(5), 'f', 3));
```
**Explanation:** After adding the wrist-roll joint, joint indexing changed. This update ensures that each GUI text field correctly displays the real-time value of the correct physical joint, preventing misalignment of displayed data.

**5)Change:** Wrist buttons enabled after ROS initialization
```cpp
ui.btn_wrist_ccw->setEnabled(true);
ui.btn_wrist_cw->setEnabled(true);
```
**Explanation:** The wrist buttons are enabled only after qnode.init() succeeds and the timer starts. This ensures the robot is ready before allowing user interaction.

**6)Change:** Updated predefined Init and Home poses
```cpp
Init pose => std::vector<double> joint_angle = {0.0, 0.0, 0.0, 0.0, 0.0};
Home pose => std::vector<double> joint_angle = {0.0, -1.0, 0.7, 0.3, 0.0};
```
**Explanation:** The predefined poses were expanded to include the new wrist-roll joint. This ensures joint-space commands execute correctly without missing joint values.

**7)Change:** Added wrist CCW/CW button handlers
```cpp
void MainWindow::on_btn_wrist_ccw_clicked(void)
{
  std::thread(
    [this]()
    {
      // Get current joint angles
      std::vector<double> current_joint_angle = qnode.getPresentJointAngle();
      
      // Rotate wrist counter-clockwise by 45 degrees (0.785 rad)
      std::vector<double> joint_angle = {
        current_joint_angle.at(0),
        current_joint_angle.at(1),
        current_joint_angle.at(2),
        current_joint_angle.at(3),
        current_joint_angle.at(4) + 0.785  // Joint 5 roll CCW
      };

      // Clamp to joint limits (±2.618 rad)
      if (joint_angle.at(4) > 2.618) {
        joint_angle.at(4) = 2.618;
      }

      if (!qnode.setJointSpacePath(joint_angle)) {
        QMetaObject::invokeMethod(
          this, [this]() {
            writeLog("[ERR!!] Failed to execute Roll +");
          }, Qt::QueuedConnection);
        return;
      }

      QMetaObject::invokeMethod(
        this, [this]() {
          writeLog("Wrist Roll + (CCW)");
        }, Qt::QueuedConnection);
    }).detach();
}

void MainWindow::on_btn_wrist_cw_clicked(void)
{
  std::thread(
    [this]()
    {
      // Get current joint angles
      std::vector<double> current_joint_angle = qnode.getPresentJointAngle();
      
      // Rotate wrist clockwise by 45 degrees (-0.785 rad)
      std::vector<double> joint_angle = {
        current_joint_angle.at(0),
        current_joint_angle.at(1),
        current_joint_angle.at(2),
        current_joint_angle.at(3),
        current_joint_angle.at(4) - 0.785  // Joint 5 roll CW
      };

      // Clamp to joint limits (±2.618 rad)
      if (joint_angle.at(4) < -2.618) {
        joint_angle.at(4) = -2.618;
      }

      if (!qnode.setJointSpacePath(joint_angle)) {
        QMetaObject::invokeMethod(
          this, [this]() {
            writeLog("[ERR!!] Failed to execute Roll -");
          }, Qt::QueuedConnection);
        return;
      }

      QMetaObject::invokeMethod(
        this, [this]() {
          writeLog("Wrist Roll - (CW)");
        }, Qt::QueuedConnection);
    }).detach();
}
```
**Explanation:** These functions make the wrist rotation buttons work. When clicked, they read the current joint angles, change only the wrist-roll joint by a fixed amount, check that it stays within safe limits, and send the updated values to the robot. Without them, the wrist buttons would do nothing.

**8)Change:** Modified on_btn_read_joint_angle_clicked
```cpp
ui.doubleSpinBox_j5->setValue(joint_angle.at(4));
ui.doubleSpinBox_gripper->setValue(joint_angle.at(5));
```
**Explanation:** Ensures the wrist-roll and gripper values are correctly read from the robot and displayed in the GUI input fields.

**9)Change:** Modified on_btn_send_joint_angle_clicked
```cpp
joint_angle.push_back(ui.doubleSpinBox_j5->value());
```
**Explanation:** Adds the wrist-roll value to the command vector before sending it to the robot. Without this, the wrist joint would not move when sending joint commands.

**10)Change:** Modified on_btn_save_pose_clicked
```cpp
file << joint_angle.at(0) << "," << joint_angle.at(1) << ","
     << joint_angle.at(2) << "," << joint_angle.at(3) << ","
     << joint_angle.at(4) << "," << joint_angle.at(5) << std::endl;
```
**Explanation:** Ensures the wrist-roll and gripper values are saved when exporting poses to CSV. This keeps stored motion data complete and consistent.

**11)Change:** Updated table row loop
```cpp
for (int i = 0; i < 6; ++i) {
```
**Explanation:** The loop count was increased to include the new wrist-roll joint, ensuring all joints are processed correctly.

**12)Change:** Updated status column index
```cpp
tableWidget->setItem(row, 6, statusItem);
```
**Explanation:** Since a new column was added, the Status column index shifted. This update ensures status messages appear in the correct column.

**13)Change:** Modified on_btn_read_task_clicked
```cpp
if (values.size() == 6) {
```
**Explanation:** Updated validation to expect six joint values (including wrist-roll). Prevents incomplete task loading.

**14)Change:** Updated table column loop
```cpp
for (int col = 0; col < 6; ++col) {
```
**Explanation:** Ensures all six joint values from CSV are loaded into the table.

**15)Change:** Modified on_btn_play_clicked condition
```cpp
if (joint_angle.size() == 6) {
```
**Explanation:** Ensures execution only starts if all six joint values are present, preventing runtime errors.

**16)Change:** Updated execution status text
```cpp
tableWidget->item(row, 6)->setText("Executing...");
```
**Explanation:** Adjusted index to match the new table structure so execution status appears in the correct column.

**17)Change:** Separated arm joints from gripper
```cpp
std::vector<double> arm_joints(
  joint_angle.begin(),
  joint_angle.begin() + 5
);
```
**Explanation:** The arm controller expects only arm joints (first 5). The gripper is controlled separately. This separation prevents incorrect command formatting.

**18)Change:** Used arm-only joints in execution
```cpp
if (!qnode.setJointSpacePath(arm_joints)) {
```
**Explanation:** Ensures only arm joints are sent to the arm motion planner, preventing control conflicts with the gripper.

**19)Change:** Updated "Done" status index
```cpp
tableWidget->item(row, 6)->setText("Done");
```
**Explanation:** Aligns completion status with the updated Status column position.

**20)Change:** Updated "Stopped" status index
```cpp
tableWidget->item(row, 6)->setText("Stopped");
```
**Explanation:** Ensures stop notifications appear under the correct Status column after table expansion.

<br>

### /src/open_manipulator_x/qnodecpp

**Change:** Added Joint 5 (wrist roll) support and updated table row and column handling to remain compatible with the additional joint.

<br>

### /ui/open_manipulator_x_main_window.ui	

**Change:** Added Joint 5 (wrist roll) support and updated table row and column handling to remain compatible with the additional joint.

---
