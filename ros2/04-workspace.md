# 从工程文件夹找到程序和配置

这页是简明回顾。扩写版用一个多文件示例讲编译与链接，再顺着 launch 解释参数怎样进入消息字段。[工作空间与构建详解](deep/05-workspace-inside.md)

## 第 09 课 · 源码存在哪里，程序又在哪里跑

| 位置 | 这个项目在这里做什么 |
| --- | --- |
| Windows | 保存和编辑项目文件，管理 Git 仓库，也可以从这里打开 WSL 终端。 |
| WSL（Windows Subsystem for Linux，适用于 Linux 的 Windows 子系统）中的 Ubuntu 22.04 | 提供 Linux 命令、编译器和 ROS 2 Humble，用模拟消息、录包或测试程序运行定位链路。 |
| 香橙派 5 Plus | 船上的 Linux 计算机，实际接串口、飞控、传感器和外设。最终构建和设备联调在板端验证。 |

Windows 磁盘常通过 WSL 的 `/mnt/c` 和 `/mnt/d` 访问。Ubuntu 的 `~` 指当前 Linux 用户的主目录。终端命令在哪个系统里运行，决定它看到哪些路径、进程和设备。`source` 把脚本造成的环境设置应用到当前终端，另开终端需要有相应设置，才能找到 ROS 2 和自建包。环境变量给程序提供搜索路径等信息，源码和可执行文件始终在磁盘上。

如果桌面电脑是 x86-64，香橙派是 ARM64，在桌面 Linux 编译的普通本机可执行文件不能直接当香橙派可执行文件用。可以传源码在板端构建，也可以使用合适的交叉编译工具链。若用 Docker，容器还会有自己的文件路径和依赖环境。容器和 WSL 中的模拟测试能验证计算和通信，板上的串口设备、GPIO 和电机接线则由板端验证。

## 第 10 课 · 为什么有两层 src

```text
orangpi_ros2/                         # 工作空间根目录
├── README.md                        # 工作空间的构建、输入输出和用法
└── src/                             # 放功能包，一个工作空间可以装多个包
    └── usv_localization/            # 无人船定位功能包
        ├── package.xml              # 包名、依赖、构建类型等元信息
        ├── CMakeLists.txt           # 生成哪些目标、编译依赖和安装位置
        ├── src/                     # 这个包自己的 C++ 源文件
        │   ├── pose_node.cpp
        │   ├── imu_test1.cpp
        │   ├── cable_angle_calibrator.cpp
        │   ├── mother_usv_msg_node.cpp
        │   └── imu_orientation.hpp
        ├── launch/localization.launch.xml
        ├── config/ekf.yaml
        └── test/test_localization.py
```

USV 是 Unmanned Surface Vehicle（无人水面艇）。`src` 来自 source，表示源码。外层让构建工具发现各个包，内层把本包的 C++ 文件放在一起。这个包只有一个内部共用头文件，因此跟源文件放在一起。库包要让其他包引用公开头文件时，常另外设置 `include/<包名>/`。本包主要是 C++ 节点，测试使用 Python。

常规工作空间构建后还会出现 `build/`（中间文件）、`install/`（安装产物与环境脚本）和 `log/`（构建或测试记录）。这些路径可以由构建命令调整。改 C++ 行为找源码，改程序依赖找清单与 CMake，改多个节点怎样一起运行找 launch，改滤波参数找 YAML 配置。

## 第 11 课 · 一个 cpp 文件怎样成为一个节点程序

从 C 的经验出发，头文件让编译器知道类型和函数怎么用。编译把各源码单元翻成目标代码，链接把它们需要的库和符号接起来，最后得到可执行文件。如果编译阶段说找不到头文件，检查依赖的开发文件和包含路径。如果链接阶段说 `undefined reference`，检查对应实现和目标的链接依赖。ROS 2 还要安装目标、配置和包索引，命令才能按包名找到它们。

| 组件 | 在这里负责什么 |
| --- | --- |
| `package.xml` | 声明构建工具、代码依赖、运行依赖和测试依赖，供工具识别包与依赖关系 |
| `rosdep` | 从依赖声明解析目标系统可安装的依赖 |
| `CMakeLists.txt` | 告诉 CMake 找哪些包、对每个目标编译哪些源文件、使用哪些依赖、把产物安装到哪里 |
| `ament_cmake` | ROS 2 的 CMake 构建基础，提供包导出和目标依赖等功能 |
| `colcon` | 发现工作空间中的包并按依赖顺序调度构建和测试 |
| `source install/setup.bash` | 把已安装工作空间加入当前终端的搜索环境 |

读原文件时先找 `foreach(node IN ITEMS ...)`。列表中的 `pose_node`、`imu_test1`、`cable_angle_calibrator` 和 `mother_usv_msg_node` 各生成一个同名目标。循环里调用 `add_executable` 指定对应的 `.cpp`，`target_compile_features` 要求 C++17，`ament_target_dependencies` 为目标添加依赖。四个目标都用 `rclcpp` 和 `std_msgs`。位置节点另外依赖 `geometry_msgs`（几何消息），两个读姿态的节点另外依赖 `sensor_msgs`（传感器消息）和 `tf2`（坐标变换库）。

`install(TARGETS ... DESTINATION lib/${PROJECT_NAME})` 把可执行程序装到 ROS 2 查找的地方，`${PROJECT_NAME}` 在这里会展开成 `usv_localization`。`launch` 和 `config` 则装到包的共享目录。源码改完需要重新构建，让安装区反映修改。找不到包时先看构建是否完成、加载的是哪个安装环境。找得到包但启动的表现还是旧的，检查当前进程和实际选中的工作空间。

## 第 16 课 · 通过 launch 组装多个节点

Launch 文件描述一次启动哪些进程、传什么参数以及怎样接线。XML 是 Extensible Markup Language（可扩展标记语言），YAML 是 YAML Ain't Markup Language（YAML 不是标记语言），两者在这里分别描述启动过程和配置值。

- **启动参数**由 launch 处理。`with_ekf` 缺省为 `false`，打开后才增加 EKF（Extended Kalman Filter，扩展卡尔曼滤波器）和一条子船机体到 IMU 的静态坐标变换。
- **节点参数**是运行节点可读取的配置。`mother_frame` 是启动参数，launch 把它传给位置节点的 `mother_frame_id`。构造函数声明并读取这个值，后续写入消息的 `header.frame_id`。这份代码把读取值缓存在 `frame_`，因此不能假定运行时改参数就会立刻改输出。
- **命名空间**为相对名字加前缀。若把角度标定器放进 `/mother`，它的相对输出 `sensor/cable_azimuth` 会变成 `/mother/sensor/cable_azimuth`。`pose_node` 当前订阅绝对名 `/sensor/cable_azimuth`，仍指向原话题。这样拆开启动时要核对两端最终名称。
- **重映射**可以在启动时替换原解析的名称。这个 launch 把滤波器的 `odometry/filtered` 输出接到 `/rov/odometry/filtered`。重映射改变连接端点，单位与坐标仍要核对。
- **参数文件**适合放成组配置。`ekf.yaml` 指定输入话题和融合字段，launch 同时传入母子船坐标系相关参数。修改节点名时，也要确认 YAML 中对应的节点配置能够匹配。

追一个具体问题时可以按三次查找走。先在 launch 找包名、可执行程序名和重映射，再到 CMake 找源文件名，最后到构造函数找创建端点和读取参数的位置。

[继续读定位链路](05-localization.md) · [课程首页](../README.md)
