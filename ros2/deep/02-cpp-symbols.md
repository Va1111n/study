# 从打印缆长开始读 C++ 符号

船上程序收到一条缆长后，你眼前的第一个值叫 `msg.data`。这一篇用它解释打印、变量、数学函数、引用、数组和可选值。你有 C 基础，可以先读左边的熟悉写法，再对照右边。代码块有完整程序和局部片段，正文会说清片段放在什么上下文。

## 01 · 怎样把一个数打到屏幕上

### 先看两个完整程序

```c
#include <stdio.h>  /* Standard Input/Output，标准输入输出头文件。 */

int main(void) {
    double length_m = 5.0;  /* 变量存数，单位由后缀 _m 和约定表达。 */
    printf("cable length: %.1f m\n", length_m);
    return 0;
}
```

```cpp
#include <iostream>  // Input/Output Stream，输入输出流。
#include <iomanip>   // Input/Output Manipulators，输入输出格式控制。

int main() {
    const double length_m{5.0};

    // 每个 << 把右侧的内容送到左侧的输出流里。
    // fixed 和 setprecision 改变浮点显示格式，随后插入实际的值。
    std::cout << "cable length: " << std::fixed << std::setprecision(1)
              << length_m << " m\n";

    return 0;
}
```

屏幕上的一行是 `cable length: 5.0 m`。`printf` 把哪些位置是文字、哪些位置要换成变量写在同一个格式串里。`std::cout` 则把每项顺次送进流。`std` 是 C++ 标准库命名空间。`cout` 是标准输出流对象，常按 character output（字符输出）记忆。`::` 叫作用域解析运算符，告诉编译器到哪里找右边的名字。

第一次看时，把一条流语句断开理解：

```text
向标准输出流放入标签 "cable length: "
同一个输出流接着采用标明的小数格式
向同一个流放入 length_m 所存的数
向同一个流放入 " m" 和换行
```

链式写法之所以成立，是因为每次插入会返回这个流的引用，能继续用下一个 `<<`。这里操作的是流和可输出的值；在两个整数上，`<<` 则有你在 C 中熟悉的左移用法。C++ 能让同一运算符对特定类型提供不同操作，这叫运算符重载。眼前先根据左边是不是输出流判别。

### 引号、反斜杠、换行和精度

`" m\n"` 是字符串，包含空格、字母和换行。`'\n'` 是一个换行字符，只有一项。C 也沿用这样的转义规则。要直接打印一个数和换行，写 `std::cout << length_m << '\n';`。`std::endl` 会换行并刷新输出缓冲，通常只换行时用 `'\n'` 就够。刚开始试运行，消息很多时可以把临时打印放在稀疏事件中。

`std::fixed` 与 `std::setprecision(1)` 合用使小数点后显示一位。它们改变流的设置，因此之后同一流上的浮点输出仍受影响。只写 `std::cout << 5.0` 常见显示为 `5`，`double` 这个类型并没有因此变成 `int`。显示精度和变量本身的精度分别由两条规则决定。

### 把打印放回船上

船的回调写 `latest = msg.data`。先分开读这三个词：

- `msg` 是回调收到的消息对象。你可以类比成 C 函数的参数。
- `data` 是对象里的数字字段，存储类型是 `double`。点号进入这个字段。
- `latest` 是已绑定到节点成员的别名。赋值目标是节点内部缓存。

这条辅助回调给三路订阅共用。你可以打印通用标签 `"sensor value: "`，因为收到的可能是缆长、角度或深度。为区分话题，日志还要带接线信息，直接把标签写成 `"length"` 会误报另两路。只想在终端观察缆长的话，可以从话题读取。`std::cout` 走进程的标准输出，`publisher->publish(...)` 发 ROS 消息给订阅端。看到屏幕打印，等于验证了某段代码执行过；要验证其他节点收到消息，还得观察对应话题。

下表区分三种看似都在“往外输出”的代码。

| 写法 | 接收者 | 在船代码里何时找它 |
| --- | --- | --- |
| `std::cout << value` | 标准输出，可以连到终端、文件或进程输出收集器 | 临时观察数值、最小 C++ 程序 |
| `RCLCPP_INFO(get_logger(), "length %.2f m", value)` | ROS 2 日志系统，格式串与 C 的打印习惯相近 | 需要节点名、严重程度和时间的诊断信息 |
| `publisher->publish(message)` | 同话题兼容的订阅端 | 发送其他节点读取的结构化数据 |

