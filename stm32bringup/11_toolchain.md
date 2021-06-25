# GNU Arm Embedded Toolchain
Arm maintains a GCC cross-compiler available as binaries that run on Linux and
Windows. It is available on their
[arm developer site](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm).

## Installation on Windows
Currently there is only a win32 version available for download either as a exe
installer or a zip archive. Naming convention
**gcc-arm-none-eabi-…-win32.exe** helps to identify if this is a preview, a
major or an update release.

By default each release installs to its own directory instead of upgrading the
previous one. This way several releases can coexist and you can select which
one you use for a specific project. One downside to this is that the directory
and filename convention is heavy. For practical use, you need to configure an
IDE or encapsulate those paths and names in **Makefile** variables.

```make
### Build environment selection

#GCCDIR = "D:/Program Files (x86)/GNU Tools ARM Embedded/9 2019-q4-major"
GCCDIR = "D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"

BINPFX  = $(GCCDIR)/bin/arm-none-eabi-
CC      = $(BINPFX)gcc

### Build rules

.PHONY: version

version:
        $(CC) --version
```

- **GCCDIR** holds the path to the folder where the toolchain is installed.
When we install a new release, we need to update this path.
- All the commands to build are located in one **bin** subfolder and they
share the same name prefix **arm-none-eabi-**. So I have created a **BINPFX**
to easily identify the commands.

```
$ make
"D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"/bin/arm-none-eabi-gcc --version
arm-none-eabi-gcc.exe (GNU Arm Embedded Toolchain 9-2020-q2-update) 9.3.1 20200408 (release)
Copyright (C) 2019 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

## Installation on Linux
Installation on Linux means downloading the Linux x86_64 tarball and extracting
it. I use the ~/Packages folder for this type of distribution. This means that
the **Makefile** on Linux will be the same as the Windows one except for the
value of the **GCCDIR** variable.

```
GCCDIR = ~/Packages/gcc-arm-none-eabi-9-2020-q2-update
```

By selecting the path based on the development environment, there is no need
to make changes while switching between OS. **Gmake** has the built-in variable
**MAKE_HOST** that can be tested for this.

```make
### Build environment selection

ifeq (linux, $(findstring linux, $(MAKE_HOST)))
#GCCDIR = ~/Packages/gcc-arm-none-eabi-9-2019-q4-major
 GCCDIR = ~/Packages/gcc-arm-none-eabi-9-2020-q2-update
else
#GCCDIR = "D:/Program Files (x86)/GNU Tools ARM Embedded/9 2019-q4-major"
 GCCDIR = "D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"
endif

BINPFX  = $(GCCDIR)/bin/arm-none-eabi-
CC      = $(BINPFX)gcc

### Build rules

.PHONY: version

version:
        $(CC) --version
```

## Toolchain’s chain of events
In order to generate a file that can be loaded in the micro-controller, we
need to sketch the chain of events that will take place.

1. **Compile** source codes (**.c**) to object modules (**.o**)
2. **Link** all the object modules (**.o**) into an executable (**.elf**)
3. **Convert** the executable (**.elf**) into a format suitable for loading or
flashing (**.bin** or **.hex**).

### 1 Compile
**Gmake** has default rules to built **.o** files out of **.c** files. As we
have already defined with CC the command to compile, we can make a simple test
of this step by creating an empty *.c* file and checking what happens when we
try to compile it.

```
$ touch empty.c

$ make empty.o
"D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"/bin/arm-none-eabi-gcc    -c -o empty.o empty.c
```

### 2 Link
To link the object module generated in the first step, we need to specify the
command to link (**ld**), the name of a link script and create a rule to call
the linker command to generate an executable **.elf** file from the object
module **.o** file.

There are sample link scripts that come with the toolchain, they are located
in the subfolder **share/gcc-arm-none-eabi/samples/ldscripts**. For now we can
use the simplest script: **mem.ld**.

```make
### Build environment selection

ifeq (linux, $(findstring linux, $(MAKE_HOST)))
#GCCDIR = ~/Packages/gcc-arm-none-eabi-9-2019-q4-major
 GCCDIR = ~/Packages/gcc-arm-none-eabi-9-2020-q2-update
