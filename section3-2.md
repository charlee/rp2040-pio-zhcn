## 3.2. PIO 程序

四个状态机执行共享指令内存的程序。系统软件将内存加载至该区域，配置状态机和 IO 映射，然后将状态机设置为运行状态。PIO 程序可以来自多个地方：
可以由用户直接汇编，可以来自 PIO 库，或者由用户的软件生成。

之后，状态机就会自动运行，系统软件通过 DMA、中断和控制寄存器与之交互，就像操作 RP2040 上的其他外设一样。
对于更复杂的接口，PIO 提供了一个短小灵活的指令集，可以让系统软件更深入地操作状态机的控制流。

| ![图39](figures/figure-39.png) |
|:--:|
| 图39. 状态机概览。数据通过一对 FIFO 流入流出。状态机执行一段程序，在这些 FIFO、一系列内部寄存器和管脚之间传输数据。时钟分割器可以按照常数因子降低状态机的执行速度。 |


### 3.2.1. PIO 程序


PIO 状态机执行短小的二进制程序。

PIO 库提供了像 UART、SPI 或 I2C 等通用接口的程序，因此许多情况下不需要自己编写 PIO 程序。但是，直接对 PIO 编程可以带来更大的灵活性，
支持许多连设计者都没有考虑供的接口。

PIO 一共有九条指令：`JMP`、`WAIT`、`IN`、`OUT`、`PUSH`、`PULL`、`MOV`、`IRQ` 和 `SET`。关于每条指令的详细信息请参见[3.4节](section3-4)。

尽管 PIO 只有九条指令，手工编写二进制 PIO 程序也非常困难。PIO 汇编是用文本形式描述的 PIO 程序，每条命令对应于二进制程序中的一条指令。
下面是一个 PIO 汇编程序的例子：

<caption>Pico 示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/squarewave.pio 行7-12 </caption>

```
 7 .program squarewave
 8     set pindirs, 1 ; Set pin to output
 9 again:
10     set pins, 1 [1] ; Drive pin high and then delay for one cycle
11     set pins, 0 ; Drive pin low
12     jmp again ; Set PC to label `again`
```

SDK 自带了 PIO 汇编器，名为 `pioasm`。该程序接受一个 PIO 汇编程序的文本文件，其中可以包含多个程序，并输出汇编后的程序。
这些汇编后的程序以 C 头文件的形式输出，头文件中包含了常量数组。更多信息请参考[3.3节](section3-3)。


### 3.2.2. 控制流

在每个时钟周期，每个状态机获取、解码并执行一条指令。每个指令精确地占用一个时钟周期，除非它显式地暂停执行（如 `WAIT` 指令）。
每条指令还可以带有最多 31 个周期的延时，推迟下一条指令的执行，用于编写精确时钟周期的程序。

程序计数器 `PC` 指向当前周期正在执行的指令的内存地址。一般而言，`PC` 每个周期加一，到达指令内存边界时自动返回开头。
跳转指令是一个例外，它显式提供了 `PC` 的下一个值。

上一节的示例汇编程序（开头为 `.program squarewave`）演示了这两个概念。它在一个 GPIO 引脚上产生占空比为 50% 的方波，每个周期占用四个时钟周期。
通过其他手段（如 side-set）可以将周期缩短至两个时钟周期。

**注释**：Side-set 可以让状态机在执行一条指令时，顺便设置少量 GPIO 的状态。详细描述请参见[3.5.1节](section3-5#351-side-set)。

系统对指令内存拥有只写的权限，用于加载程序：

<caption>Pico示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/squarewave.c 行34-38</caption>

```c
34     // Load the assembled program directly into the PIO's instruction memory.
35     // Each PIO instance has a 32-slot instruction memory, which all 4 state
36     // machines can see. The system has write-only access.
37     for (int i = 0; i < count_of(squarewave_program_instructions); ++i)
38         pio->instr_mem[i] = squarewave_program_instructions[i];
```

时钟分割器可以将状态机的执行速度降低一个常数因子，该因子用一个 16.8 的定点小时表示。在上述示例中，如果采用的时钟分割因子为 2.5，
那么方波的周期就是 4 x 2.5 = 10 个时钟周期。这在需要设置 UART 等串行接口的精确波特率时非常有用。


<caption>Pico示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/squarewave.c 行42-47</caption>

```c
42     // Configure state machine 0 to run at sysclk/2.5. The state machines can
43     // run as fast as one instruction per clock cycle, but we can scale their
44     // speed down uniformly to meet some precise frequency target, e.g. for a
45     // UART baud rate. This register has 16 integer divisor bits and 8
46     // fractional divisor bits.
47     pio->sm[0].clkdiv = (uint32_t) (2.5f * (1 << 16));
```

上述代码片段所在的整个程序可以在 GPIO 0 （或任何管脚）上产生一个 12.5MHz 的方波。我们还可以使用 `WAIT PIN` 指令根据管脚状态等待一定时间，
或使用 `JMP PIN` 指令根据管脚状态跳转，实现根据管脚状态改变控制流。

<caption>Pico示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/squarewave.c 行51-59

```c
51     // There are five pin mapping groups (out, in, set, side-set, jmp pin)
52     // which are used by different instructions or in different circumstances.
53     // Here we're just using SET instructions. Configure state machine 0 SETs
54     // to affect GPIO 0 only; then configure GPIO0 to be controlled by PIO0,
55     // as opposed to e.g. the processors.
56     pio->sm[0].pinctrl =
57         (1 << PIO_SM0_PINCTRL_SET_COUNT_LSB) |
58         (0 << PIO_SM0_PINCTRL_SET_BASE_LSB);
59     gpio_set_function(0, GPIO_FUNC_PIO0);

```

系统可以通过 CTRL 寄存器在任意时间启动或停止任意状态机。多个状态机可以同时启动，而 PIO 的确定性能保证它们之间的完美同步。

<caption>Pico示例：https://github.com/raspberrypi/pico-examples/tree/master/pio/squarewave/squarewave.c 行63-67</caption>

```c
63     // Set the state machine running. The PIO CTRL register is global within a
64     // PIO instance, so you can start/stop multiple state machines
65     // simultaneously. We're using the register's hardware atomic set alias to
66     // make one bit high without doing a read-modify-write on the register.
67     hw_set_bits(&pio->ctrl, 1 << (PIO_CTRL_SM_ENABLE_LSB + 0));
```

大多数指令都来自指令内存，但指令也可以来自其他地方，并且可以混合使用：

- 写入特殊配置寄存器（`SMx INSTR`）的指令会立即执行，中断其他指令的执行。例如，写入 `SMx INSTR` 的一条 `JMP` 指令会让状态机立即从另一个位置开始执行。
- 使用 `MOV EXEC` 指令，可以从寄存器执行指令。
- 使用 `OUT EXEC` 指令，可以从输出移位寄存器中执行指令。

后面几种方式非常灵活：指令可以嵌入到传递给 FIFO 的数据流中。I2C 的示例就在正常的数据中嵌入了 `STOP` 和 `RESTART` 等的条件。对于 `MOV` 和 `OUT EXEC` 来说，
`MOV` / `OUT` 本身需要一个时钟周期，然后再执行指定的指令。


### 3.2.3. 寄存器

每个状态机都拥有几个内部寄存器。这些寄存器用于保存输入或输出数据，以及循环变量等临时数据。

#### 3.2.3.1. 输出移位寄存器（OSR）

