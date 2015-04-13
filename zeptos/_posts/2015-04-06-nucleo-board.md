---
layout: post
title: "Part 1: The ST Micro Nucleo STM32F411 Board"
---
<img style="float: right" src="/images/nucleo411.jpg" alt="Nucleo STM32F411">
Before we look at any code, it will be useful to get a little more familiar with the hardware. This is a [Nucleo STM32F411 development board](http://www.st.com/web/catalog/tools/FM116/SC959/SS1532/LN1847/PF260320) from ST Microelectronics, and its purpose is to make it easy to develop using the STM32F411RET6 microcontroller. The useful pins are brought out to both male and female headers, with the female ones being Arduino-compatible. Attached to the main part of the board is a (detachable) programmer/debugger with a USB interface.

The microcontroller, or MCU, is based around an ARM Cortex-M4 core, which is surrounded by various peripherals, some static RAM (SRAM) and some flash memory. The core is used under license from ARM, but ST have implemented their own peripherals, such as general purpose I/O (GPIO) and various flavors of serial port (UART, SPI, I<sup>2</sup>C, etc). One of the GPIO pins has a small, green LED, which we can use for simple testing without having to attach any external components. In fact, for the first few parts of this series, the only extra hardware you'll need is a mini-USB cable.

Because this is an [mbed](http://mbed.org/) compatible board, the USB interface presents itself as three devices: a mass storage device, a virtual serial port and a proprietary ST-Link interface. Before mbed, each vendor would have their own protocol used for programming the MCU's flash memory, where the firmware is stored, which meant finding (and often building from source) the appropriate utility for your host platform. With mbed, you can simply copy the binary file to the device as though it were a USB stick. The virtual serial port is, by default, connected to one of the MCU's serial peripherals, and we'll be able to use it for debugging.

The flash memory is permanent storage just like an SD card. That's where we'll put our code so that it's available every time the board is powered up. The SRAM is volatile memory, but it's faster. That's where our code will keep the data it uses while it's running.

If you've ever looked inside an executable file you've built on Linux, you might have seen that it has several sections with names like `.text`, `.data` and `.bss`. That's because the compiler separates code and data and puts them into those sections in the final *ELF* format executable.

* `.text`: This contains the executable code.
* `.data`: This contains the initial values of non-local variables.
* `.bss`: This section kind of contains all the zero-initialized non-local variables, except there's no point in storing a lot of zeros, so they're just implied instead.

Microcontrollers don't understand the ELF format, though; in fact, all the extra metadata would be a waste of space. Instead, just the bits we actually need are extracted out into a flat binary file. Later, we'll see how we can control what gets put into that file and in what order. That's important, because the MCU expects the very first part of flash to contain a table of pointers. The first, at offset 0, tells it what the initial value of the stack pointer should be. Remember, we're completely in control here, so we get to use the SRAM however we choose, and that means we get to decide where to put the stack. The second pointer, at offset 4 (the pointers are 32-bits wide), is a pointer to something called the reset handler. It's particularly important, because when the MCU starts up (either from power-up or a reset), it takes that address and jumps to it. So to get our code running at boot time, we just put its address there. We'll see in a later part of this series that the table is actually much larger than these two words.

Our first objective is the get the on-board LED to flash. It's connected to GPIO port PA5. To make it flash, we need to toggle that I/O line high and low. As with all peripherals, this is accomplished by writing values to *registers*. These are not the registers that are internal to the CPU core; they're memory-mapped registers, which means we access them as though we were reading from and writing to memory locations. Because pointers are 32 bits wide, the full addressable space is 4GB, but of course only a fraction of that is actual memory. Here's what the rest of it looks like:

![Memory map](/images/stm32f411-memory-map.png)

You can see the flash memory starts at 0x08000000, the SRAM starts at 0x20000000 and peripheral registers are mapped into memory space starting at 0x40000000. APB1, APB2, AHB1 and AHB2 are the different peripheral buses.

In [part 2](../tookchain), we'll look at the tools we need to build our code.
