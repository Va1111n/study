# 一个从头可读完的三路输入 ROS 2 节点

这一章把前几篇学过的东西接起来。[普通 C++ 版本](09-localization-without-ros.md) 用几行直接调用模拟事件，[缆长转换包](10-build-a-teaching-package.md) 有真正的发布订阅，但每来一条就能马上转换。这里编写一份完整的教学 ROS 2 程序，接收三路再发布。它沿用相同的直角三角形近似，使用自己的话题和更简单的点消息。构造函数先把三路接收代码分别摆开，让你看得到每条回调到底写哪个成员。最后对照船的 `subscribe_scalar` 为什么可以把重复部分合在一起。

## 1. 看程序之前，先写好它的输入输出

| 端点 | 类型 | 数字按什么约定 |
| --- | --- | --- |
| `/study/pose/cable_length` | `std_msgs/msg/Float64` | 米 |
| `/study/pose/azimuth` | `std_msgs/msg/Float64` | 标定后角，弧度，船头为 0，左为正 |
| `/study/pose/depth` | `std_msgs/msg/Float64` | 相对深度，米 |
| `/study/pose/point` | `geometry_msgs/msg/PointStamped` | 母船示例坐标系，X 前、Y 左、Z 上 |

真正的定位包还发布带协方差的位姿。本例用 `PointStamped` 让字段只剩参考系、时间和点坐标，读第一遍时好找。没有传入时间戳的三个数值仍是最新缓存法，方便按[时间专题](11-time-and-freshness.md)思考下一步要增加什么信息。

## 2. 先看一份小而完整的实现

需要 ROS 2 Humble 与 C++17。以下可以作为一个包中的源文件；本例加入 `geometry_msgs` 依赖，构建描述见代码后的说明。

```cpp
#include <algorithm>
#include <cmath>
#include <memory>
#include <optional>

#include <geometry_msgs/msg/point_stamped.hpp>
#include <rclcpp/rclcpp.hpp>
#include <std_msgs/msg/float64.hpp>

using SensorValue = std_msgs::msg::Float64;
using LocationPoint = geometry_msgs::msg::PointStamped;
using SensorSubscription = rclcpp::Subscription<SensorValue>;

class StudyPositionNode : public rclcpp::Node {
public:
    StudyPositionNode() : Node("study_position") {
        // 现在只建立端点。构造函数结束以后才处理实际输入。
        point_pub_ = create_publisher<LocationPoint>("/study/pose/point", 10);

        const auto sensor_qos = rclcpp::SensorDataQoS().keep_last(10);

        // 这三个回调都拿一条 SensorValue，但它们分别保存到自己的成员。
        length_sub_ = create_subscription<SensorValue>(
            "/study/pose/cable_length", sensor_qos,
            [this](const SensorValue& message) {
                update(length_m_, message.data);
            });
        angle_sub_ = create_subscription<SensorValue>(
            "/study/pose/azimuth", sensor_qos,
            [this](const SensorValue& message) {
                update(azimuth_rad_, message.data);
            });
        depth_sub_ = create_subscription<SensorValue>(
            "/study/pose/depth", sensor_qos,
            [this](const SensorValue& message) {
                update(depth_m_, message.data);
            });
    }

private:
    void update(std::optional<double>& slot, double sample) {
        // slot 是这一次调用选定的成员的别名，sample 是当前数字的副本。
        if (!std::isfinite(sample)) {
            return;
        }
        slot = sample;
        publish_if_ready();
    }

    void publish_if_ready() {
        // 只有三路都曾收到有效测量，才有足够输入能调用公式。
        if (!length_m_ || !azimuth_rad_ || !depth_m_) {
            return;
        }

        const double length = *length_m_;
        const double angle = *azimuth_rad_;
        const double depth = *depth_m_;
        const double distance_squared = length * length - depth * depth;
        const double horizontal = std::sqrt(std::max(0.0, distance_squared));

        LocationPoint output;
        output.header.stamp = now();    // 这一条测得结果的解算时刻。
        output.header.frame_id = "study_mother_link";
        output.point.x = horizontal * std::cos(angle);
        output.point.y = horizontal * std::sin(angle);
        output.point.z = -std::abs(depth);

        point_pub_->publish(output);
    }

    // 缓存跨回调保留，默认构造时没有值。
    std::optional<double> length_m_;
    std::optional<double> azimuth_rad_;
    std::optional<double> depth_m_;

    // 成员智能指针使端点持续存在。缓存位于它们之前，在析构时晚于它们释放。
    SensorSubscription::SharedPtr length_sub_;
    SensorSubscription::SharedPtr angle_sub_;
    SensorSubscription::SharedPtr depth_sub_;
    rclcpp::Publisher<LocationPoint>::SharedPtr point_pub_;
};

int main(int argc, char** argv) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<StudyPositionNode>());
    rclcpp::shutdown();
    return 0;
}
```

