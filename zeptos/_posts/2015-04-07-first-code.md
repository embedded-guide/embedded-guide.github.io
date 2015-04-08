---
layout: post
title: "Part 2: A First Look at Code"
---
Let's take a look at the source code. Clone the following Git repository:

`git@github.com:embedded-guide/zeptos.git`

and check out the tag `part2`:

```sh
git clone git@github.com:embedded-guide/zeptos.git
cd zeptos
git checkout part2
```

This is a tiny test project that we'll use to get our toolchain working. The first file we'll look at is `src/start.c`.

## start.c

It starts with some `extern` declarations.

```c
extern uint32_t __bss_start;
extern uint32_t __bss_end;
extern uint32_t __data_init_start;
extern uint32_t __data_start;
extern uint32_t __data_end;
extern uint32_t __stack_end;
```

They're actually defined in the *linker script* we'll look at later, and they relate to the layout of our binary file.

Next is a declaration of a `main` function. Note that it returns `void`, not `int`.

```c
extern void main(void);
```

Because we're programming directly 'to the metal', there's no operating system here to return an exit status to, so we don't need a return value. Besides, we could have called it anything. Now that we've abandoned the standard C runtime, `main` no longer has any special meaning.

Finally, we get to the first bit of real code.

```c
static void __start(void) {
    // Zero out .bss section
    for (uint32_t *ptr = &__bss_start; ptr < &__bss_end; ++ptr) {
        *ptr = 0;
    }

    // Initialize .data section
    for (uint32_t *src = &__data_init_start, *dest = &__data_start; dest < &__data_end;) {
        *dest++ = *src++;
    }

    main();
}
```

It's just writing zeros and copying words from one location to another. Why? Remember, the `.bss` section was implicit, so we need to do the work of filling it with zeros, so that any code that expects an uninitialized non-local variable to be zero will not be disappointed. Normally in a C program, this would be done by the C runtime code that is supplied by the compiler. But why is the `.data` section copied? When the linker links all our object files together, it does so as though all variables are in the memory space backed by SRAM. But the code we uploaded is in flash. This second loop simply copies the initial values from flash to their expected locations in SRAM. The call to `main` just jumps into the rest of our code, which is expected to never return.

This last definition is actually the table referred to above. The first entry is the top of the stack, and the second is the address of our `__start` function.

```c
static void *__vectors[] __attribute__((section(".vectors"), unused)) = {
    &__stack_end,
    __start
};
```

I'll explain the `__attribute__` bit when I explain the linker script. For now, on to the far more interesting `src/main.c`.

## main.c

It starts by defining some pointers. 

```c
// Registers we'll be using in our minimal example
static volatile uint32_t * const RCC_AHB1ENR = (uint32_t *) 0x40023830;
static volatile uint32_t * const GPIOA_MODER = (uint32_t *) 0x40020000;
static volatile uint32_t * const GPIOA_ODR = (uint32_t *) 0x40020014;
```

There are pointers to some of those memory-mapped registers. The names, which are defined in the MCU's reference manual, are very cryptic, but that's to keep them short. These are the ones we need to use to toggle a pin on a GPIO peripheral. Note the position of the `const`: we're saying that the pointer itself is a constant, not that we're pointing to a constant. This tells the linker that the pointer can just be in flash, which saves some SRAM space.

```c
void main(void) {
    // Enable the GPIO Port A peripheral
    *RCC_AHB1ENR = 1;
```

`RCC` stands for *Real-time Clock Controller*, `AHB` is *AMBA High-performance Bus*, and `ENR` is *enable register*. The GPIO port is a peripheral attached to the AHB bus, and we need to enabled its clock. We're setting a single bit in this register, because at the moment we only care about enabling this one peripheral.

```c
    // Put pin PA5 in output mode
    *GPIOA_MODER |= 1 << 10;
```

