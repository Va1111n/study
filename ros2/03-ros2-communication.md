# 读懂节点怎样收消息和发消息

这页保留了完整的缆长转换节点。要从发送者、执行器和订阅者三个角度跟读，可以先读[话题通信详解](deep/04-topic-from-scratch.md)，再看[如何把示例编成完整功能包](deep/10-build-a-teaching-package.md)。

## 第 12 课 · 从缆长话题认识 ROS 2

ROS 是 Robot Operating System（机器人操作系统）。这个项目把采集、计算和融合拆成多个节点，它们通过约定的接口交换数据。

| 名词 | 在船上的例子 | 怎么理解 |
| --- | --- | --- |
| 功能包 | `usv_localization` | 一起构建、安装和维护的一组程序与配置 |
| 可执行程序 | `pose_node` | 编译得到的文件，能由 `ros2 run` 或 launch 启动 |
| 进程 | 运行中的 `pose_node` | 操作系统管理的一次程序运行，拥有自己的内存 |
| 节点 | `/pose_node` | ROS 2 通信图里的参与者。这里一进程一个节点，其他程序也可以把多个节点放进一个进程 |
| 话题 | `/sensor/cable_length` | 发布和订阅双方用同一个名字找这条数据流，可以有多方参与 |
| 消息类型 | `std_msgs/msg/Float64` | 约定每条数据的字段布局。这里有一个 `float64 data`，在 C++ 是 `double` |
| 消息 | 一条 `data: 5.0` | 类型定义的某个实例。这个话题的约定是数值以米计 |

`std_msgs` 是 Standard Messages（标准消息包），`msg` 是 Message（消息）。终端用斜杠写类型，C++ 用命名空间写 `std_msgs::msg::Float64`。话题名和类型需要匹配，单位与正方向也要由应用约定。ROS 2 看到两个 `Float64` 时无法推断一个是米、另一个是弧度。

话题适合连续测量和状态广播。服务（service）适合一次请求和一个响应，例如查询当前配置。动作（action）适合持续较久且需要反馈或取消的任务，例如让一套机器人执行一段导航。这两个例子用来区分接口设计，船的定位主链路走话题。

## 第 13 课 · 写一个最小的订阅和发布节点

定位节点有三路订阅，收到任一输入都调用一段计算。下面先只保留一路，写一个完整的教学程序。它读厘米，再发米，用独立的 `/study/` 话题观察处理流程。程序用到的类、模板和回调，与 `pose_node.cpp` 是同一种写法。把它作为一个 C++17 可执行目标构建时，包需要声明 `rclcpp` 与 `std_msgs` 依赖。

```cpp
#include <memory>  // std::make_shared 在这里声明。

#include <rclcpp/rclcpp.hpp>        // ROS Client Library for C++，C++ 客户端库。
#include <std_msgs/msg/float64.hpp> // Float64 消息。data 字段是 double。

using Scalar = std_msgs::msg::Float64;

class LengthConverterNode : public rclcpp::Node {
public:
    LengthConverterNode() : Node("length_converter") {
        // 创建输出端，声明它在 /study/length_m 上发布 Scalar 消息。
        // 10 是历史深度，创建端点时设置。真正发数值要等回调调用 publish。
        length_pub_ = create_publisher<Scalar>("/study/length_m", 10);

        // 创建输入端，订阅 /study/length_cm。lambda 在每次收到消息时执行。
        // [this] 让回调可以找到本对象持有的 length_pub_。
        length_sub_ = create_subscription<Scalar>(
            "/study/length_cm", 10, [this](const Scalar& input) {
                Scalar output;                    // 创建一条待发送的消息。
                output.data = input.data / 100.0; // 按该接口的约定把厘米换成米。
                length_pub_->publish(output);      // 把当前这条消息交给发布器。
            });
    }

private:
    // pub = Publisher（发布器），sub = Subscription（订阅器）。
    // SharedPtr 让这两个端点在构造函数结束后继续被节点持有。
    rclcpp::Publisher<Scalar>::SharedPtr length_pub_;
    rclcpp::Subscription<Scalar>::SharedPtr length_sub_;
};

int main(int argc, char** argv) {
    // argc = Argument Count，参数个数。argv = Argument Vector，参数列表。
    rclcpp::init(argc, argv);
    auto node = std::make_shared<LengthConverterNode>();
    rclcpp::spin(node);  // 等待通信事件，调用相应的回调。
    rclcpp::shutdown();
    return 0;
}
```