else
#GCCDIR = "D:/Program Files (x86)/GNU Tools ARM Embedded/9 2019-q4-major"
 GCCDIR = "D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"
endif

BINPFX  = $(GCCDIR)/bin/arm-none-eabi-
CC      = $(BINPFX)gcc
LD      = $(BINPFX)ld

LD_SCRIPT = $(GCCDIR)/share/gcc-arm-none-eabi/samples/ldscripts/mem.ld

### Build rules

%.elf: %.o
        $(LD) -T$(LD_SCRIPT) -o $@ $^
```

```
$ make empty.elf
"D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"/bin/arm-none-eabi-ld -T"D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"/share/gcc-arm-none-eabi/samples/ldscripts/mem.ld -o empty.elf empty.o
```

### 3 Convert
Finally, we use the command **objcopy** to convert the executable **.elf** file
into binary or intel hex format suitable to load in the micro-controller.

```make
### Build environment selection

ifeq (linux, $(findstring linux, $(MAKE_HOST)))
#GCCDIR = ~/Packages/gcc-arm-none-eabi-9-2019-q4-major
 GCCDIR = ~/Packages/gcc-arm-none-eabi-9-2020-q2-update
else
#GCCDIR = "D:/Program Files (x86)/GNU Tools ARM Embedded/9 2019-q4-major"
 GCCDIR = "D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"
endif

BINPFX  = $(GCCDIR)/bin/arm-none-eabi-
CC      = $(BINPFX)gcc
LD      = $(BINPFX)ld
OBJCOPY = $(BINPFX)objcopy

LD_SCRIPT = $(GCCDIR)/share/gcc-arm-none-eabi/samples/ldscripts/mem.ld

### Build rules

%.elf: %.o
    $(LD) -T$(LD_SCRIPT) -o $@ $(OBJS)

%.bin: %.elf
    $(OBJCOPY) -O binary $< $@

%.hex: %.elf
    $(OBJCOPY) -O ihex $< $@
```

Now, if we start in a directory that contains only this **Makefile** and an
empty **empty.c** file, we can successfully build.

```
$ make empty.bin empty.hex
"D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"/bin/arm-none-eabi-gcc    -c -o empty.o empty.c
"D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"/bin/arm-none-eabi-ld -T"D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"/share/gcc-arm-none-eabi/samples/ldscripts/mem.ld -o empty.elf empty.o
"D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"/bin/arm-none-eabi-objcopy -O binary empty.elf empty.bin
"D:/Program Files (x86)/GNU Arm Embedded Toolchain/9 2020-q2-update"/bin/arm-none-eabi-objcopy -O ihex empty.elf empty.hex
rm empty.o empty.elf
```

Notice that **gmake** automatically removes the intermediary **.o** and **.elf**
files on successful completion.

## Cleaning up
I want to keep the output of the build easy to understand without the clutter
of the long command names. Also I need a way to clean the working directory
back to its initial state.

- By prefixing **BINPFX** with *@*, commands will not be displayed by *gmake*
when they are executed. Adding an **echo** of the command target in the rules
helps to keep track of the build progression.
- A new **clean** rule will remove all generated files.

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

LD_SCRIPT = $(GCCDIR)/share/gcc-arm-none-eabi/samples/ldscripts/mem.ld

### Build rules

.PHONY: clean

clean:
    @echo CLEAN
    @rm -f *.o *.elf *.bin *.hex

%.elf: %.o
    @echo $@
    $(LD) -T$(LD_SCRIPT) -o $@ $<

%.bin: %.elf
    @echo $@
    $(OBJCOPY) -O binary $< $@

%.hex: %.elf
    @echo $@
    $(OBJCOPY) -O ihex $< $@
```

```
$ make clean
CLEAN

$ make empty.bin empty.hex
empty.elf
empty.bin
empty.hex
rm empty.o empty.elf
```

## Checkpoint
At this stage, I have a working toolchain and I am able to build from an empty
source file (**empty.c**) to an empty binary file (**empty.bin**).

[Next](https://warehouse.motd.org/?page_id=215) I will select a
micro-controller from the STM32 family and generate a binary file that it
could execute.

___
© 2020-2021 Renaud Fivet
