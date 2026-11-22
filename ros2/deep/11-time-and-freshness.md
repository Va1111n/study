# 为什么位姿还在更新，而深度已经停了

这一篇只专注时间和缓存。先找船里的 `latest = msg.data` 和 `pose.header.stamp = now()`，两行时间上相邻，含义却不同。第一行记读数，第二行给一次解算结果写时间。下面用明确的时间轴说明现在为何会出现旧值，以及要怎么设计一项可验证的改进。

## 1. 用一张事件表看旧数据怎样留下来

取本机单调时间作为例子。设三个输入分别提供：

| 时刻（秒） | 到达事件 | 计算使用的值 | 发生什么 |
| --- | --- | --- | --- |
| 0.00 | 启动 | 无 | 建立话题，三格空 |
| 1.00 | 缆长 5.0 | 缆长 5.0 | 深度和角度仍空 |
| 1.02 | 标定角 0.0 | 长度 5.0，角 0.0 | 等深度 |
| 1.05 | 深度 3.0 | 长度 5.0，角 0.0，深度 3.0 | 产生位姿，时间接近 1.05 |
| 1.10 | 缆长 5.1 | 长度 5.1，角 0.0，深度 3.0 | 再次产生位姿 |
| 1.35 | 标定角 0.1 | 长度 5.1，角 0.1，深度 3.0 | 继续解算 |
| 4.00 | 缆长 5.2 | 长度 5.2，角 0.1，深度 3.0 | 生成近 4.00 的结果，深度实际已旧约 2.95 秒 |

第 4.00 秒时，程序可以正确地说“我刚运行了一次公式”。与此同时，它也使用了近三秒前缓存的深度。只检查位姿时钟，看不出原始数据是否持续更新。这是由 `Float64` 的字段和缓存设计共同决定的。

`std::optional` 的状态只有已存值与未存值。它在第 1.05 秒装了 3.0 后，经过一秒、两秒不会自动清空。定时超时、采样年龄检测都需要程序加入规则。目前 `PoseNode` 只有“从未收到”的门禁。`robot_localization` 的 `sensor_timeout` 只能感知自己的输入，若位置节点一直报新时间，它无法推知某一份原始读数的年龄。

## 2. 一次测量可能经过哪些时间点

假设深度设备 100.000 秒完成采样，驱动 100.008 秒读到设备缓冲，处理完后 100.016 秒发布消息。定位进程 100.025 秒收到，100.031 秒真正进入回调，再在 100.032 秒执行解算。观察者 100.050 秒从 ROS 2 收到结果。六个时刻有明确含义。

| 时间 | 对应含义 | 怎样取得 |
| --- | --- | --- |
| 100.000 | 设备采样时刻 | 设备时间字段，或采样过程给出的时间 |
| 100.008 | 驱动拿到字节或读数 | 驱动进程的时间 |
| 100.016 | 驱动提交消息 | 发布端日志或消息字段 |
| 100.025 | 中间件收到消息 | 取决于接口与记录能力 |
| 100.031 | 业务回调开始运行 | 在回调内记录本机时间，常当作应用接收时间 |
| 100.032 | 解算结果被创建 | `now()` 填位姿头部 |
| 100.050 | 观察者处理输出 | 观察端自己记录 |

延迟可来自设备缓存、网络、发布周期和回调等待。软件看到的“接收时间”一般是回调开始处理读数的时间。若队列里积了 1 秒的老数据，回调时间仍很新。要判断**感测时间**是否旧，最终要由上游携带真实采样时间，并检查共享时钟和协议。

## 3. 两种时间问题，适合用不同条件

### 条件一 · 是否长时间没收到任何新数据

强制某路输入“最近一段时间有回调更新”。例如深度允许最大接收年龄 0.5 秒，缆长允许 0.2 秒，角度允许 0.15 秒。阈值要从设备正常发布间隔、抖动和解算要求一起确认，数值这里只用于讲解。用不因系统时钟校正而倒退的本机单调时钟，可以稳定计算两次处理之间过去了多久。

