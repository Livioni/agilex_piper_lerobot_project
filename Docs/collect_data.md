# 数据采集


## 前置知识： URDF 和正逆运动学基础概念 (以下只是简短的介绍请自行查找其他资料)

- **URDF**（Unified Robot Description Format，统一机器人描述格式）：一种 XML 格式，用一棵运动学树描述机器人结构。
  - `link`：刚体（连杆），挂载视觉 / 碰撞网格与惯性参数；
  - `joint`：连接两个 link，定义父子关系、关节类型（`revolute` 旋转 / `prismatic` 平移 / `fixed` 固定等）、转轴与限位；
  - `mesh`：引用 `package://` 或文件路径下的三维网格（dae / obj / stl）。
  - 本仓库的机器人回放依赖 URDF：Piper 双臂用 `aloha_tracer2_dabai_dark.urdf`，Arx5 用 `arx5_description_isaac.urdf`；脚本会把 `embodiments/` 加入 `ROS_PACKAGE_PATH` 以解析其中的 `package://` 网格路径。
- **正运动学（Forward Kinematics, FK）**：给定各关节角度 → 求末端执行器在基座（base / footprint）坐标系下的位姿（位置 + 姿态）。本仓库正是用 FK 把数据集每帧的 `state`（关节角 / 夹爪开度）逐帧摆放到对应姿态，从而回放机器人运动。
- **逆运动学（Inverse Kinematics, IK）**：给定期望的末端位姿 → 反解出一组（或多组）关节角度。本仓库只做回放不做 IK；但理解 IK 有助于读懂 LIBERO 这类用**末端位姿**（x/y/z + axis-angle）而非关节角表示 `state` 的数据集——这类数据需 IK 才能映射回关节空间，故本仓库对其仅做视频 / 信号展示（`--no-robot`），不做 URDF 关节回放。

URDF 在线查看 / 调试：https://viewer.robotsfan.com/

## 前置知识： 深度相机测距原理以及点云投影 (以下只是简短的介绍请自行查找其他资料)

### 双目深度测距原理（disparity + 内参 + baseline）

双目相机由左右两个相机组成，两光心间距称为 **baseline（基线）** $B$。空间一点 $P$ 在左右图像中的投影像素横坐标分别为 $u_L$、$u_R$，两者之差即 **disparity（视差）**：

$$d = u_L - u_R$$

由相似三角形可解出沿光轴方向的深度 $Z$：

$$Z = \frac{f \cdot B}{d}$$

其中 $f$ 为焦距（来自**内参**，单位为像素；内参还提供主点 $c_x, c_y$）。

要点：

- disparity 越大 → 物体越近（$Z$ 越小）；disparity 越小 → 物体越远。
- baseline $B$ 越大、焦距 $f$ 越长，测距精度越高、可测距离越远，但盲区（最小可测距离）也变大。
- disparity 通常由立体匹配算法计算（块匹配 SAD/SSD、半全局匹配 SGM、或深度学习网络）。
- 内参 + 标定得到的 baseline 是测距精度的关键；本仓库读取的 `calibration.<camera>.intrinsic_matrix` 与 `camera_pose_matrix` / `extrinsic_matrix` 即对应这一组量。

> 说明：本仓库 RoboTwin2 示例直接给出 RGB-D 深度流，而非左目 + 右目自行计算视差；但上述双目几何正是 RGB-D（结构光 / ToF / 双目）测距的共同物理基础，理解后即可推广到任何给出深度图的传感器。

### 从深度图得到点云

给定深度图 $D(u,v)$（像素 $(u,v)$ 处沿光轴的深度，单位米）与内参 $f_x, f_y, c_x, c_y$，逐像素反投影即得相机坐标系下的三维点：

$$
X = \frac{(u - c_x)\, D(u,v)}{f_x}, \quad
Y = \frac{(v - c_y)\, D(u,v)}{f_y}, \quad
Z = D(u,v)
$$

相机坐标系约定（OpenCV）：$+X$ 向右、$+Y$ 下、$+Z$ 向前。再用相机外参（位姿）$T_{world \leftarrow camera}$ 把点变换到世界 / 机器人 base 坐标系：

$$p_{world} = T_{world \leftarrow camera} \cdot p_{camera}$$


## 前置知识：Lerobot 数据格式以及Rerun可视化

请根据下面的repository 的readme 完成所有可视化步骤：

https://github.com/Livioni/Lerobot_Datasets

### Checklist 