如果把它放进教学 `length_study` 包，名字用 `src/study_position.cpp`。在包清单增加 `<depend>geometry_msgs</depend>`，因为包里新增程序用它。CMake 先查找依赖，再建这个独立目标：

```cmake
# 在教学包的 ament_package() 之前加入这些目标配置。
find_package(geometry_msgs REQUIRED)
add_executable(study_position src/study_position.cpp)
target_compile_features(study_position PRIVATE cxx_std_17)
ament_target_dependencies(study_position rclcpp std_msgs geometry_msgs)
install(TARGETS study_position DESTINATION lib/${PROJECT_NAME})
```

和 `pose_node` 一样，新的目标有自己的 `main`。要把它加入教学 launch，可以加一行指向该目标的 `<node>`。另外三个教学程序的厘米输入与米值输出名称跟此例不同，仅同时启动不会自动接通所有三路。要验证这个位置程序，按本章表里的三个端点提供合成输入，或显式设计重映射和其他数据源。

## 3. 每个词只问一个问题

### 读三个 using

- `SensorValue` 是标量数据消息类型。
- `LocationPoint` 是输出消息类型，附时间和参考系。看到 `LocationPoint output;` 就知道后面能用 `output.header` 与 `output.point`。
- `SensorSubscription` 是一种订阅器类型，已经给模板指定了它订阅哪种消息。再写 `::SharedPtr`，表示这个订阅器类型内提供的共享持有者类型。

别名省去重复的长名字，理解时仍能把它换回原类型。`using SensorSubscription = ...` 并没有注册任何订阅器。真正注册发生在构造函数的 `create_subscription` 处。

### 进入哪个构造函数才登记回调

`main` 里调用 `make_shared<StudyPositionNode>()`，从这里进入类构造。构造完成后三个回调都已登记，才进入 `spin`。局部 `sensor_qos` 用来给创建调用传配置，这段创建接口完成后它可以释放。回调不引用该局部变量。用于接收的端点对象由成员智能指针保存；这两者不同，避免以为所有用到的变量都应该是类成员。

### 三个回调怎么跳到相同的函数

每一个 lambda 都捕获节点指针，然后将一个独立的成员传给 `update`。缆长回调传 `length_m_`，角度回调传 `azimuth_rad_`，深度回调传 `depth_m_`。`update` 中的参数名 `slot` 在三种调用里引用不同对象。参数声明 `std::optional<double>& slot` 让它写入的是原处。若将这个 `&` 删掉，就会复制目标缓存，函数里的赋值改不到节点状态。程序可以编译，却一直等不到齐全。这类错误值得在读参数时主动判断。

### 后一次调用凭什么能读上一次的数据

收到深度之前，缆长可能已到，`update` 也已经返回。三个缓存存在节点对象里，跟着它跨过调用边界。回调的 `message` 只关联本次消息，函数返回后这段引用不应自行保留。代码只把一个 `double` 值复制到节点状态，没有把消息地址存下来。下次会拿到另一条消息，但仍然修改同一个 `StudyPositionNode`。

