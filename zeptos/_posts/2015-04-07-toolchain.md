---
layout: post
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

In [part 3](../first-code), we'll look at the minimum amount of code we need to get the board's LED flashing.
