# 第 01 课 · 用 C++ 打印收到的缆长

如果你在 C 里写过 `printf("Hello\n");`，这次只需要认识它在 C++ 里的常见写法。先看一个完整的小程序。

```cpp
#include <iostream>  // Input/Output Stream，输入输出流头文件

int main() {
    // cout 是标准输出流。<< 把右边的内容送给左边的输出流。
    std::cout << "Hello, C++!" << '\n';
    return 0;
}
```

运行后会显示 `Hello, C++!`，然后换行。

## 按 C 的经验逐行读

| 代码 | 怎么读 | 在 C 里的对应知识 |
| --- | --- | --- |
| `#include <iostream>` | 引入标准输入输出流的声明，才能使用 `std::cout` | 写 `printf` 前引入 `<stdio.h>` |
| `int main()` | 程序从这里开始，返回一个整数状态码 | 对应 C 程序里的 `int main(void)` |
| `std::` | 到标准库的命名空间里找名字。这里的 `::` 叫作用域解析运算符 | C 没有命名空间，库常用名字前缀规避重名 |
| `cout` | 常读作 character output，在标准库里是标准输出流对象 | 和 `printf(...)` 常用的 `stdout` 指向同一个标准输出 |
| `<<` | 把右边的值插入左边的流。一次插入结束后，流还能继续接收下一个值 | 类似依次把内容交给 `printf`。当两边是整数时，`<<` 才按这里熟悉的位移用法理解 |
| `"Hello, C++!"` | 要显示的一串字符 | C 的字符串字面量也是双引号 |
| `'\n'` | 一个换行字符 | C 也用同样的转义写法 |
| `return 0;` | 告诉操作系统程序成功结束 | 和 C 相同 |

## 打印一个变量

看一行船上的常见数据。变量还是用你熟悉的方式声明。

```c
/* C 写法 */
double cable_length = 5.0;
printf("Cable length: %.1f m\n", cable_length);
```

```cpp
// C++ 写法
double cable_length = 5.0;
std::cout << "Cable length: " << cable_length << " m\n";
```

C 的格式串用 `%.1f` 指定保留一位小数。C++ 的流从变量类型识别这是一个浮点数，再按流的默认格式打印。因此这行 C++ 可能显示 `5`，C 的例子会显示 `5.0`。两种写法中的 `cable_length` 都表示以米为单位的缆绳长度。

如果你希望 C++ 也显示 `5.0`，加入 `#include <iomanip>`。这个头文件提供输入输出格式控制工具，名称来自 Input/Output Manipulators。

```cpp
// fixed 让浮点数以定点格式输出。此时 setprecision(1) 指小数点后一位。
std::cout << "Cable length: " << std::fixed << std::setprecision(1)
          << cable_length << " m\n";
```

格式设置会留在同一个流里，影响后续的浮点输出。平时只想打印一个数，可以直接用 `std::cout << cable_length << '\n';`。

## 把这一行放回船代码

在 `pose_node.cpp` 里，`msg.data` 就是刚收到的数。下面是阅读回调时可用的临时诊断示例。原来的回调由三个话题共用，因此共用函数里打印时，标签用“收到的数值”。米和弧度取决于当次回调属于哪个订阅器。

```cpp
// 需要在源文件顶部加入 #include <iostream>。
// 放在回调的有效值分支里，msg.data 是这一条消息携带的 double。
std::cout << "Sensor value: " << msg.data << '\n';
```

要只观察缆长，可以读话题 `/sensor/cable_length`。在节点里留长期诊断信息时，ROS 2 的日志能附带节点名、时间和严重程度。高频消息适合降低日志频率，避免终端被同类信息淹没。

## 源文件怎样变成能运行的程序

把本课开头的完整程序保存为 `hello.cpp`。在 **WSL 的 Ubuntu 22.04 终端**进入文件所在的目录，用 `g++ hello.cpp -o hello` 编译，再用 `./hello` 运行。

- `.cpp` 是 C++ 源文件的常见扩展名，类似 C 源文件的 `.c`。
- `g++` 是 C++ 编译器的命令。它读取源文件，生成可执行程序。
- `-o hello` 指定输出文件名为 `hello`。
- `./hello` 中的 `./` 指当前目录。

回到 `pose_node.cpp` 的 `main`，可以看到同样的程序入口。船的节点要按包的依赖构建，因为它还使用 ROS 2 消息和库。

[返回定位节点](01-pose-node.md) · [继续阅读类型、引用和类](02-cpp-in-the-ship.md) · [课程首页](../README.md)
