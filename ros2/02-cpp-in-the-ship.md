# 用 C 的经验读懂船里的 C++

这页是简明回顾。想看完整的 C 和 C++ 对照程序，可以从[变量、引用和可选值详解](deep/02-cpp-symbols.md)接到[类、构造与回调详解](deep/03-objects-and-callbacks.md)。

先打开 `pose_node.cpp`，找到 `length_`、`subscribe_scalar` 和 `publish_position`。这些名字分别对应存储、接收和计算。下面按你会在它们周围遇到的写法来讲。

## 第 02 课 · 类型、常量和数值计算

`double length = 5.0;` 和 C 里的含义一致。接下来要把数值与单位一起读。C++ 只知道 `length` 是一个双精度浮点数，米这个约定来自话题说明、变量名和上游设备。把 500 厘米直接作为 500 米发进来，编译器也会接受。角度同理，`std::sin` 和 `std::cos` 的参数使用弧度，90 度应先换成约 1.571 弧度。

```cpp
#include <cmath>  // sqrt、isfinite 等数学函数声明在这里。

const double length_m{5.0};           // 大括号初始化。const 使该变量初始化后不可修改。
const double depth_m{3.0};
const double squared_m = length_m * length_m - depth_m * depth_m;
const double horizontal_m = std::sqrt(squared_m);  // 平方米开平方，得到米。
const bool valid = std::isfinite(horizontal_m);     // bool 表示 true 或 false。
```

上面的局部片段使用一组已知有效的数据。船代码会先钳制根号中的值，第 19 课解释这个分支。用大括号初始化时，`int count{3.5};` 会因缩窄转换而报错，这能提醒你区分数量和测量值。`5 / 2` 两边都是整数，结果是 2，因此写进 `double` 后也是 2.0。`5.0 / 2` 会做浮点除法，结果为 2.5。

`const` 关心“这次初始化以后能不能改”，初始值可以是运行时收到的消息。`constexpr` 要求该处初始值能在编译时求出，适合写固定系数。`std::isfinite` 排除正负无穷和 NaN（Not a Number，非数值）。有限数还有可能超出物理范围，单位、非负缆长和传感器量程需要单独约定和校验。

## 第 03 课 · 值、指针、引用和头文件

C 常通过传入地址让函数修改外部变量。C++ 可以用可写引用表示同样的意图。

```c
/* C：调用时传地址，函数内解引用。 */
void save_length(double *latest, double value) {
    *latest = value;
}
/* double length = 0.0; save_length(&length, 5.0); */
```

```cpp
// C++：latest 成为调用者那个变量的另一个名字。
void save_length(double& latest, double value) {
    latest = value;
}
// double length = 0.0; save_length(length, 5.0);
```

| 声明 | 函数拿到什么 | 在船代码中怎么读 |
| --- | --- | --- |
| `double value` | 一个新副本 | 改这个参数只影响本次调用的副本 |
| `double *value` | 一个指针值 | `*value` 访问所指对象。接口可用空指针表达无对象，解引用前须确认有效 |
| `double& value` | 已有对象的别名 | `value = 5.0` 会改调用者的对象 |
| `const Scalar& msg` | 对消息的只读引用 | `msg.data` 可以直接读，调用时无需复制整条消息 |

看 `imu_orientation.hpp` 中的 `read_orientation(const sensor_msgs::msg::Imu & imu, tf2::Quaternion & q)`。它读取 `imu`，通过 `q` 把四元数写到调用方，再用返回值 `bool` 表示结果是否可用。调用者写 `if (!read_orientation(imu, q)) { return; }`，失败就结束本次回调，只有成功后才用 `q` 计算姿态。这里 `imu` 前的 `const` 限制经由该引用的写操作。

`.hpp` 是常见的 C++ 头文件扩展名，可以放类型和函数声明。这个项目把共享的小函数定义也放进头文件，用 `inline` 允许各个编译单元包含同一份符合规则的定义。是否把调用展开由编译器自行决定。`#pragma once` 防止该头文件在同一个编译单元里被重复包含。凡是读到 `using Scalar = std_msgs::msg::Float64;`，就把 `Scalar` 看成类型别名，类似 C 的 `typedef`。

