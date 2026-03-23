# MoveIt 到上位机桥接与真机联调

面向当前 `ur5` 工作区的实机联调文档。目标只有一个：把 MoveIt 的执行轨迹稳定送到上位机，再由上位机下发给机器人。

## 概览
#下面有关路径都请换成你自己的路径，内部有些文件是绝对路径，你也自己改一改

当前工作区基于 ROS2 Humble，核心链路如下：

```text
MoveIt Execute
  -> FollowJointTrajectory action
  -> trajectory_to_marvin_cmd
  -> /control/joint_cmd_A | /control/joint_cmd_B
  -> marvin_robot_node
  -> 机器人
```

实际包名如下：

- `robotnode_marvin_ros_control`
- `robotnode_marvin_msgs`
- `marvin_real_control`
- `marvin_robot_config`

工作区路径（example）

```bash
/home/tjzn-zwt/ur5
```

## 1. 快速开始

建议第一次按完整流程跑，确认链路稳定后再精简自己的操作习惯。

### 1.1 编译

```bash
cd /home/tjzn-zwt/ur5
source /opt/ros/humble/setup.bash
colcon build --packages-select \
  robotnode_marvin_msgs \
  robotnode_marvin_ros_control \
  marvin_real_control \
  marvin_robot_config
source install/setup.bash
```

### 1.2 启动顺序

终端 A：上位机控制

```bash
cd /home/tjzn-zwt/ur5
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 launch robotnode_marvin_ros_control bringup_control_m6.launch.py
```

终端 B：桥接节点

```bash
cd /home/tjzn-zwt/ur5
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 run marvin_real_control trajectory_to_marvin_cmd
```

终端 C：MoveIt

```bash
cd /home/tjzn-zwt/ur5
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 launch marvin_robot_config demo.launch.py
```

终端 D：机器人置为可运行

```bash
cd /home/tjzn-zwt/ur5
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 service call /control/set_ready std_srvs/srv/Trigger "{}"
ros2 service call /control/set_mode robotnode_marvin_msgs/srv/Int "{data: 1}"
```

终端 E：联调检查

```bash
ros2 action info /left_arm_controller/follow_joint_trajectory
ros2 action info /right_arm_controller/follow_joint_trajectory
ros2 topic info /control/joint_cmd_A -v
ros2 topic info /control/joint_cmd_B -v
```

RViz 中操作顺序：

1. 选择规划组，例如 `left_arm`
2. 点击 `Plan`
3. 点击 `Execute`

## 2. 成功判定

满足以下条件即可认为链路已经打通：

1. MoveIt `Plan and Execute` 成功完成。
2. `/control/joint_cmd_A` 或 `/control/joint_cmd_B` 能看到 `positions`。
3. 执行期间 `ros2 topic hz /control/joint_cmd_A` 或 `/control/joint_cmd_B` 有连续频率。
4. 真实机器人跟随动作。

关键验证命令：

```bash
ros2 topic info /control/joint_cmd_A -v
```

如果输出中包含以下关系，说明桥接已经送达上位机节点：

- `Publisher`: `trajectory_to_marvin_cmd`
- `Subscription`: `marvin_robot_node`

对应链路为：

```text
MoveIt -> trajectory_to_marvin_cmd -> /control/joint_cmd_A -> marvin_robot_node
```

## 3. 运行中监控

执行轨迹时，建议额外开一个监控终端：

```bash
ros2 topic hz /control/joint_cmd_A
ros2 topic hz /control/joint_cmd_B
```

如果已经连接真机，再开一个窗口看反馈：

```bash
ros2 topic echo /joint_states
```

期望现象：

- `joint_cmd_A` 或 `joint_cmd_B` 在执行期间持续有频率
- `/joint_states` 的时间戳和关节位置持续变化

## 4. 离线验证

不连真机时，可以验证软件桥接本身是否正常，但不能验证机器人实际执行。

可以验证：

1. MoveIt action 是否送到桥接。
2. 桥接是否持续发布 `/control/joint_cmd_A|B`。
3. 消息格式和频率是否正确。

不能验证：

1. 上位机到机器人驱动的通讯。
2. 机器人是否真的运动。
3. 真机闭环反馈是否稳定。

只测桥接时，可以直接发送一个 action goal：

```bash
ros2 action send_goal /left_arm_controller/follow_joint_trajectory \
control_msgs/action/FollowJointTrajectory \
"{trajectory: {joint_names: ['Joint1_L','Joint2_L','Joint3_L','Joint4_L','Joint5_L','Joint6_L','Joint7_L'], points: [{positions: [0.1,0,0,0,0,0,0], time_from_start: {sec: 1, nanosec: 0}}]}}"
```

同时查看：

```bash
ros2 topic echo --once /control/joint_cmd_A
```

## 5. joint_states 冲突说明

如果仿真状态源和上位机状态源同时发布同名 `joint_states`，模型会来回跳动。

处理原则：

- 只改 topic 名，不改消息类型
- 消息类型保持 `sensor_msgs/msg/JointState`
- MoveIt/仿真侧可以使用独立名字，例如 `/moveit_joint_states`
- 上位机/真机侧保留原有状态话题

注意：

- 发布端改名后，订阅端必须同步 remap
- 一端改名、另一端不改，会直接收不到状态

## 6. 常见问题

### 6.1 `Package 'marvin_ros_control' not found`

当前真实包名不是 `marvin_ros_control`，而是：

```bash
robotnode_marvin_ros_control
```

正确启动命令：

```bash
ros2 launch robotnode_marvin_ros_control bringup_control_m6.launch.py
```

### 6.2 `The message type 'robotnode_marvin_msgs/msg/Jointcmd' is invalid`

先重启 daemon，并确认接口存在：

```bash
cd /home/tjzn-zwt/ur5
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 daemon stop
ros2 daemon start
ros2 interface show robotnode_marvin_msgs/msg/Jointcmd
```

### 6.3 MoveIt 显示执行成功，但真机不动

按这个顺序排查：

1. `/control/set_ready` 是否 `success=True`
2. `/control/set_mode` 是否 `success=True`
3. `/control/joint_cmd_A|B` 是否有消息
4. `/control/joint_cmd_A|B` 是否被 `marvin_robot_node` 订阅
5. `follow_joint_trajectory` 是否存在双 server 冲突

### 6.4 action 双 server 冲突

检查命令：

```bash
ros2 action info /left_arm_controller/follow_joint_trajectory
```

如果同名 action 出现多个 server，MoveIt 可能随机命中，执行会不稳定。

### 6.5 上位机未就绪

如果出现：

```text
Robot not connected
```

先检查：

1. 网线和 IP 是否正常
2. 上位机是否已经连上机器人
3. 再重新调用 `/control/set_ready`

## 7. 最短流程备忘

只想看最短步骤时，按这个顺序执行：

```bash
cd /home/tjzn-zwt/ur5
source /opt/ros/humble/setup.bash
colcon build --packages-select robotnode_marvin_msgs robotnode_marvin_ros_control marvin_real_control marvin_robot_config
source install/setup.bash
```

```bash
ros2 launch robotnode_marvin_ros_control bringup_control_m6.launch.py
```

```bash
ros2 run marvin_real_control trajectory_to_marvin_cmd
```

```bash
ros2 launch marvin_robot_config demo.launch.py
```

```bash
ros2 service call /control/set_ready std_srvs/srv/Trigger "{}"
ros2 service call /control/set_mode robotnode_marvin_msgs/srv/Int "{data: 1}"
```

```bash
ros2 topic info /control/joint_cmd_A -v
ros2 topic hz /control/joint_cmd_A
```
