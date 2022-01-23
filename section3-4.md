## 3.4. 指令集



### 3.4.1. 概要


PIO 指令为 16 位，编码方式如下：

![表376](figures/table-376.png)

所有 PIO 指令的执行时间都是一个时钟周期。

5 比特的 `Delay/side-set` 字段的含义依赖于状态机的 `SIDESET_COUNT` 配置：

- 最多 5 个 LSB 比特（5 减 `SIDESET_COUNT`）为当前指令和下一条指令之间插入的空闲周期数。
- 最多 5 个 MSB 比特（由 `SIDESET_COUNT` 设置）为 side-set（参见[3.5.1节](#351-TODO)），可以在该指令执行的同时，将某些 GPIO 管脚设置为某个常量。


### 3.4.2. JMP

#### 3.4.2.1. 编码

![JMP](figures/instruction-jmp.png)


#### 3.4.2.2. 操作

如果 `Condition` 为真，则将程序计数器设置为 `Address`，否则无操作。

`JMP` 上的延时周期不论 `Condition` 是否为真都会生效。延时在 `Condition` 被求值、程序计数器被更新后进行。

- Condition：
  - 000：（无条件）：永远跳转
  - 001：`!X`：当寄存器 X 为零时
  - 010：`X--`：当 X 非零时，在减一后判断
  - 011：`!Y`：当寄存器 Y 为零时
  - 100：`Y--`：当 Y 非零时，在减一后判断
  - 101：`X!=Y`：当 X 不等于 Y 时
  - 110：`PIN`：根据输入管脚跳转
  - 111：`!OSRE`：当输出移位寄存器非空时
- Address：要跳转到的指令地址。在指令编码中，该值为 PIO 指令内存中的绝对地址。

`JMP PIN` 会根据 `EXECCTRL_JMP_PIN` 选择的 GPIO 管脚进行跳转。
