# 从这里开始 · 读懂船的定位节点

打开船工程的 `orangpi_ros2/src/usv_localization/src/pose_node.cpp`。它收集缆长、方位角和深度，算出子船相对母船的三维位置。第一次读时先找四个位置，顺着程序运行顺序看。[工程中的定位源码](https://github.com/Va1111n/ros2-ship/blob/main/orangpi_ros2/src/usv_localization/src/pose_node.cpp)

| 先找哪里 | 程序在做什么 |
| --- | --- |
| 文件末尾的 `main` | 程序从这里开始，初始化 ROS 2，创建一个 `PoseNode` 对象，交给 `spin` 处理后续消息。类的定义写在文件上面，创建对象发生在 `main` 里面。 |
| `PoseNode() : Node("pose_node")` | 创建对象时会调用构造函数。它先初始化 ROS 2 基类，再设置坐标系参数，创建两路发布器和三路订阅器。 |
| `subscribe_scalar` 中的 `latest = msg.data` | 一条有效消息到达后，回调把消息里的数值写入对应的缓存。对缆长订阅器来说，`latest` 指的就是成员 `length_`。 |
| `publish_position` | 每次有效更新都会检查数据是否齐全，齐全就计算并发出位置。 |

假设缆长消息先到。回调把 `5.0` 写入 `length_`，然后调用 `publish_position()`。因为角度或深度还没到，这次调用直接结束。下次回调进来，`length_` 仍属于同一个节点对象，里面还是 `5.0`。这就是接下来用 C++ 类和 ROS 2 回调要解释的行为。

## 节点、话题和消息

`pose_node` 是一个**节点**，负责定位这一项计算。发布端把一条**消息**送到一个有名字的**话题**，订阅这个话题的节点就能接收它。比如 `/sensor/cable_length` 是话题名，`5.0` 是它某一条消息中的缆长读数。

| 方向 | 话题 | 消息类型 | 字段含义 |
| --- | --- | --- | --- |
| 输入 | `/sensor/cable_length` | `std_msgs/msg/Float64` | `data` 是缆长，单位米 |
| 输入 | `/sensor/cable_azimuth` | `std_msgs/msg/Float64` | `data` 是标定后的相对方位角，单位弧度，船头为零 |
| 输入 | `/sensor/relative_depth` | `std_msgs/msg/Float64` | `data` 是相对深度，单位米 |
| 输出 | `/solver/pose` | `geometry_msgs/msg/PoseWithCovarianceStamped` | 位置、时间戳、坐标系名称和协方差等信息 |
| 输出 | `/solver/debug_point` | `geometry_msgs/msg/PointStamped` | 用于调试的三维点，使用同一时间戳和坐标系 |

`ROS` 是 **Robot Operating System（机器人操作系统）**，`msg` 是 **Message（消息）**。同一个消息类型在终端里写成 `std_msgs/msg/Float64`，在 C++ 源码里写成 `std_msgs::msg::Float64`。包名 `std_msgs` 表示 **Standard Messages（标准消息）**。

## 从第一条消息看到第一行输出

### 给类型起一个短名字

```cpp
// Scalar 是这里给 Float64 消息起的别名。
using Scalar = std_msgs::msg::Float64;
```

`using` 在这里相当于 C 里常用的 `typedef`。可以把 `Float64` 消息想成一个包含 `double data` 成员的结构体。发布时给 `data` 填值，订阅后通过 `msg.data` 读取。如果想看看这个数怎么打印，读完就能把输出语句放回这行附近。[用 C++ 打印收到的缆长](00-cpp-output.md)

### 建立发布和订阅

```cpp
// pub = Publisher（发布器）。Pose 是完整位姿消息类型的别名。
pose_pub_ = create_publisher<Pose>("/solver/pose", 10);

// sub = Subscription（订阅器）。深度话题的最新数值保存在 depth_。
depth_sub_ = subscribe_scalar("/sensor/relative_depth", depth_);
```

这两行在 `PoseNode()` 构造函数里执行。构造函数类似于你在 C 里写的初始化函数，会在创建对象时自动执行。发布器上第 2 个参数 `10` 是历史队列深度。创建发布器后，代码以后用 `pose_pub_->publish(pose)` 发送实际消息。可以沿用 C 的思路理解 `->`，它让程序通过指针访问对象的成员。

`subscribe_scalar` 是项目自己封装的函数。它用引用接收 `depth_`，相当于 C 中把变量地址传进函数。每次收到消息时，回调执行 `latest = msg.data`，写入的就是原来的 `depth_`。同一个函数也被用来订阅缆长和方位角，各自传入对应的缓存变量。

### 决定数据是否到齐

```cpp
// 缆长、方位角和深度有任意一路还没收到，就结束本次计算。
if (!length_ || !angle_ || !depth_) {
    return;
}
```

三个变量的类型都是 `std::optional<double>`。可以把它类比为 C 里下面这样的结构体：

```c
#include <stdbool.h>

struct OptionalDouble {
    bool has_value;  /* 是否已保存一个值 */
    double value;    /* 实际保存的数值 */
};
```

对照起来，`!depth_` 类似判断 `!depth.has_value`，而 `*depth_` 取出保存的数值，类似读取 `depth.value`。深度为 `0.0` 时也有读数，所以照常参加定位计算。

### 从数值算到位置

缆绳拉直时，可以近似看成直角三角形。缆长是斜边，深度是竖直边。缆长 5 米、深度 3 米时，水平距离是 `√(5² − 3²) = 4` 米。若方位角是 0，子船在母船正前方，`x = 4`、`y = 0`。坐标系 Z 向上，所以水下 3 米写作 `z = -3`。

```cpp
// max 把根号中的数值限制在 0 以上，防止测量噪声导致负数开方。
const double radius = std::sqrt(std::max(0.0, length * length - depth * depth));
pose.pose.pose.position.x = radius * std::cos(*angle_);
pose.pose.pose.position.y = radius * std::sin(*angle_);
pose.pose.pose.position.z = -std::abs(depth);
```

`pose.pose.pose.position.x` 可以用 C 的嵌套结构体理解。第一个 `pose` 是 C++ 变量名，后面逐层访问带协方差的位姿、内部位姿、位置和 X 坐标。

## WSL 中的消息演示

下面使用 Ubuntu 22.04 和 ROS 2 Humble，假设船的工作空间已经完成构建。指令中的 `~/orangpi_ros2_build` 是构建目录示例。把它换成自己实际生成 `install/setup.bash` 的目录。[工程构建说明](https://github.com/Va1111n/ros2-ship/tree/main/orangpi_ros2#构建和运行)

在第一个 WSL 终端启动定位节点。

```bash
# 把 ROS 2 环境和已经编译好的船工程加载到当前终端
source /opt/ros/humble/setup.bash
source ~/orangpi_ros2_build/install/setup.bash

# 演示数据只在这台机器上通信
export ROS_LOCALHOST_ONLY=1

# 第一个名字是包名，第二个名字是可执行程序名
ros2 run usv_localization pose_node
```

让第一个终端继续运行。在第二个 WSL 终端加载相同环境后，启动监听，再依次发送三条消息。

```bash
source /opt/ros/humble/setup.bash
source ~/orangpi_ros2_build/install/setup.bash
export ROS_LOCALHOST_ONLY=1

# & 让监听命令在后台等待一条定位结果，终端仍能继续接收命令
ros2 topic echo --once /solver/pose geometry_msgs/msg/PoseWithCovarianceStamped &

# pub 是 publish（发布）。--once 表示各发送一条消息
ros2 topic pub --once /sensor/cable_length std_msgs/msg/Float64 '{data: 5.0}'
ros2 topic pub --once /sensor/cable_azimuth std_msgs/msg/Float64 '{data: 0.0}'
ros2 topic pub --once /sensor/relative_depth std_msgs/msg/Float64 '{data: 3.0}'
```

第三路数据到达后，监听命令会看到 `position` 下的 `x: 4.0`、`y: 0.0`、`z: -3.0`。`header.frame_id` 为 `mother/base_link`，说明这些坐标以母船为参考。

节点运行时，可以用 `ros2 node info /pose_node` 查看它订阅和发布的话题，也可以用 `ros2 topic info -v /sensor/relative_depth` 查看该话题的连接细节。此处 `-v` 是 **verbose（详细）**，`QoS` 是 **Quality of Service（服务质量）**，描述发布者和订阅者对消息交付的配置。演示结束后，在第一个终端按 `Ctrl+C` 停止节点。

## 这份源码里的一点改进空间

定位节点会保留每一路最后一次收到的值。如果深度数据停更，后续新的缆长或角度仍会触发计算，结果会带新的时间戳。接实船时，可以给每路输入记录接收时间，超过允许的时间就等待新数据。这是后面学到 ROS 2 参数和时间时可以继续读的地方。

[看懂回调里的 C++](02-cpp-in-the-ship.md) · [课程首页](../README.md)
