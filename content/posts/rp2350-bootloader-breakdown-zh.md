---
title: "RP2350 UART启动指南"
date: 2025-05-03
author: "Thomas Pfister"
draft: false
---

# 通过UART启动RP2350

## 2025年5月3日

RP2350是树莓派推出的一款微控制器。最近我有个新项目，需要大量PWM通道。RP2350B有24个PWM通道，确实不少——但还是不够用。我至少需要33个PWM通道，再加上一些额外的GPIO，单靠一个RP2350根本不够。于是我想到了用端口扩展器...用第二个RP2350做端口扩展器如何？

这个芯片价格便宜（大概0.90美元），外围电路简单，但固件是个问题。同一个板子上跑多个控制器，固件管理会让人头疼。我很不喜欢固件版本兼容性这类琐事，所以想找个简单的端口扩展器方案。

翻了一下RP2350的[数据手册](https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf)（第5.8章），发现这芯片不仅有USB启动加载器——多数人应该都用这个给芯片烧程序——还有个UART启动加载器（RP2350对比RP2040的新特性！）。在微控制器上实现UART比USB简单多了，而且芯片间通信用UART也很合适。

_我录了个[YouTube视频](https://youtu.be/eno0hiFSr18)讲这个话题，感兴趣的话可以看看！_

## 原理

通过UART向芯片烧录固件的流程相当简单：先用魔术码做"解锁"，然后把固件镜像切成32字节一块发送过去，传完执行就行了。

附上RP2350的启动加载器源码：[戳这里](https://github.com/raspberrypi/pico-bootrom-rp2350/blob/master/src/nsboot/nsboot_uart_client.S)

当然也有缺点：

- 固件必须在SRAM里跑，会占SRAM空间。不过RP2350有520 KiB，做端口扩展器应该够了吧？（确实够！）
- 每次启动都要花点时间。实测通过UART传镜像不到一秒就搞定（取决于文件大小）。我的应用场景没问题，但其他场景可能不行。

网上搜了一圈，发现没多少人折腾过这个，那就开干吧！

启用UART启动加载器的硬件改动很简洁：Flash接口的CSn要拉低（顺便也启用了USB启动加载器），同时把SD1拉高。

芯片这样配置启动后，会在SD2（TX）和SD3（RX）引脚上启用1MBaud的UART。

可惜手头没有Pico 2，只有自己画的板子（后续会详细分享），做这些改动很容易，用的是大封装的Flash：

![](https://pfister.dev/blog/2025/P1001178.jpg)
（两个跳线帽分别接CSn和SD1，针座是TX和RX）

我们的目标：

- 让RP2350二进制文件在SRAM运行
- 通过UART把二进制文件传进板子
- 把它嵌到另一个MCU的固件里
- 从那边启动
- 考虑长线缆下的可靠连接

## SRAM运行的二进制文件

这个其实不难。在makefile里加一行 `set(PICO_NO_FLASH 1)` 就行（按照[SDK文档](https://datasheets.raspberrypi.com/pico/raspberry-pi-pico-c-sdk.pdf)第6.4章的说法，`set(PICO_DEFAULT_BINARY_TYPE no_flash)` 应该等价，但[目前不生效](https://github.com/raspberrypi/pico-sdk/issues/2279)。）

就这些。编译完会在"build"子目录生成bin文件：

![Bildschirmfoto vom 2025-02-16 19-06-44.png](https://pfister.dev/blog/2025/Bildschirmfoto%20vom%202025-02-16%2019-06-44.png)

这个文件就是我们要通过UART发送的。测试的话，用生成的uf2文件通过USB烧录到RP2350也行。但注意它是从RAM运行的，断电就没了。

## UART传输固件

先写了个简单的Python脚本，负责发送固件，还加了校验。7.3 KiB的固件发送约160ms，校验约150ms。

不算差，但来算一下理论速度。

### 波特率和实际速率

UART模式是`8n1`：1个起始位，8个数据位，无奇偶校验，1个停止位。每个符号10位，但有效数据8位。所以实际速率是 1000000 波特/10位×8位/8 = **100 kB/s**（0.1 MB/s）

测试二进制文件7256字节，但要切成32字节一块，末尾填零后总长7264字节。每个块前的写命令产生1字节开销，总共227字节开销。7491字节理论耗时74.91ms。

不解释为啥Python脚本花了双倍时间（可能我哪里写岔了？）。总之各种中间层（Linux、USB什么的）的影响，实际没有跑满带宽。觉得这里不太关键，跳过！

## 固件嵌套

要让RP2350从另一个MCU启动，得把前者的固件嵌到后者的固件里。

可以用各种在线工具，把二进制转成包含大数组的C头文件。

我想要更自动化的方式。C++23倒是加入了 [#embed预处理指令](https://en.cppreference.com/w/c/preprocessor/embed)，但我在pico SDK 2.1.0上跑不通。设了 `set(CMAKE_CXX_STANDARD 23)` 还是报错。可能是GCC[还没实现](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=105863)C++23的#embed？

最后在[Stackoverflow](https://stackoverflow.com/a/56006001)上找到个半自动方案，试了试能用（见我的代码[这里](https://codeberg.org/retsifp/rp2350_uart/src/branch/master/main_controller/CMakeLists.txt#L61)）。

## 从另一个MCU启动RP2350

准备就绪：Python验证有了，固件二进制也嵌进另一个MCU的固件里了。把Python代码差不多翻译成C，经过几轮死循环调试...跑通了！代码在[这里](https://codeberg.org/retsifp/rp2350_uart/src/branch/master/main_controller/main_controller.cpp)。

看看速度，脚本输出：

```
Hello, world!
Device is there!
Start addr is 0x20001055
Finished sending FW, 7264 bytes were written!
The end. This took 74750 us
```

比理论最快还快0.2ms。不知道怎么做到的，不过不亏 ;-)

## 缺失的功能

启动加载器用的引脚在另一个GPIO bank，SDK里没法当串口用。发了[GitHub issue](https://github.com/raspberrypi/pico-sdk/issues/2280)后，很快就有人给了个[解决方案的链接](https://github.com/raspberrypi/pico-examples/pull/571)。试了一下，完美。估计后续会合并进SDK，关注着吧！

## 长距离可靠连接

UART长线缆信号衰减是个问题。线缆超过一米可能就不稳定了。

解法很简单：把单端UART转成差分信号。我用了TI的THVD1450做RS-485转换。做了块小PCB，两个收发器加RJ-45接口。RS-485通常用120欧姆线缆，我直接上了100欧姆网线，好买又便宜。

![P1001175.jpg](https://pfister.dev/blog/2025/P1001175.jpg)

两边各接这块板子，10米线缆都能启动远程RP2350（家里最长就10米了）。理论上能跑100米，但10米对我够用了。

![P1001181.jpg](https://pfister.dev/blog/2025/P1001181.jpg)

板子是透明工作的，跟直连一样，所以到手即用，非常可靠。我的方案就定这个了。拭目以待！

_有问题或想法可以邮件联系，或者在[YouTube视频](https://youtu.be/eno0hiFSr18)评论区留言。_