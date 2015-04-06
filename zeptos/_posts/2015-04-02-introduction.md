---
layout: post
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

Part 1 of this series will get us a working environment that allows us to compile, link and upload a firmware image. I'll be using an [ST Nucleo F411RE development board](http://www.st.com/web/catalog/tools/FM116/SC959/SS1532/LN1847/PF260320) (they're very cheap at $10, and they include an on-board USB programmer), and GCC and GNU Make. Yes, I know there are lots of nicer alternatives to Make, but for the purposes of this series, Makefiles have the advantages of being very explicit. You'll know exactly what's going on at each step and why.

By the way, the OS we're going to be building is called Zeptos, from *zepto*, which is a prefix for the very, very small.
