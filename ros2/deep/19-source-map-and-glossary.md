# 看见一个名字，去哪份源码和哪段讲解找它

这是读代码时可随时打开的索引。篇章按读源码所需的疑问组织。对课文中的简称可直接查英文全称与中文含义。带斜杠的工程路径都以船仓库为起点，实际行号随着代码修改会变化，优先搜索稳定的类名或函数名。

## 1. 打开四个节点，先问各文件为什么存在

| 文件 | 先找哪个类或函数 | 谁向它提供什么 | 谁继续使用它的结果 |
| --- | --- | --- | --- |
| `orangpi_ros2/src/usv_localization/src/pose_node.cpp` | `PoseNode`、`subscribe_scalar`、`publish_position` | 驱动或模拟源给缆长、深度，标定器给方位 | 下游使用 `/solver/pose`，调试观察 `/solver/debug_point` |
| `orangpi_ros2/src/usv_localization/src/imu_test1.cpp` | `CableImuNode`、`AngleFilter::update` | 缆绳 IMU 的姿态四元数 | 标定器订阅原始方位 |
| `orangpi_ros2/src/usv_localization/src/cable_angle_calibrator.cpp` | `CableAngleCalibrator`、`coefficients` | 缆绳方向计算节点给原始角 | 定位使用标定后的弧度 |
| `orangpi_ros2/src/usv_localization/src/mother_usv_msg_node.cpp` | `MotherYawNode`、`heading_`、`create_wall_timer` | 母船飞控桥接给 IMU 数据 | 航向消费者订阅度数，和定位直接输入独立 |
| `orangpi_ros2/src/usv_localization/src/imu_orientation.hpp` | `read_orientation` | 两处姿态回调传入 IMU 消息及可写四元数 | 共用姿态有效性检查和归一化 |

这几个文件不需要按字典顺序读。若关心结果，先开 `pose_node.cpp`，找到它要求三路输入，然后各自往前追。发现其中一路不存在代码，再去查原工作空间、配置和板上运行图。若从硬件开始读，跟随某个测量产生的位置逐步找到它发出的 ROS 消息。两个方向最终应能连到同一张图。

### 打开一个源码后，先扫这六类位置

| 找到的名字或语法 | 它回答的问题 | 本教材对应篇章 |
| --- | --- | --- |
| `using` | 后面这个短名到底是什么类型 | [符号详解](02-cpp-symbols.md) |
| `main` | 操作系统启动后，从哪一行开始构造对象 | [定位逐行篇](01-first-read.md) |
| `class ... : public ...` | 对象自身负责哪些事，哪些能力来自基类 | [对象详解](03-objects-and-callbacks.md) |
| `create_publisher`、`create_subscription` | 端点用什么名字、类型、QoS，谁处理到达事件 | [通信详解](04-topic-from-scratch.md) |
| `private:` 后的成员列表 | 节点在多次回调之间记住了什么，持有了哪些端点 | [对象与寿命](03-objects-and-callbacks.md) |
| `publish` | 消息在哪条执行路径上真正送出，输出字段从哪来 | [定位跟读](01-first-read.md)、[消息结构](12-message-contracts.md) |

如果所有源码只允许先看十行，先看 `main`、订阅初始化和发布调用附近。数学细节暂且对应到独立函数。下一次想改变行为时，才用成员的作用域、回调捕获和消息合同判断改动位置。这样每次看文件都有一个明确的问题。

## 2. 工程层文件各在哪个生命周期被读到

| 文件或位置 | 谁用它 | 读它能回答的一个实际问题 |
| --- | --- | --- |
| `orangpi_ros2/README.md` | 开发者和部署者 | 正确的系统版本、有哪些输入输出、如何按惯例启动 |
| `usv_localization/package.xml` | ROS 2 包工具 | 包名是什么，构建和运行缺哪个依赖 |
| `usv_localization/CMakeLists.txt` | CMake、构建流程 | 这个源文件编成哪个程序，哪些内容会复制到安装区 |
| `usv_localization/launch/localization.launch.xml` | ROS 2 启动工具 | 哪些程序默认起，参数在哪设置，何时增加 EKF |
| `usv_localization/config/ekf.yaml` | 滤波节点 | 从各个话题读哪几项状态，滤波三维还是二维 |
| `usv_localization/test/test_localization.py` | 测试流程 | 软件怎么提供模拟 IMU 与深度，断言哪些消息字段 |
| 旧 `mz_usv/src/auto_pwm.cpp` | 原板端推进程序 | 手柄输入如何变成左右 PWM |
| 旧 `usv/src/motor.cpp` | 原板端缆绳程序 | 串口反馈如何变成缆长话题 |
| 旧设备规则与飞控 launch | 原系统与驱动启动过程 | 设备别名如何创建，飞控启动节点在什么命名空间 |

