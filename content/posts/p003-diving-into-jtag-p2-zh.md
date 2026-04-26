---
title: "深入JTAG - 调试（第2部分/共6部分）"
date: 2026-04-26T22:21:53+08:00
author: ["FanYuxin", "Aliaksandr Kavalchuk"]
draft: false
---
正如我在上一篇文章[深入JTAG协议。第1部分 - 概述](https://interrupt.memfault.com/blog/diving-into-jtag-part1)中所述，JTAG最初是为测试集成电路和印刷电路板而开发的。然而，人们逐渐认识到它在调试方面的潜力，现在JTAG已成为微控制器调试的标准协议。许多固件和嵌入式工程师首次接触它就是在这种背景下。

在JTAG深度探讨系列的第二部分中，我们将深入了解如何与微控制器的内存交互，以及如何与处理器核心和调试寄存器交互。虽然JTAG在测试中的使用相当标准化，但在调试方面，每个处理器架构都有其独特的细微差别。考虑到这一点，本文将重点介绍在ARM Cortex-M架构上使用JTAG进行调试，具体来说是使用STM32F407VG微控制器。

要成功进行调试，仅仅了解JTAG的基础知识是不够的。理解ARM调试访问端口的工作原理也至关重要，它是允许访问控制器内部寄存器和内存的关键组件。我在文章[深入ARM调试访问端口](https://piolabs.com/blog/engineering/diving-into-arm-debug-access-port.html)中详细介绍了这个模块。有了这些基础，让我们深入JTAG调试的世界。

## 目录

- [通过JTAG访问STM32F407VG控制器](#通过jtag访问stm32f407vg控制器)
- [读取ID CODE](#读取id-code)
- [内存交互](#内存交互)
- [向内存写入变量](#向内存写入变量)
- [从内存读取变量](#从内存读取变量)
- [与处理器核心交互](#与处理器核心交互)
- [结语](#结语)

## 通过JTAG访问STM32F407VG控制器

在通过JTAG协议连接到STM32F407VG控制器之前，我们需要确定以下信息：`IR`寄存器的长度和连接到JTAG链的TAP控制器数量。官方文档，特别是[RM0090：参考手册](https://www.st.com/resource/en/reference_manual/rm0090-stm32f405415-stm32f407417-stm32f427437-and-stm32f429439-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)，可以在这方面提供帮助。

根据该文档，STM32F407VG有两个TAP控制器：STM32边界扫描和CoreSight JTAG-DP。链中的第一个是边界扫描TAP，`IR`寄存器大小为5位。紧随其后的是CoreSight JTAG-DP TAP，`IR`寄存器大小为4位。

![](https://interrupt.memfault.com/img/tap-controllers.png)

这些知识对我们至关重要，因为它决定了在访问不同TAP控制器时要在JTAG链中发送多少位以及以什么顺序发送。根据这些信息，我们可以连接到CoreSight JTAG-DP TAP控制器并尝试读取其`IDCODE`。

## 读取ID CODE

读取CoreSight JTAG-DP TAP控制器的`IDCODE`的位序列如下：

![](https://interrupt.memfault.com/img/jtag-part2/idcode-bits.png)

让我们详细分析这个序列：

- 第一步是将TAP状态机复位到初始状态`Test-Logic-Reset`，以确保我们处于已知状态。要实现这一点，只需将`TMS`保持在逻辑1状态5个`TCK`时钟周期。粉色位序列负责此步骤：

![](https://interrupt.memfault.com/img/jtag-part2/test-logic-reset-state.png)

- 接下来，我们需要将`IDCODE`命令代码加载到`IR`寄存器中，为此我们要将TAP状态机置于`Shift-IR`状态。要做到这一点，在`TMS`线上输入序列\[0, 1, 1, 0, 0\]。蓝色位序列负责此步骤：

![](https://interrupt.memfault.com/img/jtag-part2/shift-ir-state.png)

正如你可以在最后一小节的`TDO`线上看到有一个1。让我们找出它从哪里来的。事实上，在通过`Capture-IR`状态时，TAP控制器将`0b0001`的值加载到CoreSight JTAG-DP TAP的移位寄存器中，而边界扫描TAP是`0b00001`，它开始从这个测量开始沿`TDO`线移动，LSB位在前。因此，在接下来的9个时钟周期（包括这一个）中，我们将得到`TDO`序列`0b100010000`。

- 下一步是直接将`IDCODE`命令代码加载到CoreSight JTAG-DP TAP的`IR`寄存器中，并将`BYPASS`命令代码加载到边界扫描TAP的`IR`寄存器中。我认为CoreSight JTAG-DP TAP的情况很清楚，但应该解释一下为什么我们要将`BYPASS`命令加载到边界扫描TAP。你记得我们的JTAG链中有两个TAP控制器串联连接，如果不向边界扫描TAP写入某些东西，我们就无法访问CoreSight JTAG-DP TAP寄存器，而边界扫描的`IR`寄存器大小为5位，`DR`寄存器为32位，所以要向CoreSight JTAG-DP TAP的`DR`寄存器写入新数据，我们需要向JTAG电路写入32 + 32 = 64位序列。这不太方便。`BYPASS`命令在这里可以帮助我们。当这个命令在`IR`寄存器中时，所选TAP的数据寄存器变成1位长。所以现在我们需要写入33位序列而不是64位。`IDCODE`代码是`0b1110`，`BYPASS`命令代码是`0b11111`，总共需要推送9位：`0b011111111`。在输入8个LSB位期间，`TMS`输入必须保持在逻辑0，以便机器保持在`Shift-IR`状态。指令的MSB位在通过将`TMS`设置为逻辑1退出此状态时被推送。红色位序列负责此步骤：

![](https://interrupt.memfault.com/img/jtag-part2/capture-ir-state.png)

`TDO`线继续输出在`Capture-IR`状态加载的序列`0b100010000`。

此步骤后，我们处于`Exit1-IR`状态。

- 很好，必要的命令已在`IR`寄存器中设置，现在我们需要返回`Run Test/Idle`状态以便进一步工作。此时，由于`TMS`线上的最后一个1，我们处于`Exit-1-IR状态`，因此根据状态图，我们需要在`TMS`线上设置以下位序列到`Run Test/Idle`：`[1, 0]`。橙色位序列负责此步骤：

![](https://interrupt.memfault.com/img/jtag-part2/exit-1-ir-state.png)

- 在`IR`寄存器中设置`IDCODE`命令后，我们需要从`DR`寄存器读取`IDCODE`。为此，第一步是进入`Shift-DR`状态。要做到这一点，假设我们此刻处于`Run Test/Idle`状态，我们需要在`TMS`引脚上设置以下位序列`[1, 0, 0]`。黄色位序列负责此步骤：

![](https://interrupt.memfault.com/img/jtag-part2/shift-dr-state.png)

在`TDO`线上我们有序列：`[0, 0, 1]`。最后一个已经是我们芯片ID的LSB。事实上，在通过`Capture-DR`状态时，我们的芯片ID被加载到移位寄存器中，并在下一个时钟周期开始向外移动。

- 在`Shift-DR`状态下，芯片ID的其余位被卸载。要保持在`Shift-DR`状态，除了MSB位外，在所有位的输入期间`TMS`输入必须保持在逻辑0。当通过将`TMS`设置为逻辑1退出此状态时，MSB数据被推出。绿色位序列负责此步骤：

![](https://interrupt.memfault.com/img/jtag-part2/shift-dr-state-2.png)

这样我们得到了如下类型的芯片ID：

> 11101110001000000000010111010010

让我们根据文档检查这个代码：

![](https://interrupt.memfault.com/img/jtag-part2/jtag-dp-reg.png)

这里有问题。问题在于LSB在图中处于通常位置，但我们获取时是倒序的。让我们纠正这个情况：

> 01001011101000000000010001110111

- 最后一步是返回`Run Test/Idle`状态：

![](https://interrupt.memfault.com/img/jtag-part2/run-test-idle-state.png)

## 内存交互

很好，JTAG连接正常工作，我们已经读取了`IDCODE`。现在让我们继续尝试通过向内存写入值然后读取回来与微控制器的内存交互。毕竟，在调试微控制器时，读/写内存是主要操作之一。

要与微控制器的内存交互，我们需要了解其背后的机制。JTAG协议本身没有提供这方面的具体细节。对ARM架构控制器内部资源（如内存）的访问是由一个称为DAP（调试访问端口）的特殊模块促进的。我在文章[深入ARM调试访问端口](https://piolabs.com/blog/engineering/diving-into-arm-debug-access-port.html)中详细介绍了这个模块。不幸的是，没有这些知识，进一步前进可能会有挑战。因此，我强烈建议阅读那篇文章或官方文档[IHI0031 — ARM调试接口](https://developer.arm.com/documentation/ihi0031/latest/)。

对于通过DAP模块访问具有ARM架构微控制器的内部资源的任何访问操作，第一步是请求调试模块的时钟和电源。这通过在DP `CTRL/STAT`寄存器中设置`CSYSPWRUPREQ`和`CDBGPWRUPREQ`位来完成。

![](https://interrupt.memfault.com/img/jtag-part2/step-1-access-internals.png)

完成此初始设置后，我们可以开始内存操作。

在以下两个章节中，我将大量参考文章[深入ARM调试访问端口](https://piolabs.com/blog/engineering/diving-into-arm-debug-access-port.html)中的"实践部分"。本节描述了通过与DAP模块交互读/写内存的算法。此外，我将用位序列补充这个算法，说明从JTAG协议的角度来看这将是什么样子。

## 向内存写入变量

好的，让我们将值`0xAA55AA55`写入地址`0x20000000`：

首先，将`0b1010`（`DPACC`寄存器代码）写入`IR`寄存器。`AP`寄存器`CSW`中的一些设置需要配置。要访问此寄存器，我们需要在`DP SELECT`寄存器中选择相应的AP和寄存器银行。写入`DR`寄存器：

- DATA\[31:0\] = `0x00`
- APSEL\[31:24\] = `0x00`
- APBANKSEL\[7:4\] = `0x00`
- A\[3:2\] = `0b10`（`SELECT`寄存器的地址）
- RnW = `0b0`

![](https://interrupt.memfault.com/img/jtag-part2/step-2-access-internals.png)

接下来，我们需要写入`CSW`寄存器。由于这是`AP`寄存器，我们需要使用`APACC`寄存器来访问它：

- 将`0b1011`（`APACC`寄存器代码）写入`IR`寄存器。

创建配置数据：将写入的数据大小设置为32位（`Size[0:2] = 0b010`），禁用地址自动递增功能（`AddrInc[5:4] = 0b00`）。写入`DR`：

- DATA\[31:0\] = `0x00000002`
- A\[3:2\] = `0b00`（CSW寄存器的地址）
- RnW = `0b0`

![](https://interrupt.memfault.com/img/jtag-part2/step-3-access-internals.png)

然后我们需要设置要写入数据的内存单元的地址。这通过`AP`寄存器`TAR`完成。由于此寄存器属于与`CSW`相同的`AP`且在同一个银行，我们可以省略对`DP SELECT`寄存器的引用，立即写入地址值。写入DR：

- DATA\[31:0\] = `0x20000000`
- A\[3:2\] = `0b01`（`TAR`寄存器的地址）
- RnW = `0b0`

![](https://interrupt.memfault.com/img/jtag-part2/step-4-access-internals.png)

最后一步是实际写入数据。为此，我们需要将数据写入DRW寄存器。写入DR：

- DATA\[31:0\] = `0xAA55AA55`
- A\[3:2\] = `0b11`（`DRW`寄存器的地址）
- RnW = `0b0`

![](https://interrupt.memfault.com/img/jtag-part2/step-5-access-internals.png)

## 从内存读取变量

让我们尝试从地址`0x20000000`的内存单元读取值。

首先，将`0b1010`（`DPACC`寄存器代码）写入`IR`寄存器。接下来，我们需要在`AP`的`CSW`寄存器中配置一些设置。要访问此寄存器，我们需要在`DP SELECT`寄存器中选择相应的`AP`和寄存器银行。使用表中的数据，我们形成35位数据并将其写入`DR`寄存器：

- DATA\[31:0\] = `0x00`
- APSEL\[31:24\] = `0x00`
- APBANKSEL\[7:4\] = `0x00`
- A\[3:2\] = `0b10`（`SELECT`寄存器的地址）
- RnW = `0b0`

![](https://interrupt.memfault.com/img/jtag-part2/step-6-access-internals.png)

接下来，我们需要直接写入`CSW`寄存器。由于这是`AP`寄存器，我们需要使用`APACC`寄存器来访问它：

- 将`0b1011`（`APACC`寄存器代码）写入`IR`寄存器。

我们创建配置数据：将写入的数据大小设置为32位（`Size[0:2] = 0b010`），禁用地址自动递增功能（`AddrInc[5:4] = 0b00`）。写入`DR`：

- DATA\[31:0\] = `0x00000002`
- A\[3:2\] = `0b00`（`CSW`寄存器的地址）
- RnW = `0b0`

![](https://interrupt.memfault.com/img/jtag-part2/step-7-access-internals.png)

接下来，我们需要设置要从中读取数据的内存单元的地址。这通过`AP`的`TAR`寄存器完成。由于此寄存器属于与`CSW`相同的`AP`且在同一个银行，我们可以跳过`DP SELECT`寄存器访问，立即写入地址值：

- DATA\[31:0\] = `0x20000000`
- A\[3:2\] = `0b01`（`TAR`寄存器的地址）
- RnW = `0b0`

![](https://interrupt.memfault.com/img/jtag-part2/step-8-access-internals.png)

我们形成读取请求。为此，我们写入`DRW`寄存器：

- DATA\[31:0\] = `0x00`
- A\[3:2\] = `0b11`（`DRW`寄存器的地址）
- RnW = `0b1`

![](https://interrupt.memfault.com/img/jtag-part2/step-9-access-internals.png)

通过从`DR`寄存器读取35位来读取`DRW`寄存器。

![](https://interrupt.memfault.com/img/jtag-part2/read-drw-register.png)

## 与处理器核心交互

我们已经探讨了调试器如何与微控制器的内存交互。现在，让我们深入了解调试器如何暂停程序执行、单步执行代码、读取处理器寄存器等。对于STM32F407VG微控制器，这是通过特殊寄存器实现的：

![](https://interrupt.memfault.com/img/jtag-part2/register-descriptions.png)

正如[ARMv7-M架构参考手册](https://developer.arm.com/documentation/ddi0403/latest/)的C1.6调试系统寄存器部分详述的，这些寄存器在调试中起着关键作用。

我们现在需要理解的关键点是，通过设置`DHCSR`寄存器的`C_DEBUGEN`和`C_HALT`位，我们可以暂停程序的执行。同样，通过设置`C_DEBUGEN`和`C_STEP`位，我们可以逐步执行程序。

通过DAP模块访问这些寄存器本质上与访问内存相同。不是提供内存地址，而是提供所需寄存器的地址。例如，对于`DHCSR`寄存器，地址是`0xE000EDF0`。如果你允许的话，我不会深入讨论位序列的示例，因为它与内存交互章节中的示例相同。

## 结语

我们已经简要探讨了JTAG在微控制器调试中的使用。这个主题很大，单篇文章很难涵盖。在本文中，我们专注于侵入式调试，而非侵入式调试甚至没有提到。然而，撰写本文时这不是目标。对于想要更深入了解这个主题的人，我建议学习下面列出的链接列表。至于我们，我们将继续前进，在下一篇文章中，我们将讨论使用JTAG协议测试印刷电路板的主题。

感谢你的时间！:)

> **注意**：本文最初由Aliaksandr发表在他的博客上。你可以在[这里](https://medium.com/@aliaksandr.kavalchuk/diving-into-jtag-protocol-part-2-debugging-56a566db3cf8)找到原文。

## 参考链接

- [RM0090：参考手册](https://www.st.com/resource/en/reference_manual/rm0090-stm32f405415-stm32f407417-stm32f427437-and-stm32f429439-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)
- [ARMv7-M架构参考手册](https://developer.arm.com/documentation/ddi0403/latest/)
- [Cortex-M4技术参考手册](https://medium.com/@aliaksandr.kavalchuk/diving-into-jtag-protocol-part-2-debugging-56a566db3cf8#:~:text=Cortex%2DM4.Technical%20Reference%20Manual)
- [ARM CoreSight架构规范](https://documentation-service.arm.com/static/5f900a19f86e16515cdc041e?token=)
- [使用JTAG进行调试](https://www.actuatedrobots.com/debugging-with-jtag/)
- [Cortex-M上无调试器的单步调试](https://interrupt.memfault.com/blog/cortex-m-debug-monitor)
- [断点是如何工作的？](https://interrupt.memfault.com/blog/cortex-m-breakpoints)
- [深入ARM Cortex-M调试接口](https://interrupt.memfault.com/blog/a-deep-dive-into-arm-cortex-m-debug-interfaces?utm_source=platformio&utm_medium=piohome)
- [在Cortex-M上分析固件](https://interrupt.memfault.com/blog/profiling-firmware-on-cortex-m)

Aliaksandr Kavalchuk是一位拥有10多年经验的固件工程师。