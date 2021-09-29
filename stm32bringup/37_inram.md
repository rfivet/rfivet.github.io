# 3.7: In RAM Execution

So far, I have been executing either my own code from flash or the
bootloader from system memory depending of the state of the BOOT0 pin at
reset.

Using stm32flash I can request the bootloader to transfer execution to
the code in flash memory.

`stm32flash -g 0 COM6`

With my current code, this works fine as far as I don’t use interrupt
subroutine. **ledon** and **blink** both work, but **ledtick** will
reset once the `SysTick_Handler()` interrupt routine is triggered for
the first time. This is due to the fact that the system memory is still
mapped at address 0x0 where my interrupt subroutine vector should be. To
fix this, I need to insure the flash is mapped at address 0x0 before I
enable interrupts.

The memory mapping is managed through the System Configuration
controller **SYSCFG**, so I need to activate it and reconfigure the mapping
before my **SysTick** initialization code in `init()`.

```c
/* Make sure FLASH Memory is mapped at 0x0 before enabling interrupts */
    RCC_APB2ENR |= RCC_APB2ENR_SYSCFGEN ;      /* Enable SYSCFG */
    SYSCFG_CFGR1 &= ~3 ;                       /* Map FLASH at 0x0 */
```

and add the SYSCFG peripheral description.

```c
#define RCC_APB2ENR_SYSCFGEN    0x00000001  /*  1: SYSCFG clock enable */

#define SYSCFG              ((volatile long *) 0x40010000)
#define SYSCFG_CFGR1        SYSCFG[ 0]
```

With this in place, I can now switch easily from bootloader to flash
code by sending a go command via stm32flash.

## Sharing the RAM with the Bootloader

Before I can ask the bootloader to transfer execution to code in RAM, I
need first to ask it to write code there. As the bootloader data are
located in RAM too, I have to avoid overwriting them. Where is it safe
to write in RAM?

The answer is in the application note **AN2606 STM32 microcontroller
system memory boot mode**. Section 5 covers **STM32F03xx4/6 devices
bootloader** and it states in **5.1 Bootloader Configuration**: _2 Kbyte
starting from address 0x20000000 are used by the bootloader firmware_.

I am using a STM32F030F4P6, which has 4KB RAM and the bootloader
firmware is using the first 2KB. That means I have only 2KB left to use
starting from address 0x20000800.

Actually, I have only 2KB left to use until the bootloader firmware
transfer execution to my code in RAM. Once my code executes, I can
reclaim the first 2KB. This is exactly what I have to tell the linker.

I just create a new linker script **f030f4.ram.ld** by copying
**f030f4.ld** and changing the memory configuration.

```
/* FLASH means code, read only data and data initialization */
    FLASH (rx)  : ORIGIN = 0x20000800, LENGTH =  2K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH =  2K
```

I can build **ledon** or **blink** with that new linker script and check the
resulting **f030f4.map** file.

- isr vector, code, const data and data initialization are located from
  0x20000800.

- Stack Pointer initial value is 0x20000800.

- .data and .bss are located from 0x20000000.

Let’s write this code in RAM and execute it!

```
stm32flash -w blink.bin -S 0x20000800 COM6
stm32flash -g 0x20000800 COM6
```

This work just fine but of course the executable of **ledon** or
**blink** doesn’t use interrupt routines.

## ISR Vector in RAM

Like for FLASH, we need to make sure that RAM memory is mapped at
address 0x0 and start with the ISR vector.

- Use **SYSCFG** controller to map RAM at address 0x0

- Tell the linker to reserve space at beginning of RAM before locating
  .data section.

- Make a copy of `isr_vector[]` to the beginning of RAM in the space
  reserved by the linker.

To select the RAM mapping, the **MEM_MODE** bits need to be set in
**SYSCFG_CFGR1**.

```
/* Make sure SRAM Memory is mapped at 0x0 before enabling interrupts */
    RCC_APB2ENR |= RCC_APB2ENR_SYSCFGEN ;        /* Enable SYSCFG */
    SYSCFG_CFGR1 |= 3 ;                          /* Map RAM at 0x0 */
```

The ISR vector will have at most 16 + 32 entries for STM32F030xx, that
means 192 bytes need to be reserved. I add a new section before .data in
the link script.

```
    .isrdata :
    {
        ram_vector = . ;
        . = . + 192 ;
    } > RAM

    .data : AT (__etext)
    {
...
```

In the startup code, I add the code to copy the `isr_vector[]` to the
location reserved at the beginning of RAM.

```
#define ISRV_SIZE (sizeof isr_vector / sizeof *isr_vector)

extern isr_p ram_vector[] ;

/* Copy isr vector to beginning of RAM */
    for( unsigned i = 0 ; i < ISRV_SIZE ; i++)
        ram_vector[ i] = isr_vector[ i] ;
```

RAM initialization now consists of

- Stack pointer initialization

- ISR vector copy

