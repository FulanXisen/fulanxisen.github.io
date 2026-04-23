---
title: "A Guide to Using ARM Stack Limit Registers"
date: 2023-05-03T14:28:46+08:00
# weight: 1
# aliases: ["/first"]
tags: ["ARM", "Stack Over Flow"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Desc Text."
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
A Guide to Using ARM Stack Limit Registers[](#a-guide-to-using-arm-stack-limit-registers)
-----------------------------------------------------------------------------------------

14 Feb 2023 by [Jon Kurtz](/authors/jonkurtz)

堆栈溢出一直是开发过程中的一个严重问题。它们经常没有被检测到，并以难以理解的方式出现。我们已经实施了软件机制来保护它们，但这些机制有限，并且仍不能保护所有条件。

随着ARM架构的成熟，使用一个百分百可靠的机制来检测溢出难道不是更好吗？

我们将探索在ARM Cortex-M33架构上使用MSP Limit和PSP Limit寄存器来检测堆栈溢出。我们将在Renesas DA1469x上进行实现，并查看检测堆栈溢出的实际示例。此外，我们还将查看MSPLIM和PSPLIM功能不足的情况下的补充选项。


Table of Contents[](#table-of-contents)
---------------------------------------

*   [Basic Terminology](#basic-terminology)
*   [How does it work?](#how-does-it-work)
*   [Implementing the Limit Registers](#implementing-the-limit-registers)
    *   [Initializing the MSPLIM Register](#initializing-the-msplim-register)
    *   [Initializing the PSPLIM Register](#initializing-the-psplim-register)
    *   [Setting up the UsageFault\_Handler](#setting-up-the-usagefault_handler)
    *   [Testing our Implementation](#testing-our-implementation)
*   [Limitations and Further Improvements](#limitations-and-further-improvements)
    *   [Stack Canary](#stack-canary)
    *   [FreeRTOS Buffer Overflow protection](#freertos-buffer-overflow-protection)
    *   [Compiler Enabled Overflow Detection](#compiler-enabled-overflow-detection)
    *   [GCC SSP Example](#gcc-ssp-example)
*   [Practical implementations for GCC stack Canaries](#practical-implementations-for-gcc-stack-canaries)
*   [Closing](#closing)
*   [References](#references)

Basic Terminology[](#basic-terminology)
---------------------------------------

The ARM Cortex-M33 introduced two new stack limit registers, PSPLIM and MSPLIM [1](#fn:m33-psplim_msplim). ARM has included this in its ARMv8 specification, so any processors before this will not have support.

How does it work?[](#how-does-it-work)
--------------------------------------

这个新特性最妙的地方是它使用起来非常简单，而且可以避免在调试堆栈溢出时进行猜测。

We need to set the PSPLIM, and the MSPLIM registers to the boundary of the stack. 
If the MSP == MSPLIM register or the PSP == PSPLIM register, a UsageFault is generated. The UsageFault Status Register [2](#fn:m33-usfr) contains a sticky bit in position four to indicate that a stack overflow occurred.

为PSP和MSP提供硬件保护允许操作系统内的灵活性。例如，我们可以在异常和中断期间保护MSP。我们还可以在上下文切换时切换PSPLIM值，以保护每个Task的堆栈。如果你需要关于上下文切换的复习，可以在[这里](2019-10-30-cortex-m-rtos-context-switching.md)查看之前的文章。


If no RTOS is present, we can still monitor the MSP, as this will protect your whole application.

Implementing the Limit Registers[](#implementing-the-limit-registers)
---------------------------------------------------------------------

For an implementation example on the DA1469x, you can access this at Renesas’s Github [3](#fn:ARMV8_Guards).

### Initializing the MSPLIM Register[](#initializing-the-msplim-register)

We use the MSR instruction to write to these registers, which requires us to be in privileged mode. In this case, we will set the MSP Limit in the Reset\_Handler:

    Reset_Handler:
    
            ldr     r0, =__StackLimit
            add     r0, r0, #dg_configMSP_PADDING
            msr     MSPLIM, r0
    

Specifically to the DA1469x, we also need to place the same initialization in the Wakeup\_Reset\_Handler:

    Wakeup_Reset_Handler:
    
        ldr     r0, =__StackLimit
        add     r0, r0, #dg_configMSP_PADDING
        msr     MSPLIM, r0
    
    
DA1469x休眠架构并不适用于所有Cortex-M33架构。初始化后，DA1469x用Wakeup\_Reset\_Handler替换Reset\_Handler来处理Cortex-M33寄存器的恢复。


There are two definitions provided elsewhere in the project. \_\_StackLimit is defined in vector\_table\_da1469x.S:
```s
    .section .stack
                    .align  3
                    .globl  __StackTop
                    .globl  __StackLimit
    __StackLimit:
                    .space  Stack_Size
                    .size   __StackLimit, . - __StackLimit
    __StackTop:
                    .size   __StackTop, . - __StackTop
```

This definition helps us find the stack limit for setting the MSP.

We also added padding to this value. You will find this value in a configuration file:
```c
    #define dg_configMSP_PADDING                    (16)
```    

When the MSPLIM is equal to the MSP, the UsageFault exception is triggered. The padding is required to enable pushing items to the stack on Exception entry. If we don’t make space for the fault handler, nested exceptions can occur as the MSPLIM register would be continuously exceeded, usually resulting in a LOCKUP.

The alternative would be to use a naked function[4](#fn:gcc_attributes). However, I prefer to add padding as it provides more flexibility in the fault handler and allows for Memfault hooks!

### Initializing the PSPLIM Register[](#initializing-the-psplim-register)

On the DA1469x SDK, it makes use of FreeRTOS. The psp is used for each task’s stack so we can set up the PSPLIM register to protect against a task overflow. This implementation is superior to FreeRTOS’s stack overflow check[5](#fn:1) for the following reasons:

1.  FreeRTOS only checks the watermark on a context switch. Therefore, if a thread overflows the stack and isn’t yielding, it can corrupt memory, access null pointers, etc.
    
2.  FreeRTOS does not recommend using this feature in production environments because of the context switch overhead[6](#fn:2).
    

Implementing this is more straightforward. First, we must adjust the PSPLIM during a context switch in FreeRTOS.

In tasks.c we create the following above vTaskSwitchContext:
```c
    void vTaskSwitchStackGuard(void)
    {
        volatile uint32_t end_of_stack_val = (uint32_t)pxCurrentTCB->pxStack;
         __set_PSPLIM( end_of_stack_val);
    }
```    

Next, we add the call immediately after the context switch:
```c
    void xPortPendSVHandler( void ){
    
    ...
    
    "   bl vTaskSwitchContext               \n"
    "   bl vTaskSwitchStackGuard                        \n"
    
    ...
    }
```    

That’s all we need to do for the PSP.

### Setting up the UsageFault\_Handler[](#setting-up-the-usagefault_handler)

All that’s left is doing some work in the UsageFault\_Handler. We will declare the UsageFault\_Handler in exceptions\_handler.s and call a separate handler afterward.

First, we declare an application handler:
```c
    __RETAINED_CODE void UsageFault_HandlerC(uint8_t stack_pointer_mask)
    {
    
        volatile uint16_t usage_fault_status_reg __UNUSED;
    
        usage_fault_status_reg = (SCB->CFSR &         SCB_CFSR_USGFAULTSR_Msk) >> SCB_CFSR_USGFAULTSR_Pos;
    
        hw_watchdog_freeze();
    
        if(usage_fault_status_reg & (SCB_CFSR_STKOF_Msk >> SCB_CFSR_USGFAULTSR_Pos))
        {
    
            while(1){}
        }
    
            while (1) {}
    }
```    
    

Next, let’s add our UsageFault\_Handler into the exceptions\_handler.S:
```s
    #if (dg_configCODE_LOCATION == NON_VOLATILE_IS_FLASH)
                .section text_retained
    #endif
        .align  2
        .thumb
        .thumb_func
        .globl  UsageFault_Handler
        .type   UsageFault_Handler, %function
    UsageFault_Handler:
        ldr     r2,=UsageFault_HandlerC
        mrs     r1, msp
        mrs     r0, MSPLIM
        cmp     r0, r1
        beq     UsageFault_with_MSP_Overflow
        mrs     r1, psp
        mrs     r0, PSPLIM
        cmp     r0, r1
        beq     UsageFault_with_PSP_Overflow
        mov     r0, #0
        bx      r2
    UsageFault_with_PSP_Overflow:
        mov     r0, #2
        bx      r2
    UsageFault_with_MSP_Overflow:
        ldr     r1, =__StackLimit
        msr     MSPLIM, r1
        mov     r0, #1
        bx      r2
```    

Since the USFR does not indicate if the psp or the msp caused the fault, I decided to add some detection in assembly. I prefer doing this in assembly to ensure no stack pushes before the application handler call.

*   0 - General UsageFault
*   1 - MSP Overflow
*   2 - PSP Overflow (Task Overflow)

In this function, we are checking the MSP and the PSP registers against the limit registers. If the MSP matches the MSPLIM register, we restore the MSPLIM to `__StackLimit` (Removing the padding we placed initially) and then call our application fault handler.

### Testing our Implementation[](#testing-our-implementation)

We need a small piece of code to test the implementation. In our example, there is a macro provided for causing an overflow for the MSP or the PSP:
```c
    #define TOGGLE_MSP_OVERFLOW (0)     //0 Creates an application overflow in FreeRTOS task, 1 creates it on the MSP
```    

When the button is pressed, depending on this macro setting, it calls a recursive function either in interrupt context or in our main task.
```c
    static void test_overflow_func(void)
    {
            test_overflow_func();
    }
    
    static void _wkup_key_cb(void)
    {
        BaseType_t need_switch;
        /* Clear the WKUP interrupt flag!!! */
        hw_wkup_reset_interrupt();
    
    #if TOGGLE_MSP_OVERFLOW > 0
        test_overflow_func();
    #endif
    
    
        xTaskNotifyFromISR(overFlow_handle, BUTTON_PRESS_NOTIF, eSetBits, &need_switch);
        portEND_SWITCHING_ISR(need_switch);
    }
    
    ...
    
    void prvTestOverFlowTask( void *pvParameters )
    {
        _wkup_init();
    
        overFlow_handle = xTaskGetCurrentTaskHandle();
    
        for ( ;; ) {
            uint32_t notif;
            /*
            * Wait on any of the notification bits, then clear them all
            */
            xTaskNotifyWait(0, 0xFFFFFFFF, &notif, portMAX_DELAY);
    
            /* Notified from BLE manager? */
            if (notif & BUTTON_PRESS_NOTIF) {
                    test_overflow_func();
            }
        }
    }
```    

After pressing the button, we should see the UsageFault\_HandlerC get called in our application code.

Limitations and Further Improvements[](#limitations-and-further-improvements)
-----------------------------------------------------------------------------

The MSPLIM and PSPLIM registers will help against most stack overflows. Unfortunately, they do not protect us from local buffers corrupting the stack. We will look at the most common; buffer overflow. A buffer overflow occurs when a fixed buffer is allocated on the stack, and the program starts writing to memory addresses outside this boundary. This results in corrupted data and can even change the return address of a function, causing undesired execution of application code.

There are different ways to handle this condition on other architectures. For example, Zephyr uses the MPU to guard the PSP on each thread. Here, we will discuss stack canaries.

### Stack Canary[](#stack-canary)

Stack Canaries are widely implemented as a means of code hardening. A function will place a value (canary) on the end of a stack frame and will check the value is intact before it returns. This mechanism protects against buffer overflow attacks, where malicious source code could overflow the buffer to redirect the return address to its function.

This same idea can also be used to guard against buffer overflows in our application.

### FreeRTOS Buffer Overflow protection[](#freertos-buffer-overflow-protection)

FreeRTOS implements a means for overflow detection, as discussed in [Initializing the PSPLIM Register](#initializing-the-psplim-register). This uses the concept of a canary, which will periodically check the value during a context switch.

FreeRTOS has two different configurations that follow this concept:
```c
    #if( ( configCHECK_FOR_STACK_OVERFLOW > 1 ) && ( portSTACK_GROWTH < 0 ) )
    
        #define taskCHECK_FOR_STACK_OVERFLOW()                                                              \
        {                                                                                                   \
            const uint32_t * const pulStack = ( uint32_t * ) pxCurrentTCB->pxStack;                         \
            const uint32_t ulCheckValue = ( uint32_t ) 0xa5a5a5a5;                                          \
                                                                                                            \
            if( ( pulStack[ 0 ] != ulCheckValue ) ||                                                \
                ( pulStack[ 1 ] != ulCheckValue ) ||                                                \
                ( pulStack[ 2 ] != ulCheckValue ) ||                                                \
                ( pulStack[ 3 ] != ulCheckValue ) )                                             \
            {                                                                                               \
                vApplicationStackOverflowHook( ( TaskHandle_t ) pxCurrentTCB, pxCurrentTCB->pcTaskName );   \
            }                                                                                               \
        }
    
    #endif /* #if( configCHECK_FOR_STACK_OVERFLOW > 1 ) */
    /*-----------------------------------------------------------*/
    
    #if( ( configCHECK_FOR_STACK_OVERFLOW > 1 ) && ( portSTACK_GROWTH > 0 ) )
    
        #define taskCHECK_FOR_STACK_OVERFLOW()                                                                                              \
        {                                                                                                                                   \
        int8_t *pcEndOfStack = ( int8_t * ) pxCurrentTCB->pxEndOfStack;                                                                     \
        static const uint8_t ucExpectedStackBytes[] = { tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE,     \
                                                        tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE,     \
                                                        tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE,     \
                                                        tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE,     \
                                                        tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE, tskSTACK_FILL_BYTE };   \
                                                                                                                                            \
                                                                                                                                            \
            pcEndOfStack -= sizeof( ucExpectedStackBytes );                                                                                 \
                                                                                                                                            \
            /* Has the extremity of the task stack ever been written over? */                                                               \
            if( memcmp( ( void * ) pcEndOfStack, ( void * ) ucExpectedStackBytes, sizeof( ucExpectedStackBytes ) ) != 0 )                   \
            {                                                                                                                               \
                vApplicationStackOverflowHook( ( TaskHandle_t ) pxCurrentTCB, pxCurrentTCB->pcTaskName );                                   \
            }                                                                                                                               \
        }
    
    #endif /* #if( configCHECK_FOR_STACK_OVERFLOW > 1 ) */
```    

Both methods check for an expected value at the end of the stack. If this value is overwritten, then vApplicationStackOverflowHook is called, and the application should record and reset. Unfortunately, the periodicity is non-deterministic, as it relies on a context switch. Periodic checks lead to a race condition when a task doesn’t yield in time. You can test this from the previous example by setting the following:

    #define dg_configARMV8_USE_STACK_GUARDS         (0)
    #define #define TOGGLE_MSP_OVERFLOW             (0)
    

In this example, prvTestOverFlowTask will not yield, so FreeRTOS does not catch this condition.

### Compiler Enabled Overflow Detection[](#compiler-enabled-overflow-detection)

Compilers have started enabling SSP (Stack Smashing Protection) libraries. The library options will allow the compiler to use canaries within function calls. We’re going to look at GCC’s implementation[7](#fn:5) specifically. GCC provides the following compiler flags:

*   **\-fstack-protector**: This includes functions that call **alloca** and functions with buffers larger than or equal to 8 bytes. The guards are initialized when a function is entered and then checked when the function exits.
    
*   **\-fstack-protector-strong** - Like -fstack-protector but includes additional functions to be protected — those that have local array definitions, or have references to local frame addresses.
    
*   **\-fstack-protector-all**: all functions are protected.
    
*   **\-fstack-protector-explicit**: protects those functions which have the stack\_protect attribute
    

### GCC SSP Example[](#gcc-ssp-example)

Let’s take a look at using the ssp library in GCC. First, let’s add the compiler flag from the previous example: -fstack-protector. The ssp library has two externs that we define as follows.

    uint32_t__stack_chk_guard = 0xDEADBEEF;
    
    void __stack_chk_fail(void) { /* will be called if guard/canary gets corrupted */
    
        __ASM volatile ("cpsid i" : : : "memory");
    
        hw_watchdog_freeze();                           // Stop WDOG
        while(1){}
    }
    

Let’s also add the vulnerability in our application and add it to the prvTestOverFlowTask:

    __attribute__((optimize("O0"))) static uint8_t stack_buffer_test(uint16_t iters)
    {
        uint8_t buffer[16];
        uint16_t i;
    
        for(i = 0; i < iters; i++)
        {
                buffer[i] = 0xaa;
        }
    
        return buffer[8];
    }
    
    

> **NOTE:** \_\_stack\_chk\_guard should be randomized on startup if using ssp for security reasons.

This function has a fixed buffer to pass a value larger than the local buffer. Let’s add a stack\_buffer\_test(17) to our task and look at the assembly.

    (gdb) disassemble stack_buffer_test
    Dump of assembler code for function stack_buffer_test:
       0x0000ccc8 <+0>: push    {r7, lr}
       0x0000ccca <+2>: sub sp, #32
       0x0000cccc <+4>: add r7, sp, #0
       0x0000ccce <+6>: mov r3, r0
       0x0000ccd0 <+8>: strh    r3, [r7, #6]
       0x0000ccd2 <+10>:    ldr r3, [pc, #68]   ; (0xcd18 <stack_buffer_test+80>)
       0x0000ccd4 <+12>:    ldr r3, [r3, #0]
       0x0000ccd6 <+14>:    str r3, [r7, #28]
       0x0000ccd8 <+16>:    mov.w   r3, #0
       0x0000ccdc <+20>:    movs    r3, #0
       0x0000ccde <+22>:    strh    r3, [r7, #10]
       0x0000cce0 <+24>:    b.n 0xccf4 <stack_buffer_test+44>
       0x0000cce2 <+26>:    ldrh    r3, [r7, #10]
       0x0000cce4 <+28>:    adds    r3, #32
       0x0000cce6 <+30>:    add r3, r7
       0x0000cce8 <+32>:    movs    r2, #170    ; 0xaa
       0x0000ccea <+34>:    strb.w  r2, [r3, #-20]
       0x0000ccee <+38>:    ldrh    r3, [r7, #10]
       0x0000ccf0 <+40>:    adds    r3, #1
       0x0000ccf2 <+42>:    strh    r3, [r7, #10]
       0x0000ccf4 <+44>:    ldrh    r2, [r7, #10]
       0x0000ccf6 <+46>:    ldrh    r3, [r7, #6]
       0x0000ccf8 <+48>:    cmp r2, r3
       0x0000ccfa <+50>:    bcc.n   0xcce2 <stack_buffer_test+26>
       0x0000ccfc <+52>:    ldrb    r3, [r7, #20]
       0x0000ccfe <+54>:    ldr r2, [pc, #24]   ; (0xcd18 <stack_buffer_test+80>)
       0x0000cd00 <+56>:    ldr r1, [r2, #0]
       0x0000cd02 <+58>:    ldr r2, [r7, #28]
       0x0000cd04 <+60>:    eors    r1, r2
       0x0000cd06 <+62>:    mov.w   r2, #0
       0x0000cd0a <+66>:    beq.n   0xcd10 <stack_buffer_test+72>
       0x0000cd0c <+68>:    bl  0xcdb0 <__stack_chk_fail>
       0x0000cd10 <+72>:    mov r0, r3
       0x0000cd12 <+74>:    adds    r7, #32
       0x0000cd14 <+76>:    mov sp, r7
       0x0000cd16 <+78>:    pop {r7, pc}
       0x0000cd18 <+80>:    strh    r4, [r6, #44]   ; 0x2c
       0x0000cd1a <+82>:    movs    r0, #0
    
    

Here we can see the compiler loading the canary at the end of the stack frame:

       0x0000ccd2 <+10>:    ldr r3, [pc, #68]   ; (0xcd18 <stack_buffer_test+80>)
       0x0000ccd4 <+12>:    ldr r3, [r3, #0]
       0x0000ccd6 <+14>:    str r3, [r7, #28]
    
    (gdb) x /1a 0xcd18
    0xcd18 <stack_buffer_test+80>:  0x200085b4 <__stack_chk_guard>
    (gdb) x /1a 0x200085b4
    0x200085b4 <__stack_chk_guard>: 0xdeadbeef
    

Before return, we can see the function checking the canary at the end of the stack frame and calling \_\_stack\_chk\_fail if the value is corrupted:

    0x0000cd0a <+66>:   beq.n   0xcd10 <stack_buffer_test+72>
    0x0000cd0c <+68>:   bl  0xcdb0 <__stack_chk_fail>
    

Running the rest of the example should confirm the call of \_\_stack\_chk\_fail.

Practical implementations for GCC stack Canaries[](#practical-implementations-for-gcc-stack-canaries)
-----------------------------------------------------------------------------------------------------

Implementing the ssp library does provide additional overhead in execution time and code space. A function will add 7 additional instructions to make use of this feature. The developer should weigh these factors when choosing which setting to use in GCC.

My preference would be to develop and test with a stricter setting and more coverage and move to a more relaxed setting when getting closer to production. For example, you could start your development process with -fstack-protector-all, and later relax this to -fstack-protector-strong or -fstack-protector as the code matures.

Closing[](#closing)
-------------------

The PSPLIM and the MSPLIM registers are great new features from ARM and a much-needed addition to the architecture. These can also be supplemented with other techniques to fortify your application. We hope you found this helpful, and will be inspired to make use of it in your application. Implementing these features should prevent many development headaches and safeguard your application in the field!

References[](#references)
-------------------------

1.  [Cortex M33 MSPLIM PSPLIM TRM](https://developer.arm.com/documentation/100231/0002/) [↩](#fnref:m33-psplim_msplim)
    
2.  [Cortex M33 USFR](https://developer.arm.com/documentation/100235/0004/the-cortex-m33-peripherals/system-control-block/configurable-fault-status-register) [↩](#fnref:m33-usfr)
    
3.  [DA1469x Github Example](https://github.com/dialog-semiconductor/BLE_SDK10_examples/tree/main/features/armv8_stack_overflow_guards) [↩](#fnref:ARMV8_Guards)
    
4.  [GCC Attributes](https://gcc.gnu.org/onlinedocs/gcc-4.3.0/gcc/Function-Attributes.html#Function-Attributes) [↩](#fnref:gcc_attributes)
    
5.  [FreeRTOS Kernel Stack Overflow Check](https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/a8a9c3ea3e34c10c6818f654883dac3dbdae12d1/tasks.c#L3052) [↩](#fnref:1)
    
6.  [FreeRTOS Stack Overflow Check](https://www.freertos.org/Stacks-and-stack-overflow-checking.html) [↩](#fnref:2)
    
7.  [GCC Instrumentation Options](https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html) [↩](#fnref:5)
    

![](/img/author/jonkurtz.jpg) [Jon Kurtz](/authors/jonkurtz) is an FAE Connectivity manager at Renesas.  
[](https://www.linkedin.com/in/jonathankurtz1)