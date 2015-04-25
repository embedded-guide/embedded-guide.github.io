---
layout: article
title: "Part 4: Clocks"
---
Before we get into the really interesting stuff, we need to cover an important, but somewhat tedious topic: clocks. All the activity within the MCU is coordinated by clocks. We're not talking about wallclock time here (we'll refer to that as the *real-time clock*, or *RTC*), but a simple signal that oscillates between high and low at a fixed frequency -- in other words, a square wave with a 50% *duty cycle*.

# The Clock Tree

Even though the processor core runs at speeds up to 100MHz, the clock source is much slower: 4-26MHz. It is multiplied up and divided down in a byzantine flow of *phase-locked loops (PLLs)* and dividers.

Here's the full clock tree from the reference manual:

![Clock tree](/images/clock-tree.png)

Various registers control how the clocks are routed through the system. It initially seems complex, but look at the common case below as an example.

![Clock tree](/images/clock-tree-core.png)

The clock begins with an external crystal oscillator (*HSE OSC*, where HSE stands for high-speed external) and goes to a multiplexer that selects either HSE or HSI (high-speed internal). From there, it goes to PLL, where it is divided by M, multiplied by N and divided by P. After the PPL, it goes to another multiplexer that chooses between the PPL output or using the HSE or HSI directly. The output of this mux is called the *system clock*. It goes through another divider and from there to the core. 

As well as routing clock signals, the clock-related registers also enabled and disable various clocks. This is for power-saving. Driving a clock signal around the whole processor and its perhipherals consumes a significant amount of power, and the higher the frequency, the more power is used, so for greater efficiency, two things are done: turning off the clocks that go to parts of the MCU that aren't being used; and dividing down the ones that are used by slower components as early in the chain as possible.

# Default configuration
When the MCU is powered up, the initial clock source is HSI, a built-in 16MHz oscillator that just uses resistors and capacitors. It's convenient and low-power, but not as accurate as an external crystal. It's used directly, not via the PLL, so the system clock is 16MHz.

There's an important reason the internal oscillator is the default: if HSE was the default and there wasn't an external clock source, it wouldn't be possible to switch to HSI, because nothing would be running!
