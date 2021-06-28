# 2.7: C Library

So far I have used three Standard C library functions for output:
`printf()`, `putchar()` and `puts()`. It’s time to bundle them as a library.
This will give me more flexibility as I will not have to give a full
list of the modules to link, the linker will handle the missing
dependencies by looking into the libraries.

## puts()

I have already packaged `printf() and putchar()` in stand alone modules.
As I have removed my previous implementation of `puts()` from the system,
I need to create puts.c.

```c
/* puts.c -- write a string to stdout   */

#include <stdio.h>
#include "system.h" /* kputc(), kputs() */

int puts( const char *s) {
    kputs( s) ;
    kputc( '\n') ;
    return 0 ;
}
```

## Updating Makefile

I need to tell **GNU make** how to manage and use the library, which
means updating **Makefile**.

What’s the name, the content and a rule to maintain the library.

```make
AR      = $(BINPFX)ar

LIBOBJS = printf.o putchar.o puts.o
LIBSTEM = stm32

lib$(LIBSTEM).a: $(LIBOBJS)
    $(AR) rc $@ $?
```

Where to look for and which libraries to use in the link phase.

```make
LIB_PATHS = -L. -L$(LIBDIR)
LIBS = -l$(LIBSTEM) -lgcc

$(PROJECT).elf: $(OBJS) lib$(LIBSTEM).a
    @echo $@
    $(LD) -T$(LD_SCRIPT) $(LIB_PATHS) -Map=$(PROJECT).map -cref -o $@ $(OBJS) $(LIBS)
    $(SIZE) $@
    $(OBJDUMP) -hS $@ > $(PROJECT).lst
```

Library modules are implicitly part of the composition, so it’s not
necessary to list them anymore.

```make
#SRCS = startup.c uplow.2.c uptime.c printf.c putchar.c
SRCS = startup.c uplow.2.c uptime.c
```

I include libraries in the list of files to delete when doing a make
clean.

```make
clean:
    @echo CLEAN
    @rm -f *.o *.elf *.map *.lst *.bin *.hex *.a
```

## Building uptime

Build terminates successfully producing the same executable as before.

```
$ make
f030f4.elf
   text    data     bss     dec     hex filename
   1792       0      12    1804     70c f030f4.elf
f030f4.hex
f030f4.bin
```

Checking the map produced by the linker I can see that it fetched the
necessary modules for `printf()` and `putchar()` from the newly created
library.

```
Archive member included to satisfy reference by file (symbol)

.\libstm32.a(printf.o)        uptime.o (printf)
.\libstm32.a(putchar.o)       uptime.o (putchar)
D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update/lib/gcc/arm-none-eabi/9.3.1/thumb/v6-m/nofp\libgcc.a(_thumb1_case_sqi.o)
                              .\libstm32.a(printf.o) (__gnu_thumb1_case_sqi)
D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update/lib/gcc/arm-none-eabi/9.3.1/thumb/v6-m/nofp\libgcc.a(_udivsi3.o)
                              uptime.o (__aeabi_uidiv)
D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update/lib/gcc/arm-none-eabi/9.3.1/thumb/v6-m/nofp\libgcc.a(_divsi3.o)
                              uptime.o (__aeabi_idiv)
D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update/lib/gcc/arm-none-eabi/9.3.1/thumb/v6-m/nofp\libgcc.a(_dvmd_tls.o)
                              D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update/lib/gcc/arm-none-eabi/9.3.1/thumb/v6-m/nofp\libgcc.a(_udivsi3.o) (__aeabi_idiv0)
```

## Building hello

I can rebuild my **hello** application using the latest system
implementation and the newly made library.

`SRCS = startup.c uplow.2.c hello.c`

Build terminates successfully, the changes in size are due to the
difference in the system implementation.

```
$ make
f030f4.elf
   text    data     bss     dec     hex filename
    445       0       8     453     1c5 f030f4.elf
f030f4.hex
f030f4.bin
```

Checking the map file produced in the link phase, I can see that only
puts.o has been fetched from my local library.

```
Archive member included to satisfy reference by file (symbol)

.\libstm32.a(puts.o)          hello.o (puts)
```

## Checkpoint

I had to deal with linking with **gcc** library [before]( 25_prototype),
so introducing my own library implementation of the standard C library
output functions is a simple step.

[Next]( 28_clocks) I will continue on the topic of asynchronous serial
transmission and look into baud rate and clock configuration.

___
© 2020-2021 Renaud Fivet