## 第 04 课 · 一组系数和一个可能缺席的数

角度标定器保存六个系数。下面用三个系数演示同样的求值方式，便于手算。

```cpp
#include <array>

constexpr std::array<double, 3> coefficients{2.0, 3.0, 4.0};
const double angle_rad = 0.5;
double corrected = 0.0;
// 每轮取一个系数。例子依次得到 2、4、6，等价于计算 2x² + 3x + 4。
for (double coefficient : coefficients) {
    corrected = corrected * angle_rad + coefficient;
}
```

`std::array<double, 3>` 是一个模板类型，尖括号内指定元素类型和长度。它的大小固定，能用 `size()` 查询。长度要随运行中的数据变化时，可以选 `std::vector<double>`。范围 `for` 按顺序取出元素。例子复制一个很小的 `double`，大对象通常用 `const auto&` 直接读取。看到 `auto & covariance = pose.pose.covariance;`，`auto` 由编译器推导类型，`&` 保证操作的是消息中的原数组。只写 `auto covariance` 则会复制这份数组。

```cpp
#include <optional>

std::optional<double> depth;  // 刚启动，还没有深度消息。
depth = 0.0;                 // 收到水面深度。此时 has_value() 为 true。
if (depth) {
    const double depth_m = *depth;  // 先确认有值，才取里面的 double。
}
depth.reset();               // 清空后，回到没有读数的状态。
```

在 C 里可以用一个标志位加一个数值表达同样的含义。船里 `if (!length_ || !angle_ || !depth_)` 判断的是三个容器有没有值，任意一个缺席就提前返回。谁先收到消息都可以，零米和零弧度都能参加计算。

## 第 05 课 · 从状态结构体到类和对象

先看一个简化滤波器。每次更新让旧值向新测量靠近四分之一。这两个例子用于学习如何管理状态。实船的 `AngleFilter` 另外保存方差、计算增益并处理角度跨界。

```c
/* C：调用方持有状态，每次更新时显式传入它的地址。 */
struct AngleState {
    double estimate;
};

double update_angle(struct AngleState *state, double measurement) {
    state->estimate += 0.25 * (measurement - state->estimate);
    return state->estimate;
}

int main(void) {
    struct AngleState tilt = {0.0};
    struct AngleState azimuth = {0.0};
    update_angle(&tilt, 0.8);     /* tilt.estimate 变成 0.2。 */
    update_angle(&azimuth, 1.6);  /* azimuth.estimate 变成 0.4。 */
    return 0;
}
```

```cpp
class AngleSmoother {
public:
    // 外部可以调用 update。这个方法能读取并更新当前对象的私有字段。
    double update(double measurement) {
        estimate_ += 0.25 * (measurement - estimate_);
        return estimate_;
    }

private:
    double estimate_{0.0};  // 每个对象有一份，从 0.0 开始。
};

int main() {
    AngleSmoother tilt;
    AngleSmoother azimuth;
    tilt.update(0.8);
    azimuth.update(1.6);
    return 0;
}
```

`AngleSmoother` 是类，描述对象具有什么数据和操作。`tilt`、`azimuth` 是用这个类创建的两个对象，各有一个 `estimate_`。调用 `tilt.update(...)` 时，方法里的 `this` 指向 `tilt`，`estimate_` 可以理解成 `this->estimate_`。这个对象身份相当于 C 例子显式传入的 `state`。字段放到 `private` 后，外部走 `update` 这条路径，类可以集中维护状态。C++ 的 `struct` 也能定义成员函数，常用区别是 `struct` 默认公开，`class` 默认私有。

回到船上的 `AngleFilter tilt_filter_, azimuth_filter_;`。第一个对象记住倾角的历史，第二个对象记住方位角的历史，其中一个对象执行 `update` 时只更新自己的历史。

## 第 06 课 · 构造、析构、继承和组合

```cpp
class SensorScale {
public:
    // 构造函数与类同名，没有返回类型。先初始化 scale_，再运行花括号。
    explicit SensorScale(double scale) : scale_{scale} {}
    double convert(double reading) const { return reading * scale_; }

private:
    double scale_;
};

// SensorScale meters_per_centimeter{0.01};
// meters_per_centimeter.convert(500.0) 得到 5.0。
```