当测量从未出现，就不是超时测量，它的记录应为空。当出现过但接收年龄大于阈值，记录仍可供诊断看最后值，计算门禁把它视为过期。恢复新帧后重新记录时间和值。状态变化发生在数据到达或程序检查时。

### 条件二 · 三项测量是否接近同一个物理时刻

即使三路最近都有更新，时间也可能不适合混在一起。例如转弯时某条新方位比慢速深度领先很久。计算要求测量发生在时间窗内，应该比较采样时刻，或者对不同传感器时间戳做近似同步和缓冲选择。`Float64` 不带采样时间，最新值并不能自己组成这种时间对齐保证。

存在“慢速但稳定的深度”和“快速变化的角度”时，不宜简单要求同一毫秒采样。设计依据应来自速率、物理模型和能接受的空间误差。最多允许多少时间差属于数据合同。带时间戳的上游接口也要说明它的时钟来源。

## 4. 一个独立 C++17 示例用来讲清超时

这个示例显式接收时间点，`main` 传入模拟时刻，因此不需要真的等几秒便能观察各状态。它保存最近值和本机接收时刻，重点演示何时等候、何时可使用。把同样原则接回 ROS 2 前要先确认时间来源和业务阈值。

```cpp
#include <chrono>
#include <cmath>
#include <iostream>
#include <optional>

using Clock = std::chrono::steady_clock;
using TimePoint = Clock::time_point;
using Milliseconds = std::chrono::milliseconds;

struct TimedValue {
    double measurement;
    TimePoint received;
};

class Measurements {
public:
    void receive_length(double meters, TimePoint time) {
        update(length_, meters, time);
    }

    void receive_angle(double radians, TimePoint time) {
        update(angle_, radians, time);
    }

    void receive_depth(double meters, TimePoint time) {
        update(depth_, meters, time);
    }

    const char* readiness(TimePoint now) const {
        if (!length_) {
            return "missing length";
        }
        if (!angle_) {
            return "missing angle";
        }
        if (!depth_) {
            return "missing depth";
        }

        if (now - length_->received > Milliseconds{200}) {
            return "stale length";
        }
        if (now - angle_->received > Milliseconds{150}) {
            return "stale angle";
        }
        if (now - depth_->received > Milliseconds{500}) {
            return "stale depth";
        }
        return "ready";
    }

private:
    static void update(std::optional<TimedValue>& output,
                       double measurement, TimePoint time) {
        if (std::isfinite(measurement)) {
            // 值与时间一起写入，作为同一项有效结果。
            output = TimedValue{measurement, time};
        }
    }

    std::optional<TimedValue> length_;
    std::optional<TimedValue> angle_;
    std::optional<TimedValue> depth_;
};

int main() {
    const TimePoint start{};
    Measurements latest;

    latest.receive_length(5.0, start + Milliseconds{30});
    latest.receive_angle(0.0, start + Milliseconds{40});
    std::cout << latest.readiness(start + Milliseconds{50}) << '\n';

    latest.receive_depth(3.0, start + Milliseconds{60});
    std::cout << latest.readiness(start + Milliseconds{60}) << '\n';

    // 其余两路新鲜，只有深度超过了允许年龄。
    latest.receive_length(5.1, start + Milliseconds{580});
    latest.receive_angle(0.1, start + Milliseconds{590});
    std::cout << latest.readiness(start + Milliseconds{600}) << '\n';

    // 水面读数是真实 0.0，刷新时间之后能解除深度过期。
    latest.receive_depth(0.0, start + Milliseconds{610});
    std::cout << latest.readiness(start + Milliseconds{610}) << '\n';
    return 0;
}
```

