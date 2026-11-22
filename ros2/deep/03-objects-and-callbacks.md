# 从 C 的结构体、初始化函数和函数指针读 C++ 节点

读完一个节点的运行顺序后，再回来看它为什么写成一个类。下面用不依赖 ROS 的程序把状态、初始化、回调和生命周期逐个拆开。最后再把每个概念放回 `PoseNode`。

## 05 · 一个类里到底有哪些东西

先把倾角滤波器当成“读到新角度以后保存一个新的估计”。为了专注在状态管理，这里的示例用固定四分之一权重。船上的 `AngleFilter` 使用随方差变化的增益，另有角度跨界处理。

```c
#include <stdio.h>

struct AngleFilterState {
    double estimate_rad;
};

void init_filter(struct AngleFilterState *state, double initial_rad) {
    state->estimate_rad = initial_rad;
}

double update_filter(struct AngleFilterState *state, double measurement_rad) {
    /* 每个 state 都有自己的 estimate_rad。 */
    double error_rad = measurement_rad - state->estimate_rad;
    state->estimate_rad += 0.25 * error_rad;
    return state->estimate_rad;
}

int main(void) {
    struct AngleFilterState tilt;
    struct AngleFilterState azimuth;
    init_filter(&tilt, 0.0);
    init_filter(&azimuth, 1.0);

    double first_tilt = update_filter(&tilt, 0.8);
    double first_azimuth = update_filter(&azimuth, 1.8);
    printf("tilt %.2f, azimuth %.2f\n", first_tilt, first_azimuth);
    return 0;
}
```

两次输出分别是 0.20 和 1.20。传 `&tilt` 时第一个函数只操作倾角状态，传 `&azimuth` 时只操作方向状态。C 里主要靠类型、初始化函数、指针参数和调用约定把它们合在一起。C++ 可以直接定义对象如何初始化，以及哪些操作允许外部调用。

```cpp
#include <iostream>

class AngleSmoother {
public:
    // 构造函数与类同名，创建对象时自动调用。
    explicit AngleSmoother(double initial_rad) : estimate_rad_{initial_rad} {}

    double update(double measurement_rad) {
        const double error_rad = measurement_rad - estimate_rad_;
        estimate_rad_ += 0.25 * error_rad;
        return estimate_rad_;
    }

    // 末尾 const 表示读取状态时不修改当前对象的普通成员。
    double estimate() const {
        return estimate_rad_;
    }

private:
    double estimate_rad_;  // 每个对象都有自己的一份。
};

int main() {
    AngleSmoother tilt{0.0};
    AngleSmoother azimuth{1.0};

    tilt.update(0.8);
    azimuth.update(1.8);
    std::cout << "tilt " << tilt.estimate()
              << ", azimuth " << azimuth.estimate() << '\n';
    return 0;
}
```

C++ 版本把四件事移进一个类：

| 作用 | C 写法 | C++ 写法 |
| --- | --- | --- |
| 描述每个实例的状态 | `struct AngleFilterState` | `class AngleSmoother` 的成员 |
| 首次写入初始值 | 手工调用 `init_filter` | 创建对象时调用构造函数 |
| 选择修改哪个实例 | `update_filter(&tilt, 0.8)` | `tilt.update(0.8)` |
| 约束谁可以直接写状态 | 依赖接口约定，或隐藏结构体定义 | `private` 字段由类内操作 |

`tilt` 和 `azimuth` 是两个对象，也叫两个实例。进入 `tilt.update(...)` 后，编译器提供一个指向当前对象的 `this`。方法里直接写 `estimate_rad_` 可以读成 `this->estimate_rad_`。用 C 的经验理解，`this` 承担了显式传入的 `state` 指针的角色。能看懂这个对应关系，就能理解“用对象调用方法”并没有藏起状态，只是把状态和操作连到一起。

船里的 `AngleFilter tilt_filter_, azimuth_filter_;` 有两份估计和两份方差。同一个 `update` 逻辑可以给两组输入运行，上一帧倾角不会覆盖上一帧方向角。类能用 `private` 让调用者通过更新接口维护两份状态。成员名尾部的下划线是工程命名习惯，C++ 不会给它特殊意义。

## 06 · 构造、初始化、继承和组合

### 读懂构造函数冒号

`AngleSmoother(double initial_rad) : estimate_rad_{initial_rad} {}` 中，冒号后面是初始化列表。执行顺序先初始化成员，再运行最后一对花括号里的构造函数体。对于 `const` 成员和引用成员，必须在初始化阶段给值。对于一般成员，成员初始化同样把“初始是什么”写得更直接。成员的真实初始化顺序按类里声明的顺序，列表可以按同一顺序写便于核对。

