## 1. Camera

连接服务器，运行

```cmd
zellij attach workspace
```
打开 ros 窗口，左侧运行

```bash
cd ~/cobot_magic
bash tools/camera_serial.sh
```

测试通过后启动所有相机，并保持终端运行：

```bash
roslaunch astra_camera multi_camera.launch
```

右侧运行：

```bash
launch_camera
```
## 2. ROS

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

若部署时：

```bash
roslaunch piper start_ms_piper.launch mode:=1 auto_enable:=true
```
## 3. 部署

部署需要联通 server(policy) 与 client(robot)

### sever 端

```bash
zellij attach starvla

your_ckpt=checkpoints/your_path/your_ckpt.pt

python deployment/model_server/server_policy.py \
    --ckpt_path ${your_ckpt} \
    --port 8000 \
    --use_bf16
```



### client 端

调整

```
/home/agilex/Documents/phs/github/RTC-Anything/configs 
```

中 .yaml 文件 sever port 与对应的 client 文件的 port 一致，

client 文件位置: 

```
/home/agilex/Documents/phs/github/RTC-Anything/src
```

 使用时用 codex 根据 
 
 ```
 /home/agilex/Documents/phs/github/RTC-Anything/src/dual_piper_deploy.py
 ```
 
 的格式写一份针对当前模型的 client，调整端口一致

```text
请参考：

/home/agilex/Documents/phs/github/RTC-Anything/src/dual_piper_deploy.py

的代码结构和调用方式，为当前模型在同一目录下编写一份新的 client。

要求：
1. 保持 dual_piper_deploy.py 的整体部署流程和代码风格；
2. 根据当前模型修改模型输入、输出以及数据预处理/后处理；
3. 保留命令行 --port 参数；
4. client 连接的端口必须与 configs 中当前模型 .yaml 文件的 server port 一致；
5. 不要修改与当前模型无关的代码；
6. 新建 client 文件，不要直接覆盖 dual_piper_deploy.py。
```

随后运行

```bash
zellij attach rtc

uv run src/dual_piper_deploy_<your_model>.py --config configs/your/path.yaml --port 8000
```

## 4. 数据采集

可参考 

```
/home/agilex/cobot_magic/reset_pose/README.md
```

1, 2 步执行后，打开：

```
zellij attach workspace
```

中的 collect_data, 执行：

```bash
python3 collect_data/collect_data.py \
  --dataset_dir ~/data/demo_task_name \
  --task_name <task_name> \
  --episode_idx 0 \
  --max_timesteps 300 \
  --frame_rate 30
```

数据采集后，将 hdf5 文件转换为 lerobot 格式，减小硬盘空间占用

```bash
python collect_data/convert_piper_hdf5.py \
    --input-dir /home/agilex/data/<exp7_drop_bin>(exp num_task_name)  \
    --output-dir /home/agilex/data/drop_bin_lerobot \
    --image-size original
```

## 带深度的数据采集

### 采集

```bash
cd /home/agilex/cobot_magic

python3 collect_data/collect_data.py \
  --dataset_dir /home/agilex/data/demo_depth \
  --task_name drop_bin \
  --episode_idx 0 \
  --max_timesteps 300 \
  --frame_rate 30 \
  --use_depth_image True
```

### 转换

使用一个新的 py 文件

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