如果收到 `data = 500.0`，回调里 `input` 就引用着包含 500.0 的那条消息。计算结果 5.0 写入一个新的 `output`，然后发到米值话题。两边消息类型相同，因此语义写在话题名里。把这个例子放回船上，`length_pub_` 对应 `pose_pub_`，只有输出消息变成了带位置和时间戳的类型。输入回调还要先写入缓存，再检查其他两路数据。

## 第 14 课 · 启动、等待、回调和定时器

1. `rclcpp::init(argc, argv)` 初始化 ROS 2，读取相关命令行参数。
2. `make_shared` 创建节点，构造函数登记订阅器和发布器。构造过程同时决定以后收到消息时走哪段代码。
3. `spin` 把活交给执行器（executor）。有输入时执行匹配的回调，定时器到期时执行它的回调。
4. 进程收到结束信号后退出 `spin`，调用 `shutdown`，释放运行环境。

当前工程各进程用普通 `rclcpp::spin`。同一节点的这些回调在单线程执行器里依次执行。一段回调如果阻塞太久，后面等待的消息和定时器都会受到影响。将来换多线程执行器或回调组，需要重新验证哪些回调可能同时访问缓存。

母船航向节点把“更新读数”和“发布读数”分成两段。IMU（Inertial Measurement Unit，惯性测量单元）订阅回调更新 `heading_`，`create_wall_timer` 登记的回调每 20 毫秒尝试发一次 `heading_`，目标频率为每秒 50 次。若传感器只有每秒 10 帧，同一个航向就会被定时器发多次。执行器繁忙时，实际发布频率还可能低于目标。对比 `pose_node`，它只有输入更新才尝试计算。观察输出频率时，先确定是哪种触发方式。

## 第 15 课 · 为什么话题看得到却收不到

连接要先满足话题名和类型，再满足发布端提供的策略能够覆盖订阅端请求的策略。QoS 是 Quality of Service（服务质量），包含可靠性、历史深度和持久性等配置。底层通信发现负责让端点找到彼此，网络、ROS 域或仅本机设置也会影响发现。

| 发布端可靠性 | 订阅端可靠性 | 能否因这项策略匹配 |
| --- | --- | --- |
| Reliable，可靠交付 | Reliable | 能 |
| Reliable | Best Effort，尽力交付 | 能 |
| Best Effort | Best Effort | 能 |
| Best Effort | Reliable | 不能 |

船的传感器输入用 `rclcpp::SensorDataQoS().keep_last(10)`，采用 Best Effort，最多保留最近十条的历史深度设置。它可以匹配两种可靠性发布端，还要检查其他策略。教学转换器的 `10` 走普通默认 QoS 配置，可靠性为 Reliable。历史深度决定端点缓存范围。若生产快于处理，端点可丢掉旧消息，队列和频率需要结合作业量分析。默认的 Volatile（易失）持久性也意味着后来启动的监听者通常从新消息开始看。

`ros2 topic info -v /sensor/cable_length` 会列出端点、所属节点、消息类型和 QoS。主题名中的 `-v` 是详细输出选项，来自 verbose。若列表里只有订阅端，先找发布进程。两端都能看见，就对照可靠性和其他策略。ROS 2 可通过 DDS（Data Distribution Service，数据分发服务）等中间件进行发现与传输。跨设备时要核对 `ROS_DOMAIN_ID`（域标识）、`ROS_LOCALHOST_ONLY`（是否仅本机）和网络互通，再定位具体端点。

[看工作空间和启动文件](04-workspace.md) · [课程首页](../README.md)
