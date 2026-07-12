# FAST-LIO2 ROS2 工作空间

本仓库包含两个基于 [FAST-LIO2](https://github.com/hku-mars/FAST_LIO) 的 ROS2 Humble 建图节点：

| 包名 | 路径 | 说明 |
|------|------|------|
| `fast_lio` | [FAST_LIO_ROS2/](FAST_LIO_ROS2/) | 通用版本，支持 Livox（Avia/Mid-360/Horizon）、Velodyne、Ouster |
| `fast_lio_robosense` | [FAST_LIO_ROBOAIRY/](FAST_LIO_ROBOAIRY/) | 扩展版本，增加 RoboSense Airy（96线）激光雷达支持 |

> 原始 FAST_LIO 已有官方 ROS2 分支，此仓库在此基础上增加 RoboSense Airy 适配和配置调优。

## 原理

FAST-LIO2 是一种**紧耦合迭代卡尔曼滤波器 LiDAR-惯性里程计**，特点：

- 使用 **ikd-Tree** 增量建图，支持 100Hz+ 激光雷达频率
- **直接法** 点云匹配（scan-to-map），无需特征提取，精度更高
- 支持多种机械旋转式和固态激光雷达
- 支持外部 IMU
- 支持 ARM 平台（树莓派 4B、NVIDIA TX2、Khadas VIM3）

### 论文

- [FAST-LIO2: Fast Direct LiDAR-inertial Odometry](FAST_LIO_ROS2/doc/Fast_LIO_2.pdf)
- [FAST-LIO: A Fast, Robust LiDAR-inertial Odometry Package by Tightly-Coupled Iterated Kalman Filter](https://arxiv.org/abs/2010.08196)

## 先决条件

| 依赖 | 版本要求 |
|------|----------|
| Ubuntu | >= 20.04 |
| ROS2 | >= Foxy（推荐 Humble） |
| PCL | >= 1.8 |
| Eigen | >= 3.3.4 |
| [livox_ros_driver2](https://github.com/Livox-SDK/livox_ros_driver2) | 最新（必装） |

### 安装 livox_ros_driver2

```bash
git clone https://github.com/Livox-SDK/livox_ros_driver2.git
# 按官方文档编译，确保 source 到工作空间中
```

> 如果遇到单位或话题兼容性问题，可以尝试[修改版驱动](https://github.com/Ericsii/livox_ros_driver2/tree/feature/use-standard-unit)。

## 编译

```bash
# 1. 进入 ROS2 工作空间
cd ~/rosws/fast_lio2_ros2

# 2. 确保 livox_ros_driver2 已 source
#    (如果已写入 ~/.bashrc 则可跳过)
source <livox_ws>/install/setup.bash

# 3. 初始化子模块 (ikd-Tree)
cd FAST_LIO_ROS2
git submodule update --init --recursive
cd ../FAST_LIO_ROBOAIRY
git submodule update --init --recursive
cd ..

# 4. 安装 ROS 依赖
rosdep install --from-paths . --ignore-src -y

# 5. 编译
colcon build --symlink-install

# 6. Source
source install/setup.bash
```

## 项目结构

```
fast_lio2_ros2/
├── FAST_LIO_ROS2/             # fast_lio 包（通用版）
│   ├── CMakeLists.txt
│   ├── package.xml
│   ├── config/                # 各雷达的 YAML 配置文件
│   │   ├── avia.yaml          # Livox Avia
│   │   ├── mid360.yaml        # Livox Mid-360
│   │   ├── horizon.yaml       # Livox Horizon
│   │   ├── velodyne.yaml      # Velodyne
│   │   └── ouster64.yaml      # Ouster OS1-64
│   ├── launch/                # 启动文件
│   ├── src/                   # 核心源码
│   │   ├── laserMapping.cpp   # 主建图节点
│   │   ├── preprocess.cpp/h   # 点云预处理
│   │   └── IMU_Processing.hpp # IMU 处理
│   ├── include/               # 头文件（ikd-Tree、数学库等）
│   ├── rviz/                  # RViz2 配置文件
│   ├── Log/                   # 调试日志与分析脚本
│   └── msg/                   # 自定义消息 (Pose6D)
│
└── FAST_LIO_ROBOAIRY/         # fast_lio_robosense 包（RoboSense 版）
    ├── CMakeLists.txt
    ├── package.xml
    ├── config/
    │   └── robosenseAiry.yaml # RoboSense Airy 配置
    ├── launch/
    │   ├── mapping.launch.py
    │   └── mapping_robosense_airy.launch.py
    ├── src/                   # 与 FAST_LIO_ROS2 核心代码一致
    ├── include/
    ├── rviz/
    └── msg/
```

## 快速运行

详细运行示例请参见 [RUN_EXAMPLES.md](RUN_EXAMPLES.md)。

## 日志与调试

程序运行时会在 `Log/` 目录输出调试数据：

| 文件 | 说明 |
|------|------|
| `Log/mat_pre.txt` | 滤波前状态矩阵 |
| `Log/mat_out.txt` | 滤波后状态矩阵 |
| `Log/dbg.txt` | 通用调试信息 |
| `Log/imu.txt` | IMU 数据记录 |
| `Log/pos_log.txt` | 位姿轨迹日志 |

使用 `Log/plot.py` 可视化调试数据：

```bash
python3 Log/plot.py
```

## PCD 地图保存

设置启动参数 `pcd_save_en: true`，程序退出后会在 `PCD/` 目录下保存全局点云地图：

```bash
pcl_viewer PCD/scans.pcd
```

在 `pcl_viewer` 中按数字键 `1`~`5` 切换着色模式（随机/X/Y/Z/强度）。

## 许可

BSD License

## 致谢

- 原始 FAST-LIO: [hku-mars/FAST_LIO](https://github.com/hku-mars/FAST_LIO)
- ikd-Tree: [hku-mars/ikd-Tree](https://github.com/hku-mars/ikd-Tree)
- ROS2 移植: [Ericsii/FAST_LIO](https://github.com/Ericsii/FAST_LIO)
- RoboSense 适配: 本仓库维护者
