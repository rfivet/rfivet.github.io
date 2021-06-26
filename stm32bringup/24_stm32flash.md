2.4 stm32flash

So far I have been flashing boards via UART or SWD interface using STM32
Cube Programmer. An alternative to flash via UART is **stm32flash**.

## Linux Build and Install

**stm32flash** project is hosted on [SourceForge](
https://sourceforge.net/projects/stm32flash/) and the git repository is
mirrored on [gitlab]( https://gitlab.com/stm32flash/stm32flash).

I clone the repository from **Sourceforge** in my **Projects** folder.

```
$ cd ~/Projects
$ git clone https://git.code.sf.net/p/stm32flash/code stm32flash-code
Cloning into 'stm32flash-code'...
remote: Enumerating objects: 1357, done.
remote: Counting objects: 100% (1357/1357), done.
remote: Compressing objects: 100% (682/682), done.
remote: Total 1357 (delta 912), reused 996 (delta 671)
Receiving objects: 100% (1357/1357), 1.04 MiB | 74.00 KiB/s, done.
Resolving deltas: 100% (912/912), done.
```

Build on Linux doesn’t show any warnings.

```
$ cd stm32flash-code
$ make
cc -Wall -g   -c -o dev_table.o dev_table.c
cc -Wall -g   -c -o i2c.o i2c.c
cc -Wall -g   -c -o init.o init.c
cc -Wall -g   -c -o main.o main.c
cc -Wall -g   -c -o port.o port.c
cc -Wall -g   -c -o serial_common.o serial_common.c
cc -Wall -g   -c -o serial_platform.o serial_platform.c
cc -Wall -g   -c -o stm32.o stm32.c
cc -Wall -g   -c -o utils.o utils.c
cd parsers && make parsers.a
make[1]: Entering directory '~/Projects/stm32flash-code/parsers'
cc -Wall -g   -c -o binary.o binary.c
cc -Wall -g   -c -o hex.o hex.c
ar rc parsers.a binary.o hex.o
make[1]: Leaving directory '~/Projects/stm32flash-code/parsers'
cc  -o stm32flash dev_table.o i2c.o init.o main.o port.o serial_common.o serial_platform.o stm32.o utils.o parsers/parsers.a
```

I test the newly compiled command by calling it without argument
`./stm32flah` and with the serial port where the USB to UART adapter is
plugged in.

`./stm32flash` gives a detailed help of the command.

Calling it with the serial port argument where the board is plugged in
and set in bootloader mode gives a description of the chipset detected.

```
$ ./stm32flash /dev/ttyUSB0
stm32flash 0.5

http://stm32flash.sourceforge.net/

Interface serial_posix: 57600 8E1
Version      : 0x31
Option 1     : 0x00
Option 2     : 0x00
Device ID    : 0x0444 (STM32F03xx4/6)
- RAM        : Up to 4KiB  (2048b reserved by bootloader)
- Flash      : Up to 32KiB (size first sector: 4x1024)
- Option RAM : 16b
- System RAM : 3KiB
```

I install the command by moving the executable to my local bin directory.

`$ mv stm32flash ~/bin`

If everything goes well, I will later `strip` and compress (with `upx`)
the executable.

## Regression Testing

As my board has been already flashed on Windows, I can perform a simple
regression test.

- Read the content of the chipset memory as flashed with the Windows
application.

- Flash the same executable using Linux version of stm32flash.

- Read back the newly programmed chipset memory.

- Compare the two read-outs.

Reading 1 KB with stm32flash.

`$ stm32flash -r read.bin -S 0x08000000:1024 /dev/ttyUSB0`

Writing the executable in hex format.

`$ stm32flash -w f030f4.hex /dev/ttyUSB0`

Comparing the memory read-out using `od`, there is no difference

## Build and Install on Windows

There is a Windows binary that can be downloaded from **stm32flash** project
page on **SourceForge**. But I did clone and build using both **Cygwin** and
**MSYS2 64bit** environments on Windows.

The build phase gave more warnings than the Linux version, this is
mostly due to stricter warnings in the GCC compiler version.

Usage of `stm32flash` only differs in the name of the serial device, in my
case **COM4** instead of **/dev/ttyUSB0**.

## Checkpoint

There is several other Windows applications available on ST.com for
flashing STM32 chipsets: STM32 ST-Link Utility, STM32 Flash Loader
Demonstrator, ST Visual Programmer STM32. They have been marked as NRND
(Not Recommended for New Design), which means they won’t support latest
chipsets as they are replaced by STM32 Cube Programmer.

[Next]( https://warehouse.motd.org/?page_id=612) I will write an
application which make better use of transmission than **hello**.

___
© 2020-2021 Renaud Fivet
