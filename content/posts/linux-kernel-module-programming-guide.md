# Linux Kernel Module Programming Guide

[TOC]

## Easy Begin

### 内核模块包

编程内核模块首先要安装内核模块包`kmod`.

```bash
sudo apt-get install build-essential kmod
```

### 命令行的内核模块相关命令

```bash
sudo lsmod # 查看所有加载进内核的模块
```

```bash
sudo cat /proc/modules # 和lsmod有什么不同
```

```bash
sudo modinfo ABC_x.ko # 查看模块ABC的信息
```

```bash
sudo insmod ABC_x.ko # 向内核中加载模块ABC
```

```bash
sudo rmmod ABC-x # 从内核中卸载模块ABC, 注意ABC_x.ko名称中的'_'字符会被内核自动替换成'-'字符
```

```bash
sudo journalctl --since "1 hour ago" | grep kernel # 查看过去一小时内的内核日志
```

### 编码前须知

1. 版本.
   如果编译模块A时include的内核头文件版本与加载模块A的内核版本不一致, 会导致加载失败.
   如果必须要这样做, 例如模块A兼容多个内核版本, 需要启用CONFIG_MODVERSIONS编译制导.
2. 命令行.
   内核模块无法打印输出到屏幕,但可以记录日志信息然后读取日志信息到屏幕.
