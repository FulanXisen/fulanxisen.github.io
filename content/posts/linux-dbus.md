---
title: "Linux DBus IPC System"
date: 2023-09-21T22:39:17+08:00
# weight: 1
# aliases: ["/first"]
tags: ["DBus", "IPC", "Linux"]
author: ["福岚溪森"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "DBus is an inter-process communication system in Linux environment for both singleton and distributed system."
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/<path_to_repo>/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---
1\. 概述[](#overview)
-------------------------

[DBus](https://linux.die.net/man/1/dbus-daemon)是一个消息总线系统，允许不同的进程在分布式环境中进行通信。它是Linux世界广泛采用的强大工具，使应用程序能够无缝共享数据并相互交互。

在本教程中，我们将详细探讨DBus，并了解它在[Linux桌面环境](/linux/default-desktop-environment-start-up#setup-procedure)中的使用方式。

2\. DBus[](#dbus)
-----------------
DBus是一个进程间通信（[IPC](/linux/ipc-performance-comparison#overview)）系统，允许在单个系统或不同系统上运行的两个或多个进程之间进行通信。它为这些进程提供了一种简单、标准和安全的交换数据和命令的方式。因此，它消除了队复杂、容易出错的通信机制的需求，如套接字([sockets](/linux/communicate-with-unix-sockets#introduction-to-sockets))或管道([pipes](/linux/anonymous-named-pipes#what-is-a-pipe))。

此外，由于具有语言独立性，我们可以通过多种编程语言（如C、C++、Python等）使用DBus。这种灵活性使不同的系统组件能够协同工作，从而更容易创建复杂的应用程序。

此外，通过DBus，我们可以构建一个用于系统范围事件和通知的消息系统，允许应用程序订阅事件，如设备插入或移除、网络更改和电源事件。

### 2.1. Components[](#1-components)
DBus系统由三个主要组件组成 - 总线(bus)、服务(service)和接口(interface)。

第一个组件, 总线, 是所有支持DBus的应用程序的主要通信点，负责在应用程序之间路由消息，使标准化的服务注册和发现变得更容易。有两种类型的总线可用 - 系统总线(system bus)和会话总线(session bus)。

第二个组件，服务，是为其他应用程序提供一个或多个接口的应用程序。服务会自行注册到总线上，使其他应用程序能够发现它们。

最后的组件，接口, 是一个服务向外部公开的方法(methods)、信号(signals)和属性(properties)的集合。方法是其他应用程序可调用的函数，信号是服务发出的用于通知其他应用程序发生更改的事件，而属性是其他应用程序可以读取或写入的值。

除了这三个主要组件，DBus还包括对象的概念，对象是服务的实例，公开一个或多个接口。对象具有用于在总线上寻址它们的唯一对象路径。


### 2.2. Architecture[](#2-architecture)
DBus具有客户端-服务器架构，服务器充当消息总线，客户端发送和接收消息。服务器负责维护所有可用服务和对象的注册表，而客户端可以通过服务器访问这些服务和对象。

我们应该注意，DBus使用模块化架构，不同的组件负责不同的任务。架构的核心是**消息总线守护进程**，负责管理不同进程之间的通信。守护进程维护所有可用服务和对象的注册表，并在客户端和服务器之间路由消息。

但是，要与消息总线守护进程交互，应用程序可以使用**客户端库**，这些库提供了用于发送和接收消息、注册服务以及监听事件的API。

最后，服务提供者向消息总线守护进程注册其服务，使其他应用程序能够访问它们。

3\. DBus in Linux Desktop Environments[](#dbus-in-linux-desktop-environments)
-----------------------------------------------------------------------------
大多数桌面环境都集成了DBus。因此，我们可以使用不同的编程语言和环境开发系统应用程序，并通过DBus进行交互。

### 3.1. GNOME[](#1-gnome)
让我们从GNOME的示例开始，这是最流行的桌面环境。GNOME广泛使用DBus来促进应用程序之间的通信。例如，在GNOME中更改音量时，声音小部件通过DBus发送消息给声音守护进程以调整音量。

类似地，当我们插入USB驱动器时，文件管理器通过DBus向系统守护进程发送消息以挂载驱动器。

此外，GNOME Shell的扩展广泛使用DBus。Shell扩展可以添加新功能或更改桌面的外观和感觉。DBus建立了GNOME Shell与Shell扩展之间的通信。例如，在GNOME Shell中单击网络图标时，会通过DBus发送消息给网络管理器以显示可用的Wi-Fi网络。

### 3.2. KDE and Xfce[](#2-kde-and-xfce)
在 _[KDE](https://docs.kde.org/) &_ _[Xfce](https://docs.xfce.org/)_ 桌面环境中，系统应用程序使用一组DBus接口进行交互。例如，这两个桌面环境的[NetworkManager](/linux/network-manager#network-manager-components)小部件都使用DBus与NetworkManager服务通信并管理网络连接。

此外，KDE和Xfce Power Management系统提供了一个DBus接口来监视系统电源状态并调整电源设置。它们还提供了一个DBus接口来管理显示亮度。

此外，KDE Plasma Desktop Shell广泛使用DBus，允许应用程序与桌面互动并向其他应用程序公开各种系统服务。例如，Plasma通知系统使用DBus向其他应用程序发送通知，使它们能够向用户显示消息。

类似地，Xfce会话管理器使用DBus允许应用程序注册为启动应用程序，并在我们登录Linux时自动启动。它还使用DBus在我们注销并重新登录时保存和恢复用户会话。

4\. Using DBus[](#using-dbus)
-----------------------------
现在我们了解了DBus以及它在Linux桌面中的使用，让我们探讨它在Linux系统中的实际用途。

### 4.1. With System Services[](#1-with-system-services)
DBus最常见的用途是与系统服务进行交互。系统服务和守护进程通过DBus接口公开其功能，允许其他应用程序和服务与它们连接。

例如，NetworkManager服务提供了一个DBus API来配置网络连接。我们可以使用[_dbus-send_](https://manpages.debian.org/bullseye/dbus/dbus-send.1.en.html)命令行工具向NetworkManager服务发送消息以配置网络连接。

让我们看一个**如何使用DBus连接到Wi-Fi网络**的示例：

```shell
$ dbus-send --system --type=method_call \
    --dest=org.freedesktop.NetworkManager \
    /org/freedesktop/NetworkManager \
    org.freedesktop.NetworkManager.AddAndActivateConnection \
    '{"connection": {"type": "802-11-wireless", "uuid": "MyWifiConnection", "id": "My Wifi", "ssid": "MyWifiSSID"}}'
```

为了更好地理解这一点，让我们详细看看上面的_dbus-send_命令与上面的交互：

- _dbus-send_  
从命令行发送DBus消息
- _--system_  
指定我们要将消息发送到系统范围的DBus守护进程，而不是到特定用户会话守护进程
- _--type=method_call_  
指定我们要在目标对象上调用方法，而不仅仅是读取或写入属性
- _--dest=org.freedesktop.NetworkManager_  
设置我们要发送消息的DBus服务的唯一名称
- _/org/freedesktop/NetworkManager_  
指定我们要发送消息的_NetworkManager_服务的对象路径
- _org.freedesktop.NetworkManager.AddAndActivateConnection_  
定义我们要在_NetworkManager_对象上调用的接口和方法名称
- _‘{“connection”: {“type”: “802-11-wireless”, “uuid”: “MyWifiConnection”, “id”: “My Wifi”, “ssid”: “MyWifiSSID”}}’_  
是对_AddAndActivateConnection_方法的参数，这是一个描述我们要添加的新网络连接的JSON格式字符串
通过这种方式，我们可以使用DBus连接到Wi-Fi网络，甚至可以使用其他命令来管理连接。

### 4.2. With System Power Management[](#2-with-system-power-management)
我们还可以使用DBus与系统电源管理进行交互，这允许我们控制与系统电源相关的功能，如关机、重启和挂起。让我们看一个如何使用DBus关机Linux系统的示例：

```shell
$ dbus-send --system --print-reply --dest=org.freedesktop.login1 /org/freedesktop/login1 org.freedesktop.login1.Manager.PowerOff boolean:true
```
在这里，我们使用DBus发送消息给系统的登录管理器以关闭系统。

类似地，我们可以使用以下命令重新启动或挂起系统：
```shell
$ dbus-send --system --print-reply --dest=org.freedesktop.login1 /org/freedesktop/login1 org.freedesktop.login1.Manager.Reboot boolean:true
$ dbus-send --system --print-reply --dest=org.freedesktop.login1 /org/freedesktop/login1 org.freedesktop.login1.Manager.Suspend boolean:true
```    
总的来说，DBus可以成为我们与Linux系统的不同组件进行交互的强大工具。借助其简单而灵活的API，我们可以使用它来控制Linux环境中的几乎任何内容。

5\. Signal Communication Using DBus[](#signal-communication-using-dbus)
-----------------------------------------------------------------------
与之前的交互类似，DBus还为我们提供了两种应用程序之间的通信方法 - 方法调用和信号。

我们可以使用方法调用来调用远程进程中的方法或函数并获得返回值。DBus方法调用可以是同步的或异步的。同步的方法调用会阻塞调用进程，直到远程方法返回一个值。异步方法调用不会阻塞调用进程，而会立即返回，允许调用进程继续其工作。

**DBus信号本质上是异步的**。我们可以使用信号向远程进程发送异步消息。当一个进程发出信号时，它会广播给所有感兴趣的方。此外，感兴趣的方可以通过监听特定接口和对象路径上的信号来订阅接收特定事件的信号。

### 5.1. Sending a Signal[](#1-sending-a-signal)

让我们看看如何通过DBus发送信号。要开始使用DBus协议在应用程序之间进行信号通信，我们需要安装 _python-dbus_ 包：
```shell
$ pip install dbus-python
```
安装后，我们可以在Python中使用 _dbus_ 模块库与DBus进行交互并通过其通道(pathway)发送信号：
```python
    #!/usr/bin/env python3
    
    import dbus
    import dbus.service
    from dbus.mainloop.glib import DBusGMainLoop
```
首先，我们定义一个新的DBus服务，它使用继承自DBus服务对象的类来设置具有唯一名称的DBus服务，并定义一个名为 _ExampleSignal_ 的信号，它带有一个参数 _message_：
```python
class ExampleDBusService(dbus.service.Object):
    def __init__(self):
        bus_name = dbus.service.BusName('com.example.DBusExample', bus=dbus.SessionBus())
        dbus.service.Object.__init__(self, bus_name, '/com/example/DBusExample')

    @dbus.service.signal('com.example.DBusExample')
    def ExampleSignal(self, message):
        print('Signal sent:', message)
```
然后，为了使用我们定义的DBus服务，我们需要启动DBus主循环(`DBusGMainLoop`)并创建 _ExampleDBusService_ 类的实例：
```python
DBusGMainLoop(set_as_default=True)
ExampleDBusService()
```
一旦创建了 _ExampleDBusService_ 实例，我们可以使用来自DBus包的 _SessionBus()_ 对象连接到它。接下来，我们获取对DBus服务对象的引用，以允许与之前定义的DBus服务进行交互：
```python
bus = dbus.SessionBus()
dbus_service = bus.get_object('com.example.DBusExample', '/com/example/DBusExample')
```
现在，我们可以通过创建一个DBus接口对象向DBus服务发送信号，最后，我们在此对象上调用 _ExampleSignal_ 方法并传递一个参数 - 我们的消息：
```python
    dbus_iface = dbus.Interface(dbus_service, 'com.example.DBusExample')
    dbus_iface.ExampleSignal('hello world')
```
这将导致调用 _ExampleSignal_ 方法并传递我们的信号消息 - 在这种情况下，是'hello world'。

### 5.2. Testing the Signal Sender[](#2-testing-the-signal-sender)
现在，我们更好地了解了Python脚本的功能。让我们逐步执行它的步骤。

我们首先将此代码脚本保存为扩展名为 _.py_ 的文件。在本示例中，我们将其保存为 _send_signal.py._ 然后，让我们在保存文件的目录中打开终端：
```shell
$ ls
Documents  Downloads send_signal.py
```
现在，既然我们在文件的目录中，让我们使用chmod命令将文件设置为可执行：
```shell
$ chmod +x send_signal.py
```
最后，让我们运行脚本：
```shell
$ python3 send_signal.py
Signal sent: hello world
```
正如我们所看到的，脚本运行并显示输出。这样，我们成功地通过DBus发送了一个信号。

### 5.3. Receiving a Signal[](#3-receiving-a-signal)
另一方面，要在DBus系统上接收信号，我们创建一个信号处理程序，用于监听特定接口和对象路径上的信号。在这种情况下，与我们发送信号的脚本类似，我们使用了相应的库来接收信号：
```python
#!/usr/bin/env python3

import dbus
import dbus.service
from dbus.mainloop.glib import DBusGMainLoop
from gi.repository import GLib
```
我们首先定义一个类 _ExampleSignalReceiver_ , 它继承自DBus服务对象。这个类负责接收发送到DBus的信号。在这个类内部，我们定义了两个方法   _ExampleSignal_ 和 _ExampleMethod_. _ExampleSignal_ 方法将在信号被发送到DBus上时被调用, 而 _ExampleMethod_ 将用于调用DBus上的一个方法：
```python
class ExampleSignalReceiver(dbus.service.Object):
    def __init__(self):
        bus_name = dbus.service.BusName('com.example.DBusExample', bus=dbus.SessionBus())
        dbus.service.Object.__init__(self, bus_name, '/com/example/DBusExample')

    @dbus.service.signal('com.example.DBusExample')
    def ExampleSignal(self, message):
        print('Signal received:', message)

    @dbus.service.method('com.example.DBusExample')
    def ExampleMethod(self):
        print('Method called')
        return 'Example response'
```
为了启动DBus主循环，我们调用[GLib](https://en.wikipedia.org/wiki/GLib)主循环，这是当前线程的默认主循环，并创建 _ExampleSignalReceiver_ 类的实例以接收发送到DBus的信号:
```python
    DBusGMainLoop(set_as_default=True)
    ExampleSignalReceiver()
```
接下来，我们创建一个**DBus会话总线对象**并**添加一个信号接收器**来监听DBus上的信号。_ExampleSignalReceiver.ExampleSignal_ 方法用作 _ExampleSignal_ 信号的信号处理程序。此外，_dbus_interface_ 参数指定信号所属的DBus接口，而 _signal_name_ 参数指定信号的名称：

```python
bus = dbus.SessionBus()
bus.add_signal_receiver(
    ExampleSignalReceiver.ExampleSignal,
    dbus_interface='com.example.DBusExample',
    signal_name='ExampleSignal'
)
```
最后，我们启动GLib主循环以监听DBus上的传入信号：
```python
    loop = GLib.MainLoop()
    loop.run()
```
通过这样做，我们的脚本将无限地运行，以继续通过DBus接收信号消息。

### 5.4. Testing the Receiver[](#4-testing-the-receiver)
现在，我们更好地了解了脚本。让我们将其保存，例如保存为 _receive_signal.py._ 然后，我们导航到其目录，使其可执行，并运行脚本：
```shell
    $ python3 receive_signal.py
    Signal received: hello world
```
正如我们所看到的，脚本执行并显示我们发送的信号消息。每当我们通过DBus接收信号时，输出都会显示在终端上。然而，我们可以首先运行 _receive_signal.py_ 脚本来设置信号接收器，然后运行 _send_signal.py_ 脚本来发送信号。

注意, 如果我们想要通过DBus系统持续的接收消息, 我们需要保持 _receive\_signal.py_ 脚本始终运行.

以上只是我们能通过DBus进行IPC的简单例子. 还有很多高级场景, 例如在进程间传输复杂数据结构, 以及为通信创建自定义的接口和对象.

6\. DBus Limitations[](#dbus-limitations)
-----------------------------------------
尽管DBus在我们已经看到并实施的用途中非常有用，但它也存在一些挑战。让我们看一下在使用DBus时常见的一些挑战。

### 6.1. Security Concerns[](#1-security-concerns)
由于DBus是一个系统级组件，它可以直接访问我们的系统资源，并有潜在可能用于执行恶意代码。因此，确保合适的保护DBus并确保只有受信任的应用程序可以访问DBus非常重要。

幸运的是，DBus具有内置的安全功能，如消息过滤和身份验证机制，以减轻安全顾虑。我们可以配置应用程序只接受来自受信任来源的消息，并验证消息内容，以确保其符合特定的模式或格式。

### 6.2. Compatibility Issues[](#2-compatibility-issues)
虽然DBus是现代Linux桌面环境的标准组件，但不同的环境可能以略有不同的方式实现DBus，这可能导致应用程序之间的兼容性问题。

例如，为GNOME开发的应用程序如果依赖于KDE中不存在的特定DBus API或功能，则可能无法在KDE上正常工作。为了解决这些兼容性问题，很重要的是我们在开发应用程序时考虑到兼容性，并在不同环境中进行彻底测试。

### 6.3. Performance Considerations[](#3-performance-considerations)
应用程序之间的DBus通信也可能对系统性能产生重大影响，特别是如果大量应用程序同时通过DBus进行通信。由于DBus使用异步消息传递机制，因此比其他IPC机制（如远程过程调用([RPC](https://en.wikipedia.org/wiki/Remote_procedure_call))）更高效。

然而，过度使用DBus仍然可能导致性能下降。因此，为了优化性能，很重要的是我们尽量减少发送和接收的DBus消息数量，并确保应用程序有效地使用DBus。此外，我们还应确保DBus消息能够快速处理，不会阻塞应用程序的主线程或引起过多的CPU使用。

### 6.4. Debugging and Troubleshooting[](#4-debugging-and-troubleshooting)
最后，在使用DBus时，调试和故障排除可能会具有挑战性。由于它是一个底层系统组件，因此诊断与DBus通信相关的问题可能会很困难。

然而，为了解决这一挑战，很重要的是我们要对DBus的内部工具有良好的理解，例如 [_dbus-monitor_](https://manpages.debian.org/bullseye/dbus/dbus-monitor.1.en.html)和 _dbus-send_ ，以调试和排除DBus相关的问题。我们还应记录DBus错误消息并使用错误处理机制来检测和处理与DBus通信相关的潜在问题。

7\. Conclusion[](#conclusion)
-----------------------------
在本文中，我们探讨了DBus以及在Linux系统中如何实际使用它进行进程间通信。我们首先讨论了DBus的基础知识，包括其架构和组件。接下来，我们讨论了DBus在各种桌面环境中的集成，如GNOME、KDE和Xfce。

然后，我们探讨了DBus的一些实际应用，包括如何将其用于系统范围的通知和管理电源管理设置。最后，我们讨论了在使用DBus时可能遇到的一些挑战。

通过正确的知识和技能，我们可以利用DBus构建健壮高效的Linux桌面应用程序，满足现代用户的需求。