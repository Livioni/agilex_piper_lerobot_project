# Piper + Orbbec 快速操作指南

本流程已在本机三台 Orbbec 相机和两组 Piper 主从机械臂上测试。

## 1. 安全注意事项

- 清空机械臂工作区域，并确保急停可立即使用。
- 不要用手强行移动已使能的机械臂。
- 失能前先支撑从臂；电机失去力矩后机械臂可能下坠。
- 插拔主臂/控制臂连接器前，先关闭机械臂电源。
- 不要修改 `can_bottom`，它不是机械臂接口。

## 2. 连接与测试硬件

### 相机

```bash
cd ~/cobot_magic
bash tools/camera_serial.sh
```

本机可找到三台设备。该辅助程序可能在列出设备后段错误，因此最终以 ROS topic 为准。

启动所有相机，并保持终端运行：

```bash
source ~/cobot_magic/camera_ws/devel/setup.bash
roslaunch astra_camera multi_camera.launch
```

测试：

```bash
rostopic hz /camera_f/color/image_raw
rostopic hz /camera_l/color/image_raw
rostopic hz /camera_r/color/image_raw
rqt_image_view
```

### 机械臂 CAN 接口

本机测试过的映射：

```text
1-2:1.0 -> can_left
1-3:1.0 -> can_right
1-4:1.0 -> can_bottom
```

激活左右机械臂 CAN：

```bash
cd ~/cobot_magic/Piper_ros_private-ros-noetic
bash can_muti_activate.sh --ignore
```

如果脚本不能配置，可手动执行：

```bash
sudo ip link set can_left down
sudo ip link set can_left type can bitrate 1000000
sudo ip link set can_left up

sudo ip link set can_right down
sudo ip link set can_right type can bitrate 1000000
sudo ip link set can_right up
```

测试：

```bash
ip -details link show can_left | grep -E 'state|bitrate'
ip -details link show can_right | grep -E 'state|bitrate'
```

正常结果包含：`state UP`、`can state ERROR-ACTIVE` 和 `bitrate 1000000`。

## 3. Piper 模式与标志

| 设置 | 用途 |
|---|---|
| `mode:=0` | 读取主臂/从臂状态并采集示教数据 |
| `mode:=1` | 接收 ROS 指令并控制从臂运动 |
| `auto_enable:=true` | 在 mode 1 启动时自动使能从臂 |
| `/enable_flag: true` | 使能从臂电机 |
| `/enable_flag: false` | 取消电机力矩；这不是主从同步模式 |

上表以实际测试的 Python 节点行为为准。Piper 自带 README 中部分文字将 mode 0 和 mode 1 的说明写反了。

## 4. 采集相机与机械臂数据

使用三个独立终端。采集过程中保持终端 1 和终端 2 运行。

### 终端 1：相机

```bash
cd ~/cobot_magic
source camera_ws/devel/setup.bash
roslaunch astra_camera multi_camera.launch
```

### 终端 2：机械臂读取模式

```bash
cd ~/cobot_magic/Piper_ros_private-ros-noetic
source devel/setup.bash
roslaunch piper start_ms_piper.launch mode:=0 auto_enable:=false
```

### 终端 3：先测试，再采集

测试主臂和从臂 topic：

```bash
rostopic hz /master/joint_left
rostopic hz /master/joint_right
rostopic hz /puppet/joint_left
rostopic hz /puppet/joint_right
```

每个测试使用 `Ctrl-C` 停止，然后在同一个终端 3 中采集 episode 0：

```bash
cd ~/cobot_magic
source Piper_ros_private-ros-noetic/devel/setup.bash

python3 collect_data/collect_data.py \
  --dataset_dir ~/data \
  --task_name piper_demo \
  --episode_idx 0 \
  --max_timesteps 500 \
  --frame_rate 30 \
  --master_arm_left_topic /master/joint_left \
  --master_arm_right_topic /master/joint_right \
  --puppet_arm_left_topic /puppet/joint_left \
  --puppet_arm_right_topic /puppet/joint_right
```

输出文件：

```text
/home/agilex/data/piper_demo/episode_0.hdf5
```

每次采集都要增加 `--episode_idx`。重复使用编号会覆盖原文件。

偶发的 `syn fail` 可以接受。该次循环不会计入帧数，程序会继续重试。完整的 500 帧结果应为：

```text
len(timesteps): 501
len(actions): 500
```

