# 把三份 C++ 源码变成一个能启动的 ROS 2 功能包

对应第 10、11、13、16 课。这里读一份从源文件到安装、启动、观测都完整的示例。使用独立的 `/study/` 话题。各文件与船工作空间都有对应关系。

## 1. 先决定每个程序的作用

```text
length_source
    └─ /study/length_cm (Float64, 500.0)
                ↓
length_converter
    └─ /study/length_m (Float64, 5.0)
                ↓
length_observer 打印日志
```

第一个程序每半秒发一次厘米值，第二个程序收到才换算，第三个程序在有米值到达时打印。若只有发送和观察端，观察者正在等另一个话题，必然看不到米值。这个情况可以直接类比船里的原始角、标定器和位置节点。

一个学习工作空间可以按下图摆文件：

```text
任意学习工作空间/
└─ src/
   └─ length_study/
      ├─ package.xml
      ├─ CMakeLists.txt
      ├─ src/
      │  ├─ length_source.cpp       发送端
      │  ├─ length_converter.cpp    转换端
      │  └─ length_observer.cpp     下文的观察端
      └─ launch/
         └─ length_demo.launch.xml
```

发送程序取[第 13 课的定时发送端](04-topic-from-scratch.md#发送端把一次计算拆成哪些动作)。转换程序取[原教程的完整转换节点](../03-ros2-communication.md#第-13-课--写一个最小的订阅和发布节点)。两者各有 `main`，分开编成不同可执行程序。文件名与目标名之间的对应关系由 CMake 描述。每个节点的运行名称还可由构造函数或启动配置决定。

## 2. 先写给包工具看的清单

```xml
<?xml version="1.0"?>
<package format="3">
  <name>length_study</name>
  <version>0.1.0</version>
  <description>Three nodes for studying a cable length topic.</description>
  <maintainer email="study@example.com">Study example</maintainer>
  <license>Apache-2.0</license>

  <!-- 告诉包工具，此包使用 ament_cmake 构建。 -->
  <buildtool_depend>ament_cmake</buildtool_depend>

  <!-- C++ 源码使用客户端库和消息接口。 -->
  <depend>rclcpp</depend>
  <depend>std_msgs</depend>

  <!-- XML 启动描述在运行时需要这些包。 -->
  <exec_depend>ros2launch</exec_depend>
  <exec_depend>launch_ros</exec_depend>
  <exec_depend>launch_xml</exec_depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

示例维护者地址是演示占位。创建实际仓库时填写自己的维护信息与适用许可。包名要和 CMake 的项目名对上。`colcon` 通过清单识别包与构建方式。`rosdep` 可据依赖准备目标系统所需软件包。声明 `launch_xml` 是因为运行启动入口时要读取 XML，哪怕 C++ 本身只包含两个 ROS 头文件。类似地，船的可选 EKF 作为独立程序由 launch 启动，所以查它的运行依赖时要看启动入口和清单。

## 3. 再写给构建工具看的目标和安装位置

```cmake
cmake_minimum_required(VERSION 3.8)
project(length_study)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)

foreach(target IN ITEMS length_source length_converter length_observer)
  # 当前目标对应一个源码文件，每个源码只有自己的一个 main。
  add_executable(${target} src/${target}.cpp)
  target_compile_features(${target} PRIVATE cxx_std_17)
  ament_target_dependencies(${target} rclcpp std_msgs)
  install(TARGETS ${target} DESTINATION lib/${PROJECT_NAME})
endforeach()

install(DIRECTORY launch DESTINATION share/${PROJECT_NAME})
ament_package()
```

第一次循环把 `target` 替换为 `length_source`，因此 `add_executable` 加入的是发送端源码。然后给这个编译目标配置 C++17，添加 ROS 2 客户端和消息依赖，安装到包的可执行文件目录。再以相同方式处理转换和观察。`PROJECT_NAME` 在这份文件里就是 `length_study`。

只写 `find_package`，是让 CMake 在配置阶段找到某个依赖包。把依赖交给 `ament_target_dependencies`，才为指定目标配置编译和链接所需信息。船的 `pose_node` 还用几何消息，它会在共用处理之后再加自己的依赖。这样不必让所有目标都链接或声明每个目标专用的依赖。

三个 `main` 用三个目标。如果把三个源码同时写到同一个目标，链接器会遇到重复入口。以后想让一个进程里运行多个节点，可以让一个 `main` 创建并登记多个节点，所有逻辑仍然要由这同一个入口调度。教程这里选择三个进程，是为了便于观察连接关系和启动过程。

`install(DIRECTORY launch ...)` 把 XML 放到已安装包的共享区。源码里有启动文件而安装规则遗漏它时，工作空间也许能编译，但 `ros2 launch` 按包查不到入口。船的 CMake 同时安装 `launch` 和 `config`，原因就是启动时要找到滤波参数。

## 4. 加入只观察数据的第三个节点

```cpp
#include <memory>

#include <rclcpp/rclcpp.hpp>
#include <std_msgs/msg/float64.hpp>

using Scalar = std_msgs::msg::Float64;

