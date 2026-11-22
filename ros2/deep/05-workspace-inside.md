# 从打开工程目录到真正启动一个节点

本篇对应第 09～11、16 课。目标是拿到带 `package.xml` 的目录，能解释编译器读什么、产物放在哪，命令怎样找到它、启动文件怎样把四个程序接起来。

## 第 09 课 · 三个运行地点和四种文件

Windows 保存和编辑 `study` 学习仓库，也可以持有船的源码。WSL（Windows Subsystem for Linux，适用于 Linux 的 Windows 子系统）中的 Ubuntu 22.04 提供 Linux 环境。进入 WSL 后执行的 `g++`、`colcon` 和 `ros2` 是 Linux 进程。它们能通过 `/mnt/c` 访问 Windows 的 C 盘，也能访问 Linux 自己的主目录。香橙派 5 Plus 是装在船上的独立机器，它运行自己的 Linux，能看到板端串口和引脚。

| 文件或资源 | 常见位置 | 由谁读取 |
| --- | --- | --- |
| 课程 Markdown | 学习仓库 | GitHub 或编辑器，给人阅读 |
| `.cpp` 与 `.hpp` | 船仓库 | 编译器，读取代码和声明 |
| 可执行文件 | 工作空间安装区 | Linux 加载程序，决定执行环境 |
| `launch` 和 YAML 配置 | 源码中一份，构建安装区通常还有一份 | ROS 2 启动工具和节点 |
| `/dev/...` 设备入口 | 对应设备所在的 Linux 主机 | 设备驱动与用户态程序 |

当你在 Windows 浏览到 `pose_node.cpp`，只证明文件在这里。运行在哪要看终端和启动的进程。编辑器里保存代码后，原来运行中的节点进程仍在执行原来的程序。需要构建并重新启动，才会看到修改。

### 读懂终端里最常见的三个词

- `~` 是当前 shell 所用用户的主目录缩写。在 WSL 中，它通常指 Linux 的 `/home/...`，Windows 用户目录是另一处位置。进程相对路径以它的当前工作目录为起点。
- `PATH` 是一组找可执行命令的目录。输入命令时 shell 从这里找工具。包的寻找还涉及 ROS 2/ament 的索引和环境，找到 `ros2` 工具并不保证一定能找到自建包。
- `source /opt/ros/humble/setup.bash` 把 ROS 2 环境设置应用到当前 Bash。之后 `source` 工作空间的 `install/setup.bash`，可以使用已安装的包。新开普通终端不会继承前一个终端后来设置的环境。每个运行节点或检查话题的终端都要确认自己使用的环境。

Docker 容器再增加一个文件与依赖边界。容器中的 `/work` 取决于容器的挂载和工作目录，不能从名字推断它等于你电脑上的同名目录。排查环境先问“这条命令在谁的终端，由哪个进程执行，看到哪个文件系统和哪些设备”。

### 为什么电脑能编，板子还要再编

常见桌面处理器的指令集是 x86-64，香橙派目标为 ARM64。两台机器可以阅读同一份 C++ 文本，但分别需要合适的目标机器代码、系统库和 ROS 2 依赖。迁移测试分两层，电脑上软件行为能用模拟输入检查，板上的部署、设备权限和实际测量再单独验证。该项目课程使用 Ubuntu 22.04、ROS 2 Humble、C++17。换发行版时要一起核对系统、依赖包和 API（Application Programming Interface，应用程序接口）。

## 第 10 课 · 包根、源目录和安装区

从仓库打开 `orangpi_ros2/`，可以把它当成这次要构建的一组软件的根。它下面的第一层 `src/` 收集功能包。找到 `usv_localization/package.xml` 的位置，就找到了该包根目录。该目录里面的第二层 `src/` 才装这个包的 C++ 源文件。两个 `src` 约定相同名字，层级角色不同。

```text
船仓库
└─ orangpi_ros2                   工作空间
   └─ src                         给构建工具找功能包的地方
      └─ usv_localization         功能包
         ├─ package.xml           声明包名、构建类型、依赖与元信息
         ├─ CMakeLists.txt        配置编译目标、链接、安装和测试
         ├─ src                   本包的实现
         │  ├─ pose_node.cpp
         │  ├─ imu_test1.cpp
         │  ├─ cable_angle_calibrator.cpp
         │  ├─ mother_usv_msg_node.cpp
         │  └─ imu_orientation.hpp
         ├─ launch                组合启动与重映射
         ├─ config                参数
         └─ test                  集成测试
```

看到一个修改需求可以直接找到归属：