- .data initialization

- .bss clearing

I can now rebuild **ledtick** or **uptime prototype** for execution in RAM.
**f030f4.map** now shows that .data starts at 0x200000C0, after
`ram_vector[]`.

```
.isrdata        0x20000000       0xc0
                0x20000000                ram_vector = .
                0x200000c0                . = (. + 0xc0)
 *fill*         0x20000000       0xc0

.data           0x200000c0        0x0 load address 0x20000c88
                0x200000c0                __data_start__ = .
```

I can now use **stm32flash** to write those executables in RAM and request
execution.

## Memory Models

I have now the choice between four memory models when I build.

| Model   | ISRV Location      | Load address (word aligned)             |
|---------|--------------------|-----------------------------------------|
|BOOTFLASH| Beginning of FLASH | Beginning of FLASH                      |
| BOOTRAM | Beginning of RAM   | Beginning of RAM                        |
| GOFLASH | Beginning of RAM   | In FLASH                                |
| GORAM   | Beginning of RAM   | In RAM, after bootloader reserved space |

- **BOOTFLASH**: Executed at reset depending of BOOT0 pin level otherwise
  triggered by a (boot)loader.

- **BOOTRAM**: Executed at reset depending of BOOT0/BOOT1 configuration
  otherwise triggered by a (boot)loader through SWD.

- **GOFLASH**: Executed at reset if located at beginning of FLASH
  otherwise triggered by a (boot)loader. (Spoiler: IAP and multi boot)

- **GORAM**: Triggered by a (boot)loader. Useful for development if RAM
  size allows it. (Spoiler: external storage)

To avoid having to edit multiple files when switching between models or
introducing a new chipset family, I make the following changes.

1. Use a generic linker script.

2. Let the startup code handle the isr vector initialization and the
   memory mapping.

3. Maintain the FLASH and RAM information and isr vector position in the
   Makefile.

### 1. Generic Linker Script

To turn f030f4.ram.ld into a generic linker script, I need to

- abstract the memory part.

- remove the RAM isr vector hardcoded size.

```
MEMORY
{
/* FLASH means code, read only data and data initialization */
    FLASH (rx) : ORIGIN = DEFINED(FLASHSTART) ? FLASHSTART : 0x08000000,
        LENGTH =  DEFINED(FLASHSIZE) ? FLASHSIZE : 16K
    RAM  (rwx) : ORIGIN = DEFINED(RAMSTART) ? RAMSTART : 0x20000000,
        LENGTH =  DEFINED(RAMSIZE) ? RAMSIZE : 4K
}
```

The Makefile will provide the necessary addresses and sizes information
by passing parameters to the linker: `FLASHSTART`, `FLASHSIZE`,
`RAMSTART`, `RAMSIZE`.

```
    /* In RAM isr vector reserved space at beginning of RAM */
    .isrdata (NOLOAD):
    {
        KEEP(*(.ram_vector))
    } > RAM
```

The startup code will allocate ram_vector[] in .ram_vector section if
needed.

### 2. Startup Code

I create the startup code startup.ram.c from a copy of startup.txeie.c,
using conditional compiled code selected by RAMISRV whose definition
will be passed as parameter to the compiler.

```
#if RAMISRV == 2
# define ISRV_SIZE (sizeof isr_vector / sizeof *isr_vector)
isr_p ram_vector[ ISRV_SIZE] __attribute__((section(".ram_vector"))) ;
#endif

int main( void) ;

void Reset_Handler( void) {
    const long  *f ;    /* from, source constant data from FLASH */
    long    *t ;        /* to, destination in RAM */

#if RAMISRV == 2
/* Copy isr vector to beginning of RAM */
    for( unsigned i = 0 ; i < ISRV_SIZE ; i++)
        ram_vector[ i] = isr_vector[ i] ;
#endif

/* Assume:
**  __bss_start__ == __data_end__
**  All sections are 4 bytes aligned
*/
    f = __etext ;
    for( t = __data_start__ ; t < __bss_start__ ; t += 1)
        *t = *f++ ;

    while( t < &__bss_end__)
        *t++ = 0 ;

/* Make sure active isr vector is mapped at 0x0 before enabling interrupts */
    RCC_APB2ENR |= RCC_APB2ENR_SYSCFGEN ;           /* Enable SYSCFG */
#if RAMISRV
    SYSCFG_CFGR1 |= 3 ;                             /* Map RAM at 0x0 */
#else
    SYSCFG_CFGR1 &= ~3 ;                            /* Map FLASH at 0x0 */
#endif

    if( init() == 0)
        main() ;

    for( ;;)
        __asm( "WFI") ; /* Wait for interrupt */
}
```

The SYSCFG controller definition is now included through a chipset
specific header file. This way I can maintain all the chipset
controllers and peripherals in one place.

`#include "stm32f030xx.h"`

### 3. Makefile