## 4. 按实际到达顺序观察状态

```text
启动：
  main → init → 构造 → spin
  [空, 空, 空]

输入 /study/pose/depth 为 3：
  深度回调 → update(depth_m_, 3)
  [空, 空, 3] → publish_if_ready 直接返回

输入 /study/pose/azimuth 为 0：
  角度回调 → update(azimuth_rad_, 0)
  [空, 0, 3] → 长度仍空，等待

输入 /study/pose/cable_length 为 5：
  缆长回调 → update(length_m_, 5)
  [5, 0, 3] → 通过各项检查
  horizontal = sqrt(25 - 9) = 4
  point_pub_->publish 写出一条 (4,0,-3)，附解算时间和参考系。
```

把三路事件换一下，哪一路最后到就由其回调首次触发结果，数值只要一样，公式结果相同。之后若给非有限读数，`update` 在写入前退出，本事件不触发发布。下一次有效读数到来，会使用其余最后一次有效缓存。`publish_if_ready` 不负责把状态定期清空，也不检查时间。这些行为来自分支和成员寿命，不是 ROS 2 自动处理的业务规则。

## 5. 对照船里的抽象多走了哪一步

教学程序重复三遍 `create_subscription`，一眼看出哪个 lambda 处理哪项。船代码发现唯一差异是话题名与缓存，所以把它们变成 `subscribe_scalar(topic, latest)` 的两个参数。用一个函数创建订阅，回调在创建时把不同的目标引用捕获下来。变化发生在“共同的代码怎样组织”，还原到三个事件时数据流一样：

| 教学程序 | 船的程序 | 阅读时对应的问题 |
| --- | --- | --- |
| 三次写出不同的 lambda | 一段 lambda，三次创建各自绑定 | 实际登记了几个端点，绑定到谁 |
| 在回调体直接写 `update(length_m_, ...)` | `[this, &latest]` 访问给定缓存 | 目标是调用时写明还是在建立时保存 |
| 只发布 `PointStamped` | 位姿及调试点分开发 | 哪个字段承载位置、参考系和协方差 |
| 写死 `study_mother_link` | 通过节点参数配置默认 frame 名 | 更改配置时实际修改何时、怎样生效 |
| 无纯计算与通信的集成入口 | 实际包 launch 与测试从安装处运行 | 哪些生成文件和依赖会影响验证 |

读自己的代码时可以先写清楚三条，再提取重复动作。一开始就上模板、长别名和泛用回调，往往不容易看见谁在更新哪一格。已有项目代码完成了这种提取，先用展开形式理解，再回去就知道抽象省掉的是什么。

## 6. 从 0 到可读懂四个节点的最后一段路

用这三个角度继续分别读剩余文件：

- `imu_test1.cpp` 的回调取一个 IMU 消息，形成倾角与方位，更新两个滤波对象。在这里把“对象每次保存什么”换成两个上一帧状态。
- `cable_angle_calibrator.cpp` 收到一条数就处理一条。让输入经过零偏、拟合和区间规约，再发布。这里的计算与收发边界和上面相同，只是无需凑三路。
- `mother_usv_msg_node.cpp` 的订阅只改航向缓存，定时器才发。若看见 `create_wall_timer`，就去找它的回调和发布器，别在 IMU 回调末尾找发布调用。
- `localization.launch.xml` 能告诉你这些节点怎么同时起。把每个进程的输入与输出名字列开，看到上游没有发布者的数据就去查迁移指南。

理解一个节点的标准可以很具体：能说清来自哪个话题、当什么事件发生时执行哪段函数、这段函数改了哪个对象、最后哪一条调用真正发消息。把同样四句用在不同文件上，就具备了读这个工程的基本方法。

[返回定位逐行篇](01-first-read.md) · [查看函数名和缩写索引](19-source-map-and-glossary.md) · [课程首页](../../README.md)