带一个参数的构造函数加 `explicit`，要求调用处明确写出意图，例如 `AngleSmoother tilt{0.0};`。这样一个裸的 `double` 不会随意被当成滤波器。

### 节点为什么继承 rclcpp::Node

打开船代码课时，把 `class PoseNode : public rclcpp::Node` 各部分读一遍：

- `class PoseNode` 声明自己的类型。它知道缆长、角度、深度以及如何解算。
- `:` 后面是基类。`public rclcpp::Node` 让它在接口上是一种 ROS 2 节点，可以交给执行器。
- `Node("pose_node")` 出现在构造函数初始化列表，先初始化基类部分。同一对象因此拥有 ROS 节点接口和船的计算逻辑。
- 源码在构造函数体里直接用 `create_publisher`，是因为这些方法已经由基类提供。C++ 会在当前类和可访问的基类里查找方法。

对比另一句 `AngleFilter tilt_filter_;`。缆绳 IMU 节点**有一个**滤波器，这是组合。滤波器独立负责角度更新，节点负责消息接收和发布。两种写法经常同时出现。先问“对象需要被谁当成什么类型使用”，再问“对象内部持有哪些组件”，便能理解为什么这段用了继承、那段用了组合。

### 作用域与析构

```cpp
{
    AngleSmoother once{0.0};
    const double first = once.update(0.8);
    // once 和 first 在这对大括号内可直接访问。
}
// 离开大括号后，局部对象 once 的生命周期已经结束。
```

普通局部对象退出该作用域即析构。析构函数的名字是 `~类名()`，负责释放对象拥有的资源。没有额外资源时，编译器生成的析构通常足够。文件流、互斥锁持有者和智能指针能用析构自动归还资源。这种把资源的释放绑定到对象生命周期的方式叫 RAII（Resource Acquisition Is Initialization，资源获取即初始化）。

回到船节点。构造函数中的局部发布器变量退出构造即析构，而作为成员的发布器智能指针继续随节点保存。成员持有资源究竟要维持多久，应当由它的用途决定。每次发布时临时创建的消息在这一回调内有用，订阅器在整个节点运行期间有用。

## 07 · 指针、智能指针和“谁还持有它”

ROS 2 源码常用 `rclcpp::Publisher<Pose>::SharedPtr`。可以从外向内读：`rclcpp` 的 `Publisher` 模板给 `Pose` 类型生成具体发布器类型，这个类型内的别名 `SharedPtr` 表示指向它的共享智能指针。

```cpp
#include <memory>

struct LastDepth {
    double meters{0.0};
};

int main() {
    auto owner = std::make_shared<LastDepth>(); // 构造一个 LastDepth。
    owner->meters = 3.0;                       // 箭头访问目标的字段。
    {
        auto another_owner = owner;           // 两个共享指针指向同一对象。
        another_owner->meters = 3.5;          // 改的也是 owner 指向的字段。
    }                                         // 内层持有者结束，owner 仍持有它。

    return owner->meters == 3.5 ? 0 : 1;
}
```

拷贝 `SharedPtr` 拷贝的是指针式持有者，目标可能仍是同一个对象。最后一个共享所有权结束时，目标销毁。`std::unique_ptr` 表示独占所有权，能转交所有权；普通指针和引用常用来访问已存在的对象，生命周期需要由其他位置保障。这几个类型分别表达不同的所有权关系。

| 自己问一句 | 在定位节点里对应的答案 |
| --- | --- |
| 节点对象是谁创建的？ | `main` 用 `std::make_shared<PoseNode>()` 创建 |
| 回调执行期间谁保留了节点？ | 节点以共享指针交给运行它的代码，`spin` 持有并注册节点来处理事件 |
| 缆长订阅器为什么能继续有效？ | 创建接口返回的智能指针保存在 `length_sub_` |
| 缆长测量保存在指针里吗？ | 数值保存在 `length_`，订阅器负责把以后到达的数据交给回调 |
| 一帧局部消息什么时候离开作用域？ | 发完消息，`publish_position` 返回时 |

注意“对象寿命稳定”和“并发访问安全”是两件事。共享指针管理对象何时释放。若以后多个线程同时修改目标的普通字段，仍要设计同步方式。现有每个节点的常规 `spin` 串行处理回调，缓存写入和定位计算在同一个回调链内完成。

