# 用普通 C++ 重建定位的核心行为

这一章把 ROS 2 暂时拿掉。保留船上最重要的行为：三路数据分别出现，事件更新一格，三格都有值才计算。这样可以先弄懂语言和数学，再把人工调用换成订阅回调。下面的程序是学习用的独立示例，能用 C++17 编译。场景在 `main` 中按次序发生，所以每一行输出都能从源代码解释。

## 1. 一块可单测的数学，与需要记住测量的对象

定位有两项不同责任：

- 给定缆长、方位和深度，算一次 XYZ。这个动作只用传入参数。
- 每当有新测量，记住它并决定是否调用上一项计算。这个动作需要跨多次事件保存状态。

把它们分成名字明确的函数和类型，便能单独说明先后关系。原项目在节点内部实现这些职责。学习示例把边界写出来，便于看见一块纯计算如何与输入调度组合。

```cpp
#include <algorithm> // std::max，用于把平方差限制在不小于零。
#include <cmath>     // std::sqrt、cos、sin、abs、isfinite。
#include <iostream>  // 输出观察结果。
#include <optional>  // 一个可能还不存在的数或位置。

// 这是位置结果本身。三个成员的共同单位都是米。
struct RelativePosition {
    double x_m;
    double y_m;
    double z_m;
};

// 这个函数要求调用方已经得到三个有限数值。
// 它没有跨调用的内部状态，同样输入会算出同样结果。
RelativePosition solve(double length_m, double angle_rad, double depth_m) {
    const double squared_radius = length_m * length_m - depth_m * depth_m;
    const double radius_m = std::sqrt(std::max(0.0, squared_radius));

    return RelativePosition{
        radius_m * std::cos(angle_rad), // 前方分量。
        radius_m * std::sin(angle_rad), // 左方分量。
        -std::abs(depth_m),            // 水下方向采用负 Z。
    };
}

class PositionTracker {
public:
    // 三个方法模拟三条消息各自的接收回调。每次最多写入一种测量。
    // 返回 optional，告诉调用者这次事件有没有生成新位置。
    std::optional<RelativePosition> receive_length(double meters) {
        return update(length_m_, meters);
    }

    std::optional<RelativePosition> receive_angle(double radians) {
        return update(angle_rad_, radians);
    }

    std::optional<RelativePosition> receive_depth(double meters) {
        return update(depth_m_, meters);
    }

private:
    std::optional<RelativePosition> update(
        std::optional<double>& destination,
        double measurement) {
        if (!std::isfinite(measurement)) {
            // 当前数据不采用。原缓存继续存在，本事件也不生成输出。
            return std::nullopt;
        }

        // destination 引用调用方选中的那个成员，所以赋值后会留在对象内。
        destination = measurement;

        if (!length_m_ || !angle_rad_ || !depth_m_) {
            // 有任意一项尚未出现。等以后再用这个对象处理下一次事件。
            return std::nullopt;
        }

        // 三个 optional 都有值，此时 * 才安全地取出存放的 double。
        return solve(*length_m_, *angle_rad_, *depth_m_);
    }

    // 默认构造时三个成员都为空。每个 PositionTracker 对象有自己的一份状态。
    std::optional<double> length_m_;
    std::optional<double> angle_rad_;
    std::optional<double> depth_m_;
};

void show(const char* event, const std::optional<RelativePosition>& result) {
    std::cout << event << " -> ";
    if (!result) {
        std::cout << "no new position\n";
        return;
    }

    // result 是 optional，确认有值后，箭头进入它持有的位置对象。
    std::cout << '(' << result->x_m << ", " << result->y_m
              << ", " << result->z_m << ") m\n";
}

int main() {
    PositionTracker tracker; // 类定义不会自动运行。走到这里才创建一个对象。

    show("length 5", tracker.receive_length(5.0));
    show("depth 3", tracker.receive_depth(3.0));
    show("angle 0", tracker.receive_angle(0.0));
    show("depth 0", tracker.receive_depth(0.0));
    show("depth 6", tracker.receive_depth(6.0));

    PositionTracker second; // 新对象从空状态开始。
    show("second tracker length 5", second.receive_length(5.0));
    return 0;
}
```

代码中的 `PositionTracker` 类保存长期状态，`RelativePosition` 只装结果。C++ 的 `struct` 默认成员公开，适合这种只装三个数字的简单记录。`class` 里的私有成员让调用者从三个 `receive_...` 入口操作，不会绕开有效值检查直接改缓存。

## 2. 带着栈和对象的概念从第一行跑一遍

### 事件一 · 缆长 5 米

执行 `tracker.receive_length(5.0)`。方法把 `length_m_` 作为引用传给 `update` 的 `destination`。函数里的 `measurement` 自己是一个副本，值为 5.0。`destination = measurement;` 改的则是外面的 `tracker.length_m_`。

通过第一个 `if` 后，三格分别为 `[5, 空, 空]`。第二个 `if` 成立，返回 `std::nullopt`。`show` 接到本次结果为空，所以打印等待信息。`update` 与 `show` 都返回后，`length_m_` 仍是 5，因为它是一开始创建的 `tracker` 的成员。

### 事件二 · 深度 3 米

方法选定 `depth_m_` 为目的地。三格成为 `[5, 空, 3]`。角度还空，计算继续返回空。这里没有将所有成员清掉，也没有重新创建 `tracker`。这就是两条事件抵达时能积累信息的原因。

