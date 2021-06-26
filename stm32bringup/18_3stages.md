# 1.8: Three-stage Rocket

As I merge the cstartup with the ledtick code, I split the
functionalities between three files according to the three phases: boot,
initialization and main execution.

- **startup.c** is mainly concerned with the early set up of the board:
initializing the memory, calling system initialization and then
executing the main C function if initialization is successful.

- **init.c** holds the middleware implementation and will abstract the
peripherals into higher level interfaces. Its entry point is the `init()`
function.

- the `main()` C function will be the focus of the last file and it should
be written in a subset of standard C. As an example I will use
**success.c**.

```c
/* success.c -- success does nothing, successfully */
#include <stdlib.h>

int main( void) {
    return EXIT_SUCCESS ;
}
```

## startup.c

Beside the interrupt vector and the `Reset_Handler()` that calls `init()` and
`main()`, I have created stubs for all the System Exceptions listed in the
Programming Manual. For those, if there is no implementation avalable in
**init.c**, the linker will use the `Default_Handler()` provided with the
`__attribute__()` Gnu C extension.

```c
/* Memory locations defined by linker script */
extern long __StackTop ;        /* &__StackTop points after end of stack */
void Reset_Handler( void) ;     /* Entry point for execution */
extern const long __etext[] ;   /* start of initialized data copy in flash */
extern long __data_start__[] ;
extern long __bss_start__[] ;
extern long __bss_end__ ;       /* &__bss_end__ points after end of bss */

/* Stubs for System Exception Handler */
void Default_Handler( void) ;
#define dflt_hndlr( fun) void fun##_Handler( void) \
                                __attribute__((weak,alias("Default_Handler")))
dflt_hndlr( NMI) ;
dflt_hndlr( HardFault) ;
dflt_hndlr( SVCall) ;
dflt_hndlr( PendSV) ;
dflt_hndlr( SysTick) ;

/* Interrupt vector table:
 * 1  Stack Pointer reset value
 * 15 System Exceptions
 * NN Device specific Interrupts
 */
typedef void (*isr_p)( void) ;
isr_p const isr_vector[ 16] __attribute__((section(".isr_vector"))) = {
    (isr_p) &__StackTop,
/* System Exceptions */
    Reset_Handler,
    NMI_Handler,
    HardFault_Handler,
    0,  0,  0,  0,  0,  0,  0,
    SVCall_Handler,
    0,  0,
    PendSV_Handler,
    SysTick_Handler
} ;

extern int init( void) ;
extern int main( void) ;

void Reset_Handler( void) {
    const long  *f ;    /* from, source constant data from FLASH */
    long    *t ;        /* to, destination in RAM */

/* Assume:
**  __bss_start__ == __data_end__
**  All sections are 4 bytes aligned
*/
    f = __etext ;
    for( t = __data_start__ ; t < __bss_start__ ; t += 1)
        *t = *f++ ;

    while( t < &__bss_end__)
        *t++ = 0 ;

    if( init() == 0)
        main() ;

    for( ;;)
        __asm( "WFI") ; /* Wait for interrupt */
}

void Default_Handler( void) {
    for( ;;) ;
}
```

Except for the future addition of stubs for the device specific
interrupts, this file will not grow much anymore.

## init.c

This is the embryo of an hardware abstraction layer where most of the
device specific code will be added. The current code is the peripherals
part of **ledtick.c**.

```c
#define SYSTICK             ((volatile long *) 0xE000E010)
#define SYSTICK_CSR         SYSTICK[ 0]
#define SYSTICK_RVR         SYSTICK[ 1]
#define SYSTICK_CVR         SYSTICK[ 2]

#define RCC                 ((volatile long *) 0x40021000)
#define RCC_AHBENR          RCC[ 5]
#define RCC_AHBENR_IOPBEN   0x00040000  /*  18: I/O port B clock enable */

#define GPIOB               ((volatile long *) 0x48000400)
#define GPIOB_MODER         GPIOB[ 0]
#define GPIOB_ODR           GPIOB[ 5]

int init( void) {
/* By default SYSCLK == HSI [8MHZ] */

/* SYSTICK */
    SYSTICK_RVR = 1000000 - 1 ;     /* HBA / 8 */
    SYSTICK_CVR = 0 ;
    SYSTICK_CSR = 3 ;               /* HBA / 8, Interrupt ON, Enable */
    /* SysTick_Handler will execute every 1s from now on */

/* User LED ON */
    RCC_AHBENR |= RCC_AHBENR_IOPBEN ;   /* Enable IOPB periph */
    GPIOB_MODER |= 1 << (1 * 2) ;       /* PB1 Output [01], over default 00 */
    /* OTYPER Push-Pull by default */
    /* PB1 output default LOW at reset */

    return 0 ;
}

void SysTick_Handler( void) {
    GPIOB_ODR ^= 1 << 1 ;   /* Toggle PB1 (User LED) */
}
```

## Makefile

As I now build from multiple source files, I have modified the
**Makefile** to list the sources that combine together. All steps I have
done so far can be found in the commented `SRCS` lines. Single file
steps can be build explicitly (`make ledon.hex`) or implicitly (`make`)
after removing the comment on the corresponding `SRCS` line. Multiple
file steps can only be build implicitly when their `SRCS` line is
uncommented.

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

### STM32F030F4P6 based board

PROJECT = f030f4
#SRCS = boot.c
#SRCS = ledon.c
#SRCS = blink.c
#SRCS = ledtick.c
#SRCS = cstartup.c
SRCS = startup.c init.c success.c
OBJS = $(SRCS:.c=.o)
CPU = -mthumb -mcpu=cortex-m0
CFLAGS = $(CPU) -g -Wall -Wextra -Os
LD_SCRIPT = $(PROJECT).ld

### Build rules

.PHONY: clean all

all: $(PROJECT).hex $(PROJECT).bin

clean:
    @echo CLEAN
    @rm -f *.o *.elf *.map *.lst *.bin *.hex

$(PROJECT).elf: $(OBJS)
    @echo $@
    $(LD) -T$(LD_SCRIPT) -Map=$(PROJECT).map -cref -o $@ $(OBJS)
    $(SIZE) $@
    $(OBJDUMP) -hS $@ > $(PROJECT).lst

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

## Build and Test

Even if **stdlib.h** is included in **success.c**, there is no C
libraries needed to complete the build as only the constant
`EXIT_SUCCESS` from that header is used. Furthermore, default location
of the header files is derived by the compiler from the location of gcc.

```
$ make
f030f4.elf
   text    data     bss     dec     hex filename
    216       0       0     216      d8 f030f4.elf
f030f4.hex
f030f4.bin
```

Building shows an increase in code, still no data.

Once **f030f4.hex** is loaded into the board, the behavior is the same
as **ledtick.hex**. The new file structure and data initialization
didn’t introduce any ~~bugs~~ changes, just code overhead.

## Checkpoint

This step was mainly to achieve a better structure for future evolution.

[Next](https://warehouse.motd.org/?page_id=433) I will make the code
available in a git repository.

___
© 2020-2021 Renaud Fivet