class LengthObserver : public rclcpp::Node {
public:
    LengthObserver() : Node("length_observer") {
        // 与转换节点输出的名字和消息类型一致。
        subscription_ = create_subscription<Scalar>(
            "/study/length_m", 10, [this](const Scalar& sample) {
                ++received_;
                RCLCPP_INFO(get_logger(), "length %.2f m (sample %u)",
                            sample.data, received_);
            });
    }

private:
    unsigned int received_{0};  // 到目前为止通过此回调观察到的条数。
    rclcpp::Subscription<Scalar>::SharedPtr subscription_;
};

int main(int argc, char** argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<LengthObserver>());
    rclcpp::shutdown();
    return 0;
}
```

观察端和转换端都订阅一个话题。转换端把米值发给下游，观察端把值打印到日志。`received_` 是节点成员。把它放进回调体，每次进回调就会重新初始化，打印一直为 1。这和定位节点三路缓存的寿命要求相同。收到日志中的 5.00 米，证明这一组教学程序在软件层接通。设备真实缆长属于另一项验证。

用 C 的经验读 `RCLCPP_INFO`，它也是格式字符串跟参数的方式。`%.2f` 对应 `double` 字段，`%u` 对应 `unsigned int`。第一个参数 `get_logger()` 返回与节点相关的日志对象。它建立日志来源，后续参数决定文本。终端里的日志出现时间是观察者收到并处理时，跟发送端的采样时间可以不同。

## 5. 用 launch 一键组装三个端点

```xml
<launch>
  <node pkg="length_study" exec="length_source" output="screen" />
  <node pkg="length_study" exec="length_converter" output="screen" />
  <node pkg="length_study" exec="length_observer" output="screen" />
</launch>
```

三个 `pkg` 一样，表示同一个功能包。`exec` 不同，表示三个安装的程序。节点名分别由三个构造函数决定。`output="screen"` 让进程日志能出现在屏幕上。启动文件的目标是组织进程，不是声明某条消息已经交付。即使三个程序同时启动，发现与匹配也各自需要时间。教学数据源周期发数，观察者迟一点也能等到新样本。

在具备 Ubuntu 22.04 / ROS 2 Humble 的学习环境，摆好目录后，这组命令可以用于需要实际演示时：

```bash
source /opt/ros/humble/setup.bash
colcon build --packages-select length_study
source install/setup.bash
ROS_LOCALHOST_ONLY=1 ros2 launch length_study length_demo.launch.xml
```

第一步准备平台依赖的路径。第二步完成构建和安装。第三步让当前终端看到刚装好的包。第四步按包名找到 launch。少了第三步，`ros2` 命令可能存在，但查不到自己刚编的包。日志不断出现 5.00 米，可顺着依赖倒推到数据源已被启动并收到定时器事件、转换做了除法、观察端匹配成功。

## 6. 为什么转换系数有时应该成为参数

厘米到米转换固定除以 100。若上游换成设备原始计数，转换则依赖每个计数表示的米数。这里是不同的业务合同，可以使用独立的 `/study/raw_counts` 话题。程序启动时读取参数：

```cpp
// 这是节点构造函数中的局部片段。系数成员另外在类中声明。
meters_per_count_ = declare_parameter<double>("meters_per_count", 0.01);
```

回调内用 `output.data = input.data * meters_per_count_;`。在 0.01 米/计数的标定下，500 个计数是 5 米。若另一种设备每计数 0.02 米，启动时按该设备传入参数，得到 10 米。重新标定改变的是“同样设备输出如何换算”，厘米定义本身没有变。

节点启动时读取参数，然后保存到成员。终端运行时显示参数改变，并不证明该成员被业务代码同步改变。要支持运行时重配，需要参数回调或明确重新读取，并验证系数可用。船的 `mother_frame_id` 也在构造函数读取。它控制输出消息引用的参考系名称，改变配置后还要核对实际坐标含义与 TF。

## 7. 从观察结果往回查缺哪一层

| 现象 | 根因可能在哪 | 先找的证据 |
| --- | --- | --- |
| `ros2` 命令找不到 | 环境 | 当前是否在 Ubuntu 22.04 终端并加载 Humble |
| 找不到 `length_study` | 包索引或安装 | CMake 构建成功并加载的是这个工作空间吗 |
| 包可见，启动文件找不到 | 安装描述 | 已安装共享区有没有 `length_demo.launch.xml` |
| 三进程中只有两个 | 启动或链接后的可执行运行失败 | 启动日志里具体目标名和退出状态 |
| 观察端等不到米值 | 名字、类型、QoS 或中间进程 | 分别观察厘米和米两个话题的端点 |
| 能收到米值 500 而非 5 | 处理逻辑或运行了旧目标 | 转换回调是否做除法，工作空间安装版是否更新 |
| 日志只有一条后停住 | 定时发送和回调行为 | 确认定时器仍由成员持有、发送进程仍在运行 |

这张排查表可以直接迁移到船上。把厘米话题换成标定前角，米值换成标定后角，观察端换成定位节点。整个链路一次只验证一段，才能确定到哪儿断开。

[回到构建详解](05-workspace-inside.md) · [课程首页](../../README.md)