The Makefile now holds the memory model definition that is passed as
parameters to the compiler and the linker.

```make
### Memory Models
# By default we use the memory mapping from linker script

# In RAM Execution, load and start by USART bootloader
# Bootloader uses first 2K of RAM, execution from bootloader
#FLASHSTART = 0x20000800
#FLASHSIZE  = 2K
#RAMSTART   = 0x20000000
#RAMSIZE    = 2K

# In RAM Execution, load and start via SWD
# 4K RAM available, execution via SWD
#FLASHSTART = 0x20000000
#FLASHSIZE  = 3K
#RAMSTART   = 0x20000C00
#RAMSIZE    = 1K

# In Flash Execution
# if FLASHSTART is not at beginning of FLASH: execution via bootloader or SWD
#FLASHSTART = 0x08000000
#FLASHSIZE  = 16K
#RAMSTART   = 0x20000000
#RAMSIZE    = 4K

# ISR vector copied and mapped to RAM when FLASHSTART != 0x08000000
ifdef FLASHSTART
 ifneq ($(FLASHSTART),0x08000000)
  ifeq ($(FLASHSTART),0x20000000)
   # Map isr vector in RAM
   RAMISRV := 1
  else
   # Copy and map isr vector in RAM
   RAMISRV := 2
  endif
 endif
 BINLOC  = $(FLASHSTART)
else
 BINLOC  = 0x08000000
endif
```

Compiler and linker have different syntax for defining symbols through
command line parameters.

```make
CPU = -mthumb -mcpu=cortex-m0 --specs=nano.specs
ifdef RAMISRV
 CDEFINES = -DRAMISRV=$(RAMISRV)
endif
WARNINGS=-pedantic -Wall -Wextra -Wstrict-prototypes
CFLAGS = $(CPU) -g $(WARNINGS) -Os $(CDEFINES)

LD_SCRIPT = generic.ld
ifdef FLASHSTART
 LDOPTS  =--defsym FLASHSTART=$(FLASHSTART) --defsym FLASHSIZE=$(FLASHSIZE)
 LDOPTS +=--defsym RAMSTART=$(RAMSTART) --defsym RAMSIZE=$(RAMSIZE)
endif
LDOPTS +=-Map=$(subst .elf,.map,$@) -cref --print-memory-usage
comma :=,
space :=$() # one space before the comment
LDFLAGS =-Wl,$(subst $(space),$(comma),$(LDOPTS))
```

As I am revising the compilation flags, I have increased the level of
warnings by adding -pedantic, -Wstrict-prototypes.

Build rules updated with new symbols for the linker.

```make
$(PROJECT).elf: $(OBJS) libstm32.a
boot.elf: boot.o
ledon.elf: ledon.o
blink.elf: blink.o
ledtick.elf: ledtick.o
cstartup.elf: cstartup.o

%.elf:
    @echo $@
    $(CC) $(CPU) -T$(LD_SCRIPT) $(LDFLAGS) -nostartfiles -o $@ $+
    $(SIZE) $@
    $(OBJDUMP) -hS $@ > $(subst .elf,.lst,$@)
```

The projects composition need to be updated to use the new startup.

`SRCS = startup.ram.c txeie.c uptime.1.c`

Finally, to keep track of the memory model and the load location, I put
the load address in the name of the binary file generated.

`all: $(PROJECT).$(BINLOC).bin $(PROJECT).hex`

This way if I build uptime prototype in GORAM memory model

```
$ make
f030f4.elf
   text    data     bss     dec     hex filename
   1164       0      20    1184     4a0 f030f4.elf
f030f4.hex
f030f4.0x20000800.bin
```

The name of the file will remind me where to load the code.

```
$ stm32flash -w f030f4.0x20000800.bin -S 0x20000800 COM6
$ stm32flash -g 0x20000800
```

## Caveat: stm32flash v0.6 intel hex bug

At the time of writing, **stm32flash** v0.6 has a bug that prevents
writing intel hex files correctly at address other than the origin of
the Flash. A bug fix and the possibility to directly read the base
address from the intel hex file are planned to be included in v0.7.

Until v0.7 is out, I am using my own patched version of stm32flash or
the binary files when I need to test GOFLASH and GORAM memory models.

As I branched off my own patched version of **stm32flash**, I added a `-x`
option to write and execute an intel hex file:

`stm32flash -x file.hex COM#`

## Testing

I build all four memory models and check that they can be loaded and
executed using both **stm32flash** and **STM32 Cube Programmer**.

Using the USART bootloader, I validate BOOTFLASH, GOFLASH and GORAM with
**stm32flash** and **STM32 Cube Programmer**.

Using the SWD interface, I validate BOOTFLASH, GOFLASH, BOOTRAM and
GORAM with **STM32 Cube Programmer**.

## Checkpoint

[Next](38_crc32) I will add integrity check at startup by doing CRC32
validation of the code.

___
© 2020-2021 Renaud Fivet
