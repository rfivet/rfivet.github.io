# Bootstrap

## Revising the link script
To validate the toolchain, I picked up **mem.ld**, the simplest sample link
script, and used it as it is.

```
/* Linker script to configure memory regions.
 * Need modifying for a specific board.
 *   FLASH.ORIGIN: starting address of flash
 *   FLASH.LENGTH: length of flash
 *   RAM.ORIGIN: starting address of RAM bank 0
 *   RAM.LENGTH: length of RAM bank 0
 */
MEMORY
{
  FLASH (rx) : ORIGIN = 0x0, LENGTH = 0x20000 /* 128K */
  RAM (rwx) : ORIGIN = 0x10000000, LENGTH = 0x2000 /* 8K */
}
```

It needs to be modified with actual flash and ram values.

Also this link script does not contain any information for the linker to know
how to locate the output of the C compiler. Code, constant data and initial
value of variables need to be located in flash, variables and stack need to be
located in ram. We need a better link script that specify all of that.
**nokeep.ld** in the sample scripts folder is the one we need.

```
/* Linker script to configure memory regions.
 * Need modifying for a specific board.
 *   FLASH.ORIGIN: starting address of flash
 *   FLASH.LENGTH: length of flash
 *   RAM.ORIGIN: starting address of RAM bank 0
 *   RAM.LENGTH: length of RAM bank 0
 */
MEMORY
{
  FLASH (rx) : ORIGIN = 0x0, LENGTH = 0x20000 /* 128K */
  RAM (rwx) : ORIGIN = 0x10000000, LENGTH = 0x2000 /* 8K */
}

/* Linker script to place sections and symbol values. Should be used together
 * with other linker script that defines memory regions FLASH and RAM.
 * It references following symbols, which must be defined in code:
 *   Reset_Handler : Entry of reset handler
 *
 * It defines following symbols, which code can use without definition:
 *   __exidx_start
 *   __exidx_end
...
...
 *   __StackTop
 *   __stack
 */
ENTRY(Reset_Handler)

SECTIONS
{
...
...
```

From this snippet we can see that not only flash and ram parameters but also
the entry point for code execution, `Reset_Handler`, needs to be provided.

As a check, let’s change the link script to **nokeep.ld** in **Makefile** and
generate an executable **.elf** from the empty source code file **empty.c**:

```
LD_SCRIPT = $(GCCDIR)/share/gcc-arm-none-eabi/samples/ldscripts/nokeep.ld
```

```
$ make empty.elf
empty.elf
D:\Program Files (x86)\GNU Arm Embedded Toolchain\9 2020-q2-update\bin\arm-none-
eabi-ld.exe: warning: cannot find entry symbol Reset_Handler; defaulting to 0000
0000
rm empty.o
```

The linker gives a warning and fallback on a default address as entry point.

So let’s create **boot.c** with an idle loop as **Reset_Handler**:

```c
void Reset_Handler( void) {
    for( ;;) ;
}
```
In order to better understand the output of the link phase, I make the
following changes to the **Makefile**.

- Add debug information by passing the **-g** flag to the C compiler.
- Call the linker with the flags **-Map=$*.map -cref** to produce a link map
that also includes a cross reference table.
- List the size of the sections by using the **size** command.
- call **objdump -hS** to generate a disassembly listing that include a list
of the sections and make use of the debug info to mix C source with assembly
code.
- Insure that the **clean** rule removes the newly generated **.map** and
**.lst** files.

```make
OBJDUMP = $(BINPFX)objdump
SIZE    = $(BINPFX)size

CFLAGS = -g

clean:
    @echo CLEAN
    @rm -f *.o *.elf *.map *.lst *.bin *.hex

%.elf: %.o
    @echo $@
    $(LD) -T$(LD_SCRIPT) -Map=$*.map -cref -o $@ $<
    $(SIZE) $@
    $(OBJDUMP) -hS $@ > $*.lst
```

```
$ make boot.elf
boot.elf
   text    data     bss     dec     hex filename
     12       0       0      12       c boot.elf
rm boot.o
```

We can see that this build results in 12 bytes of code in the text section.
More details can be found in the **boot.map** and **boot.lst** files.

## Targeting the STM32F030F4P6
To be able to build a boot code that could bootstrap a board equipped with a
STM32F030F4P6, we need to know the following about this micro-controller:

- **Core**: Arm 32 bit **Cortex-M0** CPU.
- 16KB **Flash** located at 0x08000000.
- 4KB **Ram** located at 0x20000000.

I make a copy of the sample **nokeep.ld** link script in my working folder
under the name **f030f4.ld** and change the **MEMORY** region accordingly.

