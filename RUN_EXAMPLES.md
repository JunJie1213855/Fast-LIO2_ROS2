# 运行示例

本文档给出两种激光雷达配置的完整运行流程。其他雷达配置只需替换对应的 `config_file` 即可。

---

## 通用前置步骤

每次打开新终端都需要：

```bash
cd ~/rosws/fast_lio2_ros2
source install/setup.bash

# 如果 Livox 驱动未写入 ~/.bashrc 则还需：
source <livox_ws>/install/setup.bash
```

---

## 示例 1: Livox Mid-360 在线建图（fast_lio 包）

### 1.1 确认雷达连接

Livox Mid-360 默认 IP 为 `192.168.1.1xx`，确保网口已配置好。

### 1.2 启动 FAST-LIO

```bash
# 终端 1: 启动建图节点 + RViz
ros2 launch fast_lio mapping.launch.py config_path:=<fast_lio_install>/share/fast_lio/config/mid360.yaml
```

参数说明：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `config_path` | `config/mid360.yaml` | 配置文件路径 |
| `rviz` | `true` | 是否打开 RViz |
| `rviz_cfg` | `rviz/fastlio.rviz` | RViz 配置文件 |
| `use_sim_time` | `false` | 是否使用仿真时间 |

### 1.3 启动 Livox 驱动

```bash
# 终端 2: 启动 Mid-360 驱动
ros2 launch livox_ros_driver2 msg_MID360_launch.py
```

### 1.4 效果验证

在 RViz 中可以看到：
- `/cloud_registered` — 全局坐标系下的配准点云（地图）
- `/path` — 运动轨迹
- `/Laser_map` — 增量地图可视化

> **重要**：建图过程中 IMU 和 LiDAR 必须**时间同步**。如果出现漂移，检查 `config/mid360.yaml` 中 `extrinsic_T` 和 `extrinsic_R` 外参是否正确。

---

## 示例 2: RoboSense Airy 在线建图（fast_lio_robosense 包）

### 2.1 确认雷达话题

RoboSense Airy 默认发布话题：
- 点云：`/rslidar_points`（可在配置中修改 `common.lid_topic`）
- IMU：`/rslidar_imu_data`（可在配置中修改 `common.imu_topic`）

### 2.2 启动建图

```bash
# 在线建图（需要雷达已启动并发布数据）
ros2 launch fast_lio_robosense mapping_robosense_airy.launch.py
```

### 2.3 保存地图

```bash
# 建图结束后保存 PCD 文件
ros2 launch fast_lio_robosense mapping_robosense_airy.launch.py map_file_path:=/home/ros/maps/my_map.pcd

# 在另一个终端触发保存
ros2 service call /map_save std_srvs/srv/Trigger
```

### 2.4 RoboSense Airy 关键配置项

编辑 `config/robosenseAiry.yaml`：

```yaml
preprocess:
    lidar_type: 5       # 5 = RoboSense Airy
    scan_line: 96       # 96 线

common:
    lid_topic: "/rslidar_points"
    imu_topic: "/rslidar_imu_data"
    time_sync_en: false
    time_offset_lidar_to_imu: 0.0

mapping:
    extrinsic_T: [0.0, 0.0, 0.0]   # LiDAR 相对于 IMU 的平移
    extrinsic_R: [1., 0., 0.,       # LiDAR 相对于 IMU 的旋转矩阵
                  0., 1., 0.,
                  0., 0., 1.]
    extrinsic_est_en: true          # 是否在线估计外参
```

---

## 示例 3: Rosbag 离线建图

适用于录制好的数据包回放建图。

### 3.1 下载示例 Rosbag

- **Livox Avia**: [Google Drive](https://drive.google.com/drive/folders/1CGYEJ9-wWjr8INyan6q1BZz_5VtGB-fP) （ROS1 bag，需转换为 ROS2）
- **Velodyne NCLT**: [原始数据](http://robots.engin.umich.edu/nclt/) | [ROS1 Bag](https://drive.google.com/drive/folders/1VBK5idI1oyW0GC_I_Hxh63aqam3nocNK)

> **ROS1 Bag 转 ROS2**: 使用 [rosbags](https://ternaris.gitlab.io/rosbags/topics/convert.html) 工具：
> ```bash
> pip install rosbags
> rosbags-convert old_bag.bag new_bag/  # 输出 ROS2 格式
> ```

### 3.2 回放建图

```bash
# 终端 1: 启动 FAST-LIO
ros2 launch fast_lio mapping.launch.py config_path:=<path_to_config>.yaml

# 终端 2: 播放 bag
ros2 bag play <your_ros2_bag_dir> --clock
```

> **注意**：bag 播放时务必加 `--clock` 参数，并且建图节点的 `use_sim_time` 会自动设为 `true`。

---

## 示例 4: Velodyne HDL-32E / Ouster OS1-64

```bash
# Velodyne 建图
ros2 launch fast_lio mapping_velodyne.launch

# Ouster 建图
ros2 launch fast_lio mapping_ouster64.launch
```

---

## 常见问题

### Q1: 启动时提示 "Failed to find match for field 'time'"

说明 LiDAR 点云没有时间戳字段。对于 Livox 系列雷达，务必使用 `livox_lidar_msg.launch` 而非 `livox_lidar.launch` 启动驱动。

### Q2: 地图漂移严重

1. 检查 IMU 和 LiDAR 是否时间同步
2. 确认外参 `extrinsic_T` / `extrinsic_R` 是否正确
3. 如果外参已知且稳定，建议设置 `extrinsic_est_en: false`

### Q3: 建图帧率太低

检查 `publish_freq` 参数，Livox 驱动默认可能是 10Hz，可以调高。

### Q4: 编译找不到 livox_ros_driver2

确保 `livox_ros_driver2` 工作空间已 source：

```bash
source <livox_ws>/install/setup.bash
colcon build --symlink-install
```

### Q5: 大量点云时内存不足

- 关闭 `effect_map_en` 和 `dense_publish_en` 减少输出
- 设置 `pcd_save.interval` 将 PCD 分段保存，避免单个文件过大

---

## 支持雷达一览

| 雷达型号 | 类型 | lidar_type | 包 | 配置文件 |
|----------|------|-----------|----|----|
| Livox Avia | 固态 | 1 | fast_lio | `config/avia.yaml` |
| Livox Mid-360 | 固态 | 1 | fast_lio | `config/mid360.yaml` |
| Livox Horizon | 固态 | 1 | fast_lio | `config/horizon.yaml` |
| Velodyne HDL-32E | 机械旋转 | 2 | fast_lio | `config/velodyne.yaml` |
| Ouster OS1-64 | 机械旋转 | 3 | fast_lio | `config/ouster64.yaml` |
| RoboSense Airy | 固态 | 5 | fast_lio_robosense | `config/robosenseAiry.yaml` |
