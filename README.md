# FAST-LIO2 ROS2 建图

基于 [FAST-LIO2](https://github.com/hku-mars/FAST_LIO) 的 ROS2 Humble 建图节点，兼容 **Livox、Velodyne、Ouster、RoboSense Airy** 等多种激光雷达。

> 本仓库只保留一个包：`fast_lio_robosense`（路径 `FAST_LIO_ROBOAIRY/`）。
> 依赖库 **ikd-Tree** 与 **IKFoM toolkit** 已随仓库打包在 `include/` 下，**无需递归 clone / submodule 初始化**。

## 特性

- 基于 FAST-LIO2 ROS2 Humble 版本，紧耦合迭代卡尔曼滤波 LiDAR-惯性里程计
- **兼容多种激光雷达**：Livox（Avia / Mid-360 / Horizon）、Velodyne、Ouster、RoboSense Airy
- 新增 **RoboSense Airy（96 线）** 支持（`lidar_type: 5`）
- 使用 **ikd-Tree** 增量建图，支持高频率激光雷达
- **直接法**点云匹配（scan-to-map），无需特征提取，精度更高

### 论文

- [FAST-LIO2: Fast Direct LiDAR-inertial Odometry](https://arxiv.org/abs/2107.06829)
- [FAST-LIO: A Fast, Robust LiDAR-inertial Odometry Package by Tightly-Coupled Iterated Kalman Filter](https://arxiv.org/abs/2010.08196)

## 先决条件

| 依赖 | 要求 |
|------|------|
| Ubuntu | 22.04 |
| ROS2 | Humble |
| PCL | >= 1.8 |
| Eigen | >= 3.3.4 |
| [livox_ros_driver2](https://github.com/Livox-SDK/livox_ros_driver2) | 必装（`package.xml` 中为 `<depend>`） |
| OpenMP | 编译需要（`CMakeLists.txt` 中 `find_package(OpenMP)`） |

> 查看 PCD 地图的脚本 `view_pcd.py` 依赖 [Open3D](https://www.open3d.org/)（可选）：
> ```bash
> pip install open3d numpy
> ```

## 目录结构

```
src/
└── FAST_LIO_ROBOAIRY/           # fast_lio_robosense 包
    ├── CMakeLists.txt
    ├── package.xml
    ├── config/                  # 各雷达 YAML 配置
    │   ├── avia.yaml            # Livox Avia
    │   ├── horizon.yaml         # Livox Horizon
    │   ├── mid360.yaml          # Livox Mid-360
    │   ├── mid360_sim.yaml      # Livox Mid-360（仿真）
    │   ├── velodyne.yaml        # Velodyne
    │   ├── ouster64.yaml        # Ouster OS1-64
    │   └── robosenseAiry.yaml   # RoboSense Airy
    ├── launch/
    │   ├── mapping.launch.py                  # 通用建图（默认 mid360.yaml）
    │   ├── mapping_robosense_airy.launch.py   # RoboSense Airy 建图
    │   └── gdb_debug_example.launch
    ├── src/
    │   ├── laserMapping.cpp     # 主建图节点
    │   ├── preprocess.cpp/h     # 点云预处理
    │   └── IMU_Processing.hpp   # IMU 处理
    ├── include/                 # 头文件（已打包，无需 submodule）
    │   ├── ikd-Tree/            # ikd-Tree 增量 KD 树
    │   ├── IKFoM_toolkit/       # 迭代卡尔曼滤波流形工具箱
    │   ├── so3_math.h / use-ikfom.hpp / common_lib.h / Exp_mat.h
    │   └── matplotlibcpp.h
    ├── msg/                     # 自定义消息 Pose6D.msg
    ├── rviz/                    # fastlio.rviz
    ├── rviz_cfg/                # loam_livox.rviz
    ├── PCD/                     # 保存的点云地图
    └── view_pcd.py              # PCD 查看脚本（Open3D）
```

## 编译

```bash
# 1. 进入 ROS2 工作空间
cd ~/rosws/fast_lio2_ros2

# 2. 确保 livox_ros_driver2 已 source（已写入 ~/.bashrc 可跳过）
source <livox_ws>/install/setup.bash

# 3. 安装 ROS 依赖
rosdep install --from-paths src --ignore-src -y

# 4. 编译（依赖库已打包，无需 submodule 初始化）
colcon build --symlink-install --packages-select fast_lio_robosense

# 5. Source
source install/setup.bash
```

## 快速运行

### 通用建图（Livox / Velodyne / Ouster）

用 `mapping.launch.py` 并通过 `config_file` 指定雷达配置：

```bash
ros2 launch fast_lio_robosense mapping.launch.py config_file:=mid360.yaml
```

### RoboSense Airy 在线建图

```bash
ros2 launch fast_lio_robosense mapping_robosense_airy.launch.py
```

默认读取 `config/robosenseAiry.yaml`。关键配置项：

```yaml
preprocess:
    lidar_type: 5           # 5 = RoboSense Airy
    scan_line: 96           # 96 线

common:
    lid_topic: "/rslidar_points"
    imu_topic: "/rslidar_imu_data"
```

### 启动参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `config_path` | `<share>/fast_lio_robosense/config` | 配置目录 |
| `config_file` | `mid360.yaml`（airy 版为 `robosenseAiry.yaml`） | 配置文件名 |
| `rviz` | `true` | 是否打开 RViz |
| `rviz_cfg` | `rviz/fastlio.rviz` | RViz 配置文件 |
| `use_sim_time` | `false` | 是否使用仿真时钟 |

更完整的运行示例（含 rosbag 离线建图、常见问题）见 [RUN_EXAMPLES.md](RUN_EXAMPLES.md)。

## 发布话题与服务

| 名称 | 类型 | 说明 |
|------|------|------|
| `/cloud_registered` | `sensor_msgs/PointCloud2` | 全局坐标系配准点云（地图） |
| `/cloud_registered_body` | `sensor_msgs/PointCloud2` | IMU body 坐标系点云 |
| `/cloud_effected` | `sensor_msgs/PointCloud2` | 有效点云 |
| `/Laser_map` | `sensor_msgs/PointCloud2` | 增量地图 |
| `/Odometry` | `nav_msgs/Odometry` | 里程计（`camera_init` → `body`） |
| `/high_frequency_odometry` | `nav_msgs/Odometry` | 高频里程计 |
| `/path` | `nav_msgs/Path` | 运动轨迹 |
| `/map_save`（服务） | `std_srvs/srv/Trigger` | 触发保存 PCD 地图 |

## PCD 地图保存

1. 启动时指定保存路径（`mapping_robosense_airy.launch.py` 支持）：

   ```bash
   ros2 launch fast_lio_robosense mapping_robosense_airy.launch.py map_file_path:=/path/to/map.pcd
   ```

2. 在另一个终端触发保存：

   ```bash
   ros2 service call /map_save std_srvs/srv/Trigger
   ```

   或在配置中开启 `pcd_save.pcd_save_en: true`，退出时自动保存到 `PCD/`。

3. 查看点云：

   ```bash
   # 使用脚本（支持体素降采样 / Z 轴过滤）
   python3 view_pcd.py PCD/robo_map.pcd --voxel 0.1 --zmin -2.0

   # 或使用 pcl_viewer
   pcl_viewer PCD/robo_map.pcd
   ```

   `view_pcd.py` 参数：`--zmin/--zmax` 按高度裁剪，`--voxel` 体素降采样，`--no-view` 只打印统计。

## 许可

BSD License

## 致谢

- 原始 FAST-LIO: [hku-mars/FAST_LIO](https://github.com/hku-mars/FAST_LIO)
- ikd-Tree: [hku-mars/ikd-Tree](https://github.com/hku-mars/ikd-Tree)
- ROS2 移植: [Ericsii/FAST_LIO](https://github.com/Ericsii/FAST_LIO)
- RoboSense Airy 适配: 本仓库维护者
