---
title: "深入JTAG - 概述（第1部分/共6部分）"
date: 2026-04-26T21:45:53+08:00
author: ["FanYuxin", "Aliaksandr Kavalchuk"]
draft: false
---

作为JTAG三部分系列的第一部分，本文将对JTAG进行概述，为后续关于调试和JTAG边界扫描的更深入讨论做准备。我们将深入探讨接口的细节，如测试访问端口（TAP）、关键寄存器、指令以及JTAG的有限状态机。

## 目录

- [简介](#简介)
- [测试访问端口（TAP）](#测试访问端口-tap)
- [控制信号](#控制信号)
- [寄存器](#寄存器)
- [移位寄存器](#移位寄存器)
- [JTAG指令](#jtag指令)
  - [`IDCODE`指令](#idcode指令)
  - [`BYPASS`指令](#bypass指令)
  - [`SAMPLE/PRELOAD`指令](#samplepreload指令)
- [状态机](#状态机)
- [示例](#示例)
- [参考资料](#参考资料)

## 简介

JTAG（联合测试行动组）是基于IEEE 1149.1标准的专用硬件接口。该接口设计用于将复杂的芯片和设备连接到标准测试和调试硬件。

如今，JTAG主要用于：

- 微电路的输出控制
- 印刷电路板的测试
- 带内存的微芯片的烧录
- 芯片软件调试

标准中实现的测试方法称为边界扫描。这个名称反映了过程的思想：芯片内的功能块被隔离，特定的信号组合被应用到它们的输入。然后评估每个块输出的状态。整个过程通过JTAG接口的特殊命令执行，不需要物理干预。

## 测试访问端口（TAP）

测试访问端口（TAP）是JTAG协议的关键元素之一，设计用于控制和配置连接到JTAG链的芯片。

TAP作为一个简单的有限状态机运行，由`TMS`（测试模式选择）信号控制。它允许通过JTAG命令访问微控制器和其他设备的内部寄存器。

连接到JTAG链的每个设备都有自己的TAP，由`IR`（指令寄存器）和DR（数据寄存器）组成。`IR`寄存器用于选择要在设备上执行的指令，DR寄存器用于传输数据。

## 控制信号

测试访问端口包含四个强制信号（`TCK`、`TMS`、`TDI`、`TDO`）和一个可选信号（`TRST`）。

- `TDI`（测试数据输入）— 测试数据输入。命令和数据通过此引脚在`TCK`信号的上升沿插入芯片。
- `TDO`（测试数据输出）— 串行数据输出。命令和数据通过此引脚在`TCK`信号的下降沿从芯片输出。
- `TCK`（测试时钟）— 时钟输入。
- `TMS`（测试模式选择）— 控制TAP有限状态机的状态转换。
- `TRST`（测试复位）— TAP有限状态机的复位信号。

标准规定，JTAG模块在`TCK`线的上升沿从`TMS`和`TDI`线读取数据。任何芯片中的JTAG模块也必须在`TCK`的下降沿改变`TDO`线上的逻辑值。在下图中，JTAG模块读取数据的时刻用红色虚线表示，写入数据的时刻用绿色虚线表示。

![JTAG控制信号](https://interrupt.memfault.com/img/jtag-part1/jtag-control-signals.png)

## 寄存器

TAP状态机允许访问两个特殊寄存器：`IR`和一个称为DR的符号寄存器。

指令寄存器存储当前要执行的指令。该寄存器的值被TAP控制器用来决定如何处理输入信号。最常用的指令指定输入数据应该进入哪个数据寄存器。

数据寄存器是当前被`IR`内容选择的寄存器的占位符。因此，`IR`是多个寄存器的索引，DR是当前选择的寄存器。有三种主要类型的数据寄存器：

- BSR（边界扫描寄存器）— 测试的主寄存器。用于在芯片的引脚上传输数据。
- `BYPASS` — 一个单比特寄存器，将数据从`TDI`传输到`TDO`。它允许以最小延迟测试串联连接的其他芯片。
- `IDCODE` — 存储芯片的ID代码和修订号。

![JTAG调试TAP](https://interrupt.memfault.com/img/jtag-part1/debug-tap.gif)

在上图中，你可以看到DR寄存器工作原理的大致说明：开关SW3和SW4根据`IR`中的指令选择当前寄存器。

`IR`的大小是实现特定的，通常在4到32位之间。由于扫描DR时直接访问所选寄存器，因此DR的大小取决于当前指令。

JTAG寄存器是微控制器调试过程的重要组成部分，因为它们允许你在程序执行期间控制和监视微控制器的状态。每个微控制器制造商可能使用自己的JTAG寄存器，因此你应该查阅特定微控制器的文档，了解它支持的JTAG寄存器的详细信息。

## 移位寄存器

JTAG协议中的数据传输（读/写）是通过移位寄存器原理执行的。在移位寄存器中，数据按顺序逐位传输，每个时钟周期一位。

![JTAG调试TAP](https://interrupt.memfault.com/img/jtag-part1/shift-reg.gif)

该寄存器位于`TDI`和`TDO`引脚之间，用于从`TDI`引脚接收信息并向`TDO`引脚输出信息。每次你想通过JTAG协议向TAP写入内容时，你将必要的信号设置到`TDI`引脚 - 这些信号从最高位开始同步写入移位寄存器，并随着每个新时钟逐渐移向寄存器的最低位，而移位寄存器的最低位的值随着每个时钟移到`TDO`引脚，我们可以从那里读取它。

> **注意**：我没有在任何地方找到关于这方面的详细信息，但从一般理解来看，我可以假设对于每个可通过JTAG协议访问的寄存器，都有一个单独版本的移位寄存器。例如，假设我们有一个TAP，其`IR`寄存器长度为4位，`BYPASS`寄存器长度为1位，`IDCODE`寄存器长度为32位，那么对于这些寄存器中的每一个，都会有一个单独版本的移位寄存器，每当选择相应的`IR`、`BYPASS`或`IDCODE`寄存器时，它都会被集成到`TDI`和`TDO`引脚之间的扫描链中。但这只是我的假设，尚未记录。

## JTAG指令

JTAG指令是与TAP交互的命令，启用测试、调试、编程和配置功能。

如前一章所述，选择指令通常不会直接触发任何动作，只是选择适当的寄存器作为DR。

让我们看看一些最常见的指令。

### `IDCODE`指令

JTAG中的`IDCODE`指令用于获取连接到JTAG电路的设备的唯一标识符。每个支持JTAG的设备都有自己唯一的ID代码，可以使用`IDCODE`命令读取。这对于识别设备类型、制造商和版本很有用。

此标识符大小为32位，由以下字段组成：

![IDCODE](https://interrupt.memfault.com/img/jtag-part1/idcode-instruction-reg.png)

因此，当你在`IR`寄存器中加载`IDCODE`指令时，这将强制选择`IDCODE`寄存器作为数据寄存器。

### `BYPASS`指令

JTAG协议中的`BYPASS`指令允许你绕过JTAG链中的一个或多个组件，而不将它们包含在扫描链中。当设备不支持JTAG协议命令或你要检查链中的其他组件时，这很有用。

当`BYPASS`指令传递到JTAG链时，它会跳过它所针对的设备，并将控制权传递给链中的下一个设备。因此，`BYPASS`命令避免寻址无法被JTAG协议扫描的设备，并继续扫描链中更上游的设备。

此外，`BYPASS`指令可用于加速JTAG链扫描，因为跳过设备减少了通过链所需的周期数。

因此，当你在`IR`寄存器中加载`BYPASS`指令时，这将强制选择1位`BYPASS`寄存器作为数据寄存器。

### `SAMPLE/PRELOAD`指令

> **注意**：此命令和其他几个命令：`EXTEST`、`INTEST`、`HIGHZ`在板测试过程中被积极使用，将在本文章的第二部分详细讨论，主题是使用JTAG协议进行板测试。在同一篇文章中，它们通过引用被指定。

此命令将`TDI`和`TDO`连接到BSR（边界扫描寄存器）。然而，芯片保持正常工作状态。在执行此命令期间，BSR寄存器可用于捕获芯片在正常操作期间交换的数据。换句话说，通过此命令，我们可以从微控制器的引脚上读取信号，而不会干扰其操作。

因此，当你在`IR`寄存器中加载`SAMPLE/PRELOAD`指令时，这将强制选择BSR寄存器作为数据寄存器。

## 状态机

JTAG协议的有限状态自动机由一组TAP可以假设的状态组成，取决于其输入接收的信号。每个状态对应于`TMS`和`TDI`输入的特定信号值组合。

状态之间的转换取决于`TCK`电平上升时刻的`TMS`信号。

复位后的初始状态是Test Logic-Reset。根据标准，对于所有移位寄存器，LSB首先被推入和拉出。

状态机相当简单，有两种工作方式：

- 指令寄存器选择（蓝色块）用于选择当前命令。
- 数据寄存器选择（绿色块）用于在数据寄存器中读/写数据。

![JTAG调试TAP](https://interrupt.memfault.com/img/jtag-part1/jtag-state-machine.png)

所有状态都有两个输出，转换的安排使得任何状态都可以通过单个`TMS`信号（由`TCK`同步）控制分配器来达到。有两个不同的状态序列：一个用于读取或写入数据寄存器，一个用于处理指令寄存器。

让我们描述最重要的状态。但由于`IR`路径和DR路径具有相同的状态，我将同时描述这两个路径的这些状态，并在必要时指定差异。

- **Test-Logic-Reset** — 所有测试逻辑被禁用，芯片正常工作。
- **Run-Test/Idle** — 初始化测试逻辑的第一个状态和默认空闲状态
- **Select-DR/IR-Scan** — 此状态对于选择当前路径（数据或指令）是必要的。我认为这可以可视化为开关的操作：SW1、SW2、SW3、SW4。当命中Select-DR-Scan状态时，开关SW1、SW2、SW3、SW4切换到相应的DR寄存器。当达到Select-IR-Scan状态时 - 开关SW1、SW2切换到`IR`寄存器。

![选择DR和IR切换](https://interrupt.memfault.com/img/jtag-part1/debug-tap-2.gif)

- **Capture-DR** — 在这个状态中，如果遵循Select-DR-Scan状态分支，所选DR寄存器中存储的值会并行加载到移位寄存器中；如果遵循Select-IR-Scan状态路径，则加载特殊模式，通常选择0x01作为模式。

![Capture-DR状态期间的DR寄存器](https://interrupt.memfault.com/img/jtag-part1/capture-dr-state-dr.gif)

- **Shift-DR** — 寄存器将数据从`TDI`向前移动一位到`TDO`。Shift-DR和Shift-IR状态是将数据串行加载到数据寄存器或指令寄存器的主要状态。

![Shift-DR状态期间的DR寄存器](https://interrupt.memfault.com/img/jtag-part1/shift-dr-state.gif)

- **Update-DR** — 移位寄存器中的数据被写入芯片中相应寄存器的状态。Update-DR和Update-IR状态将数据锁存到寄存器中，将指令寄存器中的数据设置为当前指令

![Update-DR状态期间的DR寄存器](https://interrupt.memfault.com/img/jtag-part1/update-dr-state-ir.gif)

- **Pause-DR/IR** — 暂时停止从`TDI`到`TDO`的数据移位

状态机在测试时钟（`TCK`）边缘进行，测试模式选择（`TMS`）引脚的值控制行为。

> **注意**：无论TAP控制器的初始状态如何，始终可以通过将TMS逻辑1保持5个TCK时钟周期来进入Test-Logic-Reset状态。

## 示例

现在我们已经涵盖了理论，是时候看看JTAG协议的实际应用了。让我们考虑一个从芯片读取ID代码值的例子，`IR`长度为4位。引脚`TMS`、`TDI`、`TDO`上的位序列、状态机转换以及开关SW1-SW4的状态在下图中显示：

![JTAG状态转换示例](https://interrupt.memfault.com/img/jtag-part1/jtag-example.gif)

最初，我们处于`Run-Test/Idle`状态。为了读取芯片ID代码，我们需要将指令代码`IDCODE`写入`IR`（在我们的例子中是`0b1110`）。要将指令写入`IR`，我们需要选择状态机的蓝色分支。图像`2`和`3`显示了这种转换。图像`3`显示了当进入`Select-IR-Scan`状态时，键`SW1`和`SW2`如何切换。接下来，在步骤`4`的`Capture-IR`状态中，`0b0001`模式被加载到移位寄存器中。在步骤`5`中，转换到`Shift-IR`状态，在这个转换中，加载模式的位`1`被推进到`TDO`引脚。

步骤`6-7`显示了`IDCODE (0b1110)`指令逐位顺序移入移位寄存器，最后一位在转换到`Exit1-IR`状态（步骤`8`）时被移入。在步骤`9`（`Update-IR`状态），写入移位寄存器的指令代码被锁存到`IR`寄存器中。在`10`，我们返回初始状态。我们已经写入了指令代码，现在需要读取与此指令对应的数据，为此我们将使用自动机的绿色分支。在步骤`11`，我们进入`Select-DR-Scan`状态，此时键`SW1`和`SW2`切换到`DR`寄存器，并且`ID`寄存器被选择，因为在`IR`阶段我们选择了`IDCODE`指令。在步骤`12`的`Capture-DR`状态，32位`ID`代码被加载到移位寄存器中。在步骤`13`，执行到`Shift-DR`状态的转换，在这个转换中，`ID`代码的低位被推进到`TDO`输出。步骤`14-20`显示了芯片ID代码`(0b111001101)`的逐位顺序移位。在步骤`21`，转换到`Exit1-DR`状态，代码`ID`的最后一位被提升。步骤`22`（`Update-DR`状态）- 应该有一个将写入移位寄存器的代码锁存到所选`DR`寄存器的操作，但在`IDCODE`命令的情况下，这不会发生。在步骤`23`，我们再次返回初始状态。

在本系列的第2部分中，我们将深入探讨JTAG调试。

> **注意**：本文最初由Aliaksandr发表在他的博客上。你可以在[这里](https://medium.com/@aliaksandr.kavalchuk/diving-into-jtag-protocol-part-1-overview-fbdc428d3a16)找到原文。

## 参考资料

- [JTAG和测试访问端口（TAP）简介](https://www.allaboutcircuits.com/technical-articles/introduction-to-jtag-test-access-port-tap/)
- [JTAG测试访问端口（TAP）状态机](https://www.allaboutcircuits.com/technical-articles/jtag-test-access-port-tap-state-machine/)
- [JTAG.FPGA4Fun](https://www.fpga4fun.com/JTAG.html)
- [EEVblog — 什么是JTAG和边界扫描？](https://youtu.be/TlWlLeC5BUs)
- [将JTAG边界扫描带入2021年](https://circuitcellar.com/research-design-hub/design-solutions/bringing-jtag-boundary-scan-into-2021/)
- [Arm核心设备中的JTAG实现](https://www.allaboutcircuits.com/technical-articles/jtag-implementation-arm-core-devices/)
- [Jworker — 它如何工作](http://www.seabrooks.plus.com/jworker/internals.pdf)
- [使用JTAG进行调试](https://www.actuatedrobots.com/debugging-with-jtag/)
- [Intel JTAG原语 — 使用JTAG而无需虚拟JTAG](https://tomverbeure.github.io/2021/10/30/Intel-JTAG-Primitive.html)
- [通过JTAG编程Spartan-6 FPGA](https://www.cyrozap.com/2015/05/31/programming-a-spartan-6-fpga-via-jtag/)
- [边界扫描/JTAG](https://semiconshorts.com/2023/02/11/boundary-scan-jtag/)
- [黑盒JTAG逆向工程](https://fahrplan.events.ccc.de/congress/2009/Fahrplan/attachments/1435_JTAG.pdf)
- [黑盒JTAG逆向工程 — 视频](https://media.ccc.de/v/26c3-3670-en-blackbox_jtag_reverse_engineering#t=238)
- [嵌入式分析的贫民窟工具 — Nathan Fain — REcon 2011](https://www.youtube.com/watch?v=ZmBfahwV3ss)

Aliaksandr Kavalchuk是一位拥有10多年经验的固件工程师。