## 08 · 回调与 lambda 为什么能留到后来运行

### 一个用 C 写的登记和调用过程

这个小程序用函数指针加上下文。看 `register_callback` 结束时留下了什么，再看稍后的 `deliver` 做了什么。

```c
#include <stdio.h>

typedef void (*ReceiveCallback)(void *context, double measurement);

struct Endpoint {
    ReceiveCallback callback;
    void *context;
};

struct Cache {
    double last_value;
};

void store_value(void *context, double measurement) {
    struct Cache *state = context;
    state->last_value = measurement;
}

struct Endpoint register_callback(ReceiveCallback callback, void *context) {
    struct Endpoint endpoint = {callback, context};
    return endpoint;
}

void deliver(struct Endpoint *endpoint, double measurement) {
    endpoint->callback(endpoint->context, measurement);
}

int main(void) {
    struct Cache length = {0.0};
    struct Endpoint receiver = register_callback(store_value, &length);
    deliver(&receiver, 5.0);
    printf("%.1f\n", length.last_value);
    return 0;
}
```

登记时只有函数和“应该操作谁”，5.0 在 `deliver` 那一刻才出现。`length` 属于 `main`，在登记函数返回后还存在，因此保留下来的地址可以使用。

### 同一过程用 C++ 写

```cpp
#include <functional>
#include <iostream>
#include <optional>

int main() {
    std::optional<double> length; // long-lived target，活到 main 结束。
    std::function<void(double)> callback;

    // 这里登记回调。创建可调用对象时，记住将来要访问哪个 length。
    callback = [&length](double measurement) {
        length = measurement;
    };

    // 这里模拟消息到达，执行登记好的操作。此时才传入 measurement。
    callback(5.0);
    if (length) {
        std::cout << "length " << *length << " m\n";
    }
    return 0;
}
```

`std::function<void(double)>` 这个类型能保存符合“收一个 double、不返回值”的可调用对象。这里用它展示“登记”和“调用”可以分开。ROS 2 接口保存传给它的 lambda，接口会在数据就绪时交给执行器调用。`[&length]` 把要改的对象绑到这个 lambda 内，效果类似 C 例子保存了 `&length`。

### 捕获的是谁，参数又是谁

| 写法 | 创建时留下什么 | 调用时发生什么 |
| --- | --- | --- |
| `[](double x) { ... }` | 无外部捕获 | 得到一次调用的 `x` |
| `[scale](double x) { ... }` | 保存 `scale` 的副本 | 用登记时保留的比例处理这次 `x` |
| `[&latest](double x) { latest = x; }` | 允许访问原对象 | 写到绑定的目标 |
| `[this](double x) { process(x); }` | 保存指向当前对象的指针 | 调用当前对象的方法 |

在船的辅助函数里，`latest` 是引用形参。调用时来的是 `length_`、`angle_` 或 `depth_`，它们存在于节点对象内部。传给三次 `create_subscription` 的是三个登记好的可调用对象。你在文件里只看见一段 lambda，运行时每个订阅器各有自己的绑定。

### 看一个反例，理解它为何需要成员

下面的函数返回一个待调用的 lambda。写法用来说明悬空引用，正文不执行它。

```cpp
auto bad_delayed_update() {
    double temporary = 0.0;
    return [&temporary](double value) { temporary = value; };
}
```

函数返回时，`temporary` 的生命周期结束。以后调用返回的函数对象会访问已经结束生命周期的变量，属于未定义行为。即使某次测试看起来能跑，下次编译或换机器也可能出错。

船上的设计让目标留在节点成员中，订阅器也由该节点持有。把代码改成“在构造函数里申请临时缓存，然后按引用给长期订阅器”会破坏这种关系。更改回调时先确认目标由谁拥有、谁先销毁。

## 把这些概念放回船文件

读一份新节点时按这个顺序：

1. 在 `main` 找创建了哪个类和谁执行它。
2. 在类的初始化列表找基类和初始值。
3. 在成员列表分清操作所需的长期状态、发布器、订阅器和定时器。
4. 在构造函数找每个端点的消息类型、名字、服务质量和回调。
5. 在回调里沿字段找到读取来源与赋值目标。
6. 在计算函数里找提前返回条件，再找真正输出的一行。

这样读出的是执行关系。类定义、模板语法、引用和智能指针各解决其中一项需要。

[继续读 ROS 2 通信](04-topic-from-scratch.md) · [课程首页](../../README.md)