方法末尾的 `const` 表示这个方法不能通过 `this` 修改普通成员，适合读取状态或纯计算。构造函数的 `explicit` 要求显式构造，例如上面的花括号写法。成员按类内的声明顺序初始化，可以按这个顺序写初始化列表。析构函数的名字是 `~SensorScale()`，对象生命周期结束时自动调用。这个例子由编译器生成默认析构。资源交给对象随生命周期释放的做法叫 RAII（Resource Acquisition Is Initialization，资源获取即初始化），例如智能指针和文件流。

在项目的 `class PoseNode : public rclcpp::Node` 里，`PoseNode` 继承 ROS 2 的节点能力。构造函数初始化列表里的 `Node("pose_node")` 先建立基类部分，随后构造函数体调用它提供的 `create_publisher`、`create_subscription`、`declare_parameter`。这里的 `public` 继承让 `PoseNode` 可以作为 ROS 2 节点交给执行器。另一方面，缆绳 IMU 节点**包含**两个滤波器，定位节点**持有**发布器和订阅器。这是组合。读源码时可以先问“它是一种什么对象”，再问“它内部持有哪些对象”。

## 第 07 课 · 点号、箭头和智能指针

`pose.header` 用点号访问对象成员。`pose_pub_->publish(pose)` 的左边是指针式对象，箭头访问它所指向的发布器。在 C 中 `pointer->field` 等价于 `(*pointer).field`。C++ 的智能指针也提供这样的箭头访问。

| 写法 | 对生命周期有什么影响 |
| --- | --- |
| `PoseNode node;` | 此处是局部对象，离开作用域会析构 |
| `PoseNode& ref = node;` | 别名，使用范围受原对象生命周期约束 |
| `PoseNode* ptr = &node;` | 原始指针，这里的观察者无需负责释放对象 |
| `auto node = std::make_shared<PoseNode>();` | 构造节点并得到共享智能指针，最后一个强拥有者释放时销毁对象 |

`SharedPtr` 是 Shared Pointer（共享智能指针）的缩写。复制这种指针会共享同一个对象。它的对应类型通常是 `std::shared_ptr<T>`，另有独占资源的 `std::unique_ptr<T>`。在构造函数里调用 `create_subscription` 后，把返回的指针存入成员 `length_sub_`、`angle_sub_` 或 `depth_sub_`，订阅器会在构造函数返回以后继续被节点持有。若只放在局部变量里且没有其他强引用，退出构造函数时这个订阅就会消失。普通局部测量值则可以很快销毁，例如每次发布时临时创建的 `Pose pose;`。

## 第 08 课 · 从 C 函数指针到 lambda

C 的异步接口经常收两个东西，一是函数指针，二是 `void *context`（上下文指针），以后一起传回来。C++ 可以把操作和需要的上下文装进可调用对象。lambda 是在表达式中定义这种对象的常见写法。

```cpp
double last = 0.0;
// [&last] 捕获对外部变量的引用。
// (double value) 是将来每次调用时才传进来的参数。
auto remember = [&last](double value) {
    last = value;
};
remember(5.0);  // 走到这一行才执行花括号里的代码。
```

`[]` 表示无需捕获外部变量，`[value]` 保存一个副本，`[&value]` 经由引用使用原对象。`[this]` 保存指向当前对象的指针，因此可以访问它的成员。引用捕获和 `this` 捕获都要求实际目标活到回调运行时。把一个短命的局部变量按引用交给长期运行的回调，会留下悬空引用。

定位节点里 `[this, &latest](const Scalar & msg)` 的三个角色值得逐一读。`this` 让回调能调用当前节点的 `publish_position()`。`latest` 原本就是引用参数，捕获后仍指向调用方传进来的那个成员。`msg` 则是收到每条新消息时由 ROS 2 传入的参数。三次调用 `subscribe_scalar` 各创建一个可调用对象，分别指向 `length_`、`angle_` 和 `depth_`。这就是同一段文字能在三路消息里写入不同成员的原因。

[继续看 ROS 2 通信](03-ros2-communication.md) · [课程首页](../README.md)
