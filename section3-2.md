## 3.2. PIO 程序


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
 8 set pindirs, 1 ; Set pin to output
 9 again:
10 set pins, 1 [1] ; Drive pin high and then delay for one cycle
11 set pins, 0 ; Drive pin low
12 jmp again ; Set PC to label `again`
```

SDK 自带了 PIO 汇编器，名为 `pioasm`。该程序接受一个 PIO 汇编程序的文本文件，其中可以包含多个程序，并输出汇编后的程序。
这些汇编后的程序以 C 头文件的形式输出，头文件中包含了常量数组。更多信息请参考[3.3节](section3-3)。


### 3.2.2. 控制流

在每个时钟周期，每个状态机获取、解码并执行一条指令。每个指令精确地占用一个时钟周期，除非它显式地暂停执行（如 `WAIT` 指令）。
每条指令还可以带有最多 31 个周期的延时，推迟下一条指令的执行，用于编写精确时钟周期的程序。

程序计数器 `PC` 指向当前周期正在执行的指令的内存地址。一般而言，`PC` 每个周期加一，到达指令内存边界时自动返回开头。
跳转指令是一个例外，它显式提供了 `PC` 的下一个值。