3. SecureBoot.
   SecureBoot是UEFI的一个设置, 要求只会引导安全密钥签名过的模块.
   `[Shell]ERROR: could not insert module`.
   `[dmesg]Lockdown: insmod: unsigned module loading is restricted; see man kernel lockdown.7`.
   [SecureBoot - Debian Wiki](https://wiki.debian.org/SecureBoot)

### 内核头文件

编译模块必需内核头文件, 编译时Makefile也需要include内核头文件所在文件夹.

```bash
// 查看系统内核版本
> uname -r
6.1.21-v8+
```

```bash
sudo apt-get update 
// 查看对于内核头文件是否在源仓库中
sudo apt-cache search linux-headers-`uname -r`
// 下载(到/usr/src文件夹下)
sudo apt-get install linux-headers-`uname -r`
```

rpi的rpi-os中的Debian源里没有对应版本的内核头文件, 但是rpi自己提供了一份.

```bash
// 不需要指定版本
sudo apt-get install raspberrypi-kernel-headers
```

### 编码规范

1. 缩进使用tabs而非spaces. 提交patch到上游时必须遵守.
2. `pr_info`和`pr_debug`in `include/linux/printk.h`
3. `kbuild`, `Documentation/kbuild/modules.rst`
4. `Makefile`, `Documentation/kbuild/makefiles.rst`

### 一个极简内核模块示例

```C
#include <linux/module.h> /* needed by all modules */
#include <linux/init.h> /* needed for macros */
#include <linux/printk.h>  /* needed for pr_info */
/* __initdata macro修饰下的变量在init完成时会被丢弃并释放内存(适用于built-in driver, 对loadable driver没有影响) */
static int hello_data __initdata = 3;

/* 模块接受一个基本类型参数的声明方法 */
static int input_data = 1;
module_param(input_data, int, S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP);
MODULE_PARM_DESC(input_data, "a int input");

/* 模块接收一个字符串参数的声明方法 */
static char* input_string = "???"; // 字符串的最大长度不需要定义
module_param(input_string, charp, 0000);
MODULE_PARM_DESC(input_string, "a string input");

/* 模块接受一个数组类型参数的声明方法 */
static int input_array[2] = {0, 1}; // 预先定义数组的最大长度, 输入数组不足最大长度的位置依然保持这里静态声明的默认值
static int input_array_argc = 0; // number of array elements
module_param_array(input_array, int, &input_array_argc, 0000);
MODULE_PARM_DESC(input_array, "a int array input");
/* __init macro 修饰下的init函数在完成时会被丢弃并释放内存(适用于built-in driver, 对loadable driver没有影响)
 因为built-in driver只加载进内核一次 */
static int __init hello_init(void) 
{
  pr_info("hello %d %d\n", hello_data, input_data);
  return 0;
}
/* __exit macro 修饰下的exit函数会被忽略(适用于built-in driver, 对loadable driver没有影响) 
	因为built-in driver不会被卸载出内核 */
static int __exit hello_exit(void)
{
  pr_info("bey\n");
  return 0;
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("fanyx");
MODULE_DESCRIPTION("A simple driver");
```

```bash
# 加载内核时, 按照名称赋参数值
sudo insmod hello.ko input_int=1 input_string="hehe" input_array=-1,-1
```

### 编译多个文件组成的模块

```bash
> ls
start.c
stop.c
```

```makefile
obj-m += hello.o # 增加一个object hello
hello-objs := start.o stop.o # 声明hello object包含的文件
PWD := %(CURDIR)
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

## 设备驱动程序视角下的内核模块

Linux Kernel Module的一个重要类型是Device Driver.
物理设备抽象为文件，存放在/dev目录下面。通过 cat /proc/devices 可以查看所有设备。

Device分为两种，Char Device和Block Device

## 模块如何开始和结束

## 模块可用的函数

## `dmesg`读取模块日志

[Message logging with printk — The Linux Kernel documentation](https://www.kernel.org/doc/html/next/core-api/printk-basics.html)



| Name         | String | Alias Function                                               |
| ------------ | ------ | ------------------------------------------------------------ |
| KERN_EMERG   | "0"    | [`pr_emerg()`](https://www.kernel.org/doc/html/next/core-api/printk-basics.html#c.pr_emerg) |
| KERN_ALERT   | "1"    | [`pr_alert()`](https://www.kernel.org/doc/html/next/core-api/printk-basics.html#c.pr_alert) |
| KERN_CRIT    | "2"    | [`pr_crit()`](https://www.kernel.org/doc/html/next/core-api/printk-basics.html#c.pr_crit) |
| KERN_ERR     | "3"    | [`pr_err()`](https://www.kernel.org/doc/html/next/core-api/printk-basics.html#c.pr_err) |
| KERN_WARNING | "4"    | `pr_warn()`                                                  |
| KERN_NOTICE  | "5"    | [`pr_notice()`](https://www.kernel.org/doc/html/next/core-api/printk-basics.html#c.pr_notice) |
| KERN_INFO    | "6"    | [`pr_info()`](https://www.kernel.org/doc/html/next/core-api/printk-basics.html#c.pr_info) |
| KERN_DEBUG   | "7"    | [`pr_debug()`](https://www.kernel.org/doc/html/next/core-api/printk-basics.html#c.pr_debug) and [`pr_devel()`](https://www.kernel.org/doc/html/next/core-api/printk-basics.html#c.pr_devel) if DEBUG is defined |

```bash
# 查看当前console loglevel
$ cat /proc/sys/kernel/printk
4        4        1        7
#current,default,minimun,boot-time-default
```

```bash
# 修改当前console loglevel
$ dmesg -n 5
```

优先级高于(越小越高)当前`console_loglevel`的日志会立刻出现在console上.

## User Space vs Kernel Space

## Name Space

## Code Space

## Device Driver

## 字符设备驱动程序
>
> Charactor Device Driver
>

1. `struct file_operation`
2. `struct file`
3. 注册一个设备
4. 注销一个设备
5. 示例程序
6. 兼容多个内核版本

## `/proc`: Module向Process发送信息

1. `struct proc_ops`
2. 读、写`/proc` 文件
3. 通过标准文件系统管理`/proc`文件
4. 通过`seq_file`管理`/proc`文件

## `/sys`：读写Module内部的变量

## 与设备文件对话

- using device file to write things to the physical device
  - write device's commands
  - write data to be sent through the device
- using device file to read things from the physical device
  - read responses for device's commands
  - read data received through the device

Unix ioctl机制实现以上功能

## System Calls

System Call是真正的process to kernel communication mechanism
一般来说，process不能访问kernel(既不能访问kernel memory也不能调用kernel function)，这是CPU从硬件上保证的(称为protected mode 或者 page protection)
system call是一个例外。process向特定的寄存器中填充参数，然后调用一个特别的指令跳转到kernel中预先定义好的位置。当程序跳转到这个位置的时候，硬件就知道程序不再以受限模式运行，而是处于kernel中。这个可以让process跳转到kernel当中的特别的指令就是`system_call`. `system_call`的过程是这样的:

1. 检查system call number, 从而告诉kernel是哪个服务被请求
2. 查看sys_call_table获得对应kernel function的地址
3. 调用查找到的kernel function
4. 等待调用的kernel function返回
5. 做一些system check,然后返回到process当中
6. 如果process时间用完了,会返回到别的process.
这一部分代码在`arch/$(architecture)/kernel/entry.S`中`ENTRY(system_call)`的位置.

如果我想改变一个system call的行为, 假设这个system call的符号是`system_call_1`, 对应的kernel function符号是`system_call_1_cb`. 我需要自己实现一个函数`my_system_call_1_cb`(通常只用在`my_system_call_1_cb`里写一点点定制的代码,然后在末尾调用`system_call_1_cb`), 然后在`sys_call_table`中存的`system_call_1_cb`的指针换成`my_system_call_1_cb`就可以了. 
**注意**:如果我在module里覆盖了一个system call, 当这个module加载进kernel然后再卸载出kernel之后, `sys_call_table`中被修改的指针没有改回去, 务必记得在`cleanup_module`中将`sys_call_table`恢复到原来的状态.

如何修改`sys_call_table`的内容? 

```bash
# 不是su权限查看不到kernel中的symbol和address信息, address会返回全零
$> cat /proc/kallsyms | grep sys_call_table
0000000000000000 R sys_call_table
0000000000000000 R compat_sys_call_table

$> sudo cat /proc/kallsyms | grep sys_call_table
ffffffebcd190938 R sys_call_table
ffffffebcd196a38 R compat_sys_call_table

```



注意:不要在生产用途中篡改任何system call. 举一个导致系统崩溃的例子, 假设module A实现了`A_openat`用于篡改打开文件的system call `openat`, module B实现了`B_openat`用于篡改`openat`. 如果module A先加载进kernel, module B再加载进kernel, 然后module A先从kernel中卸载, module B再从kernel中卸载. 最终这个system call对应的kernel function pointer指向`A_openat`, 但是module A已经从kernel中卸载, 就变成了一个野指针, 再次调用这个system call就会导致kernel crash.

 





## 阻塞Processes和Threads

### Sleep

当module当前无法回应process的请求的时候,可以将这个process sleep, 直到可以回应的时候再唤醒.

```c
static DECLARE_WAIT_QUEUE_HEAD(waitq);

wait_event_interruptible(waitq,);
wake_up(&waitq);
module_put(THIS_MODULE);
```



### Completions

在一个module内部,有时需要面对多线程执行的先后顺序的问题. Kernel有一种允许timeout和interrupt的实现方式, 就是Completion.

```c
static struct {
  struct completion a_comp;
  struct completion b_comp;
} machine; // 声明machine变量(具有匿名类型)
static int machine_thread_a(void *arg)
{
  completion_all(&machine.a_comp);
  kthread_complete_and_exit(&machine.a_comp, 0);
}
static int machine_thread_b(void *arg)
{
  // 等待completion a_comp完成
  wait_for_completion(&machine.a_comp);
  completion_all(&machine.b_comp);
  kthread_complete_and_exit(&machine.b_comp, 0);
}
static int mod_use_completion_init(void)
{
  // 声明两个task
	struct task_struct *thread_a;
  struct task_struct *thread_b;
  pr_info("example of completion\n");
  // 初始化completion
  init_completion(&machine.a_comp);
  init_completion(&machine.b_comp);
  // 创建thread_a
  thread_a = kthread_create(machine_thread_a, NULL, "KThread a");
  if (IS_ERR(thread_a))
  {
    goto ERROR_THREAD_A;
  }
  // 创建thread_b
  thread_b = kthread_create(machine_thread_b, NULL, "KThread b");
  if (IS_ERR(thread_b))
  {
    goto ERROR_THREAD_B;
  }
  // 
  wake_up_process(thread_a);
  wake_up_process(thread_b);
  return 0;
ERROR_THREAD_B:
  kthread_stop(thread_a);
ERROR_THREAD_A:
  return -1;
}
static void mod_use_completion_exit(void)
{
  wait_for_completion(&machine.a_comp);
  wait_for_completion(&machine.b_comp);
}
module_init(mod_use_completion_init);
module_exit(mod_use_completion_exit);
```

此外还有一些基于`wait_for_completion`函数而来的变体函数(可以设置timeout或者可以设置interrupt). 一般来说`wait_for_completion`足以满足绝大部分场景.





## 避免Collision和Deadlock

1. [Mutex](#mutex)
2. [Spinlocks](#spinlocks)
3. [Read and write locks](#read-and-write-locks)
4. [Atomic operations](#atomic-operations)

### Mutex

```C
static DEFINE_MUTEX(my_mutex);
mutex_trylock(&my_mutex);
mutex_is_locked(&my_mutex);
mutex_unlock(&my_mutex);
```

### Spinlocks

自旋锁会锁住代码正在运行的CPU, 并占用其100%的资源. 最好只在预期运行时间不超过几毫秒, 并且用户看不出任何事物明显变慢的代码上使用.

### Read and write locks

读写锁是一种特化的自旋锁. 

```C
static DEFINE_RWLOCK(my_rwlock);
unsigned long flags;
read_lock_irqsave(&my_rwlock, flags);
read_unlock_irqrestore(&my_rwlock, flags);
write_lock_irqsave(&my_rwlock, flags);
write_unlock_irqrestore(&my_rwlock, flags);
```

### Atomic operations



## 练习: Replacing Print Macros

`printk.h`中定义的Print Macros将字符串发送到日志当中, 当是有时, 我想让模块将字符串发送到执行了加载这个模块的命令的tty(teletype, text stream abstraction of )当中.

```C
static void print_string(char *str)
{
	/* The tty for the current task */
	struct tty_struct *my_tty = get_current_tty();

	/* If my_tty is NULL, the current task has no tty you can print to (i.e.,
	* if it is a daemon). If so, there is nothing we can do.
	*/
	if (my_tty) {
		const struct tty_operations *ttyops = my_tty->driver->ops;
    /* my_tty->driver is a struct which holds the tty's functions,
    * one of which (write) is used to write strings to the tty.
    * It can be used to take a string either from the user's or
    * kernel's memory segment.
    *
    * The function's 1st parameter is the tty to write to, because the
    * same function would normally be used for all tty's of a certain
    * type.
    * The 2nd parameter is a pointer to a string.
    * The 3rd parameter is the length of the string.
    *
    * As you will see below, sometimes it's necessary to use
    * preprocessor stuff to create code that works for different
    * kernel versions. The (naive) approach we've taken here does not
    * scale well. The right way to deal with this is described in
    * section 2 of
    * linux/Documentation/SubmittingPatches
    */
    (ttyops->write)(my_tty, str, strlen(str));
    
    /* ttys were originally hardware devices, which (usually) strictly
    * followed the ASCII standard. In ASCII, to move to a new line you
    * need two characters, a carriage return(回车) and a line feed(换行). 
    * On Unix, the ASCII line feed is used for both purposes - so we can not
    * just use \n, because it would not have a carriage return and the
    * next line will start at the column right after the line feed.
    *
    * This is why text files are different between Unix and MS Windows.
    * In CP/M and derivatives, like MS-DOS and MS Windows, the ASCII
    * standard was strictly adhered to, and therefore a newline requires
    * both a LF and a CR.
    */
    (ttyops->write)(my_tty, "\015\012", 2);
	}
}
```

## 练习: Flashing keyboard LEDs

```c
#include <linux/init.h>
#include <linux/kd.h> /* For KDSETLED, */
#include <linux/module.h>
#include <linux/tty.h> /* For tty_struct */
#include <linux/vt.h> /* For MAX_NR_CONSOLES, the max number of virtual console supported by kerntl */
#include <linux/vt_kern.h> /* for fg_console, fg_console is the current virtual console */
#include <linux/console_struct.h> /* For vc_cons, the list of virtual consoles */

static struct tty_driver *this_tty_driver;
static struct timer_list this_timer;
#define BLINK_DELAY (HZ / 5)
#define ALL_LED_ON 0x07
#define RESTORE_LEDS 0xFF
static void timer_fn(struct timer_list *unused)
{
  struct tty_struct *t = vc_cons[fg_console].d->port.tty;
  if (kbledstatus == ALL_LEDS_ON){
    kbledstatus = RESTORE_LEDS;
  }else{
    kbledstatus = ALL_LEDS_ON;
  }
  (this_tty_driver->ops->ioctl)(t, KDSETLED, kbledstatus);
  this_timer.expires = jiffies + BLINK_DELAY;
  add_timer(&this_timer);
}
static int __init kbleds_init(void)
{
  int i;
  pr_info("kbleds: scanning consoles...\n");
  pr_info("kbleds: fg_console %x\n", fg_console);
  for (int i = 0; i < MAX_NR_CONSOLES; i++) {
    if (NULL == vc_cons[i].d){
      break;
    }
    pr_info("poet_atkm: console[%i/%i] #%i, tty %p\n", i, MAX_NR_CONSOLES, vc_cons[i].d->vc_num, (void *)vc_cons[i].d->port.tty);
  }
  pr_info("kbleds: scan consoles ends");
  this_tty_driver = vc_cons[fg_console].d->port.tty->driver;
  pr_info("kbleds: tty driver magic %x\n", this_tty_driver->magic);
  
  timer_setup(&this_timer, timer_fn, 0);
  this_timer.expires = jiffies + BLINK_DELAY;
  add_timer(&this_timer);
  return 0;
}
static int __exit kbleds_exit(void)
{
  pr_info("kbleds: exit\n");
  del_timer(&this_timer);
  (this_tty_driver->ops->ioctl)(vc_cons[fg_console].d->port.tty, KDSETLED, RESTORE_LEDS);
}
```

## Task Schedule
1. [Tasklets](#tasklets)
2. [Work queues](#work-queues)

### Tasklets

```C
static void my_task_fn(unsigned long data)
{
  pr_info("example tasklet starts\n");
  mdelay(5000);
  pr_info("example tasklet ends");
}
static DECLEARE_TASKLET(my_task, my_task_fn, 0L);
static int example_tasklet_init(void)
{
  pr_info("tasklet example init\n");
  tasklet_schedule(&my_task);
  mdelay(2000);
  pr_info("example tasklet continues...\n");
  return 0;
}
static void example_tasklet_exit(void)
{
  pr_info("tasklet example exit\n");
  tasklet_kell(&my_task);
}
```

tasklet的callback不能sleep, 并且不能访问user space data. 因为tasklet的callback运行atomic context中运行,在software interrupt里. 

此外, kernel只允许任意时间给定tasklet只有一个instance在运行,多个不同tasklet的callback可以并行运行.

tasklet可以替换成workqueue、timer、threaded interrupt. 

kernel正在移除tasklet, 但目前保留了tasklet兼容.

### Work queues

一个Work queue中的work由Completely Fair Scheduler(CFS)方式调度.

```C
static struct workqueue_struct *queue = NULL;
static struct work_struct work;
static void work_handler(struct work_struct *data)
{
  pr_info("work handler function.\n")
}
static int __init sched_init(void)
{
  queue = alloc_workqueue("Hello World", WQ_UNBOUND, 1);
  INIT_WORK(&work, work_handler);
  schedule_work(&work);
  return 0;
}
static void __exit sched_exit(void)
{
  destory_workqueue(queue);
}
```

大而全的万金油多线程实现.

## Interrupt Handler

CPU和其他硬件的交互有两个类型,一种是CPU主动发送命令给硬件,另一种是硬件需要告诉CPU一些东西. 第二种称为interrupt. Interrupt类型的交互更难实现,因为硬件通常RAM数量很少, 必须在硬件方便而非CPU方便的情况下完成处理, 否则当一条信息可访问的时候CPU没有读进来, 这条信息就丢失了.

Linux的硬件interrupt称为IRQ. IRQ有short和long两种. short IRQ预计花费非常短的时间, 在此期间机器的其余部分将被阻塞,并且不会处理其他中断. long IRQ预计花费更长的时间, 在此期间其他中断可能发生(除了来自同一设备的中断). 如何可能的话, 尽量将中断处理程序声明为long IRQ. 

当CPU收到一个中断, 它停止正在进行的程序(除非正在处理优先级更高的中断), 将某些参数保存在堆栈上, 然后调用中断处理程序. 这意味着在中断处理程序内部, 某些事情是不被允许的, 因为当前系统处于一种未知的状态. Linux通过将中断处理过程分为两部分来解决这个问题. 第一部分立刻执行并屏蔽中断线. 硬件中断必须快速处理, 第二部分处理被推迟的繁重工作. 例如Softirq. 

有关IRQ的更详细内容请参考"APCI". 

## 练习: Detecting Button Presses



## Crypto
### SHA256
```C
#define SHA256_LENGTH 256

char *plaintext="this is a test";
char hash_sha256[SHA256_LENGTH];

struct crypto_shash *sha256;
struct shash_desc *shash;
sha256=crypto_alloc_shash("sha256",0,0);
shash=kmalloc(sizeof(struct shash_desc)+crypto_shash_descsize(sha256), GFP_KERNEL);
shash->tfm=sha256;
crypto_shash_init(shash);
crypto_shash_update(shash, plaintex, strlen(plaintext));
crypto_shash_final(shash,hash_sha256);
// print plaintext "plaintex" and hash result shash.
kfree(shash);
crypto_free_shash(sha256);

```

### AES256

这个更复杂一些, AES变种太多, 初始化过程很漫长.

```C
#define SYMMETRIC_KEY_LENGTH 32
#define CIPHER_BLOCK_SIZE 16

```



## Virtual Input Output Device Driver
通过Event的方式与Device交互

```C
int send(struct vinput *, char *, int);
int read(struct vinput *, char *, int);
```



## The Device Model
以上介绍了各种各样的模块做各种各样的事, 但是缺少一致性. 一致性要由我自己定义的, 控制设备启动挂起结束等等功能的一致性模型称为设备模型. 以下是一个样例:

```C
#include <linux/kernel>
#include <linux/module.h>
#include <linux/platform_device.h>

struct dm_data{
  char *greeting;
  int number;
};
static int dm_probe(struct platform_driver *dev)
{
  struct dm_da *pa = (struct dm_data*)(dev->dev.platform_data);
  pr_info("dm probe\n");
  pr_info("dm greeting: %s; %d\n", pd->greeting, pd->number);
  /* Your device initialization code */
  return 0;
}
static int dm_remove(struct platform_driver *dev)
{
  pr_info("dm example removed\n");
  /* Your device removal code */
  return 0;
}
static int dm_suspend(struct device *dev)
{
  pr_info("dm example suspend\n");
  /* Your device suspend code */
  return 0;
}
static int dm_resume(struct device *dev)
{
  pr_info("dm example resume\n");
  /* Your device resume code */
  return 0;
}
static const struct dev_pm_ops dm_pm_ops = {
  .suspend = dm_suspend,
  .resume = dm_resume,
  .poweroff = dm_suspend,
  .freeze = dm_suspend,
  .thaw = dm_resume,
  .restore = dm_resume,
};
static struct platform_driver dm_driver = {
  .driver = {
    .name = "dm_example",
    .pm = &dm_pm_ops,
  },
  .probe = dm_probe,
  .remove = dm_remove,
};
static int dm_init(void)
{
  int ret;
  pr_info("device model init\n");
  ret = platform_driver_register(&dm_driver);
  if (ret){
    pr_err("Unable to register driver\n");
    return ret;
  }
  return 0;
}
static int dm_exit(void)
{
  pr_info("dm exit\n");
  platform_driver_unregister(&dm_driver);
}
module_init(dm_init);
module_exit(dm_exit);
```



## Optimizations

注意:如果计算性能不是瓶颈, 就不要优化.

### Likely and Unlikely conditions

```C
// I know allocating memory always expecting succeed.
bvl = bvec_alloc(gfp_mask, nr_iovecs, &idx);
if (unlikely(!bvl)){
  mempool_free(bio, bio_pool);
  bio = NULL;
  goto out;
}

```

编译器会改变生成的机器码, 当使用unlikely的时候, 机器会直接执行false分支, 当condition是true的时候, 才会发生一次跳转. 显然, 正确的情况下避免了一次分支跳转, 错误的情况下延迟也会比分支跳转高.

### Static Keys

How to enable:

1. gcc support `asm goto` inline assembly.
2. `CONFIG_JUMP_LABEL=y`
3. `CONFIG_HAVE_ARCH_JUMP_LABEL=y`
4. `CONFIG_HAVE_ARCH_JUMP_LABEL_RELATIVE=y`

```C
// API
DEFINE_STATIC_KEY_FALSE(x);
DEFINE_STATIC_KEY_TRUE(x);
DEFINE_STATIC_KEY_FALSE_RO(x);// read only
DEFINE_STATIC_KEY_TRUE_RO(x); // read only
// 在一些情况下, 一个static key在module init阶段disable(或enable)之后就不会再改变, 这种情况可以声明为只读. 这时, 这个static key就只能在init阶段改值. 在运行时修改一个只读的static key的值会导致一个page fault.
```



```C
DEFINE_STATIC_KEY_FALSE(fkey);
pr_info("fastpath 1\n");
if (static_branch_unlikely(&fkey)){
  pr_info("slowpath\n");
}
pr_info("fastpath 2\n");
```

以这个例子, fkey是static key, 并且值为false. 那么上述代码在fastpath 1之后会直接进入fastpath 2, 不会再做分支检查(应该是通过`asm goto`来实现的). 当fkey被修改为true时, 上述代码则一定会做分支检查.



## Common Pitfalls

### Using standard libraries

You can not do that.

在kernel module中, 我只能使用kernel function. 所有的kernel function都可以在`/proc/kallsyms`中看到.

### Disabling interrupts

我可以禁用一小小会儿, 问题不大. 但是在禁用之后必须记得启用, 否则系统会卡死哦, 那就只能重启了.

## Where To Go From Here?

1. Kernel newbies.org
2. Documentation subdirectory within the kernel source code. (It is not always easy to understand, but is a good starting point for further investigation)
3. Linus said, the best way to learn the kernel is to read the source code yourself.