## 5. 回放动作

使用两个独立终端。使用 `--only_pub_master` 回放时不需要相机节点。

先停止采集程序，并在 mode 0 机械臂终端按 `Ctrl-C`。准备机械臂并清空工作区域。

### 终端 1：机械臂控制模式

```bash
cd ~/cobot_magic/Piper_ros_private-ros-noetic
source devel/setup.bash
roslaunch piper start_ms_piper.launch mode:=1 auto_enable:=true
```

保持终端 1 运行，等待左右机械臂成功使能，并确认没有 CAN 错误。

### 终端 2：动作回放

```bash
cd ~/cobot_magic
source Piper_ros_private-ros-noetic/devel/setup.bash

python3 collect_data/replay_data.py \
  --dataset_dir ~/data \
  --task_name piper_demo \
  --episode_idx 0 \
  --only_pub_master \
  --frame_rate 30 \
  --control_rate 200
```

- `frame_rate`：原始动作的采集频率，30 Hz。
- `control_rate`：平滑控制指令的发布频率，200 Hz。

修改后的回放代码：

```python
rate = rospy.Rate(args.control_rate)
interpolation_steps = max(1, round(args.control_rate / args.frame_rate))
new_actions = np.linspace(last_action, action, interpolation_steps + 1)[1:]
```

不使用 `linspace` 时，30 Hz 下每隔约 33 ms 直接跳到下一个目标。使用插值后，每两个目标之间会分成约七个较小的 200 Hz 指令，因此动作更平滑，并且总时长接近原始记录。

按 `Ctrl-C` 停止回放。机械臂仍保持使能并锁定在最后位置。

## 6. 停止、解锁并恢复同步模式

mode 1 终端运行时，可取消从臂电机力矩：

```bash
rostopic pub --once /enable_flag std_msgs/Bool "data: false"
```

执行前必须先支撑机械臂，因为失能后可能下坠。失能后仍可能存在减速器阻力。

恢复本机已测试的主从同步流程：

1. 停止回放终端。
2. 停止 Piper mode 1 终端。
3. 关闭机械臂电源。
4. 重新插拔主臂/控制臂连接器。
5. 打开机械臂电源。
6. 再次启动读取模式：

```bash
cd ~/cobot_magic/Piper_ros_private-ros-noetic
source devel/setup.bash
roslaunch piper start_ms_piper.launch mode:=0 auto_enable:=false
```

小幅移动左右主臂，确认对应从臂恢复同步。

## 7. 本地代码修改

`Piper_ros_private-ros-noetic/can_muti_activate.sh`：

```bash
USB_PORTS["1-2:1.0"]="can_left:1000000"
USB_PORTS["1-3:1.0"]="can_right:1000000"
```

`collect_data/replay_data.py`：

- 增加 `--control_rate` 参数。
- 将固定 20 步插值改为根据 `control_rate / frame_rate` 自动计算插值步数。

## 8. LeRobot Dataset v2.0 转换

LeRobot v2.0 仓库克隆在：

```text
/home/agilex/lerobot-v2
```

激活独立 Conda 环境：

```bash
conda activate lerobot_v2
```

转换程序位置：

```text
/home/agilex/lerobot-v2/convert_piper_hdf5.py
```

每次转换时需要修改以下参数：

- `--input-dir`：包含 `episode_*.hdf5` 文件的目录。
- `--output-dir`：LeRobot 数据集输出目录；该目录不能已存在。
- `--repo-id`：写入 LeRobot metadata 的数据集名称。
- `--fps`：采集数据时使用的帧率。
- `--task`：写入数据集的任务描述。

示例：

```bash
cd ~/lerobot-v2
conda activate lerobot_v2

python convert_piper_hdf5.py \
  --input-dir /home/agilex/data/piper_demo \
  --output-dir /home/agilex/lerobot-v2-data/agilex/piper_demo_v2 \
  --repo-id agilex/piper_demo_v2 \
  --fps 30 \
  --task "Piper bimanual demonstration"
```

该命令的输出目录是：

```text
/home/agilex/lerobot-v2-data/agilex/piper_demo_v2
```

目录内容：

```text
piper_demo_v2/
├── data/
├── meta/
└── videos/
```

采集和转换内容包括 RGB 视频、关节位置、速度、力矩和动作，不采集深度图像。

最后更新：`2026-07-10 CST`。
