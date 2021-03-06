---
layout: article
title: "Part 0: Introduction"
---
Last year, I found myself with a good excuse to dive into writing some embedded software. Like many people, I'd started out with an [Arduino](http://www.arduino.cc/), using their Wiring programming environment. After some long evenings cozying up to Atmel datasheets, I felt comfortable abandoning the Arduino IDE and writing everything from the ground up as a learning exercise. When I felt confident enough with that, I moved on to ARM microcontrollers. They're a little more difficult to initialize, but I have a nostalgic fondness for them having learned ARM programming on an ARM 2 computer ([Acorn A3000](http://en.wikipedia.org/wiki/Acorn_Archimedes#A3000_and_A5000)) in my early teens.

But they say you don't truly understand something unless you can explain it to others, so in this series of articles, I'm going to use what I've learned about microcontroller programming to demonstrate the basics of creating a small embedded operating system. I'll start from getting some minimal LED-flashing code running, and then add the features necessary to run useful tasks, such as:

* Initialization
* Task switching
* Synchronization
* Message passing
* Interrupt handling
* Peripheral drivers

Although I'll be implementing it in C, this isn't a lesson in C programming. I'll assume you're already familiar with it, but I'll point out things that you might not have used outside of embedded programming.

If you're already at all familiar with what an operating system does, you'll know that it runs at the lowest level, directly controlling the processor. There are many kinds of OS, and one way in which they differ is in how much functionality is contained within the core of the OS, called the *kernel*. *Monolithic kernels* as found in most Unix-like operating systems contain all of the core functionality and device drivers. *Microkernels*, such as Mach and L4 contain a small subset of functionality, such as task switching, memory protection and IPC, and device drivers are services than run in user space. Some operating systems, such as Windows and Mac OS X use elements of both of these, with services running in kernel space, but on top of the NT and XNU microkernels respectively.

The kind of OS we'll be developing is called an *exokernel*. It's similar to a microkernel in that it contains only core functionality, not device drivers. Where it differs, though, is that instead of drivers being implemented as services, they are libraries linked to the running application. This approach works well here, because a microcontroller is usually dedicated to only one application, and the less featured memory controller means we'll only be using minimal memory protection, so libraries linked to tasks will be able to directly access hardware.

[Part 1](part1-nucleo-board.html) introduces the hardware I'll be using, which is an [ST Nucleo F411RE](http://www.st.com/web/catalog/tools/FM116/SC959/SS1532/LN1847/PF260320) development board (they're very cheap at $10, and they include an on-board USB programmer).

[Part 2](part2-toolchain.html) will get the GCC and Make toolchain working. Yes, I know there are lots of nicer alternatives to Make, but for the purposes of this series, Makefiles have the advantages of being very explicit. You'll know exactly what's going on at each step and why.

[Part 3](part3-first-code.html) will go through the code required to get the on-board LED blinking.

[Part 4](clocks.html) describes the all-important clock tree

# Parts still to come

These upcoming parts are planned or in progress.

* Part 5: The GPIO Driver
* Part 6: The UART Driver
* Part 7: Task Switching
* Part 8: Blocking Tasks
* Part 9: Message Passing

# Documentation

You'll want to have the following documents handy, from ST:

* [User manual for the Nucleo F411RE board](http://www.st.com/st-web-ui/static/active/en/resource/technical/document/user_manual/DM00105823.pdf)
* [Datasheet for the STM32F411 microcontroller itself](http://www.st.com/web/en/resource/technical/document/datasheet/DM00115249.pdf)
* [Reference manual for the STM32F411 microcontroller](http://www.st.com/web/en/resource/technical/document/reference_manual/DM00119316.pdf)

And from ARM:

* [Cortex-M4 Devices Generic User Guide](http://infocenter.arm.com/help/topic/com.arm.doc.dui0553a/DUI0553A_cortex_m4_dgug.pdf)
* [ARM Cortex-M4 Technical Reference Manual](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0439b/DDI0439B_cortex_m4_r0p0_trm.pdf)
* [ARM v7-M Architecture Reference Manual](https://silver.arm.com/download/ARM_and_AMBA_Architecture/AR580-DA-70000-r0p0-05rel0/DDI0403E_B_armv7m_arm.pdf)

The reference manual is going to be our main source of information. It gives in depth details of all the registers that are used to control the microcontroller.

By the way, the OS we're going to be building is called Zeptos, from *zepto*, which is a prefix for the very, very small.
