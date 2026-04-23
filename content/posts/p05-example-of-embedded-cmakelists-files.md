---
title: "Example of Embedded CMakeLists Files"
date: 2023-05-03T23:38:47+08:00
# weight: 1
# aliases: ["/first"]
tags: ["CMake"]
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
The most thoroughly commented embedded CMakeLists file
======================================================

January 05, 2023 - [CMake](https://dnedic.github.io/tags/cmake/) [Build System](https://dnedic.github.io/tags/build-system/) [Embedded](https://dnedic.github.io/tags/embedded/) [STM32](https://dnedic.github.io/tags/stm32/)

Many embedded software engineers have had to deal with build systems that are either non cross-platform or a part of the IDE they are using, don't offer easy composability or per-library configuration, provide no testing support or ways to generate or package files without using a third party scripting language.

[CMake](https://cmake.org/) is a build system that offers solutions to many of those problems and is already well adopted in the world of traditional software development. Inspired by [The most thoroughly commented linker script](https://blog.thea.codes/the-most-thoroughly-commented-linker-script/) blogpost and having used CMake in two different professional environments, I have decided to write an overview of a minimal CMakeLists file for an embedded project.

[Here](https://github.com/DNedic/most_commented_embedded_cmakelists/blob/main/CMakeLists.txt) is the CMakeLists file we are going to be taking a look at. It is a part of a [CMake STM32 project](https://github.com/DNedic/most_commented_embedded_cmakelists) i created as a demonstration and uses an STM32F103 MCU.

The minimum version
-------------------
```cmake
    cmake_minimum_required(VERSION 3.21)
```

这个调用语句必须位于每个CMakeLists文件的顶部，它设置了我们的项目构建所需要的最低CMake版本，并决定了可以使用的CMake功能。
通常建议将其设置的尽可能低，除非需要新版本CMake的功能。
另一个好的方法是检查[repology](https://repology.org/project/cmake/versions)获取大多数Linux发行版中所使用的CMake版本。

Project declaration
-------------------
```cmake
    project(most_commented_embedded_cmakelists
        VERSION 1.0.0
        DESCRIPTION "This is a demo project"
    )
```    

The [project()](https://cmake.org/cmake/help/latest/command/project.html) call sets up the project name and optionally version, description, homepage and more and stores them in variables that can be used later. For instance, the project name is stored in `PROJECT_NAME`, description in `PROJECT_DESCRIPTION` and so on.

Language settings
-----------------
```cmake
    enable_language(C ASM)
```    

This sets the languages the project is going to be using, if C++ support is desired add `CXX` to the list.
```cmake
    set(CMAKE_C_STANDARD 11)
    set(CMAKE_C_STANDARD_REQUIRED ON)
```    

这个设置会在全局范围内指定使用的语言标准和是否将其作为必须条件。尽管这不是必须的，但由于 C 和 C++ 不向后兼容的特性，推荐使用。

C11通常是项目的首选，因为它提供了有用的附加功能，如原子操作([atomics](https://en.cppreference.com/w/c/atomic))，但它取消了可选功能，如可变长度数组([VLAs](https://en.wikipedia.org/wiki/Variable-length_array))，因此在处理旧项目时，C99或更低版本可能更适合。

```cmake
    set(CMAKE_C_EXTENSIONS OFF)
```    

这个命令用于全局控制是否包含C语言的[GNU Extensions](https://gcc.gnu.org/onlinedocs/gcc/C-Extensions.html)，
设置为OFF时更倾向于使用更标准和可移植的C，然而GNU Extensions提供的功能在基本标准之上提供了有用的东西。

Compiler options
----------------
```cmake
    set(MCU_OPTIONS
        -mcpu=cortex-m3
        -mthumb
    )
```    
这里设置了一个自定义的[microcontroller specific compiler flags](https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html)变量。
首先，设置了CPU变体，然后告诉编译器发出thumb指令([thumb instructions](https://developer.arm.com/documentation/ddi0337/e/Programmer-s-Model/Instruction-set))。
```cmake
    set(EXTRA_OPTIONS
        -fdata-sections
        -ffunction-sections
    )
```   
这里设置了一个包含额外功能编译器标志的变量。此处添加的标志确保所有对象都放在单独的linker section中。稍后我们将看到这为什么很重要。
```cmake
    set(OPTIMIZATION_OPTIONS
        $<$<CONFIG:Debug>:-Og>
    )
```   
这里创建了编译器优化标志的自定义变量。当使用Debug([configuration](https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html#build-configurations))配置进行构建时, 这个奇怪的表达式有条件地将编译器优化级别设置为`-Og`。

由于大多数微控制器在闪存大小和CPU速度方面都相当有限，因此默认优化级别`-O0`不适合。`-Og`是一种最佳折衷方案，在大小、速度和调试可行性之间有最佳的优化。

其他配置使用它们的默认编译器优化级别，`Release`配置为`-O2`，`MinSizeRel`配置为`-Os`，因此不需要添加其他内容。

```cmake
    set(DEPENDENCY_INFO_OPTIONS
```   
这里创建了一个包含Dependency Info Options的预处理器标志的自定义变量。
```cmake
        -MMD
```    
这一行告诉预处理器为兼容Make的构建系统生成依赖文件，而不是完整的预处理器输出，同时删除系统头文件的提及。

> **Note:** 如果我们自己需要预处理器的输出，可以向编译器传递`-E`参数.
```cmake
        -MP
```    
此选项指示预处理器为除主文件以外的每个依赖项添加虚拟目标，以解决删除头文件而没有更新时构建系统出现的错误。
```cmake
        -MF "$(@:%.o=%.d)"
    )
```    
The last line specifies that we want the dependency files to be generated with the same name as the corresponding object file.
```cmake
    set(DEBUG_INFO_OPTIONS
        -g3
        -gdwarf-2
    )
```  

This creates a custom variable containing [compiler flags for generating debug information](https://gcc.gnu.org/onlinedocs/gcc/Debugging-Options.html). The first line tells the compiler to produce the debugging output and the level of detail while the second line configures the debug output format and version. We are using dwarf version 2 for best debugger compatibility.
```cmake
    add_compile_options(
        ${MCU_OPTIONS}
        ${EXTRA_OPTIONS}
        ${DEBUG_INFO_OPTIONS}
        ${DEPENDENCY_INFO_OPTIONS}
        ${OPTIMIZATION_OPTIONS}
    )
```    

Finally, all the created variables are used to set the global compiler options.

> **Note:** It is possible to override these per-target by using `target_compile_options()`, for instance to apply a higher optimization level to third party libraries we are not going to be debugging.

Linker options
--------------
```cmake
    add_link_options(
```    

Here we are adding global linker options.
```cmake
        ${MCU_OPTIONS}
```    

First, it is required to pass the previously created microcontroller specific flags variable created earlier.
```cmake
        -specs=nano.specs
```    

This tells the linker to include the nano variant of the [newlib](https://sourceware.org/newlib/) standard library which is optimized for minimal binary size and RAM use. The regular newlib variant is used simply by not passing this.
```cmake
        -T${CMAKE_SOURCE_DIR}/STM32F103C8Tx_FLASH.ld
```    

Here the linkerscript of the chip is passed. The linkerscript tells the linker where to store the objects in memory. I recommend reading the already mentioned [most thoroughly commented linker script](https://blog.thea.codes/the-most-thoroughly-commented-linker-script/) blogpost.
```cmake
        -Wl,-Map=${PROJECT_NAME}.map,--cref
```    

This directive instructs the linker to generate a mapfile. Mapfiles contain information about the final layout of the firmware binary and are an invaluable resource for development. I recommend reading [this excellent Interrupt blogpost](https://interrupt.memfault.com/blog/get-the-most-out-of-the-linker-map-file) for a quick introduction.
```cmake
        -Wl,--gc-sections
    )
```    

This is the linker flag for removing the unused sections from the final binary, it works in conjunction with the previously mentioned `-fdata-sections` and `-ffunction-sections` compiler flags to remove all unused objects from the final binary.

Linking the standard library
----------------------------
```cmake
    link_libraries("-lc -lm -lnosys")
```    

This call tells the linker to link the standard library components to all the libraries and executables added after the call.

The order of the standard library calls is important as the linker evaluates arguments one by one:

*   \-lc is the libc containing most standard library features
*   \-lm is the libm containing math functionality
*   \-lnosys provides stubs for the syscalls, essentially placeholders for what would be operating system calls

Adding library subprojects
--------------------------
```cmake
    add_subdirectory(Drivers)
```

When a subdirectory is added, the CMakeLists file contained in that subdirectory is evaluated, this is usually chained for composability. In this case the `Drivers` CMakeLists file creates a static library called `Drivers`.
```cmake
    target_include_directories(Drivers
    PRIVATE
        Core/Inc
    )
```    

We need to include the `Core/Inc` for the `Drivers` library, as it depends on the definitions inside the `Core` headers.

The executable
--------------
```cmake
    set(EXECUTABLE ${PROJECT_NAME}.elf)
```    

This creates a variable containing the executable name by appending the [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) extension to the project name.
```cmake
    add_executable(${EXECUTABLE}
        Core/Src/main.c
        Core/Src/stm32f1xx_hal_msp.c
        Core/Src/stm32f1xx_it.c
        Core/Src/system_stm32f1xx.c
        startup_stm32f103xb.s
    )
```    

Here we are creating the executable target and adding top-level sources to it. An executable is the final product of the build process, in this case our firmware. Note the startup file gets added along with the C sources.
```cmake
    target_include_directories(${EXECUTABLE}
    PRIVATE
        Core/Inc
    )
```    

Add the include directories for the executable. Since we don't need the includes to propagate up as our executable is the final product of the build process, we are including as `PRIVATE`.
```cmake
    target_compile_options(${EXECUTABLE}
    PRIVATE
        -Wall
        -Wextra
        -Wshadow
        -Wconversion
        -Wdouble-promotion
    )
```    

This adds [additional warning compiler flags](https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html) to our executable target, enabling them for our sources, but not third party libraries. All of these provide much more safety if not ignored.
```cmake
    target_link_libraries(${EXECUTABLE}
    PRIVATE
        Drivers
    )
```    

Here we link the libraries to the executable. Same principle applies to the reasoning behind `PRIVATE`.

Post-build commands
-------------------
```cmake
    add_custom_command(TARGET ${EXECUTABLE}
        POST_BUILD
        COMMAND ${CMAKE_SIZE_UTIL} ${EXECUTABLE}
    )
```    

This creates a custom command that prints out the firmware binary size information. Example:
```
    text   data    bss    dec    hex filename
    3432     20   1572   5024   13a0 most_commented_embedded_cmakelists.elf
```    

`text` is the code, `data` stores variables that have a non-zero initial value and have to be stored in flash, `bss` stores zero initial values that only take up ram. `dec` and `hex` are just the cumulative size in decimal and hexadecimal notation respectively.

[Here](https://mcuoneclipse.com/2013/04/14/text-data-and-bss-code-and-data-size-explained/) is a good resource for understanding these sections and to learn more about output options.
```cmake
    add_custom_command(TARGET ${EXECUTABLE}
        POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -O ihex ${EXECUTABLE} ${PROJECT_NAME}.hex
        COMMAND ${CMAKE_OBJCOPY} -O binary ${EXECUTABLE} ${PROJECT_NAME}.bin
    )
```    

This creates a custom command to generate binary and hex files. These can be used depending on which method of loading the firmware to the MCU is used.

Closing
-------

Hopefully this gives you an overview of how a minimal CMakeLists file for an embedded project looks like and peaked your curiosity if you've never used CMake for embedded projects.

It is worth keeping in mind that this article is not a complete guide to CMake and doesn't go into toolchain files, library CMakeLists and other things required for a complete CMake-based embedded project, however the [project on GitHub](https://github.com/DNedic/most_commented_embedded_cmakelists) used for demonstration contains all of those and can be used as a template.

* * *

© 2023 - Djordje Nedic | Find me on [LinkedIn](https://www.linkedin.com/in/djordje-nedic/) or [GitHub](https://github.com/DNedic)