```
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 16K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH =  4K
}
```
The **Makefile** needs the following changes:

- Specify f030f4.ld as the link script
- C compiler flags to generate thumb Cortex-M0 code.
- Request the C compiler to generate extra warning and optimize for size.

```
CPU = -mthumb -mcpu=cortex-m0
CFLAGS = $(CPU) -g -Wall -Wextra -Os
LD_SCRIPT = f030f4.ld
```

At boot time, the Arm core fetches the initial address of the stack pointer
and the address where to start execution from the first two entries of its
interrupt routine table. We have to modify **boot.c** to initialize such a
table in accord with the symbols defined in the link script.

```c
/* Memory locations defined by linker script */
extern long __StackTop ;        /* &__StackTop points after end of stack */
void Reset_Handler( void) ;     /* Entry point for execution */

/* Interrupt vector table:
 * 1  Stack Pointer reset value
 * 15 System Exceptions
 * NN Device specific Interrupts
 */
typedef void (*isr_p)( void) ;
isr_p const isr_vector[ 2] __attribute__((section(".isr_vector"))) = {
    (isr_p) &__StackTop,
/* System Exceptions */
    Reset_Handler
} ;

void Reset_Handler( void) {
    for( ;;) ;
}
```

**__StackTop** is defined by the linker script and is located after the end of
the RAM. I use the GNU C extension **__attribute__()** to name the section
where I want the interrupt vector to be included. If you check the linker
script you will see that it places **.isr_vector** at the start of the **text**
area which is located at the beginning of the flash memory. I chose to name the
interrupt vector table **isr_vector** to match the section name **.isr_vector**,
but it is really the section name that matters here.

```
$ make boot.hex
boot.elf
   text    data     bss     dec     hex filename
     10       0       0      10       a boot.elf
boot.hex
rm boot.elf boot.o
```

A build produce 10 bytes of code, we can check the disassembly in the
**boot.lst** file.

```
Disassembly of section .text:

08000000 <isr_vector>:
 8000000:   00 10 00 20 09 00 00 08                             ... ....

08000008 <Reset_Handler>:
/* System Exceptions */
    Reset_Handler
} ;

void Reset_Handler( void) {
    for( ;;) ;
 8000008:   e7fe        b.n 8000008 <Reset_Handler>
```

- The interrupt vector table is at address **0x08000000**, the beginning of the
flash.
- First entry, the initial stack pointer value, is the address **0x20001000**,
which is the first location after the end of the 4K RAM.
- Next entry is **0x08000009**. We expect **0x08000008** here, as this is the
first location after the small interrupt vector table we created. The lowest
bit is set to 1 to mark that it is indeed the address of an interrupt routine.
So the value is correct even if an odd address value is not possible for an
opcode location.
- Finally, we found the code for our idle loop at **0x08000008** as expected.

## Checkpoint
I have built a first executable targeting a member of the STM32 family.

[Next](13_flash) I will take a board with a
STM32F030F4P6 and check if this code behaves as expected.

Below is the **Makefile** for reference. If you happen to cut&paste from this
web page to a file, remember that **GNU make** expects rules to be tab indented.

```make
### Build environment selection

ifeq (linux, $(findstring linux, $(MAKE_HOST)))
#GCCDIR = ~/Packages/gcc-arm-none-eabi-9-2019-q4-major
 GCCDIR = ~/Packages/gcc-arm-none-eabi-9-2020-q2-update
else
#GCCDIR = "D:/Program Files (x86)/GNU Tools ARM Embedded/9 2019-q4-major"
 GCCDIR = "D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"
endif

BINPFX  = @$(GCCDIR)/bin/arm-none-eabi-
CC      = $(BINPFX)gcc
LD      = $(BINPFX)ld
OBJCOPY = $(BINPFX)objcopy
OBJDUMP = $(BINPFX)objdump
SIZE    = $(BINPFX)size

CPU = -mthumb -mcpu=cortex-m0
CFLAGS = $(CPU) -g -Wall -Wextra -Os
LD_SCRIPT = f030f4.ld

### Build rules

.PHONY: clean

clean:
    @echo CLEAN
    @rm -f *.o *.elf *.map *.lst *.bin *.hex

%.elf: %.o
    @echo $@
    $(LD) -T$(LD_SCRIPT) -Map=$*.map -cref -o $@ $<
    $(SIZE) $@
    $(OBJDUMP) -hS $@ > $*.lst

%.bin: %.elf
    @echo $@
    $(OBJCOPY) -O binary $< $@

%.hex: %.elf
    @echo $@
    $(OBJCOPY) -O ihex $< $@
```

___
© 2020-2021 Renaud Fivet
