## 3.3. PIO 汇编器（pioasm）


PIO 汇编器能够解析一段 PIO 源文件，并输出汇编后的代码，该代码可以包含到某个 RP2040 应用程序中，
可以是使用 SDK 构建的 C 或 C++ 程序，也可以是 RP2040 MicroPython 上运行的 Python 程序。

本节简要介绍 `pioasm` 中可以使用的伪指令（directives）和指令（instructions）。有关怎样使用 `pioasm`、
怎样将其集成到 SDK 构建系统中、怎样扩展其特性以及它能生成何种输出格式等的深入讨论，请参考 [Raspberry Pi Pico C/C++ SDK](https://datasheets.raspberrypi.org/pico/raspberry-pi-pico-c-sdk.pdf) 一书。


### 3.3.1. 伪指令

PIO 程序中的汇编语言可以使用以下伪指令：

`.define ( PUBLIC ) <symbol> <value>`

定义一个名为 `<symbol>` 的整数符号，其值为 `<value>` （参见[3.3.2节](section3-3#TODO)）。
如果 `.define` 出现在输入文件中的第一个程序之前，那么定义就是全局的，对所有程序生效；否则，
定义就是局部的，仅对其所在的程序生效。如果指定了 `PUBLIC`，那么符号将输出到输出的汇编中，可以由用户代码使用。
对于 SDK 而言，程序符号的定义形式为 `#define <program_name>_<symbol> value`，而全局符号的定义形式为
`#define <symbol> value`。

`.program <name>`

开始一个名为 `<name>` 的新程序。注意名称会在代码中使用，所以应当由字母、数字或下划线组成，并且不以数字开始。
程序直到出现下一个 `.program` 伪指令或源文件末尾时结束。PIO 指令只能在程序内使用。

`.origin <offset>`

可选的伪指令，指示 PIO 指令内存的偏移量，程序*必须*加载到此偏移量处。绝大多数情况下，该伪指令用于指定程序必须加载到偏移量 0，
这样这些程序才能使用基于数据的 JMP 指令，并且只用几个比特来存储跳转的（绝对）地址。在程序外部，该伪指令无效。

`.side_set <count> (opt) (pindirs)`

如果出现该伪指令，则 `<count>` 指示使用的 side-set 比特数。还可以通过 `opt` 以指定 `side <value>` 对于指令是可选的
（注意这样做会使 side-set 在原本需要占用的 `<count>` 个比特之外，再额外占用一个比特。这些被占用的比特均来自指令的延时比特）。
最后，`pindirs` 可以用来指示 side-set 值应当应用于 PINDIR，而不是 PIN 上。该伪指令只能出现在程序的第一条指令之前。


`.wrap_target`

该伪指令置于某条指令之前，用于指示在程序折返时应当从哪一条指令继续执行。该伪指令只能在程序内部使用，且每个程序只能使用一次。
如果不指定，折返目的地为程序的开头。

`.wrap`

该伪指令置于某条指令之后，指示在正常的控制流中（即 `jmp` 条件为假，或没有 `jmp` 的情况）执行完该指令后，程序应当折返（至 `.warp_target` 指示的指令）。
该伪指令只能在程序内部使用，且每个程序只能使用一次。如果不指定，则折返点为程序的最后一条指令之后。

`.lang_opt <lang> <name> <option>`

为程序指定与特定语言生成器有关的选项（参见[SDK文档](https://datasheets.raspberrypi.org/pico/raspberry-pi-pico-c-sdk.pdf)中的"Language generators"一节）。
该伪指令只能在程序内使用。


`.word <value>`

将一个 16 位值作为一条指令存储在程序中。该伪指令只能在程序内部使用。


### 3.2.2. 值


下述值类型可以用于定义整数，或分支目的地。

| `integer` | 一个整数值，如 `3` 或 `-7` |
| `hex`     | 一个十六进制值，如 `0xf` |
| `binary`  | 一个二进制值，如 `0b1001` |
| `symbol`  | 由 `.define` 定义的一个值（参见[3.3.1节](#331-伪指令)） |
| `<label>` | 程序中的标签所对应的指令偏移量。通常在 JMP 指令中使用（参见[3.4.2节](#TODO) |
| `( <expression> )` | 一个可求值的表达式；参见[3.3.3节](#333-表达式)。注意括号是必须的。 |


### 3.3.3. 表达式

表达式可以与 pioasm 值一同使用。

| `<expression> + <expression>` | 两个表达式的和 |
| `<expression> - <expression>` | 两个表达式的差 |
| `<expression> * <expression>` | 两个表达式的积 |
| `<expression> / <expression>` | 两个表达式的整数除法商 |
| `- <expression>` | 表达式的负值 |
| `:: <expression>` | 表达式的按位取反 |
| `<value>` | 任意的值（参见[3.3.2节](#332-值) |



### 3.3.4. 注释

行注释以 `//` 或 `;` 开头。

C 语言风格的注释放在 `/*` 和 `*/` 之间。


### 3.3.5. 标签

标签的形式如下：

`<symbol>:`

或

`PUBLIC <symbol>:`

标签必须从行首开始。

**提示**：标签实际上只是自动的 `.define`，其值设置为当前程序指令的偏移量。`PUBLIC` 的标签可以通过与 `PUBLIC` 的 `.define` 同样的方式从用户代码访问。



### 3.3.6. 指令

所有的 pioasm 指令都遵循以下格式：

`<instruction> (side <side_set_value>) ([<delay_value])`

其中：

`<instruction>` 是下一节介绍的汇编指令。（参见[3.4节](section3-4)）

`<side_set_valie>` 是