| 我希望什么发生变化 | 从哪里入手 | 为什么 |
| --- | --- | --- |
| 改定位计算分支 | `src/pose_node.cpp` | 接收后的计算逻辑在这里 |
| 改四个节点一起启动的配置 | `launch/localization.launch.xml` | 决定起什么进程、传什么启动参数 |
| 改哪些信息给滤波器使用 | `config/ekf.yaml` | 描述输入话题与融合字段 |
| 增加一个 C++ 程序并让包可启动 | `src/*.cpp`、`CMakeLists.txt`，有新依赖时也改 `package.xml` | 源文件需要列入构建和安装 |
| 为共同代码提供头文件 | 参考 `src/imu_orientation.hpp` | 它在本包两处编译时共用，公开给其他包时通常要配置安装头文件和导出 |
| 验证改完还能按安装入口运行 | `test/test_localization.py` | 启动实际安装区的 launch 排查接线或行为回归 |

生成后的 `build/` 存中间产物和 CMake 构建状态。`install/` 存安装产物、启动配置、每个包的索引和环境入口。`log/` 有构建及测试日志。各种产物位置可以由命令指定。因此“工作空间目录”指组织源码的地方，“构建输出在哪”要看实际执行方式。修改 `install/` 下的复制产物可能在下次安装时被覆盖，稳定的修改应该落实到源码树和构建描述。

## 第 11 课 · 从源码走到可执行文件

### 把整条链先念一遍

```text
package.xml 声明依赖和构建类型
  → colcon 发现包、安排先后次序
  → CMake 根据 CMakeLists.txt 配置每项编译与链接
  → 编译器读取每个翻译单元，生成目标代码
  → 链接器配上该程序需要的实现与库
  → 安装规则把程序、配置及索引放进 install
  → 当前终端加载安装环境
  → ros2 按包索引找到并运行可执行文件
```

`.hpp` 给编译器声明接口，或者放适合共享的短函数定义。每个 `.cpp` 与它实际包含的内容组成一个翻译单元。即使每行都能编译，链接还可能因为某实现没加进目标或缺库而失败。安装和查找又是后续步骤。所以“构建失败”和“运行找不到包”须分开追。

如果要在普通项目中演示这一区别，可以用两个 C++ 文件。头文件声明一个函数，第一个源文件调用它，另一份实现定义它。仅编译调用它的文件时，编译器已经有类型信息。把那个目标文件单独链接成程序时还缺函数定义。将实现文件也加入目标，链接器才找得到函数体。

```cpp
// meters.hpp：声明函数接受什么、返回什么。
#pragma once
double centimeters_to_meters(double centimeters);
```

```cpp
// meters.cpp：定义这个函数。编译后提供可链接的实现。
#include "meters.hpp"

double centimeters_to_meters(double centimeters) {
    return centimeters / 100.0;
}
```

```cpp
// main.cpp：只需知道函数的声明就能生成调用。
#include <iostream>
#include "meters.hpp"

int main() {
    std::cout << centimeters_to_meters(500.0) << '\n';
    return 0;
}
```

船的 `read_orientation` 采取另一种组织方式，直接把短定义放进共享头并写 `inline`。这里共享的目的是让两路 IMU 节点遵循同一种输入检查约定，而不是重复维护两份检查。

### 用包清单读出三种依赖

在 `package.xml` 搜 `ament_cmake`、`rclcpp` 与 `robot_localization`，它们代表不同需要：

1. `buildtool_depend` 表明用什么构建基础。这个包用 `ament_cmake`。
2. 普通 `depend` 声明 C++ 代码要用的接口，如 `rclcpp`、消息包、`tf2`。
3. `exec_depend` 表明运行入口还会用的东西。可选 EKF 由 launch 启动已有程序，定位节点本身没有把滤波器源码编进可执行文件。
4. `test_depend` 让测试工具、Python 客户端与测试所需消息可以单独准备。

`rosdep` 读取这些声明，帮助把依赖映射到目标系统可安装的包。它负责准备依赖。`colcon` 负责有顺序地构建和测试工作空间。`CMake` 描述每个包内部怎么构建。文件、声明和工具处于不同环节。新增一个消息依赖时，包清单和 CMake 目标要一致，分别让包管理和编译器知道。

### 按 pose_node 追一遍构建描述

打开 `CMakeLists.txt`，在循环列表里找 `pose_node`，然后逐项看：

| 构建动作 | 对此目标意味着什么 |
| --- | --- |
| `find_package(... REQUIRED)` | 配置时必须找到所列的依赖包信息 |
| `add_executable` | 将对应的 `src/pose_node.cpp` 生成 `pose_node` 可执行目标 |
| `target_compile_features(... PRIVATE cxx_std_17)` | 此目标编译至少需要 C++17，`std::optional` 等接口才可用 |
| `ament_target_dependencies` | 把对应依赖传给目标。定位节点使用 `geometry_msgs` 保存位置消息 |
| `install(TARGETS ...)` | 把生成的程序装进该包的可执行文件位置 |
| `install(DIRECTORY launch config ...)` | 把启动文件和参数文件一并安装 |
| `ament_package()` | 写出 ROS 2 包所需的元信息，使工具能查找 |