`GPIOA_MODER` is the *GPIO port A mode register*. Each pin on port A has two bits that control its mode (input or output). Because pin PA5 is the sixth pin on that port (they're numbered from PA0), we're setting the sixth pair of bits to 01, which means "make PA5 an output".

Now that the port is enabled and the pin is configured as an output, we can start toggling it.

```c
    for (;;) {
        // Toggle pin PA5
        *GPIOA_ODR ^= 1 << 5;

        // Spend some time doing nothing
        for (int i = 1000000; i; --i);
    }
}
```

`GPIOA_ODR` stands for *GPIO port A output data register*. Ones in this register set the respective output pins high, which in the case of PA5 lights the LED. Here we're using the exclusive-OR operator to toggle it. If we did it without a delay, though, it would happen so quickly we wouldn't see the flashing, so we use an empty loop to slow it down to approximately one flash per second, or 1Hz. This happens in an infinite loop, because there's nowhere to return to if our functions exit.

## link.ld

If you haven't done embedded programming before, it's likely you've never encountered a linker script before. Usually they're hidden away and used automatically, but in a project like this, they give us the control we need to get the things we need in the right places in the firmware blob we upload. To quote [the ld manual](https://sourceware.org/binutils/docs/ld/Scripts.html):

> The main purpose of the linker script is to describe how the sections in the input files should be mapped into the output file, and to control the memory layout of the output file.
 
They look intimidating at first, and they can get quite complex, but the one I've written here has been pared down to a more manageable size. It doesn't include, for example, sections used by C++.

```
__stack_size = 1024;
```

This is just a variable for our own use, so that if we change the size of the stack we want our OS to use, we don't have to go searching through the file for it.

```
MEMORY {
    flash(rx) : org = 0x08000000, len = 512k
    sram(rwx) : org = 0x20000000, len = 128k
}
```

The `MEMORY` command defines output sections that correspond to the areas of memory that are present on the target device, and their properties. You should recognize the start (or *origin*) addresses from [Part 1](../nucleo-board). The lengths come from the memory sizes given in the datasheet. The attributes, `r`, `w` and `x`, mean readable, writeable and executable respectively. Both memory areas are readable, and code can be executed from either, but the flash memory is read-only. Flash memory can be written to, of course, because we're putting our own code in it, but because of the particular technology it uses, writing to it involves going through a process of reading a block, modifying the bytes that need to change, erasing the block, and then writing the new block. The programmer takes care of that when we upload our code, but it does mean that it can't be written to as a memory-mapped device. It can be read as one just fine, though.

```
SECTIONS {
    .text : {
        KEEP(*(.vectors));
        *(.text);
        *(.rodata .rodata.*);
    } >flash
```

The `SECTIONS` command defines mappings from input sections (in the object files that GCC builds) to the output sections, and specifies which of the above regions the section goes in. This one generates an output `.text` section that contains: the `vectors` array we defined in `start.c`; all the actual ARM code; read-only data. Because this section comes first and is assigned to the `flash` region, it will come first in our firmware binary. That places the vector table at the beginning of flash, right where the MCU is expecting it.

```
    .data : {
        __data_init_start = LOADADDR(.data);
        __data_start = .;
        *(.data .data.*);
        __data_end = .;
    } >sram AT >flash
```

This output section contains the initialized data. Because it's writeable, it needs to be loaded into flash and copied into SRAM at boot time. The `>sram` attribute causes the program to be linked as though the data was at an address in SRAM, while the `AT >flash` attribute specifies that it also needs to be included in the flash region at a different address. `LOADADDR(.data)` gives us the address in flash that this section will have, so we assign that to the symbol we use in our initialization code. Two more symbols, `__data_start` and `__data_end` point to the start and end SRAM addresses of where the data will be copied to.

```
    .bss : {
        __bss_start = .;
        *(.bss .bss.*);
        __bss_end = .;
    } >sram
```

This one says that zero-initialized non-local variables live in SRAM. We also define symbols, `__bss_start` and `__bss_end` that mark the beginning and end of the BSS section. The BSS sections that GCC generates do not have a `LOAD` flag, so although space is reserved for these zero-initialized variables, all those zeros are not needlessly included in our firmware image.

```
    .stack (NOLOAD) : ALIGN(4) {
        __stack_top = .;
        . += __stack_size;
        __stack_end = .;
    } >sram
}
```

There's no input section for our stack, so we have to explicitly specify that it shouldn't be included in the image (`NOLOAD`) and that it should be aligned on a word boundary.

## Makefile

Admittedly, Makefiles can get a little hairy, but at least for now we're going to stick it out, because they're very explicit about what's happening. Here are the highlights. One thing to be careful of: the indentations are tabs, not spaces. Make is peculiarly picky about that.

```makefile
SRC := $(shell find src -name *.c -print)
```

This populates the variable `SRC` with the names of all our C source files.

```makefile
CPU_FLAGS = -mcpu=cortex-m4 -mthumb

CC_FLAGS = -std=gnu99 $(CPU_FLAGS) -Wall -ffreestanding -fno-common -fstack-usage
LD_FLAGS = -nostdlib -Wl,-Map=$(TARGET).map -T link.ld
```

`-mcpu=cortex-m4` and `-mthumb` specify what kind instructions are generated by the compiler. Originally, all ARM instructions were 32 bits long. Thumb is a compact 16-bit instruction set. The Cortex-M4 supports Thumb-2, which extends Thumb with some additional 32-bit instructions.

We tell the linker that we don't want to link with the standard library, that we want it to output a map file, and that we have our own linker script to use.

```makefile
all: $(TARGET).bin
```

The final target is a flat binary file that is our firmware image that we can upload to the MCU's flash memory.

```makefile
$(TARGET).elf: $(OBJ) link.ld
    $(CC) $(LD_FLAGS) -o $@ $(OBJ)
    $(SIZE) -Ax $@
```

When the ELF file is built, the `size` utility lists the sections with their sizes and addresses.

```makefile
$(TARGET).bin: $(TARGET).elf
    $(OBJCOPY) -O binary $< $@
```

The binary file is created by copying the contents of each section of the ELF file that has the `LOAD` attribute, ordered by its address.

```makefile
.PHONY: flash
flash: $(TARGET).bin
    @if [ -z "$(MBED_PATH)" ]; then echo You must define MBED_PATH as the path to the root of the mbed mass storage; exit 1; fi
    $(COPY) $< $(MBED_PATH)
```

In order to flash our image to the MCU, we need to know the path to the mbed mass stoage. It will be different on different host platforms, so rather than hardcode it in the Makefile, it's taken from the environment. This Makefile will abort if it's not set.
