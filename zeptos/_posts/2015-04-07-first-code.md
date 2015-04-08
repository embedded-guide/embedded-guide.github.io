---
layout: post
title: "Part 2: A First Look at Code"
---
Let's take a look at the source code. Clone the following Git repository:

`git@github.com:embedded-guide/zeptos.git`

and check out the tag `part1`:

```sh
git clone git@github.com:embedded-guide/zeptos.git
cd zeptos
git checkout part1
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
static void *__vectors[16] __attribute__((section(".vectors"), unused)) = {
    &__stack_end,
    __start
};
```

I'll explain the `__attribute__` bit when I explain the linker script. For now, on to the far more interesting `src/main.c`.

## main.c

It starts by defining some pointers. 

```c
// Registers we'll be using in our minimal example
static volatile uint32_t *RCC_AHB1ENR = (uint32_t *) 0x40023830;
static volatile uint32_t *GPIOA_MODER = (uint32_t *) 0x40020000;
static volatile uint32_t *GPIOA_ODR = (uint32_t *) 0x40020014;
```

There are pointers to some of those memory-mapped registers. The names, which are defined in the MCU's reference manual, are very cryptic, but that's to keep them short. These are the ones we need to use to toggle a pin on a GPIO peripheral.

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

If you haven't done embedded programming before, it's likely you've never encountered a linker script before. Usually they're hidden away and used automatically, but in a project like this, they give us the control we need to get the things we need in the right places in the firmware blob we upload.

```
__stack_size = 1024;
```

```
MEMORY {
    flash(rx) : org = 0x08000000, len = 512k
    sram(rwx) : org = 0x20000000, len = 128k
}
```

```
SECTIONS {
    .bss : {
        __bss_start = .;
        *(.bss .bss.*);
        __bss_end = .;
    } > sram
```

```
    .text : {
        KEEP(*(.vectors));
        *(.rodata .rodata.*);
    } > flash
```

```
    .data : {
        __data_init_start = LOADADDR(.data);
        __data_start = .;
        *(.data .data.*);
        __data_end = .;
    } > sram AT > flash
```

```
    .stack : {
        . += __stack_size;
        __stack_end = .;
    } > sram
}
```
