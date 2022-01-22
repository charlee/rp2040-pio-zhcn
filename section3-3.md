## 3.3. PIO 汇编器（pioasm）


PIO 汇编器能够解析一段 PIO 源文件，并输出汇编后的代码，该代码可以包含到某个 RP2040 应用程序中，
可以是使用 SDK 构建的 C 或 C++ 程序，也可以是 RP2040 MicroPython 上运行的 Python 程序。

本节简要介绍 `pioasm` 中可以使用的指示（directives）和指令（instructions）。