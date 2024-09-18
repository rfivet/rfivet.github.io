# 3.6: Toolchain Update

While I was experimenting with the internal temperature sensor, a major
release of GNU ARM Embedded Toolchain came out and I had to do some
adaptations when switching to it.

To switch from release 9 update to release 10 major, I made three
changes to **Makefile**.

● Update the Linux base directory location:

```
#GCCDIR = $(HOME)/Packages/gcc-arm-none-eabi-9-2020-q2-update
GCCDIR = $(HOME)/Packages/gcc-arm-none-eabi-10-2020-q4-major
```

● Update the Windows base directory location:

```
#GCCDIR = D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update
GCCDIR = D:/Program Files (x86)/GNU Arm Embedded Toolchain/10 2020-q4-major
```

● Update the library subdirectory location:

```
#LIBDIR  = $(GCCDIR)/lib/gcc/arm-none-eabi/9.3.1/thumb/v6-m/nofp
LIBDIR  = $(GCCDIR)/lib/gcc/arm-none-eabi/10.2.1/thumb/v6-m/nofp
```

Unfortunately when I did some regression testing by recompiling the
projects so far, I found that the new release optimizes further the C
startup clearing of BSS data by calling `memset()` from the distribution
libraries. So I had to add **libc.a** on top of **libgcc.a** to the list
of libraries.

As I don’t want to turn off size optimization and I am not willing to
always pay the full 180 bytes for a production ready `memset()` when it
is called only once at startup to clear a few bytes, I ended up adding
my own version of `memset()` to my local library.

```c
#include <string.h>

void *memset( void *s, int c, size_t n) {
    char *p = s ;
    while( n--)
        *p++ = c ;

    return s ;
}
```

As I was investigating the compilation flags to find if there was a
better way to solve this issue, I figure out I could let **gcc** handle
the distribution libraries selection and their location based on the CPU
type. So I changed the linker invocation accordingly and got rid of LD,
LIBDIR and LIB_PATHS definitions.

```make
#   $(LD) -T$(LD_SCRIPT) $(LIB_PATHS) -Map=$(PROJECT).map -cref -o $@ $(OBJS) $(LIBS)
    $(CC) $(CPU) -T$(LD_SCRIPT) -L. -Wl,-Map=$(PROJECT).map,-cref \
        -nostartfiles -o $@ $(OBJS) $(LIBS)
```

As the compiler front end is now controlling the libraries selection it is
possible to give it a hint how to select a better optimized memset().  The
libc library comes in two flavors: regular and nano.

```make
OBJS = $(SRCS:.c=.o)
LIBOBJS = printf.o putchar.o puts.o # memset.o

CPU = -mthumb -mcpu=cortex-m0 --specs=nano.specs
```

memset() included in the nano version of libc occupies the same space as my
own implementation.

Finally, I revised the way I specify the commands location by updating the
PATH environment variable instead of giving the full path of each command.
On Windows, I make sure that drive specification matches the development
environment in use (Cygwin, MSYS2 and other).

```make
### Build environment selection

ifeq (linux, $(findstring linux, $(MAKE_HOST)))
 INSTALLDIR = $(HOME)/Packages
#REVDIR = gcc-arm-none-eabi-9-2019-q4-major
#REVDIR = gcc-arm-none-eabi-9-2020-q2-update
 REVDIR = gcc-arm-none-eabi-10-2020-q4-major
else
 DRIVE = d
ifeq (cygwin, $(findstring cygwin, $(MAKE_HOST)))
 OSDRIVE = /cygdrive/$(DRIVE)
else ifeq (msys, $(findstring msys, $(MAKE_HOST)))
 OSDRIVE = /$(DRIVE)
else
 OSDRIVE = $(DRIVE):
endif
 INSTALLDIR = $(OSDRIVE)/Program Files (x86)
#REVDIR = GNU Tools ARM Embedded/5.4 2016q2
#REVDIR = GNU Tools ARM Embedded/6 2017-q2-update
#REVDIR = GNU Tools ARM Embedded/7 2017-q4-major
#REVDIR = GNU Tools ARM Embedded/7 2018-q2-update
#REVDIR = GNU Tools ARM Embedded/9 2019-q4-major
#REVDIR = GNU Arm Embedded Toolchain/9 2020-q2-update
 REVDIR = GNU Arm Embedded Toolchain/10 2020-q4-major
endif

GCCDIR = $(INSTALLDIR)/$(REVDIR)
export PATH := $(GCCDIR)/bin:$(PATH)

BINPFX  = @arm-none-eabi-
AR      = $(BINPFX)ar
CC      = $(BINPFX)gcc
OBJCOPY = $(BINPFX)objcopy
OBJDUMP = $(BINPFX)objdump
SIZE    = $(BINPFX)size
```

## Checkpoint

Invoking the compiler instead of the linker gives more flexibility in
case the toolchain directory structure changes or if I target a
different core. The compiler is aware of the location of the toolchain
libraries while the linker need explicit parameters to handle those
changes.

[Next]( 37_inram) I will (re)build to execute code in RAM instead of
FLASH.

___
© 2020-2021 Renaud Fivet