`INFO` 表示 Information（信息级别）。日志格式符要和传入类型匹配。大量高频输出会让重要消息难找，也会增加回调执行时间。正式诊断可按需降低日志级别、限频或只在状态变化时输出。

## 02 · 类型、初始化、常量和物理单位

### 先认识一条声明中的每个位置

```cpp
const double length_m{5.0};
```

`double` 是浮点数类型，`length_m` 是变量名字，`{5.0}` 是初始化值。前面的 `const` 让这个变量初始化后不能再赋值。`m` 在这个名字里是米的标记。这只是人为约定，C++ 不会因为名字的后缀自动做单位转换。

大括号对缩窄转换尤其有用。`int meters{5.5};` 会报错，提醒你这个初始化可能丢掉小数。`int meters = 5.5;` 在常见编译设置下能编译并变成 5，编译器可以给诊断。实际物理量常用 `double`。事件次数、数组下标、队列深度是整数，布尔门禁用 `bool`。

| 代码 | 类型或结果 | 为什么 |
| --- | --- | --- |
| `5` | `int` | 字面量写成整数 |
| `5.0` | `double` | 小数形式的浮点字面量 |
| `5.0f` | `float` | `f` 后缀明确用单精度 |
| `double q = 5 / 2;` | `q` 为 2.0 | 整数除法发生在赋值之前 |
| `double q = 5.0 / 2;` | `q` 为 2.5 | 参与除法的值转为浮点再运算 |
| `bool valid = std::isfinite(q);` | 有限值时为 `true` | `isfinite` 来自 `<cmath>` |

### 哪些值应该固定

从消息中取得的缆长需要在每次计算中更新，生成这一帧的局部数值可以用 `const double length = *length_;`。前者是会变化的成员缓存，后者是一次计算中固定的副本。两者不会相互矛盾。`constexpr` 要求常量初始化能在编译时求值。例如一段校准多项式的固定系数可以存成 `constexpr std::array`。若将来系数从 YAML 或 ROS 参数读取，值到运行时才知道，适合改为运行时存储，读取时仍可以用只读引用。

物理单位也能沿名字和接口检查。`std::sin` 的输入是弧度，传入 90.0 会被理解成 90 弧度。度转弧度用 `degrees * pi / 180.0`。传感器可能报毫米，定位话题约定用米。收到 5000 毫米时需要先换成 5.0 米。把单位转换放在驱动发布前或独立转换节点里，可以让下一层接口一直使用相同约定。

### 有效数和可用数

`std::isfinite` 会排除 NaN（Not a Number，非数值）及正负无穷。`-7.0`、`1e100` 通常也是有限值，仍需按缆长的物理意义和量程检查。以定位节点为例，先做有限值检查能防止异常值污染缓存。设计输入接口时还要决定负缆长、超量程读数和深度大于缆长分别怎么处理。第 19 课会解释现有代码采用的策略。

## 03 · 为什么三个 & 看起来一样

### 在声明和表达式里对照

```cpp
double reading = 0.0;
double* pointer = &reading;  // & 出现在表达式：取 reading 的地址。
double& alias = reading;     // & 出现在声明：alias 引用 reading。
*pointer = 3.0;              // * 出现在表达式：访问地址指向的对象。
alias = 5.0;                 // 直接写 reading，因为它们是同一对象。
```

`reading` 最后是 5.0。第一个赋值改的是它，第二个赋值也是。`double* pointer` 可以在稍后指向另一个 `double`，`double& alias` 初始化时绑定，`alias = another` 的意思是把另一个数赋进 `reading`。引用类型需要一个有效对象，引用不会负责延长普通目标对象的生命。

```cpp
void read_copy(double value) {
    value += 1.0;         // 只改形参副本。
}

void write_length(double& output) {
    output = 5.0;         // 改调用者传进来的对象。
}

double to_centimeters(const double& input) {
    return input * 100.0; // 经由只读引用访问。单个 double 也可以直接按值传。
}
```

按值传适合小类型，尤其你明确要副本时。`const T&` 适合需要读取但不复制较大的对象，例如完整 IMU 消息。`T&` 很直接地表达“允许操作原对象”。像 `read_orientation(imu, q)` 那样有两个方向时，读签名就能分辨。调用方持有 `q`，共用函数填写它并告诉调用方是否成功。根据函数约定，先看返回的 `bool`，确认成功再使用输出。

### 与逻辑和按位运算区分

