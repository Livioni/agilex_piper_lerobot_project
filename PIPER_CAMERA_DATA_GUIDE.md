# Piper + Orbbec Quick Guide

Tested on this machine with three Orbbec cameras and two Piper arm pairs.

## 1. Safety

- Clear the arm workspace and keep the emergency stop ready.
- Do not force an enabled arm by hand.
- Support the follower arms before disabling them; they can fall when torque is removed.
- Stop arm power before unplugging or reconnecting the master/control-arm connectors.
- Do not change `can_bottom`; it is not an arm interface.

## 2. Connect and test hardware

### Cameras

```bash
cd ~/cobot_magic
bash tools/camera_serial.sh
```

The helper found three devices on this machine. It may segfault after listing them; confirm operation with ROS topics.

Start all cameras and keep the terminal running:

```bash
source ~/cobot_magic/camera_ws/devel/setup.bash
roslaunch astra_camera multi_camera.launch
```

Test:

```bash
rostopic hz /camera_f/color/image_raw
rostopic hz /camera_l/color/image_raw
rostopic hz /camera_r/color/image_raw
rqt_image_view
```

### Arm CAN ports

The tested mapping is:

```text
1-2:1.0 -> can_left
1-3:1.0 -> can_right
1-4:1.0 -> can_bottom
```

Activate the two arm interfaces:

```bash
cd ~/cobot_magic/Piper_ros_private-ros-noetic
bash can_muti_activate.sh --ignore
```

If needed, configure them manually:

```bash
sudo ip link set can_left down
sudo ip link set can_left type can bitrate 1000000
sudo ip link set can_left up

sudo ip link set can_right down
sudo ip link set can_right type can bitrate 1000000
sudo ip link set can_right up
```

Test:

```bash
ip -details link show can_left | grep -E 'state|bitrate'
ip -details link show can_right | grep -E 'state|bitrate'
```

Expected: `state UP`, `can state ERROR-ACTIVE`, and `bitrate 1000000`.

## 3. Piper modes

| Setting | Use |
|---|---|
| `mode:=0` | Read master/follower states and record demonstrations |
| `mode:=1` | Accept ROS commands and move follower arms |
| `auto_enable:=true` | Enable follower motors automatically in mode 1 |
| `/enable_flag: true` | Enable follower motors |
| `/enable_flag: false` | Remove motor torque; this is not sync mode |

The table follows the tested Python node behavior. Some text in the bundled Piper README reverses modes 0 and 1.

## 4. Record camera and arm data

Use three separate terminals. Keep Terminals 1 and 2 running during recording.

### Terminal 1: cameras

```bash
cd ~/cobot_magic
source camera_ws/devel/setup.bash
roslaunch astra_camera multi_camera.launch
```

### Terminal 2: arm read mode

```bash
cd ~/cobot_magic/Piper_ros_private-ros-noetic
source devel/setup.bash
roslaunch piper start_ms_piper.launch mode:=0 auto_enable:=false
```

### Terminal 3: test, then record

Test both master and follower topics:

```bash
rostopic hz /master/joint_left
rostopic hz /master/joint_right
rostopic hz /puppet/joint_left
rostopic hz /puppet/joint_right
```

Stop each test with `Ctrl-C`. Then record episode 0 in the same Terminal 3:

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
  --puppet_arm_right_topic /puppet/joint_right \
  --use_depth_image True
```

Output:

```text
/home/agilex/data/piper_demo/episode_0.hdf5
```

Use a new `--episode_idx` for every recording. Reusing an index overwrites the file.

Occasional `syn fail` is acceptable. The loop retries without counting that iteration. A complete 500-frame recording ends with:

```text
len(timesteps): 501
len(actions): 500
```

## 5. Replay actions

Use two separate terminals. Cameras are not required for `--only_pub_master` replay.

First stop the recording program and stop the mode 0 arm terminal with `Ctrl-C`. Prepare the arms and clear the workspace.

### Terminal 1: arm control mode

```bash
cd ~/cobot_magic/Piper_ros_private-ros-noetic
source devel/setup.bash
roslaunch piper start_ms_piper.launch mode:=1 auto_enable:=true
```

Keep Terminal 1 running and wait until both arms enable without CAN errors.

### Terminal 2: action replay

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

- `frame_rate` is the recorded action rate: 30 Hz.
- `control_rate` is the smooth arm-command rate: 200 Hz.

The edited replay code uses:

```python
rate = rospy.Rate(args.control_rate)
interpolation_steps = max(1, round(args.control_rate / args.frame_rate))
new_actions = np.linspace(last_action, action, interpolation_steps + 1)[1:]
```

Without `linspace`, each target jumps directly to the next target every 33 ms at 30 Hz. With interpolation, each transition is divided into about seven smaller commands at 200 Hz, producing smoother motion while keeping approximately the recorded duration.

Stop replay with `Ctrl-C`. The arms remain enabled and hold the final position.

## 6. Stop, unlock, and return to sync mode

To remove follower motor torque while the mode 1 terminal is running:

```bash
rostopic pub --once /enable_flag std_msgs/Bool "data: false"
```

Support the arms first because they may fall. Disabled arms can still have gearbox resistance.

To return to the tested native master/follower sync workflow:

1. Stop the replay terminal.
2. Stop the Piper mode 1 terminal.
3. Turn arm power off.
4. Reconnect/unplug-replug the master/control arms.
5. Turn arm power on.
6. Start read mode again:

```bash
cd ~/cobot_magic/Piper_ros_private-ros-noetic
source devel/setup.bash
roslaunch piper start_ms_piper.launch mode:=0 auto_enable:=false
```

Move each master arm slightly and confirm that the matching follower arm synchronizes.

## 7. Local edits

`Piper_ros_private-ros-noetic/can_muti_activate.sh`:

```bash
USB_PORTS["1-2:1.0"]="can_left:1000000"
USB_PORTS["1-3:1.0"]="can_right:1000000"
```

`collect_data/replay_data.py`:

- Added `--control_rate`.
- Replaced fixed 20-step replay interpolation with steps calculated from `control_rate / frame_rate`.