最重要的目录关系是两层 `src`。第一个是工作空间里面的包集合，第二个是单个包里面的源文件。`install` 归编译与安装过程，`launch` 是配置入口，不是所有节点的源码堆放处。按[构建篇](05-workspace-inside.md)的流程把它们串上，看到“源码改了、运行没改”的问题就知道检查哪几步。

## 3. 船的信号索引

| 信号名 | 数据形状 | 读它时先问 |
| --- | --- | --- |
| `/imu` | IMU 消息 | 有效姿态已经给出吗，传感器下向量与真实缆绳是否对齐 |
| `/sensor/cable_azimuth_original` | Float64，弧度 | 原始角来自竖直默认分支，还是来自水平投影 |
| `/sensor/cable_azimuth` | Float64，弧度 | 是真实标定结果，还是绕过标定的合成输入 |
| `/sensor/cable_length` | Float64，米 | 原板软件或测试源谁在发布，正负与比例对吗 |
| `/sensor/relative_depth` | Float64，米 | 深度基准、更新频率和实际发布端是什么 |
| `/solver/pose` | 带时间与协方差的位姿 | 解算是哪条新输入触发，参考系是什么 |
| `/solver/debug_point` | 带头部的三维点 | 能否同一时间戳与位姿对照 |
| `/mother/mavros/imu/data` | 母船飞控 IMU | 母船桥接是否正常，安装偏置在下一步才加 |
| `/mother_usv/imu/yaw` | Float64，度 | 数值是首帧前默认，还是最近姿态转换后的航向 |
| `/mavros/imu/data` | 子船 IMU | 与母船话题区分，坐标和时间可供 EKF 使用吗 |
| `/rov/odometry/filtered` | 滤波后的里程计 | 是否启用了可选滤波，姿态来自哪里，哪个是子 frame |

这些是默认根命名空间下的接口名。使用自定义 launch 改名字后，用实际端点信息作为排查依据。深度接口在已备份的旧工程没有定位到发布者。源码索引给出约定和待核对位置。

## 4. C++ 写法快速对照

| 写法 | 全句怎么读 | 一个常见的读错方式 |
| --- | --- | --- |
| `int main()` | 进程从这里进入普通 C++ 程序 | 按源码顶部写着哪个类就认为先运行哪个方法 |
| `std::cout` | 标准库命名空间中的标准输出流对象 | 把流当成函数名，直接用圆括号传格式串 |
| `stream << value` | 向输出流插入值，并继续返回流引用 | 在此处按整数位移读 |
| `object.field` | 进入对象的一个成员 | 对指针包装直接使用点号找目标字段 |
| `pointer->field` | 通过指针式对象进入所指对象 | 对消息值或引用无条件使用箭头 |
| `const T&` | 只读访问已存在的 T 对象 | 当成取地址或在回调里尝试改只读输入 |
| `T&` | 访问原对象，允许修改 | 当成副本，忽略调用者的状态被改变 |
| `&value` | 取一个已有变量的地址 | 把它与声明中出现的引用符号混在一起 |
| `std::optional<double>` | 这个容器可以装一个 double 或为空 | 把其中的 0.0 当成空 |
| `if (!optional_value)` | 测试当前有没有装值 | 把它读成内部数值是否是零 |
| `*optional_value` | 取出已经存在的内部对象 | 空时也直接解引用 |
| `auto& values = message.covariance;` | 推导元素容器类型，并绑定原数组 | 漏掉 &，修改到副本 |
| `[this, &latest]` | 将节点指针和目标引用保留下来，供以后调用 | 把回调注册当成已有一条传感器输入 |
| `create_subscription<Scalar>` | 用 Scalar 定义订阅端点接受的消息类型 | 认为尖括号里填话题名字 |
| `SharedPtr` | 持有共享所有权的智能指针 | 认为它代表一整条 Float64 消息 |
| `public` 与 `private` | 类的接口与内部实现的访问范围 | 误以为 public 的方法在启动时会自动执行 |
| `PoseNode() : Node("pose_node")` | 构造 PoseNode 时先初始化基类 | 把初始化列表读成继承声明本身 |
| `return;` | 离开当前 void 函数 | 以为会自动结束整个节点 |
| `std::isfinite(msg.data)` | 判定数字既非无穷也非 NaN | 以为它同时验证了缆长非负、单位和设备量程 |