`[&latest]` 里的符号出现在 lambda 捕获列表，意思是引用方式访问原对象。`if (have_length && have_depth)` 有两个 `&`，是逻辑与，会在左侧为假时跳过右侧。单个 `&` 夹在两个整数表达式之间表示按位与。读符号时先找它处在**声明、取地址、捕获还是运算表达式**，再解释含义。

### 头文件为什么要放在这些位置

C 常把声明放头文件、实现在源文件。C++ 也如此。`#include <cmath>` 让编译器认识标准数学函数。`#include "imu_orientation.hpp"` 让两个姿态节点共同使用同一个读取函数定义。`""` 的搜索规则通常先看包含它的源文件附近，再看包含路径。`<>` 通常查配置好的包含路径。`#pragma once` 防同一编译单元重复包含。船的共享头文件中定义 `inline` 函数，允许按 C++ 的规则被多个编译单元包含同一份定义。编译阶段看得见声明与最后链接找得到实现是两个关口，完整过程见[构建篇](05-workspace-inside.md)。

## 04 · 数组、模板、auto 和 optional

### 先看一个你会读的 C 循环

```c
const double coefficients[3] = {2.0, 3.0, 4.0};
double result = 0.0;
double angle_rad = 0.5;
/* 从最高次项系数向常数项遍历。 */
for (int i = 0; i < 3; ++i) {
    result = result * angle_rad + coefficients[i];
}
```

C++ 的固定数组类可以这样写：

```cpp
// 需要 <array>。元素类型和数组长度都写在尖括号中。
constexpr std::array<double, 3> coefficients{2.0, 3.0, 4.0};
double result = 0.0;
const double angle_rad = 0.5;
for (double coefficient : coefficients) {
    result = result * angle_rad + coefficient;
}
```

第 1 次循环得到 2，第 2 次得到 4，第 3 次得到 6。乘完再加恰好算出二次多项式 `2x² + 3x + 4`。船里同样的三步结构由六个系数扩展为六次循环，数学上是五次多项式。第一次先把最高次系数放进去，后五次逐步乘出幂。`array` 大小固定，`vector` 可按需增长，原始 C 数组也仍能用。这里选择 `std::array`，因为有固定数量的系数，还能直接用于范围 `for`。

`std::array<double, 3>` 由一个类模板加两个参数确定一个具体类型。模板在源代码层描述一族写法。读 `create_publisher<Pose>` 时也用同样的方法，先看尖括号约定的类型，在这个例子里就是能发布的消息类型。圆括号里的话题名和队列深度属于这次创建传入的值。

### auto 有时会复制

```cpp
auto copied = pose.pose.covariance;   // 局部片段。复制消息中的数组。
copied[0] = 0.1;                     // 改的是副本。

auto& original = pose.pose.covariance;
original[0] = 0.1;                   // 改的是消息内部的数组。

const auto& view = pose.pose.covariance;
// 只读查看原数组，避免复制整个数组。
```

等号右边的类型可通过查看消息定义确定。`auto` 让编译器替你写类型。加不加 `&` 会决定对副本还是原对象操作。原程序需要填写消息后发布，因此这里保留引用。

### optional 与旧 C 写法一一对应

```c
#include <stdbool.h>

struct OptionalDouble {
    bool present;
    double value;
};
```

```cpp
#include <optional>

std::optional<double> depth;
depth = 0.0;
if (depth.has_value()) {
    const double meters = depth.value();
    // 这里 meters 为 0.0，表示确实有一次读数。
}
depth.reset();
```

| 在 C 结构体里 | C++ 写法 | 什么时候用 |
| --- | --- | --- |
| 初始 `present = false` | 默认构造 `optional` | 刚启动，还没收到这一项 |
| 一起写 `value` 和 `present` | `depth = 0.0` | 收到读数，数值可以为零 |
| `if (present)` | `if (depth)` 或 `has_value()` | 检查是否可取值 |
| 确认后读 `value` | 确认后读 `*depth` 或 `depth.value()` | 参与计算 |
| `present = false` | `depth.reset()` | 清除之前的读数 |

`value()` 在空时会抛出 `std::bad_optional_access`。用 `*depth` 应先保证存在值。三路检查通过后源码再取值，前后有明确的条件关系。以后扩展数据新鲜度，可把“数、时间、是否有效”设计为一个简小的记录，再用 `optional` 表达整条记录是否出现过。

[返回定位逐行篇](01-first-read.md) · [继续读类、生命周期与回调](03-objects-and-callbacks.md) · [课程首页](../../README.md)
