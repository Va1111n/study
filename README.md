# 用无人船工程学 C++ 和 ROS 2

从船的 `pose_node.cpp` 开始。假设母船已经知道缆长 5 米，过一会儿才知道深度 3 米，最后收到标定后方位角 0 弧度。为什么前两次没有结果，第三次却发出了 `(4, 0, −3)`？逐行跟读会带你从 `main` 跟到最后一次 `publish`。遇到 `std::cout`、`class`、`&`、`[this]` 和 `->`，随时回看对应的 C 写法和完整小程序。课程使用 Ubuntu 22.04、ROS 2 Humble 和 C++17。

**第一次打开，从 [定位节点逐行跟读](ros2/deep/01-first-read.md) 开始。** 想先弄懂打印、构造函数或话题时，可以跳到下表。看到不熟悉的工程简称和符号，可随时翻[源码、写法和缩写索引](ros2/deep/19-source-map-and-glossary.md)。

## 两天读一遍

按每天约 3～4 小时安排。这两天用来走通工程主线，标定推导、完整教学包、测试、实船迁移等详解可以带着问题回查。

| 时间 | 怎么读 |
| --- | --- |
| 第一天前半段 | [从定位节点逐行入手](ros2/deep/01-first-read.md)。先找 `main`，然后找构造函数和 `subscribe_scalar`。读到 `msg.data` 后接着看[怎么打印数值](ros2/deep/02-cpp-symbols.md)。 |
| 第一天后半段 | [从 C 结构体读到类和回调](ros2/deep/03-objects-and-callbacks.md)。再看[不依赖 ROS 的三路定位程序](ros2/deep/09-localization-without-ros.md)，专心追哪次函数调用改了哪个对象。 |
| 第二天前半段 | 先看[一个话题怎样连接两个程序](ros2/deep/04-topic-from-scratch.md)，再看[传感器数值怎样变成位置](ros2/deep/06-localization-step-by-step.md)。 |
| 第二天后半段 | [从工程目录找到启动入口](ros2/deep/05-workspace-inside.md)，然后读[一次完整运行记录](ros2/deep/15-reading-a-real-run.md)。遇到具体问题再翻测试、实船和维护篇。 |

## 24 课详解

| 课次 | 正文 | 带代码或数值的配套阅读 |
| --- | --- | --- |
| 01～04 | [从打印、变量到引用、数组和可选值](ros2/deep/02-cpp-symbols.md) | [逐行定位入门](ros2/deep/01-first-read.md) |
| 05～08 | [C 的结构体与函数指针对照 C++ 类、构造、智能指针与 lambda](ros2/deep/03-objects-and-callbacks.md) | [用普通 C++ 重建三路定位行为](ros2/deep/09-localization-without-ros.md) |
| 09～11、16 | [Windows、WSL、工程目录、构建、参数和 launch](ros2/deep/05-workspace-inside.md) | [从头搭一个功能包](ros2/deep/10-build-a-teaching-package.md)、[从编译到运行怎么读报错](ros2/deep/18-errors-from-compile-to-run.md) |
| 12～15 | [节点、发布订阅、执行器与 QoS](ros2/deep/04-topic-from-scratch.md) | [从头读完一个三路 ROS 2 节点](ros2/deep/21-three-input-ros2-node.md)、[消息字段和接口合同](ros2/deep/12-message-contracts.md) |
| 17～20 | [从缆绳 IMU 到相对位置](ros2/deep/06-localization-step-by-step.md) | [带数字读角度算法](ros2/deep/13-angle-by-numbers.md)、[坐标变换、TF 与 EKF](ros2/deep/14-coordinate-and-fusion.md) |
| 21～22 | [实船硬件与迁移](ros2/deep/08-ship-hardware-and-migration.md) | [设备字节怎样变成缆长话题](ros2/deep/20-device-data-to-topic.md) |
| 23～24 | [从症状查证据，再读项目集成测试](ros2/deep/07-debug-by-evidence.md) | [跑一遍整条链](ros2/deep/15-reading-a-real-run.md)、[时间与输入过期](ros2/deep/11-time-and-freshness.md)、[测试怎么写和怎么读](ros2/deep/16-testing-why-and-how.md)、[三个代码修改案例](ros2/deep/17-code-change-walkthrough.md) |

## 简明回顾

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

[直接开始读定位节点 →](ros2/deep/01-first-read.md)

## 对照项目源码

教程中的文件路径以船工程为准。ROS 2 包位于 `orangpi_ros2/src/usv_localization/`。仓库需要访问权限，在自己的电脑上也可以用同名文件对照。当前 ROS 2 版本包含四个定位计算节点和可选的扩展卡尔曼滤波器。实船章节用 ROS 1 备份解释推进与缆绳电机。相对深度的发布端和新系统上的硬件接入仍待确认。[无人船工程](https://github.com/Va1111n/ros2-ship/tree/main/orangpi_ros2)
