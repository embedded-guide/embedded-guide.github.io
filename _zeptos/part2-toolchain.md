---
layout: article
title: "Part 2: The Toolchain"
---
The process of building firmware isn't too different to building a native executable. We'll use `gcc` and `ld` to compile and link C sources, but it will be a *cross-compiler* that outputs ARM code instead of code for our host platform (usually x86 or x86-64). We'll also use GCC's assembler to assemble the few bits of code we'll need to write in ARM code. (Don't worry, there isn't much ARM code involved, and it's much nicer than x86 assembly anyway.)

You might have the compiler available as a package. In Homebrew on OS X and on Ubuntu, for example, it's `gcc-arm-none-eabi`. You can always download the latest version from the [GCC ARM Embedded](https://launchpad.net/gcc-arm-embedded) homepage.

You'll also need GNU Make installed.

Here are all the tools we'll use and what each one does:

* **`gcc`** Compile C sources into ELF object files.
* **`as`** Assemble ARM assembly code into ELF object files.
* **`ld`** Link several object files into one ELF executable file.
* **`objcopy`** Transform the ELF executable into a flat binary or hex file.
* **`make`** Build our code according to rules we define in a `Makefile`.

There are also some non-essential tools that are still useful:

* **`objdump`** Examine the headers or contents of sections of ELF files.
* **`nm`** List symbols defined or referenced in ELF files.
* **`size`** Show the size of text, data and BSS sections in an ELF file.

Apart from `make`, these are prefixed with the target platform, so that GCC executable that we'll actually be using is `arm-none-eabi-gcc`.

## ELF files

When GCC compiles source code, it builds it into an executable format called ELF (executable linkable format). 
If you've ever looked inside an executable file you've built on Linux, you might have seen that it has several sections with names like `.text`, `.data` and `.bss`. That's because the compiler separates code and data and puts them into those sections in the final *ELF* format executable.

* `.text`: This contains the executable code.
* `.data`: This contains the initial values of non-local variables.
* `.bss`: This section kind of contains all the zero-initialized non-local variables, except there's no point in storing a lot of zeros, so they're just implied instead.

Microcontrollers don't understand the ELF format, though; in fact, all the extra metadata would be a waste of space. Instead, just the bits we actually need are extracted out into a flat binary file. Later, we'll see how we can control what gets put into that file and in what order. That's important, because the MCU expects the very first part of flash to contain a table of pointers. The first, at offset 0, tells it what the initial value of the stack pointer should be. Remember, we're completely in control here, so we get to use the SRAM however we choose, and that means we get to decide where to put the stack. The second pointer, at offset 4 (the pointers are 32-bits wide), is a pointer to something called the reset handler. It's particularly important, because when the MCU starts up (either from power-up or a reset), it takes that address and jumps to it. So to get our code running at boot time, we just put its address there. We'll see in a later part of this series that the table is actually much larger than these two words.

In [part 3](../first-code), we'll look at the minimum amount of code we need to get the board's LED flashing.