把看到的符号先按**声明、表达式、参数、成员和控制流**分类。例如 `const Scalar& msg` 完整讲的是有一个名叫 msg 的只读引用；`msg.data` 只是在读这个参数的字段；`latest = msg.data` 在另一个对象上持久保存数。按三个层次展开，原来挨着写的表达式就各有角色。

## 5. 缩写和工程词汇

| 写法 | 英文全称或来源 | 在这套代码中的意义 |
| --- | --- | --- |
| ROS / ROS 2 | Robot Operating System / 第二代 ROS | 组织节点、消息、运行和工具的机器人软件生态 |
| rclcpp | ROS Client Library for C++ | C++ 使用节点、发布器与执行器的库 |
| rclpy | ROS Client Library for Python | Python 集成测试创建发布、订阅和节点 |
| USV | Unmanned Surface Vehicle | 无人水面艇，出现在定位包和话题名称 |
| ROV | Remotely Operated Vehicle | 常指遥控作业设备，这里用作子船机体和输出命名的一部分 |
| IMU | Inertial Measurement Unit | 惯性测量单元，本文用其消息中的姿态求方向 |
| RPY | Roll, Pitch, Yaw | 横滚、俯仰、偏航，描述旋转的三个欧拉角 |
| EKF | Extended Kalman Filter | 扩展卡尔曼滤波器，组合相对位置和子船 IMU |
| TF | Transform | 坐标变换，`tf2` 管理参考系之间的关系 |
| QoS | Quality of Service | 服务质量约定，包括可靠性、历史与持久性 |
| DDS | Data Distribution Service | 数据分发服务，常见于 Humble 中间件 |
| MAVLink | Micro Air Vehicle Link | 飞控与上位系统之间可用的消息协议 |
| MAVROS | MAVLink 与 ROS 的桥接软件 | 把控制器相关信息提供给 ROS 节点 |
| PWM | Pulse-Width Modulation | 脉宽调制，旧母船推进链使用 |
| ESC | Electronic Speed Controller | 电子调速器，将控制信号作用到电机 |
| GPIO | General-Purpose Input/Output | 通用输入输出，与物理引脚和复用配置相关 |
| WSL | Windows Subsystem for Linux | 在 Windows 电脑上运行 Linux 环境 |
| ROS_DOMAIN_ID | ROS domain identifier | 通信域标识，区分中间件参与者 |
| XML | Extensible Markup Language | 可扩展标记语言，当前 launch 的文件格式 |
| YAML | YAML Ain't Markup Language | 常用配置格式，当前 EKF 参数位置 |
| NaN | Not a Number | 浮点非数值，定位输入过滤的情况之一 |
| RAII | Resource Acquisition Is Initialization | 让资源释放跟着持有对象的生命周期 |
| CLI | Command-Line Interface | 命令行接口，诊断中使用 ros2 工具 |
| Hz | Hertz | 赫兹，描述每秒更新次数 |
| rad | radian | 弧度，定位使用的角度单位 |
| odom | odometry | 里程计，输出估计位姿并可提供速度 |
| pub / sub | Publisher / Subscription | 发布器、订阅器，常见于端点成员名 |
| msg | Message | 消息，常见于消息包命名和回调参数 |
| src | Source | 源文件或源工作空间目录 |
| argc / argv | Argument Count / Argument Vector | 程序入口的参数个数与参数列表 |

[返回课程首页](../../README.md) · [用一个运行记录串起来](15-reading-a-real-run.md)