> 目标：从零开始，使用 [Lerobot_Datasets](https://github.com/Livioni/Lerobot_Datasets) 仓库完成本地 LeRobot 数据集的**可视化、点云重建、机器人回放**。每完成一项打勾。

- [ ] 环境准备
- [ ] 运行内置示例
    - [ ] **LIBERO（单臂 Franka，仅视频/信号）** : 
      ```bash
        python visualize_lerobot_rerun.py --root assets/example/LIBERO --episode 0 --no-robot
        ```
    - [ ] **RoboTwin2（完整 Arx5 机器人 + RGB-D）**：
      ```bash
      python visualize_lerobot_rerun.py --root assets/example/RoboTwin2 --episode 0
      ```
    - [ ] **RoboTwin2 RGB-D 点云重建**（在 base 坐标系下重建主相机 + 双腕部点云）：
      ```bash
      python visualize_lerobot_rerun.py --root assets/example/RoboTwin2 --episode 0 --point-cloud
      ```
    - [ ] **W2（Piper 双臂，含 14 维位姿/速度/力/动作信号）**：
      ```bash
      python visualize_lerobot_rerun.py --root assets/example/W2 --episode 0
      ```
    - [ ] **W2 标定相机视角回放**（固定主相机视角）：
      ```bash
      python visualize_lerobot_rerun.py \
        --root assets/example/W2 --episode 0 \
        --camera-calibration caliberations/w2_demo.yaml \
        --camera-resolution 480 640
      ```
    - [ ] **RoboTwin 标定相机视角回放**（含点云）：
      ```bash
      python visualize_lerobot_rerun.py \
        --root assets/example/RoboTwin2 --episode 0 \
        --camera-calibration caliberations/robotwin.yaml \
        --camera-resolution 240 320 \
        --point-cloud
      ```

- [ ] 理解Lerobot v2.1和Lerobot v3.0 的区别
- [ ] W2系列的数据是Agilex-aloha 真机采集的数据，观察有哪些值？
- [ ] Action和State有什么区别？
- [ ] 理解Rerun中标定相机视角回放的原理
  
## 真机采集

下面的代码均在Agilex-aloha 真机上运行：

### 1. Camera

连接服务器，运行

```bash
zellij attach workspace
```

> **Zellij** 是一个用 Rust 编写的终端复用器（terminal multiplexer，类似 tmux / screen）：可在单个终端里管理多个窗格（pane）/标签页（tab）/窗口，会话在断开后仍然存活，可用 `zellij attach` 重新接入。上面的 `zellij attach workspace` 即重新接入名为 `workspace` 的已有会话，其中已预先排布好 camera / ros 等窗格。

!!! 请注意 退出zellij 请使用CTRL+O 然后按D !!!

!!! 请注意 退出zellij 请使用CTRL+O 然后按D !!!

!!! 请注意 退出zellij 请使用CTRL+O 然后按D !!!

![zellij 窗格布局](../assets/images/zellij.png)

打开 ros 窗口，左侧运行

```bash
cd ~/cobot_magic
bash tools/camera_serial.sh
```

测试通过后启动所有相机，并保持终端运行：
切换到

```bash
roslaunch astra_camera multi_camera.launch
```

右侧运行：

```bash
launch_camera
```
### 2. ROS

与 任务一 同一个 zellij，打开 ros 窗口

若重启电脑，需激活左右机械臂 CAN，否则可跳过此步：

```bash
cd ~/cobot_magic/Piper_ros_private-ros-noetic
bash can_muti_activate.sh --ignore
```

若采集数据：

```bash
roslaunch piper start_ms_piper.launch mode:=0 auto_enable:=false
```

### 3. 采集数据脚本


```bash
python3 collect_data/collect_data_interactive.py \
   --dataset_dir ~/data/exp7_demo \
   --task_name demo \
   --episode_idx 0
```

请查看 ~/data 文件夹，确认命名规范：

```bash
.
├── exp1_put_mongo
│   ├── m2w-put-mongo
│   └── m2w-put-mongo-lerobot
│       ├── data
│       │   └── chunk-000
│       ├── images
│       │   ├── observation.images.cam_high
│       │   ├── observation.images.cam_left_wrist
│       │   └── observation.images.cam_right_wrist
│       ├── meta
│       │   └── episodes
│       │       └── chunk-000
│       └── videos
│           ├── observation.images.cam_high
│           │   └── chunk-000
│           ├── observation.images.cam_left_wrist
│           │   └── chunk-000
│           └── observation.images.cam_right_wrist
│               └── chunk-000
├── exp2_table_clean
│   ├── table_clean
│   └── table_clean_lerobot
│       ├── data
│       │   └── chunk-000
│       ├── images
│       │   ├── observation.images.cam_high
│       │   ├── observation.images.cam_left_wrist
│       │   └── observation.images.cam_right_wrist
│       ├── meta
│       │   └── episodes
│       │       └── chunk-000
│       └── videos
│           ├── observation.images.cam_high
│           │   └── chunk-000
│           ├── observation.images.cam_left_wrist
│           │   └── chunk-000
│           └── observation.images.cam_right_wrist
│               └── chunk-000
....
```

Save videos

```bash
bash /home/agilex/cobot_magic/collect_data/convert_place_bag_lerobot_224.sh \
  --image-size original \
  --output-dir /home/agilex/data/demo_place_bag/place_bag_lerobot \
  --repo-id agilex/place_bag_v2 --overwrite
```

### 4. hdf5数据格式转化为Lerobot v3.0


```bash
/home/agilex/miniconda3/envs/lerobot/bin/python \
  /home/agilex/cobot_magic/collect_data/convert_piper_hdf5_depth.py \
  --input-dir /home/agilex/data/demo_depth/drop_bin \
  --output-dir /home/agilex/data/demo_depth/drop_bin_lerobot_depth \
  --repo-id agilex/drop_bin_depth \
  --task "Throw the batteries into the trash bin." \
  --fps 30 \
  --image-size original \
  --depth-unit mm
```