为什么 `readiness` 末尾有 `const`？它只读取现有缓存，把解释通过返回值交给调用者。查询本身不改变任何一个传感器的最后有效读数。`static void update(...)` 表示这个辅助函数不直接使用隐式 `this`，它所需的目标和值已在参数里给出。调用它依然会修改通过引用传入的目标，这是引用的预期效果。

`TimePoint start{}` 在例子里只是可加时间间隔的人造起点。真实进程会用 `Clock::now()` 得到本机单调时间。`now - depth_->received` 是时长，拿毫秒间隔比较会保持单位清楚。`readiness` 返回的是字符串常量地址，用 `const char*` 足够表达课程示例所需。真正要提供给其他程序作逻辑判断时，更适合用 `enum class` 表达状态，再把状态转成展示文字。否则业务逻辑容易依赖文本和翻译。

第四行恢复用 0 米，是为了说明缺值、过期和零分别由不同信息决定。有限值检查只影响是否更新。若 610 毫秒收到 NaN，缓存仍是 60 毫秒那份深度，状态会继续是过期。只记录最后有效接收时间，还无法区分“设备停止发消息”和“持续发无效消息”。需要排查这类故障时，可另存最后见到任何消息的时间，让两种现象都可判断。

## 5. 选用时钟的时候问四个问题

| 问题 | 较合适的时间 | 需要留意 |
| --- | --- | --- |
| 我想知道同一进程两次处理之间经过多久 | C++ 单调时钟 | 本机进程使用，不能直接和另一台机器比较绝对值 |
| 我想写 ROS 2 输出时间戳 | 节点时钟 | 可能使用系统时间或仿真时间，跟整个系统约定一致 |
| 我想让三台设备的测量按同一时刻比较 | 同步后的采样时钟 | 消息要提供采样时间，检查偏差与同步来源 |
| 我希望录包回放时重现超时 | 按约定随回放驱动的时间或可注入的时间 | 加速、暂停回放和系统单调时间的关系要明确 |

实际船的代码用了 `now()`。这里的示例用了 `steady_clock`，两者分工不同。可在节点同时保存“用于检查本机经过多久”的时间和“对外标记测量或计算”的时间。不要直接相减不同来源的时间点。系统时钟重新校正、仿真时间暂停、回放倍速变化，会让看似相同的“0.5 秒”得到不同结果。用可注入时刻测试能明确各分支，系统集成测试则继续验证实际环境采用了预期时钟。

## 6. 什么时候要主动报告失效

如果定位节点只在任一路更新时检查，一旦所有传感器同时停更，它没有新事件去运行 `readiness`。业务可能已经停止输出位置，但也不主动告诉下游“全部停了”。这时可以让下游为位姿流设计超时，或者在定位节点加一个小频率的诊断定时器定期报告状态。前者反映下游自身收到信息的情况，后者能指出哪路上游已失效。两边观测各有证据。

对于导航或控制消费者，要明确定义“没有新位置”和“收到过期标志”分别怎么处理。只停发数据可能让另一个节点也永远保留上次位置。新鲜度合同应沿整个链路传递。项目当前独立计算层可以先验证停止输出与恢复，再约定与控制层的接口。

## 7. 在测试中怎样把条件一项项分开

1. 先发长度和角度，深度缺席，确认没有可用结果。
2. 加入深度后，计算可用，检查数值未受新规则影响。
3. 停掉深度，保留另外两路，并选两个时刻：刚低于阈值时仍可用，明确超过时进入过期。
4. 发送一次非有限深度，验证它无需刷新已有记录。
5. 重新给出真实 0 米，恢复计算，检查零值含义不变。
6. 让所有输入都停更，若加了主动诊断，再验证是否会独立报告过期。

这些是为改进功能写的验收推导。当前教程中的代码片段只在学习文件，原船工程的处理逻辑要经过单独需求与测试确认后再改。

[回到故障分析](07-debug-by-evidence.md) · [看消息字段怎么设计](12-message-contracts.md) · [课程首页](../../README.md)
