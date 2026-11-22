# 用无人船工程学 C++ 和 ROS 2

从船的 `pose_node.cpp` 开始。你会先跟着程序看一次收消息、存数据、算位置和发消息，再用你熟悉的 C 写法理解类、引用和回调。沿着这个节点向上找传感器，向下找融合结果，最后回来看构建、调试和实船迁移。以下内容使用 Ubuntu 22.04、ROS 2 Humble 和 C++17。

## 两天读一遍

按每天约 3～4 小时安排。第一遍边读正文边在船工程里搜索文中给出的函数名，碰到符号可以随时回到对应一节。

| 时间 | 怎么读 |
| --- | --- |
| 第一天前半段 | [从定位节点入手](ros2/01-pose-node.md)。打开 `pose_node.cpp`，先找末尾的 `main`，再找构造函数和 `subscribe_scalar`。看到 `msg.data` 后，接着看[如何打印这个数](ros2/00-cpp-output.md)。 |
| 第一天后半段 | [用 C 的经验读懂 C++](ros2/02-cpp-in-the-ship.md)。重点读引用、`std::optional`、`AngleFilter`、构造函数、`SharedPtr` 和 `[this]`。把每个概念放回定位节点对应的一行。 |
| 第二天前半段 | [读懂节点怎样通信](ros2/03-ros2-communication.md)，再看[船的定位数据怎么来](ros2/05-localization.md)。把缆绳角度、缆长、深度、位置和可选融合输出画在同一条链路上。 |
| 第二天后半段 | [看懂文件夹、编译和启动](ros2/04-workspace.md)，最后读[实船、移植和调试](ros2/06-migration-and-debugging.md)。从 launch 找节点，辨认哪些读数需要由外部节点提供，再看相对深度停更的例子。 |

## 课程目录

| 课次 | 讲解 | 对应的船代码 |
| --- | --- | --- |
| 01 | [用 C++ 打印一条缆长消息](ros2/00-cpp-output.md) | `pose_node.cpp` 的 `msg.data` |
| 02～04 | [类型与单位、函数与引用、数组与可选值](ros2/02-cpp-in-the-ship.md) | `pose_node.cpp`、`imu_orientation.hpp`、`cable_angle_calibrator.cpp` |
| 05～08 | [类与对象、构造与析构、智能指针、lambda 回调](ros2/02-cpp-in-the-ship.md) | `AngleFilter`、`PoseNode` 和 `subscribe_scalar` |
| 09～11 | [Windows、WSL 和香橙派、文件夹、编译与链接](ros2/04-workspace.md) | `orangpi_ros2`、`package.xml`、`CMakeLists.txt` |
| 12～15 | [通信接口、编写节点、执行器和定时器、服务质量](ros2/03-ros2-communication.md) | 四个节点的 `main` 与构造函数、母船航向定时器 |
| 16 | [参数、命名空间、重映射和 launch](ros2/04-workspace.md#第-16-课--通过-launch-组装多个节点) | `localization.launch.xml` 和 `ekf.yaml` |
| 17～20 | [惯性测量、角度标定、相对位置、坐标变换和融合](ros2/05-localization.md) | 四个节点、`imu_orientation.hpp`、`ekf.yaml` |
| 21～24 | [实船设备、ROS 1 到 ROS 2、诊断与测试](ros2/06-migration-and-debugging.md) | 原 `auto_pwm.cpp`、`motor.cpp` 与新 `test_localization.py` |

[直接开始读定位节点 →](ros2/01-pose-node.md)

## 对照项目源码

教程中的文件路径以船工程为准。ROS 2 包位于 `orangpi_ros2/src/usv_localization/`。仓库需要访问权限，在自己的电脑上也可以用同名文件对照。当前 ROS 2 版本包含四个定位计算节点和可选的扩展卡尔曼滤波器。实船章节用 ROS 1 备份解释推进与缆绳电机。相对深度的发布端和新系统上的硬件接入仍待确认。[无人船工程](https://github.com/Va1111n/ros2-ship/tree/main/orangpi_ros2)
