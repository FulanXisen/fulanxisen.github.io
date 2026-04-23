---
title: Control Bluetooth Device via Bluetoothctl
description: Bluez and Bluetoothclt in Linux Bluetooth System
tags:
    - Bluetooth
    - Linux
categories:
    - Technology
---
# Control a Bluetooth Device via Bluetoothctl

------

1\. Overview[](#overview)
-------------------------

In this tutorial, we’ll learn how to connect to a Bluetooth device via the terminal. This process involves configuring our Bluetooth controller, pairing it to our target device, and then finally connecting.

2\. Using the _bluetoothctl_ Command[](#using-the-bluetoothctlcommand)
----------------------------------------------------------------------

[BlueZ](https://git.kernel.org/pub/scm/bluetooth/bluez.git/about/) provides support for Bluetooth functionality and protocols through the [_bluetoothd_](https://www.mankier.com/8/bluetoothd) daemon. To interact with _bluetoothd_ from the terminal, we use the [_bluetoothctl_](https://manpages.debian.org/stretch/bluez/bluetoothctl.1.en.html) command.

Let’s begin by running _bluetoothctl_ without any arguments:

    $ bluetoothctl
    Agent registered
    [bluetooth]#
    

Using _bluetoothctl_ by itself will open the interactive shell. **This is called interactive mode and is where we can run commands to configure our Bluetooth settings.**

To get more information about using interactive mode, let’s run the _help_ command in the interactive shell:

freestar.config.enabled\_slots.push({ placementName: "baeldung\_leaderboard\_mid\_1", slotId: "baeldung\_leaderboard\_mid\_1" });

    [bluetooth]# help
    Menu main:
    Available commands:
    -------------------
    advertise                                         Advertise Options Submenu
    monitor                                           Advertisement Monitor Options Submenu
    ...
    

Running _help_ in the shell will list all available commands for _bluetoothctl_ interactive mode with a short summary of what they do.

Sometimes, however, we want to run a command outside of the interactive shell. Luckily, there’s also a non-interactive mode in _bluetoothctl_, which we can use by running an individual command:

    $ bluetoothctl --help
    bluetoothctl ver 5.64
    Usage:
            bluetoothctl [--options] [commands]
    Options:
            --agent         Register agent handler: 
    ...

3\. Preparing Our Bluetooth Controller[](#preparing-our-bluetooth-controller)
-----------------------------------------------------------------------------

Now that we know the basics of using _bluetoothctl_, let’s begin preparing our Bluetooth controller.

### 3.1. Configuring a Bluetooth Controller[](#1-configuring-a-bluetooth-controller)

Let’s begin to configure our Bluetooth controller by using the _show_ command:

    $ bluetoothctl show 00:1A:7D:DA:71:15
    Controller 00:1A:7D:DA:71:15 (public)
            Name: pc-m
            Alias: pc-m
            Class: 0x00000000
            Powered: no
            Discoverable: no
            DiscoverableTimeout: 0x00000000
            Pairable: no
    ...
            Discovering: no
    ...
    

Typically, the _bluetoothctl show_ command will output a large amount of information. However, **we just need to ensure our controller is powered-on, discoverable, and pairable**.

Let’s start by powering on our controller:

    $ bluetoothctl power on
    [CHG] Controller 00:1A:7D:DA:71:15 Class: 0x006c0104
    Changing power on succeeded
    

We can use the _bluetoothctl_ _power on_ command to power on our controller. **It’s important to power on our Bluetooth controller before modifying other controller attributes.**

Next, we should set the controller to be discoverable and pairable:

    $ bluetoothctl discoverable on
    Changing discoverable on succeeded
    $ bluetoothctl pairable on
    Changing pairable on succeeded
    

We set the controller to discoverable using the command _bluetoothctl discoverable on_, and then we use the _bluetoothctl pairable on_ command to set our controller to pairable. The output of these commands shows that they were successful.

### 3.2. Using Multiple Bluetooth Controllers[](#2-using-multiple-bluetooth-controllers)

When using multiple Bluetooth controllers, we must ensure we select the correct one before configuring.

freestar.config.enabled\_slots.push({ placementName: "baeldung\_leaderboard\_mid\_3", slotId: "baeldung\_leaderboard\_mid\_3" });

Let’s use the _bluetoothctl list_ command to get a list of connected Bluetooth controllers:

    $ bluetoothctl list
    Controller 00:1A:7D:DA:71:15 pc-m [default]
    Controller 34:02:86:03:7C:F2 pc-m #2 
    

This command outputs information about the connected Bluetooth controllers, including their [MAC addresses](http://baeldung.com/linux/get-mac-address#what-is-mac-address) and what the default controller is. **The default controller is the controller that will be operated on when we run a command.**

**To change the default controller, we use the _bluetoothctl_ _select_ command**, passing the MAC address of the controller we want to connect to:

    [bluetooth]# select 34:02:86:03:7C:F2
    Controller 34:02:86:03:7C:F2 pc-m [default]
    [bluetooth]# list
    Controller 00:1A:7D:DA:71:15 pc-m 
    Controller 34:02:86:03:7C:F2 pc-m #2 [default]
    

In this example, we used the _select_ command in interactive mode. Non-interactive mode opens up a new session per command, but the interactive shell maintains the same session until exited. **Since the _select_ command only changes the default controller for the current session, it will only work within interactive mode**.

However, there is a way we can use _select_ in a similar way to non-interactive mode:

    $ bluetoothctl <<< $'select 34:02:86:03:7C:F2\nlist\n'
    ...
    [bluetooth]# select 34:02:86:03:7C:F2
    Controller 34:02:86:03:7C:F2 pc-m [default]
    [bluetooth]# list
    Controller 00:1A:7D:DA:71:15 pc-m 
    Controller 34:02:86:03:7C:F2 pc-m #2 [default]
    

In this example, we use a [here-string](/linux/heredoc-herestring#here-string) to redirect our string to stdin. This causes the _bluetoothctl_ shell to treat the string as user input and allows us to automate the process of using interactive mode. Our string can contain as many commands as want as long as we end each with a newline.

4\. Connect a Bluetooth Device Using the Terminal[](#connect-a-bluetooth-device-using-the-terminal)
---------------------------------------------------------------------------------------------------

After configuring our Bluetooth controller, we can begin pairing and connecting our device.

### 4.1. Pairing a Bluetooth Device[](#1-pairing-a-bluetooth-device)

Now, we can begin to pair our Bluetooth device to our controller. To start, we should turn our controller to discovery mode:


    $ bluetoothctl scan on
    Discovery started
    [CHG] Controller 00:1A:7D:DA:71:15 Discovering: yes
    ^Z
    [1]+  Stopped                 bluetoothctl scan on
    

To set the controller to discovery mode, we use the _bluetoothctl scan on_ command. However, this command is a [foreground job](/linux/foreground-background-process#background-and-foreground-jobs), which means that we won’t be able to use the terminal until it is finished. So, we put it in the background using Ctrl-Z.

Now, let’s output the discovered devices using _bluetoothctl devices_:

    $ bluetoothctl devices
    Device 3C:4D:BE:84:1F:BC MyEarbuds
    Device 60:B7:6E:35:39:0D MyPhone
    

The output shows us any discovered devices with their names and MAC addresses. **The important part is the MAC address, which we use to pair devices.**

Once we know the MAC address of our device, we can begin pairing. Let’s use the _bluetoothctl pair_ command to pair to our device with the name “MyEarbuds”:

    $ bluetoothctl pair 3C:4D:BE:84:1F:BC
    Attempting to pair with 3C:4D:BE:84:1F:BC
    [CHG] Device 3C:4D:BE:84:1F:BC Connected: yes
    ...
    [CHG] Device 3C:4D:BE:84:1F:BC Paired: yes
    Pairing successful
    

We can pair a device by using the device’s MAC address as an argument to the _bluetoothctl pair_ command. The output will tell us if our device paired successfully or not.

Now that our device is paired, we don’t need to be in discovery mode anymore. To exit discovery mode, we must end the _bluetoothctl scan_ command that we put into the background:

    $ fg
    bluetoothctl scan on
    ^C
    

To stop a background job, we use the [_fg_](/linux/kill-background-process#1-fg) command to bring the _bluetoothctl scan_ into the foreground. Then we press Ctrl-C to stop the program.

### 4.2. Connecting a Bluetooth Device[](#2-connecting-a-bluetooth-device)

Let’s start connecting to our Bluetooth device using _bluetoothctl connect_:

freestar.config.enabled\_slots.push({ placementName: "baeldung\_incontent\_2", slotId: "baeldung\_incontent\_2" });

    $ bluetoothctl connect 3C:4D:BE:84:1F:BC
    Attempting to connect to 3C:4D:BE:84:1F:BC
    ...
    Connection successful
    

We can connect a paired Bluetooth device by using its MAC address with the _bluetoothctl_ _connect_ command. The output will show whether we successfully connected to our device or not.

### 4.3. Disconnecting a Bluetooth Device[](#3-disconnecting-a-bluetooth-device)

Likewise, we can disconnect our device using the _bluetoothctl disconnect_ command, again passing the MAC address of the device we want to disconnect:

    $ bluetoothctl disconnect 3C:4D:BE:84:1F:BC
    Attempting to disconnect from 3C:4D:BE:84:1F:BC
    ...
    Successful disconnected
    

The output shows us that our device was successfully disconnected.

5\. Overview[](#overview-1)
---------------------------

In this article, we learned how to connect a Bluetooth device via the Linux terminal. We began by learning the basics of the _bluetoothctl_ command. Then, we learned about configuring a Bluetooth controller. Finally, we learned how to pair and then connect a Bluetooth device.