四个目标共用同一段循环配置。各自另外声明所需的消息或坐标变换依赖。这样可以只运行定位节点验证几何解算，也能一次启动完整链路。

### 看到错误文字时用它判断环节

| 现象 | 所处环节 | 第一个要看的证据 |
| --- | --- | --- |
| CMake 提示找不到依赖包配置 | 配置 | 目标 ROS 发行版是否加载，依赖是否安装并正确声明 |
| 编译提示找不到头文件 | 编译 | 包与目标依赖、包含路径和文件名 |
| 编译说类型或成员不存在 | 编译 | 消息类型、命名空间、字段拼写，是否混写 ROS 1 与 ROS 2 接口 |
| 链接提示 `undefined reference` | 链接 | 函数定义、源文件是否属于目标、库是否链接 |
| `ros2 run` 找不到包 | 查找 | 安装成功没有，当前终端加载了哪个工作空间 |
| 包在，但启动提示可执行文件不在 | 安装与启动 | 目标名、安装规则和 `exec` 一致吗 |
| 启动成功，还像旧版 | 运行 | 进程是否重启、是否从别的覆盖工作空间启动 |
| 程序有输出，但没有位置 | 业务逻辑 | 回到三路话题和数据条件 |

## 第 16 课 · launch 与参数如何把程序接好

Launch 是一份如何启动的描述。源代码决定“程序会怎么处理传入的参数”，launch 决定“这次启动传什么并运行几个进程”。这个工程用 XML（Extensible Markup Language，可扩展标记语言）写启动描述，用 YAML（YAML Ain't Markup Language，一种常用配置格式）写成组的滤波参数。

### 两种参数怎样走完一整条路

以母船坐标系名为例，依次找：

```text
用户传入 mother_frame 参数，或采用 launch 中的默认值
  → localization.launch.xml 读这个 launch 参数
  → 在 pose_node 进程上设置 mother_frame_id 节点参数
  → PoseNode 构造函数 declare_parameter 取得本次值
  → 保存到成员 frame_
  → 发布时写到 pose.header.frame_id
```

这条说明了两点。`mother_frame` 与 `mother_frame_id` 分别处在 launch 和节点两层。改变启动参数，会影响下次构建消息所用的默认坐标标签。源码只在构造时取值并缓存，运行时即使参数系统接受更新，字段也未必立即跟着变。要支持热更新需让应用明确处理更新，并核对更换坐标标签时坐标值如何转换。

`with_ekf` 是另一个 launch 开关。默认四个计算节点启动，打开它才加滤波进程与静态变换。若没有看到 `/rov/odometry/filtered`，先检查这次进程是否真的启动了 EKF，然后再进 YAML 核对输入和坐标。

### 名字和重映射是怎么改变实际接线的

| 源码名字与启动方式 | 最终话题示例 | 对船的含义 |
| --- | --- | --- |
| 相对名 `sensor/cable_azimuth`，在根命名空间 | `/sensor/cable_azimuth` | 默认能接到定位节点 |
| 相对名 `sensor/cable_azimuth`，放进 `/mother` | `/mother/sensor/cable_azimuth` | 需要让订阅端也使用对应最终名字 |
| 绝对名 `/sensor/cable_azimuth`，在 `/mother` 节点内 | `/sensor/cable_azimuth` | 简单改命名空间不改变这个绝对话题 |
| 滤波器 `odometry/filtered` 经 launch 重映射 | `/rov/odometry/filtered` | 既有滤波程序按新接口名输出 |

重映射重新接到不同的名字，既不会换消息字段，也不会把度换成弧度。命名空间可以组织多台设备的局部接口。把所有绝对名字复制进两个节点时，两份输入可能混到同一个话题；分船部署前要复核实际解析出来的名字。

### 如何从 launch 追到一个函数

看到 `pkg="usv_localization"`，告诉你从哪个包查。看到 `exec="pose_node"`，告诉你在 CMake 找哪个目标。找到 `add_executable` 对应源文件后，依次在 `main` 定位节点类，再到构造函数核对参数、发布器和订阅器。对滤波器而言，执行程序属于另一个包，接线更多由 YAML 指定。参数值、参数所属节点、最终话题名三项都匹配，才能去追算法输出。

[继续读定位算法](06-localization-step-by-step.md) · [课程首页](../../README.md)
