---
title: "Embedded System from Zero to Main()"
date: 2023-10-14T16:40:08+08:00
# weight: 1
# aliases: ["/first"]
tags: ["Cortex-M", "Embedded", "Bare matel"]
author: "fanyuxin"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Setup a bare matel C environment, linker scripts, bootloader, libc"
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

Bare metal C
============

Power on
------------

There must be behavior intrinsic to the chip that defines how code is executed.

Digging into the [ARMv6-M Technical Reference Manual](https://static.docs.arm.com/ddi0419/d/DDI0419D_armv6m_arm.pdf), which is the underlying architecture manual for the Cortex-M0+, we can find some pseudo-code that describes reset behavior:

```
// B1.5.5 TakeReset()
// ============
TakeReset()
    VTOR = Zeros(32);
    for i = 0 to 12
        R[i] = bits(32) UNKNOWN;
    bits(32) vectortable = VTOR;
    CurrentMode = Mode_Thread;
    LR = bits(32) UNKNOWN; // Value must be initialised by software
    APSR = bits(32) UNKNOWN; // Flags UNPREDICTABLE from reset
    IPSR<5:0> = Zeros(6); // Exception number cleared at reset
    PRIMASK.PM = '0'; // Priority mask cleared at reset
    CONTROL.SPSEL = '0'; // Current stack is Main
    CONTROL.nPRIV = '0'; // Thread is privileged
    ResetSCSRegs(); // Catch-all function for System Control Space reset
    for i = 0 to 511 // All exceptions Inactive
        ExceptionActive[i] = '0';
    ClearEventRegister(); // See WFE instruction for more information
    SP_main = MemA[vectortable,4] AND 0xFFFFFFFC<31:0>;
    SP_process = ((bits(30) UNKNOWN):'00');
    start = MemA[vectortable+4,4]; // Load address of reset routine
    BLXWritePC(start); // Start execution of reset routine

```

In short, the chip does the following:

- Reset the vector table address to 0x00000000
- Disable all interrupts
- Load the SP from address 0x00000000
- Load the PC from address 0x00000004

PC has value 0x00000004, is our `main` function addressed at 0x00000004?

Let us check, dump our `bin` file to see what address 0x0000000 and 0x00000004 contain:

```shell
$ xxd build/minimal/minimal.bin  | head
00000000: 0020 0020 c100 0000 b500 0000 bb00 0000  . . ............
00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000060: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000070: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000080: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000090: 0000 0000 0000 0000 0000 0000 0000 0000  ................
```

Initial SP is `0x20002000`, and our start address pointer is `0x000000c1`.

Let’s dump our symbols to see which one is at `0x000000c1`.

```shell
$ arm-none-eabi-objdump -t build/minimal.elf | sort
...
000000b4 g     F .text  00000006 NMI_Handler
000000ba g     F .text  00000006 HardFault_Handler
000000c0 g     F .text  00000088 Reset_Handler
00000148 l     F .text  0000005c system_pinmux_get_group_from_gpio_pin
000001a4 l     F .text  00000020 port_get_group_from_gpio_pin
000001c4 l     F .text  00000022 port_get_config_defaults
000001e6 l     F .text  0000004e port_pin_set_output_level
00000234 l     F .text  00000038 port_pin_toggle_output_level
0000026c l     F .text  00000040 set_output
000002ac g     F .text  0000002c main
...
```

看起来很奇怪, Our `main` function is found at `0x000002ac`. No symbol at `0x000000c1`, but a `Reset_Handler` symbol at `0x000000c0`.

事实证明, the lowest bit of the PC is used to indicate thumb2 instructions, which is one of the two instruction sets supported by ARM processors, so `Reset_Handler` is what we’re looking for (for more details check out section A4.1.1 in the ARMv6-M manual).

Write a Reset_Handler
------------

不幸的是，Reset_Handler通常是一堆难以理解的汇编代码。这个贴nRF52 SDK的startup file展示:

```S
/*
 
Copyright (c) 2009-2018 ARM Limited. All rights reserved.

    SPDX-License-Identifier: Apache-2.0

Licensed under the Apache License, Version 2.0 (the License); you may
not use this file except in compliance with the License.
You may obtain a copy of the License at

    www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an AS IS BASIS, WITHOUT
WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

NOTICE: This file has been modified by Nordic Semiconductor ASA.

*/

    .syntax unified
    .arch armv7e-m

#ifdef __STARTUP_CONFIG
#include "startup_config.h"
#ifndef __STARTUP_CONFIG_STACK_ALIGNEMENT
#define __STARTUP_CONFIG_STACK_ALIGNEMENT 3
#endif
#endif

    .section .stack
#if defined(__STARTUP_CONFIG)
    .align __STARTUP_CONFIG_STACK_ALIGNEMENT
    .equ    Stack_Size, __STARTUP_CONFIG_STACK_SIZE
#elif defined(__STACK_SIZE)
    .align 3
    .equ    Stack_Size, __STACK_SIZE
#else
    .align 3
    .equ    Stack_Size, 8192
#endif
    .globl __StackTop
    .globl __StackLimit
__StackLimit:
    .space Stack_Size
    .size __StackLimit, . - __StackLimit
__StackTop:
    .size __StackTop, . - __StackTop

    .section .heap
    .align 3
#if defined(__STARTUP_CONFIG)
    .equ Heap_Size, __STARTUP_CONFIG_HEAP_SIZE
#elif defined(__HEAP_SIZE)
    .equ Heap_Size, __HEAP_SIZE
#else
    .equ Heap_Size, 8192
#endif
    .globl __HeapBase
    .globl __HeapLimit
__HeapBase:
    .if Heap_Size
    .space Heap_Size
    .endif
    .size __HeapBase, . - __HeapBase
__HeapLimit:
    .size __HeapLimit, . - __HeapLimit

    .section .isr_vector
    .align 2
    .globl __isr_vector
__isr_vector:
    .long   __StackTop                  /* Top of Stack */
    .long   Reset_Handler
    .long   NMI_Handler
    .long   HardFault_Handler
    .long   MemoryManagement_Handler
    .long   BusFault_Handler
    .long   UsageFault_Handler
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   SVC_Handler
    .long   DebugMon_Handler
    .long   0                           /*Reserved */
    .long   PendSV_Handler
    .long   SysTick_Handler

  /* External Interrupts */
    .long   POWER_CLOCK_IRQHandler
    .long   RADIO_IRQHandler
    .long   UARTE0_UART0_IRQHandler
    .long   SPIM0_SPIS0_TWIM0_TWIS0_SPI0_TWI0_IRQHandler
    .long   SPIM1_SPIS1_TWIM1_TWIS1_SPI1_TWI1_IRQHandler
    .long   NFCT_IRQHandler
    .long   GPIOTE_IRQHandler
    .long   SAADC_IRQHandler
    .long   TIMER0_IRQHandler
    .long   TIMER1_IRQHandler
    .long   TIMER2_IRQHandler
    .long   RTC0_IRQHandler
    .long   TEMP_IRQHandler
    .long   RNG_IRQHandler
    .long   ECB_IRQHandler
    .long   CCM_AAR_IRQHandler
    .long   WDT_IRQHandler
    .long   RTC1_IRQHandler
    .long   QDEC_IRQHandler
    .long   COMP_LPCOMP_IRQHandler
    .long   SWI0_EGU0_IRQHandler
    .long   SWI1_EGU1_IRQHandler
    .long   SWI2_EGU2_IRQHandler
    .long   SWI3_EGU3_IRQHandler
    .long   SWI4_EGU4_IRQHandler
    .long   SWI5_EGU5_IRQHandler
    .long   TIMER3_IRQHandler
    .long   TIMER4_IRQHandler
    .long   PWM0_IRQHandler
    .long   PDM_IRQHandler
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   MWU_IRQHandler
    .long   PWM1_IRQHandler
    .long   PWM2_IRQHandler
    .long   SPIM2_SPIS2_SPI2_IRQHandler
    .long   RTC2_IRQHandler
    .long   I2S_IRQHandler
    .long   FPU_IRQHandler
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */
    .long   0                           /*Reserved */

    .size __isr_vector, . - __isr_vector

/* Reset Handler */


    .text
    .thumb
    .thumb_func
    .align 1
    .globl Reset_Handler
    .type Reset_Handler, %function
Reset_Handler:


/* Loop to copy data from read only memory to RAM.
 * The ranges of copy from/to are specified by following symbols:
 *      __etext: LMA of start of the section to copy from. Usually end of text
 *      __data_start__: VMA of start of the section to copy to.
 *      __bss_start__: VMA of end of the section to copy to. Normally __data_end__ is used, but by using __bss_start__
 *                    the user can add their own initialized data section before BSS section with the INTERT AFTER command.
 *
 * All addresses must be aligned to 4 bytes boundary.
 */
    ldr r1, =__etext
    ldr r2, =__data_start__
    ldr r3, =__bss_start__

    subs r3, r3, r2
    ble .L_loop1_done

.L_loop1:
    subs r3, r3, #4
    ldr r0, [r1,r3]
    str r0, [r2,r3]
    bgt .L_loop1

.L_loop1_done:

/* This part of work usually is done in C library startup code. Otherwise,
 * define __STARTUP_CLEAR_BSS to enable it in this startup. This section
 * clears the RAM where BSS data is located.
 *
 * The BSS section is specified by following symbols
 *    __bss_start__: start of the BSS section.
 *    __bss_end__: end of the BSS section.
 *
 * All addresses must be aligned to 4 bytes boundary.
 */
#ifdef __STARTUP_CLEAR_BSS
    ldr r1, =__bss_start__
    ldr r2, =__bss_end__

    movs r0, 0

    subs r2, r2, r1
    ble .L_loop3_done

.L_loop3:
    subs r2, r2, #4
    str r0, [r1, r2]
    bgt .L_loop3

.L_loop3_done:
#endif /* __STARTUP_CLEAR_BSS */

/* Execute SystemInit function. */
    bl SystemInit

/* Call _start function provided by libraries.
 * If those libraries are not accessible, define __START as your entry point.
 */
#ifndef __START
#define __START _start
#endif
    bl __START

    .pool
    .size   Reset_Handler,.-Reset_Handler

    .section ".text"


/* Dummy Exception Handlers (infinite loops which can be modified) */

    .weak   NMI_Handler
    .type   NMI_Handler, %function
NMI_Handler:
    b       .
    .size   NMI_Handler, . - NMI_Handler


    .weak   HardFault_Handler
    .type   HardFault_Handler, %function
HardFault_Handler:
    b       .
    .size   HardFault_Handler, . - HardFault_Handler


    .weak   MemoryManagement_Handler
    .type   MemoryManagement_Handler, %function
MemoryManagement_Handler:
    b       .
    .size   MemoryManagement_Handler, . - MemoryManagement_Handler


    .weak   BusFault_Handler
    .type   BusFault_Handler, %function
BusFault_Handler:
    b       .
    .size   BusFault_Handler, . - BusFault_Handler


    .weak   UsageFault_Handler
    .type   UsageFault_Handler, %function
UsageFault_Handler:
    b       .
    .size   UsageFault_Handler, . - UsageFault_Handler


    .weak   SVC_Handler
    .type   SVC_Handler, %function
SVC_Handler:
    b       .
    .size   SVC_Handler, . - SVC_Handler


    .weak   DebugMon_Handler
    .type   DebugMon_Handler, %function
DebugMon_Handler:
    b       .
    .size   DebugMon_Handler, . - DebugMon_Handler


    .weak   PendSV_Handler
    .type   PendSV_Handler, %function
PendSV_Handler:
    b       .
    .size   PendSV_Handler, . - PendSV_Handler


    .weak   SysTick_Handler
    .type   SysTick_Handler, %function
SysTick_Handler:
    b       .
    .size   SysTick_Handler, . - SysTick_Handler


/* IRQ Handlers */

    .globl  Default_Handler
    .type   Default_Handler, %function
Default_Handler:
    b       .
    .size   Default_Handler, . - Default_Handler

    .macro  IRQ handler
    .weak   \handler
    .set    \handler, Default_Handler
    .endm

    IRQ  POWER_CLOCK_IRQHandler
    IRQ  RADIO_IRQHandler
    IRQ  UARTE0_UART0_IRQHandler
    IRQ  SPIM0_SPIS0_TWIM0_TWIS0_SPI0_TWI0_IRQHandler
    IRQ  SPIM1_SPIS1_TWIM1_TWIS1_SPI1_TWI1_IRQHandler
    IRQ  NFCT_IRQHandler
    IRQ  GPIOTE_IRQHandler
    IRQ  SAADC_IRQHandler
    IRQ  TIMER0_IRQHandler
    IRQ  TIMER1_IRQHandler
    IRQ  TIMER2_IRQHandler
    IRQ  RTC0_IRQHandler
    IRQ  TEMP_IRQHandler
    IRQ  RNG_IRQHandler
    IRQ  ECB_IRQHandler
    IRQ  CCM_AAR_IRQHandler
    IRQ  WDT_IRQHandler
    IRQ  RTC1_IRQHandler
    IRQ  QDEC_IRQHandler
    IRQ  COMP_LPCOMP_IRQHandler
    IRQ  SWI0_EGU0_IRQHandler
    IRQ  SWI1_EGU1_IRQHandler
    IRQ  SWI2_EGU2_IRQHandler
    IRQ  SWI3_EGU3_IRQHandler
    IRQ  SWI4_EGU4_IRQHandler
    IRQ  SWI5_EGU5_IRQHandler
    IRQ  TIMER3_IRQHandler
    IRQ  TIMER4_IRQHandler
    IRQ  PWM0_IRQHandler
    IRQ  PDM_IRQHandler
    IRQ  MWU_IRQHandler
    IRQ  PWM1_IRQHandler
    IRQ  PWM2_IRQHandler
    IRQ  SPIM2_SPIS2_SPI2_IRQHandler
    IRQ  RTC2_IRQHandler
    IRQ  I2S_IRQHandler
    IRQ  FPU_IRQHandler

  .end
```

let’s see if we can write a minimal Reset_Handler from first principles.
ARM’s Technical Reference Manuals are useful. [Section 5.9.2 of the Cortex-M3 TRM](https://developer.arm.com/docs/ddi0337/e/exceptions/resets/intended-boot-up-sequence) contains the following table:
| Reset boot-up behavior|
| Action | Description |
| --- | --- |
| Initialize variables| Any global/static variables must be setup. This includes initializing the BSS variable to 0, and copying initial values from ROM to RAM for non-constant variables.|
| [Setup stacks] | If more than one stack is be used, the other banked SPs must be initialized. The current SP can also be changed to Process from Main.|
| [Initialize any runtime] | Optionally make calls to C/C++ runtime init code to enable use of heap, floating point, or other features. This is normally done by __main from the C/C++ library.|

So, our ResetHandler is responsible for initializing static and global variables, and starting our program.
We rely on the compiler (technically, the linker) to put all those variables in the same place so we can initialize them in one fell swoop.

For static variables that must be zeroed, the linker gives us

- `_sbss` as the start addresses the static variables live at
- `_ebss` as the end addresses the static variables live at

We can do

```c
/* Clear the zero segment */
for (uint32_t *bss_ptr = &_sbss; bss_ptr < &_ebss;) {
    *bss_ptr++ = 0;
}
```

For static variables with an init value, the linker gives us:

- `_etext` as the address the init values are stored at
- `_sdata` as the address the static variables live at
- `_edata` as the end of the static variables memory

We can do

```c
uint32_t *init_values_ptr = &_etext;
uint32_t *data_ptr = &_sdata;
if (init_values_ptr != data_ptr) {
    for (; data_ptr < &_edata;) {
        *data_ptr++ = *init_values_ptr++;
    }
}
```

Putting it together, and with a call to `main()` to start our program, we write our Reset_Handler:

```c
void Reset_Handler(void)
{
    /* Copy init values from text to data */
    uint32_t *init_values_ptr = &_etext;
    uint32_t *data_ptr = &_sdata;

    if (init_values_ptr != data_ptr) {
        for (; data_ptr < &_edata;) {
            *data_ptr++ = *init_values_ptr++;
        }
    }

    /* Clear the zero segment */
    for (uint32_t *bss_ptr = &_sbss; bss_ptr < &_ebss;) {
        *bss_ptr++ = 0;
    }

    /* Overwriting the default value of the NVMCTRL.CTRLB.MANW bit (errata reference 13134) */
    NVMCTRL->CTRLB.bit.MANW = 1;

    /* Branch to main function */
    main();

    /* Infinite loop */
    while (1);
}
```

We add two more things:

1. An infinite loop after main(), so we do not run off into the weeds if the main function returns
2. Workaround for chip bugs which are best taken care of before our program starts. Sometimes these are wrapped in a `SystemInit` function called by the `Reset_Handler` before `main`.

Firmware Linker Scripts
===

Linking is the last stage in compiling a program. It takes a number of compiled object files and merges them into a single program, filling in addresses so that everything is in the right place.

Prior to linking, the compiler will have taken your source files one by one and compiled them into machine code. In the process, it leaves placeholders for addresses as (1) it does not know where the code will end up within the broader structure of the program and (2) it knows nothing about symbols outside of the current file or compilation unit.

The linker takes all of those object files and merges them together along with external dependencies like the C Standard Library into your program. To figure out which bits go where, the linker relies on a linker script - a blueprint for your program. Lastly, all placeholders are replaced by addresses.

Anatomy of a Linker Script
---

A linker script contains four things:

1. Memory layout: what memory is available where
2. Section definitions: what part of a program should go where
3. Options: commands to specify architecture, entry point, …etc. if needed
4. Symbols: variables to inject into the program at link time

Memory Layout
---

In order to allocate program space, the linker needs to know how much memory is available, and at what addresses that memory exists. This is what the MEMORY definition in the linker script is for.

The syntax for MEMORY is defined in the binutils docs and is as follow:

```ld
MEMORY
  {
    name [(attr)] : ORIGIN = origin, LENGTH = len
    …
  }
```

- `name` is a name you want to use for this region. Names do not carry meaning, so you’re free to use anything you want. You’ll often find “flash”, and “ram” as region names.
- `(attr)` are optional attributes for the region, like whether it’s writable (w), readable (r), or executable (x). Flash memory is usually (rx), while ram is rwx. Marking a region as non-writable does not magically make it write protected: these attributes are meant to describe the properties of the memory, not set it.
- `origin` is the start address of the memory region.
- `len` is the size of the memory region, in bytes.

Translate SAMD21G18 Chip Memory Map
| Memory | Start | Address Size |
| --- | --- | --- |
| Internal Flash | 0x00000000 | 256 Kbytes|
| Internal SRAM  | 0x20000000 | 32 Kbytes|

as

```ld
MEMORY
{
  rom      (rx)  : ORIGIN = 0x00000000, LENGTH = 0x00040000
  ram      (rwx) : ORIGIN = 0x20000000, LENGTH = 0x00008000
}
```

Section Definitions
---

Code and data are bucketed into sections, which are contiguous areas of memory. There are no hard rules about how many sections you should have, or what they should be, but you typically want to put symbols in the same section if:

1. They should be in the same region of memory, or
2. They need to be initialized together.

In our previous post, we learned about two types of symbols that are initialized in bulk:

1. Initialized static variables which must be copied from flash
2. Uninitialized static variables which must be zeroed.

Our linker script concerns itself with two more things:

1. Code and constant data, which can live in read-only memory (e.g. flash)
2. Reserved sections of RAM, like a stack or a heap

By convention, we name those sections as follow:

1. `.text` for code & constants
2. `.bss` for uninitialized data
3. `.stack` for our stack
4. `.data` for initialized data

The [elf spec](http://refspecs.linuxbase.org/elf/elf.pdf) holds a full list. Your firmware will work just fine if you call them anything else, but your colleagues may be confused and some tools may fail in odd ways. The only constraint is that you may not call your section `/DISCARD/`, which is a reserved keyword.

First, let’s look at what happens to our symbols if we do not define any of those sections in the linker script.

    MEMORY
    {
      rom      (rx)  : ORIGIN = 0x00000000, LENGTH = 0x00040000
      ram      (rwx) : ORIGIN = 0x20000000, LENGTH = 0x00008000
    }
    
    SECTIONS
    {
        /* empty! */
    }

The linker is perfectly happy to link our program with this. Probing the resulting elf file with objdump, we see the following:

    $ arm-none-eabi-objdump -h build/minimal.elf
    build/minimal.elf:     file format elf32-littlearm
    
    SYMBOL TABLE:
    no symbols

No symbols! While the linker is able to make assumptions that will allow it to link in symbols with little information, but it at least needs to know either what the entry point should be, or what symbols to put in the text section.

### `.text` Section

Let’s start by adding our `.text` section. We want that section in ROM. The syntax is simple:

    SECTIONS
    {
        .text :
        {
    
        } > rom
    }

This defines a section named `.text`, and adds it to the ROM. We now need to tell the linker what to put in that section. This is accomplished by listing all of the sections from our input object files we want in `.text`.

To find out what sections are in our object file, we can once again use `objdump`:

    $ arm-none-eabi-objdump -h
    build/objs/a/b/c/minimal.o:     file format elf32-littlearm
    
    Sections:
    Idx Name          Size      VMA       LMA       File off  Algn
      0 .text         00000000  00000000  00000000  00000034  2**1
                      CONTENTS, ALLOC, LOAD, READONLY, CODE
      1 .data         00000000  00000000  00000000  00000034  2**0
                      CONTENTS, ALLOC, LOAD, DATA
      2 .bss          00000000  00000000  00000000  00000034  2**0
                      ALLOC
      3 .bss.cpu_irq_critical_section_counter 00000004  00000000  00000000  00000034
    2**2
                      ALLOC
      4 .bss.cpu_irq_prev_interrupt_state 00000001  00000000  00000000  00000034
    2**0
                      ALLOC
      5 .text.system_pinmux_get_group_from_gpio_pin 0000005c  00000000  00000000
    00000034  2**2
                      CONTENTS, ALLOC, LOAD, READONLY, CODE
      6 .text.port_get_group_from_gpio_pin 00000020  00000000  00000000  00000090
    2**1
                      CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
      7 .text.port_get_config_defaults 00000022  00000000  00000000  000000b0  2**1
                      CONTENTS, ALLOC, LOAD, READONLY, CODE
      8 .text.port_pin_set_output_level 0000004e  00000000  00000000  000000d2  2**1
                      CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
      9 .text.port_pin_toggle_output_level 00000038  00000000  00000000  00000120
    2**1
                      CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
     10 .text.set_output 00000040  00000000  00000000  00000158  2**1
                      CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
     11 .text.main    0000002c  00000000  00000000  00000198  2**2
                      CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE

We see that each of our symbol has a section. This is due to the fact that we compiled our firmware with the `-ffunction-sections` and `-fdata-sections` flags. Had we not included them, the compiler would have been free to merge several functions into a single `text.<some identifier>` section.

To put all of our functions in the `.text` section in our linker script, we use the following syntax: `<filename>(<section>)`, where `filename` is the name of the input files whose symbols we want to include, and `section` is the name of the input sections. Since we want all `.text...` sections in all files, we use the wildcard `*`:

    .text :
    {
        KEEP(*(.vector*))
        *(.text*)
    } > rom

Note the `.vector` input section, which contains functions we want to keep at the very start of our `.text` section. This is so the `Reset_Handler` is where the MCU expects it to be. We’ll talk more about the vector table in a future post.

Dumping our elf file, we now see all of our functions (but no data)!

    $ arm-none-eabi-objdump -t build/minimal.elf
    
    build/minimal.elf:     file format elf32-littlearm
    
    SYMBOL TABLE:
    00000000 l    d  .text  00000000 .text
    ...
    00000000 l    df *ABS*  00000000 minimal.c
    00000000 l     F .text  0000005c system_pinmux_get_group_from_gpio_pin
    0000005c l     F .text  00000020 port_get_group_from_gpio_pin
    0000007c l     F .text  00000022 port_get_config_defaults
    0000009e l     F .text  0000004e port_pin_set_output_level
    000000ec l     F .text  00000038 port_pin_toggle_output_level
    00000124 l     F .text  00000040 set_output
    00000000 l    df *ABS*  00000000 port.c
    00000190 l     F .text  00000028 system_pinmux_get_config_defaults
    00000000 l    df *ABS*  00000000 pinmux.c
    00000208 l     F .text  0000005c system_pinmux_get_group_from_gpio_pin
    00000264 l     F .text  00000110 _system_pinmux_config
    00000164 g     F .text  0000002c main
    000001b8 g     F .text  0000004e port_pin_set_config
    00000374 g     F .text  00000040 system_pinmux_pin_set_config
    ...

### `.bss` Section

Now, let’s take care of our `.bss`. Remember, this is the section we put uninitialized static memory in. `.bss` must be reserved in the memory map, but there is nothing to load, as all variables are initialized to zero. As such, this is what it should look like:

    SECTION {
        ...
        .bss (NOLOAD) :
        {
            *(.bss*)
            *(COMMON)
        } > ram
    }

You’ll note that the `.bss` section also includes `*(COMMON)`. This is a special input section where the compiler puts global uninitialized variables that go beyond file scope. `int foo;` goes there, while `static int foo;` does not. This allows the linker to merge multiple definitions into one symbol if they have the same name.

We indicate that this section is not loaded with the `NOLOAD` property. This is the only section property used in modern linker scripts.

### `.stack` Section

We do the same thing for our `.stack` memory, since it is in RAM and not loaded. As the stack contains no symbols, we must explicitly reserve space for it by indicating its size. We also must align the stack on an 8-byte boundary per ARM Procedure Call Standards ([AAPCS](https://static.docs.arm.com/ddi0403/ec/DDI0403E_c_armv7m_arm.pdf)).

In order to achieve these goals, we turn to a special variable `.`, also known as the “location counter”. The location counter tracks the current offset into a given memory region. As sections are added, the location counter increments accordingly. You can force alignment or gaps by setting the location counter forward. You may not set it backwards, and the linker will throw an error if you try.

We set the location counter with the `ALIGN` function, to align the section, and use simple assignment and arithmetic to set the section size:

    STACK_SIZE = 0x2000; /* 8 kB */
    
    SECTION {
        ...
        .stack (NOLOAD) :
        {
            . = ALIGN(8);
            . = . + STACK_SIZE;
            . = ALIGN(8);
        } > ram
        ...
    }

Only one more section to go!

### `.data` Section

The `.data` section contains static variables which have an initial value at boot. You will remember from our previous article that since RAM isn’t persisted while power is off, those sections need to be loaded from flash. At boot, the `Reset_Handler` copies the data from flash to RAM before the `main` function is called.

To make this possible, every section in our linker script has two addresses, its _load_ address (LMA) and its _virtual_ address (VMA). In a firmware context, the LMA is where your JTAG loader needs to place the section and the VMA is where the section is found during execution.

You can think of the LMA as the address “at rest” and the VMA the address during execution i.e. when the device is on and the program is running.

The syntax to specify the LMA and VMA is relatively straightforward: every address is two part: AT . In our case it looks like this:

```ld
    .data :
    {
        *(.data*);
    } > ram AT > rom  /* "> ram" is the VMA, "> rom" is the LMA */
```

Note that instead of appending a section to a memory region, you could also explicitly specify an address like so:

```ld
    .data ORIGIN(ram) /* VMA */ : AT(ORIGIN(rom)) /* LMA */
    {
        . = ALIGN(4);
        _sdata = .;
        *(.data*);
        . = ALIGN(4);
        _edata = .;
    }
```

Where `ORIGIN(<region>)` is a simple way to specify the start of a region. You can enter an address in hex as well.

And we’re done! Here’s our complete linker script with every section:

Complete Linker Script
---

    MEMORY
    {
      rom      (rx)  : ORIGIN = 0x00000000, LENGTH = 0x00040000
      ram      (rwx) : ORIGIN = 0x20000000, LENGTH = 0x00008000
    }
    
    STACK_SIZE = 0x2000;
    
    /* Section Definitions */
    SECTIONS
    {
        .text :
        {
            KEEP(*(.vectors .vectors.*))
            *(.text*)
            *(.rodata*)
        } > rom
    
        /* .bss section which is used for uninitialized data */
        .bss (NOLOAD) :
        {
            *(.bss*)
            *(COMMON)
        } > ram
    
        .data :
        {
            *(.data*);
        } > ram AT >rom
    
        /* stack section */
        .stack (NOLOAD):
        {
            . = ALIGN(8);
            . = . + STACK_SIZE;
            . = ALIGN(8);
        } > ram
    
        _end = . ;
    }

You can find the full details on linker script sections syntax in the [ld manual](https://sourceware.org/binutils/docs/ld/SECTIONS.html#SECTIONS).

Variables
---

In the first post, our `ResetHandler` relied on seemingly magic variables to know the address of each of our sections of memory. It turns out, those variable came

In order to make section addresses available to code, the linker is able to generate symbols and add them to the program.

You can find the syntax in the [linker documentation](https://sourceware.org/binutils/docs/ld/Simple-Assignments.html#Simple-Assignments), it looks exactly like a C assignment: `symbol = expression;`

Here, we need:

1. `_etext` the end of the code in `.text` section in flash.
2. `_sdata` the start of the `.data` section in RAM
3. `_edata` the end of the `.data` section in RAM
4. `_sbss` the start of the `.bss` section in RAM
5. `_ebss` the end of the `.bss` section in RAM

They are all relatively straightforward: we can assign our symbols to the value of the location counter (`.`) at the start and at the end of each section definition.

The code is below:

        .text :
        {
            KEEP(*(.vectors .vectors.*))
            *(.text.*)
            *(.rodata.*)
            _etext = .;
        } > rom
    
        .bss (NOLOAD) :
        {
            _sbss = . ;
            *(.bss .bss.*)
            *(COMMON)
            _ebss = . ;
        } > ram
    
        .data :
        {
            _sdata = .;
            *(.data*);
            _edata = .;
        } > ram AT >rom

One quirk of these linker-provided symbols: you must use a reference to them, never the variable themselves. For example, the following gets us a pointer to the start of the `.data` section:

    uint8_t *data_byte = &_sdata;

You can read more details about this in the [binutils docs](https://sourceware.org/binutils/docs/ld/Source-Code-Reference.html).

------------

Write a Bootloader from Scratch
===

Previously, we

1. wrote a startup file to bootstrap our C environment, and
2. a linker script to get the right data at the right addresses.

These two will allow us to write a monolithic firmware which we can load and run on our microcontrollers.

In practice, this is not how most firmware is structured. Digging through vendor SDKs, you’ll notice that they all recommend using a _bootloader_ to load your applications. A bootloader is a small program which is responsible for loading and starting your application.

In this part, we will explain why you may want a bootloader, how to implement one, and cover a few advanced techniques you may use to make your bootloader more useful.

Why you may need a bootloader
---

Bootloaders serve many purposes, ranging from security to software architecture.

Most commonly, you may need a bootloader to load your software. Some microcontrollers like Dialog’s [DA14580](https://www.dialog-semiconductor.com/products/connectivity/bluetooth-low-energy/smartbond-da14580-and-da14583) have little to no onboard flash and instead rely on an external device to store firmware code. In that case, it is the bootloader’s job to copy code from non-executable storage, such as a SPI flash, to an area of memory that can be executed from, such as RAM.

Bootloaders also allow you to decouple parts of the program that are mission critical, or that have security implications, from application code which changes regularly. For example, your bootloader may contain firmware update logic so your device can recover no matter how bad a bug ships in your application firmware.

Last but certainly not least, bootloaders are an essential component of a trusted boot architecture. Your bootloader can, for example, verify a cryptographic signature to make sure the application has not been replaced or tampered with.

A minimal bootloader
---
Let’s build a simple bootloader together. To start, our bootloader must do two things:

1. Execute on MCU boot
2. Jump to our application code

We’ll need to decide on a memory map, write some bootloader code, and update our application to make it bootload-able.

### Setting the stage

For this example, we’ll be using the same setup as we did in our previous Zero to Main posts:

- Adafruit’s [Metro M0 Express](https://www.adafruit.com/product/3505) as our development board,
- a simple [CMSIS-DAP Adapter](https://www.adafruit.com/product/2764)
- OpenOCD (the [Arduino fork](https://github.com/arduino/OpenOCD)) for programming

### Deciding on a memory map

We must first decide on how much space we want to dedicate to our bootloader. Code space is precious - your application may come to need more of it - and you will not be able to change this without updating your bootloader, so make this as small as you possibly can.

Another important factor is your flash sector size: you want to make sure you can erase app sectors without erasing bootloader data, or vice versa. Consequently, your bootloader region must end on a flash sector boundary (typically 4kB).

I decided to go with a 16kB region, leading to the following memory map:

```txt
            0x0 +---------------------+
                |                     |
                |     Bootloader      |
                |                     |
         0x4000 +---------------------+
                |                     |
                |                     |
                |     Application     |
                |                     |
                |                     |
        0x30000 +---------------------+
```

We can transcribe that memory into a linker script:

```ld
    /* memory_map.ld */
    MEMORY
    {
      bootrom  (rx)  : ORIGIN = 0x00000000, LENGTH = 0x00004000
      approm   (rx)  : ORIGIN = 0x00004000, LENGTH = 0x0003C000
      ram      (rwx) : ORIGIN = 0x20000000, LENGTH = 0x00008000
    }
    
    __bootrom_start__ = ORIGIN(bootrom);
    __bootrom_size__ = LENGTH(bootrom);
    __approm_start__ = ORIGIN(approm);
    __approm_size__ = LENGTH(approm);
```

Since linker scripts are composable, we will be able to `include` that memory map into the linker scripts we write for our bootloader and our application.

You’ll notice that the linker script above declares some variables. We’ll need those for our bootloader to know where to find the application. To make them accessible in C code, we declare them in a header file:

```c
    /* memory_map.h */
    #pragma once
    
    extern int __bootrom_start__;
    extern int __bootrom_size__;
    extern int __approm_start__;
    extern int __approm_size__;
```

### Implementing the bootloader itself

Let’s write some bootloader code.
Our bootloader needs to start executing on boot and then jump to our app.

We know how to do the first part from our previous post: we need a valid stack pointer at address `0x0` , and a valid `Reset_Handler` function setting up our environment at address `0x4`. We can reuse our previous startup file and linker script, with one change: we use `memory_map.ld` rather than define our own `MEMORY` section.

We also need to put our code in the `bootrom` region from our memory rather than the `rom` region in our previous post.

Our linker script therefore looks like this:

    /* bootloader.ld */
    INCLUDE memory_map.ld
    
    /* Section Definitions */
    SECTIONS
    {
        .text :
        {
            KEEP(*(.vectors .vectors.*))
            *(.text*)
            *(.rodata*)
            _etext = .;
        } > bootrom
      ...
    }

To jump into our application, we need to know where the `Reset_Handler` of the app is, and what stack pointer to load. Again, we know from our previous post that those should be the first two 32-bit words in our binary, so we just need to dereference those addresses using the `__approm_start__` variable from our memory map.

```c
    /* bootloader.c */
    #include <inttypes.h>
    #include "memory_map.h"

    int main(void) {
      uint32_t *app_code = (uint32_t *)__approm_start__;
      uint32_t app_sp = app_code[0];
      uint32_t app_start = app_code[1];
      /* TODO: Start app */
      /* Not Reached */
      while (1) {}
    }
```

Next we must load that stack pointer and jump to the code. This will require a bit of assembly code.

ARM MCUs use the [`msr` instruction](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289882044.htm) to load immediate or register data into system registers, in this case the MSP register or “Main Stack Pointer”.

Jumping to an address is done with a branch, in our case with a [`bx` instruction](http://www.keil.com/support/man/docs/armasm/armasm_dom1361289866466.htm).

We wrap those two into a `start_app` function which accepts our `pc` and `sp` as arguments, and get our minimal bootloader:

```c
    /* bootloader.c */
    #include <inttypes.h>
    #include "memory_map.h"
    
    static void start_app(uint32_t pc, uint32_t sp) __attribute__((naked)) {
        __asm("           \n\
              msr msp, r1 /* load r1 into MSP */\n\
              bx r0       /* branch to the address at r0 */\n\
        ");
    }
    
    int main(void) {
      uint32_t *app_code = (uint32_t *)__approm_start__;
      uint32_t app_sp = app_code[0];
      uint32_t app_start = app_code[1];
      start_app(app_start, app_sp);
      /* Not Reached */
      while (1) {}
    }
```

> Note: hardware resources initialized in the bootloader must be de-initialized before control is transferred to the app. Otherwise, you risk breaking assumptions the app code is making about the state of the system

### Making our app bootloadable

We must update our app to take advantage of our new memory map. This is again done by updating our linker script to include `memory_map.ld` and changing our sections to go to the `approm` region rather than `rom`.

    /* app.ld */
    INCLUDE memory_map.ld
    
    /* Section Definitions */
    SECTIONS
    {
        .text :
        {
            KEEP(*(.vectors .vectors.*))
            *(.text*)
            *(.rodata*)
            _etext = .;
        } > approm
      ...
    }

We also need to update the [_vector table_](https://developer.arm.com/docs/dui0552/latest/the-cortex-m3-processor/exception-model/vector-table) used by the microcontroller. The vector table contains the address of every exception and interrupt handler in our system. When an interrupt signal comes in, the ARM core will call the address at the corresponding offset in the vector table.

For example, the offset for the Hard fault handler is `0xc`, so when a hard fault is hit, the ARM core will jump to the address contained in the table at that offset.

By default, the vector table is at address `0x0`, which means that when our chip powers up, only the bootloader can handle exceptions or interrupts! Fortunately, ARM provides the [Vector Table Offset Register](https://developer.arm.com/docs/dui0552/latest/cortex-m3-peripherals/system-control-block/vector-table-offset-register) to dynamically change the address of the vector table. The register is at address `0xE000ED08` and has a simple layout:

```txt
    31                                  7              0
    +-----------------------------------+--------------+
    |                                   |              |
    |              TBLOFF               |   Reserved   |
    |                                   |              |
    +-----------------------------------+--------------+
```

Where `TBLOFF` is the address of the vector table. In our case, that’s the start of our text section, or `_stext`. To set it in our app, we add the following to our `Reset_Handler`:

```c
    /* startup_samd21.c */
    /* Set the vector table base address */
    uint32_t *vector_table = (uint32_t *) &_stext;
    uint32_t *vtor = (uint32_t *)0xE000ED08;
    *vtor = ((uint32_t) vector_table & 0xFFFFFFF8);
```

One quirk of the ARMv7-m architecture is the alignment requirement for the vector table, as specified in section B1.5.3 of the [reference manual](https://static.docs.arm.com/ddi0403/eb/DDI0403E_B_armv7m_arm.pdf):

> The Vector table must be naturally aligned to a power of two whose alignment value is greater than or equal to (Number of Exceptions supported x 4), with a minimum alignment of 128 bytes.The entry at offset 0 is used to initialize the value for SP\_main, see The SP registers on page B1-8. All other entries must have bit \[0\] set, as the bit is used to define the EPSR T-bit on exception entry (see Reset behavior on page B1-20 and Exception entry behavior on page B1-21 for details).

Our SAMD21 MCU has 28 interrupts on top of the 16 system reserved exceptions, for a total of 44 entries in the table. Multiply that by 4 and you get 176. The next power of 2 is 256, so our vector table must be 256-byte aligned.

### Putting it all together

Because it is hard to witness the bootloader execute, we add a print line to each of our programs:

```c
    /* boootloader.c */
    #include <inttypes.h>
    #include "memory_map.h"
    
    static void start_app(uint32_t pc, uint32_t sp) {
        __asm("           \n\
              msr msp, r1 /* load r1 into MSP */\n\
              bx r0       /* branch to the address at r0 */\n\
        ");
    }
    
    int main() {
      serial_init();
      printf("Bootloader!\n");
      serial_deinit();
    
      uint32_t *app_code = (uint32_t *)__approm_start__;
      uint32_t app_sp = app_code[0];
      uint32_t app_start = app_code[1];
    
      start_app(app_start, app_sp);
    
      // should never be reached
      while (1);
    }
```

and:

```c
    /* app.c */
    int main() {
      serial_init();
      set_output(LED_0_PIN);
    
      printf("App!\n");
      while (true) {
        port_pin_toggle_output_level(LED_0_PIN);
        for (int i = 0; i < 100000; ++i) {}
      }
    }
```

> Note that the bootloader _must_ deinitialize the serial peripheral before starting the app, or you’ll have a hard time trying to initialize it again.

You can compile both these programs and load the resulting elf files with `gdb` which will put them at the correct address. However, the more convenient thing to do is to build a single binary which contains both programs.

To do that, you must go through the following steps:

1. Pad the bootloader binary to the full 0x4000 bytes
2. Create the app binary
3. Concatenate the two

Creating a binary from an elf file is done with `objcopy` . To accommodate our use case, `objcopy` has some handy options:

```shell
    $ arm-none-eabi-objcopy --help | grep -C 2 pad
      -b --byte <num>                  Select byte <num> in every interleaved block
         --gap-fill <val>              Fill gaps between sections with <val>
         --pad-to <addr>               Pad the last section up to address <addr>
         --set-start <addr>            Set the start address to <addr>
        {--change-start|--adjust-start} <incr>
```

The `—pad-to` option will pad the binary up to an address, and `—gap-fill` will allow you to specify the byte value to fill the gap with. Since we are writing our firmware to flash memory, we should fill with `0xFF` which is the erase value of flash, and pad to the max address of our bootloader.

We implement those rule in our Makefile, to avoid having to type them out each time:

```makefile
    # Makefile
    $(BUILD_DIR)/$(PROJECT)-app.bin: $(BUILD_DIR)/$(PROJECT)-app.elf
      $(OCPY) $< $@ -O binary
      $(SZ) $<
    
    $(BUILD_DIR)/$(PROJECT)-boot.bin: $(BUILD_DIR)/$(PROJECT)-boot.elf
      $(OCPY) --pad-to=0x4000 --gap-fill=0xFF -O binary $< $@
      $(SZ) $<
```

Last but not least, we need to concatenate our two binaries. As funny as that may sound, this is best achieved with `cat`:

```makefile
    # Makefile
    $(BUILD_DIR)/$(PROJECT).bin: $(BUILD_DIR)/$(PROJECT)-boot.bin $(BUILD_DIR)/$(PROJECT)-app.bin
      cat $^ > $@
```

Beyond the MVP
---

Now, this bootloader isn’t too useful, it only loads our application. We could do just as well without it.

In the following sections, I will go through a few useful things you can do with a bootloader.

### Message passing to catch reboot loops

A common thing to do with a bootloader is monitor stability. This can be done with a relatively simple setup:

1. On boot, the bootloader increments a persistent counter
2. After the app has been stable for a while (e.g. 1 minute), it resets the counter to 0
3. If the counter gets to 3, the bootloader does not start the app but instead signals an error.

This requires shared, persistent data between the application and the bootloader which is retained across reboots. On some architectures, non volatile registers are available which make this easy. This is the case on all STM32 microcontrollers which have RTC backup registers.

大多数情况下, we can use a region of RAM to get the same result. As long as the system remains powered, the RAM will keep its state even if the device reboots.

First, we carve some RAM for shared data in our memory map:

```ld
    /* memory_map.ld */
    MEMORY
    {
      bootrom  (rx)  : ORIGIN = 0x00000000, LENGTH = 0x00004000
      approm   (rx)  : ORIGIN = 0x00004000, LENGTH = 0x0003C000
      shared   (rwx) : ORIGIN = 0x20000000, LENGTH = 0x1000 # new added memory region
      ram      (rwx) : ORIGIN = 0x20001000, LENGTH = 0x00007000
    }
    
    /* shared data starts point at the origin of the shared region */
    _shared_data_start = ORIGIN(shared);
```

We can then create a data structure and assign it to this section, with getters to read it:

```c
    /* shared.h */
    
    #include <inttypes.h>
    
    uint8_t shared_data_get_boot_count(void);
    
    void shared_data_increment_boot_count(void);
    
    void shared_data_reset_boot_count(void);
    
    /* shared.c */
    
    #include "shared.h"
    
    extern uint32_t _shared_data_start;
    
    #pragma pack (push)
    struct shared_data {
      uint8_t boot_count;
    };
    #pragma pack (pop)
    
    struct shared_data *sd = (struct shared_data *)&_shared_data_start;
    
    uint8_t shared_data_get_boot_count(void) {
      return sd->boot_count;
    }
    
    void shared_data_increment_boot_count(void) {
      sd->boot_count++;
    }
    
    void shared_data_reset_boot_count(void) {
      sd->boot_count = 0;
    }
```

We compile the `shared` module into both our app and our bootloader, and can read the boot count in both programs.

### Relocating our app from flash to RAM

More commonly, bootloaders are used to relocate applications before they are executed. Relocations involves copying the application code from one place to another in order to execute it. This is useful when your application is stored in non-executable memory like a SPI flash.

Consider the following memory map:

```c
    /* memory_map.ld */
    MEMORY
    {
      bootrom  (rx)  : ORIGIN = 0x00000000, LENGTH = 0x00010000
      approm   (rx)  : ORIGIN = 0x00010000, LENGTH = 0x00004000
      ram      (rwx) : ORIGIN = 0x20000000, LENGTH = 0x00004000
      eram     (rwx) : ORIGIN = 0x20004000, LENGTH = 0x00004000
    }
    
    __bootrom_start__ = ORIGIN(bootrom);
    __bootrom_size__ = LENGTH(bootrom);
    __approm_start__ = ORIGIN(approm);
    __approm_size__ = LENGTH(approm);
    __eram_start__ = ORIGIN(eram);
    __eram_size__ = LENGTH(eram);
```

In this case, `approm` is our app storage and `eram` is our executable RAM, where we want to copy our program. Our bootloader needs to copy the code from `approm` to `eram` before executing it.

We know from our previous blog post that executable code typically ends up in the `.text` section so we must tell the linker that this section is **stored** in `approm` but executed from `eram` so our program can execute correctly.

This is similar to our `.data` section, which is stored in `rom` but lives in `ram` while the program is running. We use the `AT` linker command to specify the storage region and the `>` operator to specify the load region. This is the resulting linker script section:

```c
    /* app.ld */
    SECTIONS {
        .text :
        {
            KEEP(*(.vectors .vectors.*))
            *(.text*)
            *(.rodata*)
        } > eram AT > approm
        ...
    }
```

We then update our bootloader to copy our code from one to the other before starting the app:

```c
      /* booloader.c */
    
      /* copy app code to eram */
      uint32_t *src = (uint32_t*) &__approm_start__;
      uint32_t *dst = (uint32_t*) &__eram_start__;
      int size = (int) &__approm_size__;
      printf("Copying firmware from %p to %p\n", src, dst);
      memcpy(dst, src, size);
    
      /* find app start & SP */
      uint32_t app_sp = dst[0];
      uint32_t app_start = dst[1];
    
      /* cleanup peripherals here we may have initialized */
    
      /* start the app */
      start_app(app_start, app_sp);
```

### Locking the bootloader with the MPU
>
> Memory Protect Unit

Last but not least, we can protect the bootloader using the memory protection unit to make it inaccessible from the app. This prevents accidentally erasing the bootloader during execution.

If you do not know about the MPU, check out [Chris’s excellent blog post from a few weeks ago](/blog/fix-bugs-and-secure-firmware-with-the-mpu).

Remember that our MPU regions must be power-of-2 sized. Thankfully, our bootloader already is! `0x4000` is 2^14 bytes.

We add the following MPU code to our bootloader:

```c
    /* bootloader.c */
    int main(void) {
      /* ... */
      base_addr = 0x0;
      *mpu_rbar = (base_addr | 1 << 4 | 1);
      //  AP=0b110 to make the region read-only regardless of privilege
      //  TEXSCB=0b000010 because the Code is in "Flash memory"
      //  SIZE=13 because we want to cover 16kiB
      //  ENABLE=1
      *mpu_rasr = (0b110 << 24) | (0b000010 << 16) | (13 << 1) | 0x1;
    
      start_app(app_start, app_sp);
    
      /* Not reached */
      while (1) {}
    }
```

Bootstrapping libc with Newlib
===

So far, we bootstrapped a C environment, wrote a linker script from scratch, and implemented our own bootloader.

And yet, we cannot even write a hello world program! Consider the following `main.c` file:

```c
    #include <stdio.h>
    
    int main() {
      printf("Hello, World\n");
      while (1) {}
    }
```

Compiling this using our Makefile and linker script from [previous](/blog/zero-to-main-1) [posts](/blog/how-to-write-linker-scripts-for-firmware), we hit the following error:

```shell
    $ make
    ...
    Linking build/minimal.elf
    arm-none-eabi/bin/ld: build/objs/a/b/c/minimal.o: in function `main':
    /minimal/minimal.c:4: undefined reference to `printf'
    collect2: error: ld returned 1 exit status
    make: *** [build/minimal.elf] Error 1
```

Undefined reference to `printf`! How could this be?

Our firmware's C environment doesn't contains a working C standard library.
This means that commonly used functions such as `printf`, `memcpy`, or `strncpy` are all out of reach of our program so far.

In firmware-land, nothing comes free with the system: just like we had to explicitly zero out the `bss` region to initialize some of our static variables, we’ll have to port a `printf` implementation alongside a C standard library if we want to use it.

In this post, we will

1. add RedHat’s Newlib to our firmware and highlight some of its features.
2. implement syscalls, learn about constructors, and finally print out “Hello, World”!
3. also learn how to replace parts or all of the standard C library.

Setup
---

As we did in previous sections, we are using Adafruit’s Metro M0 development board to run our examples. We use a cheap CMSIS-DAP adapter and openOCD to program it.

You can find a step by step guide [in our previous post](/blog/getting-started-with-ibdap-and-atsamd21g18).

As with previous examples, we start with our “minimal” example which you can find [on GitHub](https://github.com/memfault/zero-to-main/tree/master/minimal). I’ve reproduced the source code for `main.c` below:

```c
    #include <samd21g18a.h>
    
    #include <port.h>
    #include <string.h>
    
    #define LED_0_PIN PIN_PA17
    
    static void set_output(const uint8_t pin) {
      struct port_config config_port_pin;
      port_get_config_defaults(&config_port_pin);
      config_port_pin.direction = PORT_PIN_DIR_OUTPUT;
      port_pin_set_config(pin, &config_port_pin);
      port_pin_set_output_level(pin, false);
    }
    
    int main() {
      memcpy(NULL, NULL, 0);
      set_output(LED_0_PIN);
      while (true) {
        port_pin_toggle_output_level(LED_0_PIN);
        for (int i = 0; i < 100000; ++i) {}
      }
    }
```

Implementing Newlib
---

### Why Newlib?

There are several implementations of the C Standard Library, starting with the venerable `glibc` found on most GNU/Linux systems. Alternative implementations include Musl libc[1](#fn:2), Bionic libc[2](#fn:3), ucLibc[3](#fn:4), and dietlibc[4](#fn:5).

Newlib is an implementation of the C Standard Library targeted at bare-metal embedded systems that is maintained by RedHat. It has become the de-facto standard in embedded software because it is complete, has optimizations for a wide range of architectures, and produces relatively small code.

Today Newlib is bundled alongside toolchains and SDK provided by vendors such as ARM (`arm-none-eabi-gcc`) and Espressif (ESP-IDF for ESP32).

> Note: when code-size constrained, you may choose to use a variant of newlib, called newlib-nano, which does away with some C99 features, and some `printf` bells and whistles to deliver a more compact standard library. Newlib-nano is enabled with the `—specs=nano.specs` CFLAG. You can read more about it in our [code size blog post](/blog/code-size-optimization-gcc-flags#c-library)

### Enabling Newlib

Newlib is enabled **by default** when you build a project with `arm-none-eabi-gcc`.
Indeed, you must explicitly opt-out with `-nostdlib` if you prefer to build your firmware without it.

This is what we do for our “minimal” example, to guarantee we do not include any libc functionality by mistake.

```makefile
    PROJECT := minimal
    BUILD_DIR ?= build
    
    CFLAGS += -nostdlib
    
    SRCS = \
     startup_samd21.c \
     $(PROJECT).c
    
    include ../common-standalone.mk
```

It is very easy to add a dependency on the C standard library without meaning to, as GCC will sometimes use standard C functions implicitly. For example, consider this code used to zero-initialize a struct:

```c
    int main() {
      int b[50] = {0}; // zero initialize a struct
      /* ... */
    }
```

We added no new `#include`, nor any call to C library functions. Yet if we compile this code with `-nostdlib`, we’ll get the following error:

```shell
    ...
    Linking build/minimal.elf
    /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/bin/ld: build/objs/a/b/c/minimal.o: in function `main':
    /minimal/minimal.c:16: undefined reference to `memset'
    collect2: error: ld returned 1 exit status
    make: *** [build/minimal.elf] Error 1
```

If we remove `-nostdlib`, the program compiles and link without problems.

```shell
    Linking build/minimal.elf
    arm-none-eabi-objdump -D build/minimal.elf > build/minimal.lst
    arm-none-eabi-objcopy build/minimal.elf build/minimal.bin -O binary
    arm-none-eabi-size build/minimal.elf
       text    data     bss     dec     hex filename
       1292       0    8192    9484    250c build/minimal.elf
```

So here we are, using Newlib, and we did not have to do anything. Could it really be this simple?

> Note: the variant of Newlib bundled with `arm-none-eabi-gcc` is not compiled with `-g`, which can make debugging difficult. For that reason, you may chose to replace it with your own build of Newlib. You can read more about that process in the [Implementing our own C standard library section](#implementing-our-own-c-standard-library) of this article.

System Calls
---

Let’s go back to our “Hello World” example:

```c
    int main() {
      printf("Hello World!\n");
      while(1) {}
    }
```

Removing `-nostdlib` is not quite enough. Instead of `printf` being undefined, we now see a whole mess of undefined symbols:

```shell
    Linking build/minimal.elf
    /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/bin/ld: /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/lib/thumb/v6-m/nofp/libg_nano.
    a(lib_a-sbrkr.o): in function `_sbrk_r':
    sbrkr.c:(.text._sbrk_r+0xc): undefined reference to `_sbrk'
    /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/bin/ld: /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/lib/thumb/v6-m/nofp/libg_nano.
    a(lib_a-writer.o): in function `_write_r':
    writer.c:(.text._write_r+0x10): undefined reference to `_write'
    /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/bin/ld: /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/lib/thumb/v6-m/nofp/libg_nano.
    a(lib_a-closer.o): in function `_close_r':
    closer.c:(.text._close_r+0xc): undefined reference to `_close'
    /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/bin/ld: /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/lib/thumb/v6-m/nofp/libg_nano.
    a(lib_a-lseekr.o): in function `_lseek_r':
    lseekr.c:(.text._lseek_r+0x10): undefined reference to `_lseek'
    /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/bin/ld: /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/lib/thumb/v6-m/nofp/libg_nano.
    a(lib_a-readr.o): in function `_read_r':
    readr.c:(.text._read_r+0x10): undefined reference to `_read'
    /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/bin/ld: /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/lib/thumb/v6-m/nofp/libg_nano.
    a(lib_a-fstatr.o): in function `_fstat_r':
    fstatr.c:(.text._fstat_r+0xe): undefined reference to `_fstat'
    /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/bin/ld: /usr/local/Cellar/arm-none-eabi-gcc/8-2018-q4-major/gcc/bin/../lib/gcc/arm-none-eabi/8.2.1/../../../../arm-none-eabi/lib/thumb/v6-m/nofp/libg_nano.
    a(lib_a-isattyr.o): in function `_isatty_r':
    isattyr.c:(.text._isatty_r+0xc): undefined reference to `_isatty'
    collect2: error: ld returned 1 exit status
```

Specifically, the compiler is asking for `_fstat`, `_read`, `_lseek`, `_close`, `_write`, and `_sbrk`.

The newlib documentation[5](#fn:6) calls these functions “system calls”. In short, they are the handful of things newlib expects the underlying “operating system”. The complete list of them is provided below:

    _exit, close, environ, execve, fork, fstat, getpid, isatty, kill,
    link, lseek, open, read, sbrk, stat, times, unlink, wait, write

You’ll notice that several of the syscalls relate to filesystem operation or process control. These do not make much sense in a firmware context, so we’ll often simply provide a stub that returns an error code.

Let’s look at the ones our “Hello, World” example requires.

### fstat

`fstat` returns the status of an open file. The minimal version of this should identify all files as character special devices. This forces one-byte-read at a time.

    #include <sys/stat.h>
    int fstat(int file, struct stat *st) {
      st->st_mode = S_IFCHR;
      return 0;
    }

### lseek

`lseek` repositions the file offset of the open file associated with the file descriptor `fd` to the argument `offset` according to the directive `whence`.

Here we can simply return 0, which implies the file is empty.

    int lseek(int file, int offset, int whence) {
      return 0;
    }

### close

`close` closes a file descriptor `fd`.

Since no file should have gotten `open`\-ed, we can just return an error on close:

    int close(int fd) {
      return -1;
    }

### write

This is where things get interesting! `write` writes up to `count` bytes from the buffer starting at `buf` to the file referred to by the file descriptor `fd`.

Functions like `printf` rely on `write` to write bytes to `STDOUT`. In our case, we will want those bytes to be written to serial instead.

On the SAMD21 chip we are using, writing bytes to serial is done using the `usart_serial_putchar` function. We can use it to implement `write`:

    static struct usart_module stdio_uart_module;
    
    int _write (int fd, char *buf, int count) {
      int written = 0;
    
      for (; count != 0; --count) {
        if (usart_serial_putchar(&stdio_uart_module, (uint8_t)*buf++)) {
          return -1;
        }
        ++written;
      }
      return written;
    }

We’ll also need to initialize the USART peripheral prior to calling printf:

    static void serial_init(void) {
      struct usart_config usart_conf;
    
      usart_get_config_defaults(&usart_conf);
      usart_conf.mux_setting = USART_RX_3_TX_2_XCK_3;
      usart_conf.pinmux_pad0 = PINMUX_UNUSED;
      usart_conf.pinmux_pad1 = PINMUX_UNUSED;
      usart_conf.pinmux_pad2 = PINMUX_PB22D_SERCOM5_PAD2;
      usart_conf.pinmux_pad3 = PINMUX_PB23D_SERCOM5_PAD3;
    
      usart_serial_init(&stdio_uart_module, SERCOM5, &usart_conf);
      usart_enable(&stdio_uart_module);
    }
    
    int main() {
      serial_init();
    
      printf("Hello, World!\n");
      while (1) {}
    }

### read

`read` attempts to read up to `count` bytes from file descriptor `fd` into the buffer at `buf`.

Similarly to `write`, we want `read` to read bytes from serial:

    int _read (int fd, char *buf, int count) {
      int read = 0;
    
      for (; count > 0; --count) {
        usart_serial_getchar(&stdio_uart_module, (uint8_t *)buf++);
        read++;
      }
    
      return read;
    }

### sbrk

`sbrk` increases the program’s data space by `increment` bytes. In other words, it increases the size of the heap.

What does `printf` have to do with the heap, you will justly ask? It turns out that newlib’s `printf` implementations allocates data on the heap and depends on a working `malloc` implementation.

The source for `printf` is hard to follow, but you will find that indeed [it calls malloc](https://github.com/bminor/newlib/blob/6497fdfaf41d47e835fdefc78ecb0a934875d7cf/newlib/libc/stdio/vfprintf.c#L226)!

Here’s a simple implementation of `sbrk`:

    void *_sbrk(int incr) {
      static unsigned char *heap = HEAP_START;
      unsigned char *prev_heap = heap;
      heap += incr;
      return prev_heap;
    }

More often than not, we want the heap to use all the RAM not used by anything else. We therefore set `HEAP_START` to the first address not spoken for in our linker script. In our [previous post](/blog/how-to-write-linker-scripts-for-firmware#complete-linker-script), we had added the `_end` variable in our linker script to that end.

We replace `HEAP_START` with `_end` and get:

    void *_sbrk(int incr) {
      static unsigned char *heap = NULL;
      unsigned char *prev_heap;
    
      if (heap == NULL) {
        heap = (unsigned char *)&_end;
      }
      prev_heap = heap;
    
      heap += incr;
    
      return prev_heap;
    }

Putting it all together, we get the following `main.c` file:

    static struct usart_module stdio_uart_module;
    
    // LIBC SYSCALLS
    /////////////////////
    
    extern int _end;
    
    void *_sbrk(int incr) {
      static unsigned char *heap = NULL;
      unsigned char *prev_heap;
    
      if (heap == NULL) {
        heap = (unsigned char *)&_end;
      }
      prev_heap = heap;
    
      heap += incr;
    
      return prev_heap;
    }
    
    int _close(int file) {
      return -1;
    }
    
    int _fstat(int file, struct stat *st) {
      st->st_mode = S_IFCHR;
    
      return 0;
    }
    
    int _isatty(int file) {
      return 1;
    }
    
    int _lseek(int file, int ptr, int dir) {
      return 0;
    }
    
    void _exit(int status) {
      __asm("BKPT #0");
    }
    
    void _kill(int pid, int sig) {
      return;
    }
    
    int _getpid(void) {
      return -1;
    }
    
    int _write (int file, char * ptr, int len) {
      int written = 0;
    
      if ((file != 1) && (file != 2) && (file != 3)) {
        return -1;
      }
    
      for (; len != 0; --len) {
        if (usart_serial_putchar(&stdio_uart_module, (uint8_t)*ptr++)) {
          return -1;
        }
        ++written;
      }
      return written;
    }
    
    int _read (int file, char * ptr, int len) {
      int read = 0;
    
      if (file != 0) {
        return -1;
      }
    
      for (; len > 0; --len) {
        usart_serial_getchar(&stdio_uart_module, (uint8_t *)ptr++);
        read++;
      }
      return read;
    }
    
    
    // APP
    ////////////////////
    
    static void __attribute__((constructor)) serial_init(void) {
      struct usart_config usart_conf;
    
      usart_get_config_defaults(&usart_conf);
      usart_conf.mux_setting = USART_RX_3_TX_2_XCK_3;
      usart_conf.pinmux_pad0 = PINMUX_UNUSED;
      usart_conf.pinmux_pad1 = PINMUX_UNUSED;
      usart_conf.pinmux_pad2 = PINMUX_PB22D_SERCOM5_PAD2;
      usart_conf.pinmux_pad3 = PINMUX_PB23D_SERCOM5_PAD3;
    
      usart_serial_init(&stdio_uart_module, SERCOM5, &usart_conf);
      usart_enable(&stdio_uart_module);
    }
    
    int main() {
      serial_init();
      printf("Hello, World!\n");
    }

This compiles fine, and can be run on our MCU. Hello, World!

Initializing State with Constructors & Destructors
---

Although we could perfectly well stop here, we can improve a bit over the above.

In our example, `printf` depends implicitly on `serial_init` being called. This isn’t the end of the world, but it goes against the spirit of a standard library function which should be usable anywhere in our program.

Instead, let’s see what we can do so that this works:

    int main() {
      printf("Hello, World\n");
    }

Can you think of a solution?

If we want `printf` to work anywhere in our `main` function, then `serial_init` must be run **before** main. What runs before main? We know from our previous post that it is the `Reset_Handler`. A simple solution might therefore be:

    void Reset_Handler(void)
    {
      /* ... */
      /* Hardware Initialization */
      serial_init();
    
      /* Branch to main function */
      main();
    
      /* Infinite loop */
      while (1);
    }

The GNU compiler collection and Newlib offer an alternative solution: constructors.

Constructors are functions which should be run before `main`. Conceptually, they are similar to the constructors of statically allocated C++ objects.

A function is marked as a constructor using the attribute syntax: `__attribute__((constructor))`. We can thus update `serial_init`:

    static void __attribute__((constructor)) serial_init(void) {
      struct usart_config usart_conf;
    
      usart_get_config_defaults(&usart_conf);
      usart_conf.mux_setting = USART_RX_3_TX_2_XCK_3;
      usart_conf.pinmux_pad0 = PINMUX_UNUSED;
      usart_conf.pinmux_pad1 = PINMUX_UNUSED;
      usart_conf.pinmux_pad2 = PINMUX_PB22D_SERCOM5_PAD2;
      usart_conf.pinmux_pad3 = PINMUX_PB23D_SERCOM5_PAD3;
    
      usart_serial_init(&stdio_uart_module, SERCOM5, &usart_conf);
      usart_enable(&stdio_uart_module);
    }

But how do these constructors get invoked? We know that in firmware, we do not get anything for free. This is where newlib comes in.

By default, GCC will put every constructor into an array in their own section of flash. Newlib then offers a function, `__libc_init_array` which iterates over the array and invokes every constructor. You can find out more about it by reading [the source code](https://github.com/bminor/newlib/blob/master/newlib/libc/misc/init.c).

All we need to do is call `__libc_init_array` prior to `main` in our `Reset_Handler`, and we are good to go.

    void Reset_Handler(void)
    {
      /* ... */
      /* Run constructors / initializers */
      __libc_init_array();
    
      /* Branch to main function */
      main();
    
      /* Infinite loop */
      while (1);
    }

Newlib and Multi-threading
---
We have not yet talked much about multi-threading (e.g. with an RTOS) in this series, and going into details is outside of the scope of this article. However, there are a few things worth knowing when using Newlib in a multi-threaded environment.

### `_impure_ptr` and the `_reent` struct

Most Newlib functions are _reentrant_. This means that they can be called by multiple processes safely.

For the functions that cannot be easily made re-entrant, newlib depends on the operating system correctly setting the `_impure_ptr` variable whenever a context switch occur. That variable is expected to hold a `struct _reent` for the current thread. That struct is used to store state for standard library functions being used by that thread.

### Locking shared memory

Some standard library functions depend on global memory which would not make sense to hold in the `_reent` struct. This is especially important when using `malloc` to allocate memory out of the heap. If mutliple threads try modifying the heap at the same time, they risk corrupting it.

To allow multiple threads to call `malloc`, Newlib provides the `__malloc_lock` and `__malloc_unlock` APIs[6](#fn:1). A good implementation of these APIs would lock and unlock a recursive mutex.

Implementing our own C standard library
---

In some cases, you may want to take different tradeoffs than the ones taken by the implementers of Newlib. Perhaps you are willing to sacrifice some functionality for code space, or are willing to trade performance for security. In most cases it is easier to replace a few functions, though you may end up with a fully custom C library.

### Replacing a function

Because Newlib is a static library with a separate object file for every function, all you need to do to replace a function is define it in your program. The linker won’t go looking for it in static libraries if it finds it in your code.

For example, we may want to replace Newlib’s `printf` implementation, either because it is too large or because it depends on dynamic memory management. Using Marco Paland’s excellent alternative[7](#fn:7) is as simple as a Makefile change.

We first clone it in our firmware’s folder under `lib/printf`, and update our Makefile to reflect the change:

    PROJECT := with-libc
    BUILD_DIR ?= build
    
    INCLUDES = \
      ... \
      lib/printf
    
    SRCS = \
      ... \
      lib/printf/printf.c \
     startup_samd21.c \
     $(PROJECT).c
    
    include ../common-standalone.mk

### Full replacement

In some cases, you may want to do away with Newlib altogether. Perhaps you don’t want any dynamic memory allocation, in which case you could use Embedded Artistry’s solid alternative[8](#fn:8). Another good reason to replace the version of Newlib provided by your toolchain is to use your own build of it because you would like to use different compile-time flags.

Once we have copied the static lib (`.a`) for our selected libc, we disable Newlib with `-nostdlib` and explicitly link in our substitute library. You can find the resulting Makefile below:

    PROJECT := with-libc
    BUILD_DIR ?= build
    
    CFLAGS += -nostdlib
    LDFLAGS += -L../lib/embeddedartistry_libc -lc
    
    INCLUDES = \
     $(ASF_PATH)/sam0/drivers/sercom \
     $(ASF_PATH)/sam0/drivers/sercom/usart \
     $(ASF_PATH)/common/services/serial \
     $(ASF_PATH)/common/services/serial/sam0_usart
    
    SRCS = \
     $(ASF_PATH)/sam0/drivers/sercom/usart/usart.c \
     $(ASF_PATH)/sam0/drivers/sercom/sercom.c \
     startup_samd21.c \
     $(PROJECT).c
    
    include ../common-standalone.mk

> Note that the `__libc_init_array` functionality is not found in every standard C library. You will either need to avoid using it, or bring in Newlib’s implementation.

Reference Links
---

1. [musl libc](https://www.musl-libc.org/) [↩](#fnref:2)

2. [bionic libc](https://android.googlesource.com/platform/bionic/) [↩](#fnref:3)

3. [ucLibc](https://www.uclibc.org/) [↩](#fnref:4)

4. [dietlibc](https://www.fefe.de/dietlibc/) [↩](#fnref:5)

5. [newlib documentation](https://sourceware.org/newlib/libc.html#Syscalls) [↩](#fnref:6)

6. [\_\_malloc\_lock documentation](https://sourceware.org/newlib/libc.html#g_t_005f_005fmalloc_005flock) [↩](#fnref:1)

7. [Marco Paland’s printf](https://github.com/mpaland/printf) [↩](#fnref:7)

8. [Embedded Artistry’s libc](https://github.com/embeddedartistry/libc) [↩](#fnref:8)

Bare metal Rust
=========

For the past thirty years or so, the choice of languages for embedded systems developers has been relatively slim.
Languages like C++ and Ada have found a home in some niche areas, such as telecommunications and safety critical fields, while higher level languages like Lua, Python, and JavaScript have found a home for scripting and prototyping.

Despite these options, most developers working on bare metal systems have been using the same two languages as long as I can remember: Assembly and C.

But not for no reason! Languages often make trade-offs to fit the needs of the developers working with them: an interpreter to allow for rapid iteration, a heap for ease of memory management, exceptions for simplifying control flow, etc. These trade-offs usually come with a price: whether it is code size, RAM usage, low level control, power usage, latency, or determinism.

Since 2015, Rust has been redefining what it means to combine the best-in-class aspects of performance, correctness, and developer convenience into one language, without compromise. In this post, we’ll bootstrap a Rust environment on a Cortex-M microcontroller from scratch, and explain a few of the language concepts you might not have seen before.

As a compiled systems language (based on LLVM), it is also capable of reaching down to the lowest levels of embedded programming as well, without losing built-in features that feel more at home in higher level languages, like a package manager, helpful compile time diagnostics, correctness through powerful static analysis, or documentation tooling.

This post is meant as a complement to the original [Zero to main()](/blog/zero-to-main-1) post on the Interrupt blog, and will elide some of the explanations of hardware level concepts. If you’re new to embedded development, or haven’t seen that post, go read it now!

Setting the stage
-----------------

Most of the concepts and code presented in this series should work for all Cortex-M series MCUs, though these examples target the nRF52 processor by Nordic. This is a Cortex-M4F chip found on several affordable development boards.

Specifically, we are using:

- Decawave’s [DWM1001-DEV](https://www.decawave.com/product/dwm1001-development-board/) as our development board
- The built-in JLink capabilities of the board
- Segger’s JLinkGDBServer for programming

Software wise, we will be using:

- The 1.39.0 version of Rust, though any stable version 1.31.0 or newer should work
- We’ll also use some of the `arm-none-eabi` binutils, such as `arm-none-eabi-gdb` and `arm-none-eabi-objdump`, which are compatible with the binaries produced by Rust

We’ll also be implementing a simple blinking LED application. The full Rust source used for this blog post is available [here, on GitHub](https://github.com/ferrous-systems/zero-to-main/blob/master/from-scratch/src/main.rs). This is what the source code looks like for our application:

```rust
    #![no_std]
    #![no_main]
    
    use nrf52::gpio::{Level, Pins};
    
    fn main() -> ! {
        let gpios = Pins::take();
        let mut led = gpios.p0_31;
    
        led.set_push_pull_output(Level::Low);
    
        loop {
            led.set_high();
            delay(2_000_000);
    
            led.set_low();
            delay(6_000_000);
        }
    }
```

Power on
---------

Let’s build our Rust application, and see what the binary contains:

```shell
    cargo build --release
       Compiling from-scratch v0.1.0 (/home/james/memfault/blog-1/examples/from-scratch)
        Finished release [optimized] target(s) in 0.62s
    
    arm-none-eabi-objcopy -O binary target/thumbv7em-none-eabihf/release/from-scratch  target/thumbv7em-none-eabihf/release/from-scratch.bin
    
    xxd target/thumbv7em-none-eabihf/release/from-scratch.bin | head -n 5
    00000000: 0000 0120 dd00 0000 0000 0000 0000 0000  ... ............
    00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
```

Reading this, our initial stack pointer is `0x20010000`, and our start address pointer is `0x000000dd`. Let’s see what symbol is there. We will also pass `-C` to objdump, which will demangle our symbols (we’ll explain demangling a bit more later):

```shell
    arm-none-eabi-objdump -Ct target/thumbv7em-none-eabihf/release/from-scratch | sort
    ...
    00000004 g     O .vector_table  00000004 __RESET_VECTOR
    000000dc g       .vector_table  00000000 _stext
    000000dc l     F .text  0000005c from_scratch::reset_handler
    00000138 l     F .text  0000006a from_scratch::main
    000001a4 g       *ABS*  00000000 __sidata
    ...
```

Same as in the original post, our compiler has set the lowest bit of our reset handler to one to indicate a thumb2 function, so `from_scratch::reset_handler` is what we’re looking for.

Writing a Reset\_Handler
------------------------

The Cortex-M processor on our board doesn’t actually know the difference between C and Rust, so our responsibilities for starting the firmware are the same when we build a Rust program, we need to:

1. Provide an entry point, stored in the second word of flash
2. Zero-initialize the `.bss` section
3. Set items with static storage duration to their initial values. In Rust the destination (in RAM) is referred to as `.data`, and the source values (in Flash) is referred to as `.rodata`

Let’s go through the code required to write this reset handler, one chunk at a time.

### Starting at the top

The first line in many embedded applications and libraries in Rust will look like this:

    #![no_std]

This is called a “global attribute”, and it is stating that this Rust code will not be using the Rust Standard Library. Attributes in Rust are sometimes used similarly to `#pragma` in C, in that they can change certain behaviors of the compiler.

Rust provides a number of built-in library components, but the two main ones are:

1. The Rust Standard Library
2. The Rust Core Library

While the Standard Library contains a number of useful components, such as data structures, and interfaces for opening files and sockets, it generally requires your target to have these things! For bare metal applications, we can instead forego this library and only use the Rust Core Library, which does not have these requirements.

Rust as a language has a concept of “modules”, which can be used to organize and provide namespaces for your code. Libraries or applications in Rust are called “Crates”, and each has its own namespace. This is why we saw the symbol `from_scratch::reset_handler` in our linker script: It was referring to the `reset_handler` function in the `from_scratch` crate (which is the application crate in this example).

To use items from another crate, including the [`core` library](https://doc.rust-lang.org/core/index.html), you can import these items in using the `use` syntax:

    use core::{
        mem::zeroed,
        panic::PanicInfo,
        ptr::{read, write_volatile},
    };

This imports the symbols into the current context so that they can be used. Most symbols in Rust are not available in a global namespace, which helps to avoid naming collisions.

However in some cases, it is important to have a globally defined symbol. As part of the ABI of the Cortex-M platform, we need to provide the address of the reset handler in a very specific location. Let’s look at how we do that in Rust:

### Setting the Reset Vector

    #[link_section = ".vector_table.reset_vector"]
    #[no_mangle]
    pub static __RESET_VECTOR: fn() -> ! = reset_handler;

Let’s unpack that from the bottom up:

    pub static __RESET_VECTOR: fn() -> ! = reset_handler;

This line defines a symbol called `__RESET_VECTOR` at static scope. The type of this symbol is `fn() -> !`, which is a pointer to a function that takes no arguments and that never returns, or that “diverges”. The value of this symbol is `reset_handler`, which is the name of a function in our program. Functions are a first class items in Rust (similar to Python), so we can use the names of functions as a value that represents a function pointer.

    #[no_mangle]

This is another attribute, like our `#![no_std]`. By starting with `#[` instead of `#![`, we can tell this is a local attribute instead of a global attribute, which means it only applies to the next item, instead of the whole module.

The `#[no_mangle]` attribute tells the compiler **not** to mangle this symbol, so it will show up as `__RESET_VECTOR`, not `from_scratch::__RESET_VECTOR`. [Name mangling](https://en.wikipedia.org/wiki/Name_mangling) is a technique used by languages like C++ and Rust to emit unique names for things like functions, generic data type parameters, or symbols for data at static scope, no matter where or how often they are used in the resulting binary.

    #[link_section = ".vector_table.reset_vector"]

This is another attribute that is informing the compiler to place this symbol in the `.vector_table.reset_vector` section of our linker script, which will place it right where we need it. This is similar to gcc’s `__attribute__((section(...)))`.

### The Reset Handler, for real

Now let’s look at our actual reset handler, from top to bottom:

    pub fn reset_handler() -> ! {
        extern "C" {
            // These symbols come from `linker.ld`
            static mut __sbss: u32; // Start of .bss section
            static mut __ebss: u32; // End of .bss section
            static mut __sdata: u32; // Start of .data section
            static mut __edata: u32; // End of .data section
            static __sidata: u32; // Start of .rodata section
        }
    
        // Initialize (Zero) BSS
        unsafe {
            let mut sbss: *mut u32 = &mut __sbss;
            let ebss: *mut u32 = &mut __ebss;
    
            while sbss < ebss {
                write_volatile(sbss, zeroed());
                sbss = sbss.offset(1);
            }
        }
    
        // Initialize Data
        unsafe {
            let mut sdata: *mut u32 = &mut __sdata;
            let edata: *mut u32 = &mut __edata;
            let mut sidata: *const u32 = &__sidata;
    
            while sdata < edata {
                write_volatile(sdata, read(sidata));
                sdata = sdata.offset(1);
                sidata = sidata.offset(1);
            }
        }
    
        // Call user's main function
        main()
    }

Phew! That was a lot at once, especially if you aren’t familiar with Rust! Let’s break that down one chunk at a time to explain the concepts in a little more detail:

### Defining a function in Rust

    pub fn reset_handler() -> ! {

This defines a function that is public, named `reset_handler`, that takes no arguments `()`, and that never returns `-> !`.

In Rust, functions normally either don’t return a value like this:

    /// Returns nothing
    fn foo() { /* ... */ }

Or do return a value like this:

    /// Returns a 32-bit unsigned integer
    fn bar() -> u32 { /* ... */ }

The `!` type, called the “Never type”, means that this function will never return, or diverges. Since our reset handler never will return (where would it go?) we can tell Rust this, which may allow it to make certain optimizations at compile time.

### A little help from the linker

    extern "C" {
        // These symbols come from `linker.ld`
        static mut __sbss: u32; // Start of .bss section
        static mut __ebss: u32; // End of .bss section
        static mut __sdata: u32; // Start of .data section
        static mut __edata: u32; // End of .data section
        static __sidata: u32; // Start of .rodata section
    }

This section defines a number of static symbols which will be provided by our linker, namely the start and end of the sections that are important for our reset handler to know about. These symbols are defined in an `extern "C"` scope, which means two things:

1. They will be provided sometime later, by another piece of code, or in this case, the linker itself
2. They will be defined using the “C” style of ABI and naming conventions, which means they are implicitly `#[no_mangle]`

Some of these symbols are also declared as `mut`, or “mutable”. By default in Rust, all variables are immutable, or read-only. To make a variable mutable in Rust, you must explicitly mark it as `mut`. This is the opposite of languages like C and C++, where variables are by default mutable, and must be marked with `const` to prevent them from being modified.

### Zeroing the BSS section

    // Initialize (Zero) BSS
    unsafe {
        let mut sbss: *mut u32 = &mut __sbss;
        let ebss: *mut u32 = &mut __ebss;
    
        while sbss < ebss {
            write_volatile(sbss, zeroed());
            sbss = sbss.offset(1);
        }
    }

As a language, Rust makes some pretty strong guarantees around memory safety, correctness, and freedom from Undefined Behavior. However, when working directly with the hardware, which has no knowledge of Rust’s guarantees, it is necessary to work in Rust’s `unsafe` mode, which allows some additional behaviors, but requires the developer to uphold certain correctness guarantees manually.

Rust has two ways of referring to data by reference:

1. References
2. Raw Pointers

In most Rust code, we use references, which can be statically guaranteed for correctness and memory safety. However in this case, we are given the raw integers, which we want to treat as pointers.

In this code, we take a mutable reference to the `__sbss` and `__ebss` symbols provided by the linker, and convert these Rust references into raw pointers.

We then use these pointers to make volatile writes of zero across the range, one 32-bit word at a time.

This section zeros our entire `.bss` section, as defined by the linker.

### Initializing static data

    // Initialize Data
    unsafe {
        let mut sdata: *mut u32 = &mut __sdata;
        let edata: *mut u32 = &mut __edata;
        let mut sidata: *const u32 = &__sidata;
    
        while sdata < edata {
            write_volatile(sdata, read(sidata));
            sdata = sdata.offset(1);
            sidata = sidata.offset(1);
        }
    }

This section of code initializes our `.data` section, copying directly from the `.rodata` section. This is similar to the code above, however we also walk the pointer in the initializer section as well as the pointer in the destination section.

### Ready for launch

Finally, at the end of our reset handler, we get to call main!

    // Call user's main function
    main()

Since the main function we defined above is also divergent (`fn main() -> !`), we can simply call the function. If we had called a non-divergent function, we would get a compile error here!

### Something just for Rust

Rust does have one additional requirement for a bare metal program: You must define the panic handler.

    /// This function is called on panic.
    #[panic_handler]
    fn panic(_info: &PanicInfo) -> ! {
        // On a panic, loop forever
        loop {
            continue;
        }
    }

Rust has a concept of a `panic`, which is like failing an assert in C. This happens when the program has hit an unrecoverable error case, and must be stopped in some way.

Unlike Exceptions in C++, panics are usually not designed to be recovered from gracefully, and therefore do not require the overhead necessary to unwind.

Still, we must define a “panic handler” in case our program ever panics. For this example, we go into an endless loop, though you could choose to do something different, like logging the error to flash, or soft-rebooting the system immediately.

### Programming without Compromise

Earlier I mentioned that Rust brings convenience without compromise. To demonstrate this, let’s take a quick look at the size and total contents of our code once we compile for `opt-level = "s"`, which is equivalent to `-Os` in C or C++:

    arm-none-eabi-size target/thumbv7em-none-eabihf/release/from-scratch
       text    data     bss     dec     hex filename
        420       0       8     428     1ac target/thumbv7em-none-eabihf/release/from-scratch
    
    arm-none-eabi-nm -nSC target/thumbv7em-none-eabihf/release/from-scratch
    00000004 00000004 R __RESET_VECTOR
    00000008 R __reset_vector
    000000dc R _stext
    000000dc 0000005c t from_scratch::reset_handler
    00000138 0000006a t from_scratch::main
    000001a4 T __erodata
    000001a4 T __etext
    000001a4 A __sidata
    20000000 T __edata
    20000000 B __sbss
    20000000 T __sdata
    20000000 00000004 b from_scratch::delay::DUMMY
    20000004 00000001 b from_scratch::nrf52::gpio::Pins::take::TAKEN
    20000008 B __ebss
    20000008 B __sheap
    20010000 A _stack_start

This is 420 bytes of `.text`, which boils down to 220 bytes for the vector table, and 198 bytes of actual code.

Wrapping up
-----------

Although we spent this post talking about how to write support for scratch in Rust, we almost never need to actually do this in practice!

Instead we can leverage Cargo, the package manager and build system for Rust, to use existing libraries that support Cortex-M and Nordic components, a board support crate that handles configuration for our specific development board, and provide a panic handler.

These libraries include an initialization runtime with pre-init hooks, a template linker script that can be modified and extended, access to common Cortex-M components like the NVIC, and more, without having to copy and paste boilerplate reference code into our project.

Now we instead end up with a program that looks like this:

    #![no_std]
    #![no_main]
    
    // Panic provider crate
    use panic_reset as _;
    
    // Provides definitions for our development board
    use dwm1001::{
        cortex_m_rt::entry,
        nrf52832_hal::prelude::*,
        DWM1001
    };
    
    #[entry]
    fn main() -> ! {
        // Set up the board, initializing the LEDs on the board
        let mut board = DWM1001::take().unwrap();
    
        // Obtain a microsecond precision timer
        let mut timer = board.TIMER0.constrain();
    
        loop {
            // board.leds.D10 - Bottom LED BLUE
            board.leds.D10.enable();
            timer.delay(2_000_000);
    
            board.leds.D10.disable();
            timer.delay(6_000_000);
        }
    }

All of the code used in this blog post is available [on GitHub](https://github.com/ferrous-systems/zero-to-main). If you’re looking for information on how to get started with embedded Rust, check out the [Embedded Working Group](https://github.com/rust-embedded/wg)’s [bookshelf](https://docs.rust-embedded.org/) for documentation on how to install Rust, connect to your device, and build and run your first application.

In future posts we’ll talk about how libraries like [`r0`](https://docs.rs/r0), [`cortex-m`](https://docs.rs/cortex-m), and [`cortex-m-rt`](https://docs.rs/cortex-m-rt) provide common functionality when writing embedded programs, and how libraries like [`nrf52832-pac`](https://docs.rs/nrf52832-pac), [`nrf52832-hal`](https://docs.rs/nrf52832-hal), and [`dwm1001`](https://docs.rs/dwm1001) provide compile-time safe abstractions over hardware interfaces!
