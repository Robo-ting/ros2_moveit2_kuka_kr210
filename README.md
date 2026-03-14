# ROS2 MoveIt2 KUKA KR210

本仓库包含 KUKA KR210 机器人在 ROS2 与 MoveIt2 下的仿真与运动规划配置，主要用于木结构加工等研究。感谢 ZJUI 肖岩教授的指导。

## 系统要求

- **ROS2**：Humble 或更高版本
- **Ubuntu**：22.04（推荐）
- **MoveIt2**：随 ROS2 安装

## 一、初始化

### 1.1 安装依赖

```bash
# 安装 ROS2 基础依赖
sudo apt update
sudo apt install -y ros-humble-moveit ros-humble-moveit-visual-tools

# 若需 Gazebo 仿真，安装以下包
sudo apt install -y ros-humble-gazebo-ros-pkgs ros-humble-gazebo-ros2-control
```

### 1.2 克隆与编译

```bash
# 克隆仓库
git clone https://github.com/Robo-ting/ros2_moveit2_kuka_kr210.git
cd ros2_moveit2_kuka_kr210

# 编译工作空间
colcon build --symlink-install

# 加载环境（每次新开终端都需要执行）
source install/setup.bash
```

### 1.3 验证安装

```bash
# 检查包是否可用
ros2 pkg list | grep kuka
# 应看到 kuka_description 和 kuka_moveit2
```

---

## 二、启动 RViz（MoveIt2 可视化与规划）

### 2.1 方式一：完整 Demo（推荐）

一键启动 MoveIt2、RViz 和交互式规划界面：

```bash
source install/setup.bash
ros2 launch kuka_moveit2 demo.launch.py
```

### 2.2 方式二：仅 MoveIt + RViz

```bash
source install/setup.bash
ros2 launch kuka_moveit2 moveit_rviz.launch.py
```

### 2.3 RViz 中的操作

1. 等待 RViz 和 MoveGroup 完全启动
2. 在 RViz 左侧 **MotionPlanning** 插件中：
   - 使用 **Planning** 选项卡设置目标位姿（拖拽交互式标记或输入关节角度）
   - 点击 **Plan** 进行运动规划
   - 点击 **Execute** 执行轨迹（仿真模式下会驱动虚拟机器人）
3. 可切换 **Scene Robot**、**Planning Scene** 等显示选项查看机器人与规划场景

---

## 三、启动 Gazebo 仿真

### 3.1 启动 Gazebo 与机器人

```bash
source install/setup.bash
ros2 launch kuka_moveit2 gazebo.launch.py
```

**双系统用户**：若出现找不到模型或 mesh 文件，可增加启动延迟（默认 5 秒，可调高）：

```bash
ros2 launch kuka_moveit2 gazebo.launch.py start_delay:=10.0
```

该命令会依次启动：

- Gazebo 仿真环境
- `robot_state_publisher`（发布机器人状态）
- 在 Gazebo 中生成 KUKA KR210 模型
- `joint_state_broadcaster` 与 `arm_controller`

### 3.2 Gazebo 与 MoveIt 联合使用

若要在 Gazebo 中执行 MoveIt 规划的轨迹，需要：

**终端 1：启动 Gazebo**

```bash
source install/setup.bash
ros2 launch kuka_moveit2 gazebo.launch.py
```

**终端 2：启动 MoveGroup（连接 Gazebo 的控制器）**

```bash
source install/setup.bash
ros2 launch kuka_moveit2 move_group.launch.py use_sim_time:=true
```

**终端 3：启动 RViz 进行规划与执行**

```bash
source install/setup.bash
ros2 launch kuka_moveit2 moveit_rviz.launch.py use_sim_time:=true
```

> 与 Gazebo 联用时需设置 `use_sim_time:=true`，以保持时间同步。

在 RViz 中规划并 **Execute** 时，轨迹会发送到 Gazebo 中的 `arm_controller`，机器人将在仿真中运动。

---

## 四、运行结果说明

| 启动方式 | 预期效果 |
|---------|---------|
| `demo.launch.py` | RViz 中显示 KR210 模型，可进行运动规划与仿真执行 |
| `moveit_rviz.launch.py` | 同上，适合与 Gazebo 等外部仿真配合 |
| `gazebo.launch.py` | Gazebo 中加载 KR210，可接收关节轨迹控制 |
| `move_group.launch.py` | 仅启动 MoveGroup 节点，供 RViz 或其他客户端连接 |

---

## 五、包结构说明

```
src/
├── kuka_description/     # 机器人 URDF、mesh、xacro
│   ├── urdf/
│   ├── meshes/
│   └── launch/
└── kuka_moveit2/        # MoveIt2 配置与启动
    ├── config/          # SRDF、运动学、控制器等配置
    └── launch/          # 各类 launch 文件
```

---

## 六、双系统与启动延迟

在 **Windows + Linux 双系统** 环境下，从 Windows 重启进入 Linux 后，文件系统或包路径可能尚未完全就绪，Gazebo 启动时容易出现：

- `Could not find model` / 找不到模型
- mesh 文件路径解析失败
- `robot_description` 话题为空，spawn 失败

**处理方式**：在 `gazebo.launch.py` 中已加入可配置的启动延迟，在 Gazebo 启动后等待数秒再加载机器人，给文件系统留出就绪时间。

```bash
# 默认延迟 5 秒
ros2 launch kuka_moveit2 gazebo.launch.py

# 若仍失败，可尝试 10 秒或更长
ros2 launch kuka_moveit2 gazebo.launch.py start_delay:=10.0
```

---

## 七、常见问题与解决方式

| 现象 | 可能原因 | 解决方式 |
|------|----------|----------|
| Gazebo 报错找不到 `gazebo` 或插件 | 未安装 Gazebo 相关包 | `sudo apt install ros-humble-gazebo-ros-pkgs ros-humble-gazebo-ros2-control` |
| 双系统下 Gazebo 找不到模型 / mesh | 文件系统未就绪，包路径解析失败 | 使用 `start_delay:=10.0` 增加启动延迟 |
| RViz 中看不到机器人 | MoveGroup 未就绪或 Fixed Frame 错误 | 等待启动完成；将 Fixed Frame 设为 `base_link` 或 `world` |
| Execute 无反应 | `arm_controller` 未加载或名称不一致 | 确认 Gazebo 日志中控制器已加载；核对 `moveit_controllers.yaml` 与 `ros2_controllers.yaml` |
| `source install/setup.bash` 后仍找不到包 | 未在正确目录执行 source | 在 `ros2_moveit2_kuka_kr210` 根目录执行，或使用绝对路径 |
| Host key verification failed（git push） | SSH 未添加 GitHub 主机密钥 | `ssh-keyscan github.com >> ~/.ssh/known_hosts` |

---

## 许可证

本项目采用 [MIT License](LICENSE)。