### 事件三 · 方位 0 弧度

`angle_rad_` 现在有一个有效值，虽然值恰好是 0。三个 `optional` 的 `has_value()` 都为真，第二个条件不成立。函数依次取出 5、0、3 调用 `solve`，返回 `RelativePosition`，语言在返回位置时把它包装进 `std::optional<RelativePosition>`。随后 `show` 用 `result->x_m` 之类的字段访问取出三维数值，打印 `(4, 0, -3)`。

`->` 在之前见过智能指针语境。这里是 `optional` 也提供同名操作，让你从持有的数据进入结果成员，前提是已经检查有值。要写得更直接，也可以 `(*result).x_m`。

### 事件四 · 深度变为 0

第 3 次收到的是角度事件，这次换深度。深度 0 代表同一高度，不会被视作缺席。缓存是 `[5, 0, 0]`。平方差为 25，前方位置约是 5 米。某些终端会将负零显示为 `-0`，因为 `-std::abs(0.0)` 可得到带负号的浮点零，数值比较等于零。

### 事件五 · 深度大小大于缆长

缓存是 `[5, 0, 6]`。平方差为 −11。钳成零以后，X 和 Y 为 0，而 Z 是 −6。示例保留船代码的几何边界处理，方便和原结果对照。在实际驱动确认输入是否合理时，要把“防止负数开方”和“测量符合物理假设”分别判断。

### 事件六 · 新对象对先前数据一无所知

`PositionTracker second;` 创建第二个实例。它的三个成员从空开始。第一个定位对象已经有缆长、方向和深度，第二个对象仍需自己接收。可用它解释船代码中的两个滤波对象为何不会相互污染，也可用来理解重启一个节点后之前测量要重新获取。

| 到达事件 | 第一个对象的长、角、深 | 第二个对象 | 这一事件有结果吗 |
| --- | --- | --- | --- |
| 启动 | 空、空、空 | 尚未创建 | 无 |
| 长度 5 | 5、空、空 | 尚未创建 | 无 |
| 深度 3 | 5、空、3 | 尚未创建 | 无 |
| 角度 0 | 5、0、3 | 尚未创建 | (4, 0, −3) |
| 深度 0 | 5、0、0 | 尚未创建 | 前方 5 米 |
| 深度 6 | 5、0、6 | 尚未创建 | (0, 0, −6)，水平钳零 |
| 新对象收到长度 5 | 5、0、6 | 5、空、空 | 无 |

## 3. 同一段代码有几种“返回”

`return std::nullopt;` 表示这一次事件没有产生新结果。第一处是无效输入，第二处是输入未齐。两处在当前示例使用同一种返回值，所以调用者只知道这回没结果。想让调用者进一步区分原因，可以设计一个含状态码的返回对象。在 ROS 2 的实现中，日志或诊断状态也可说明为何不发消息。

`return solve(...);` 返回一个真实位置。它发生在全部输入有效、状态已齐的路径。`return;` 是 `void show` 中的写法，只结束这次打印。`return 0;` 在 `main` 返回进程结束状态。之后你在文件里看见同一个词，可以先看它位于哪个函数以及函数的返回类型。

## 4. 把模拟事件换成 ROS 2，哪些步骤对得上

| 学习程序 | 真正的 ROS 2 节点 | 原因 |
| --- | --- | --- |
| `main` 依次直接调用三种接收方法 | 构造订阅器，随后由执行器调用回调 | 真正输入来自异步外部进程，不能约定写死的到达顺序 |
| `double measurement` | `const Scalar & msg`，再访问 `msg.data` | ROS 2 按消息类型交付，这里应用层才取单个字段 |
| `update(member, value)` | `subscribe_scalar` 在登记时绑定 `latest`，回调执行时赋值 | 复用处理逻辑，同时标明各路写不同状态 |
| 返回 `RelativePosition` | 填完整位姿消息，附时间和参考系再发布 | 跨进程发送要说明坐标从哪里看、结果何时产生 |
| `show` 直接打印 | 在同一话题开观察订阅、查看字段或日志 | 终端打印与应用端点通信是不同的输出来路 |

此例 `solve` 能被普通 C++ 直接调用，因此可以把公式的错误与“收不到话题”的错误分开。真实项目还有包、工作空间与构建这些层。它们帮助独立程序可启动、可通信，但数字代入的因果关系没有变。

## 5. 如果改成命令式的 C 写法会怎样

核心结构的 C 版本通常由三个 `has_*` 标志和三个数值组成。接收函数拿到一份结构体指针，写对应的值和标志，再根据三个标志决定是否填输出结构体。C++ 写法把已有的两个概念分别交给 `optional` 和类成员。你已经有 C 的内存和函数经验，读这两种语言时可一直保留这条对应关系。

如果把数学函数中的 `radius_m` 写成静态或全局，再跨事件复用，程序就多了一个隐藏依赖。输入每次变化，半径由本次值重新算更直接。适合留成员的，是跨次事件确实需要记住的数据；可以每次重新得到的，留作局部计算。这样写也让读者一眼看到哪个数需要在两次回调之间存在。

[返回定位逐行篇](01-first-read.md) · [继续看完整的 ROS 2 教学包](10-build-a-teaching-package.md) · [课程首页](../../README.md)
