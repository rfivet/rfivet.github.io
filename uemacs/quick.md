# Quick µEMACS build and install

## Dependencies and build environment

To build µEMACS, you need to have gcc, gmake and ncurses development library installed.

## Checking environment

gcc and gmake are often preinstalled with gmake set as the default make. Use your favorite
package manager to check their availability.

```
% which gcc make
/usr/bin/gcc
/usr/bin/make
% apt list gcc make
gcc/focal,now 4:9.3.0-1ubuntu2 amd64 [installed]
make/focal,now 4.2.1-1.2 amd64 [installed]
```

Use your favorite package manager if they need to be installed.

```
% sudo apt install gcc make
```

To check that make is actually GNU make:

```
% make --version
GNU Make 4.2.1
Built for x86_64-pc-linux-gnu
Copyright (C) 1988-2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later http://gnu.org/licenses/gpl.html
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```

ncurses development library usually need to be installed. Query your favorite package
manager to check which packages are available for installation:

```
% apt search libncurses
```

Use your favorite package manager to install the needed package:

```
% sudo apt install libncurses-dev
```

On Ubuntu, apt will select the package that matches your architecture (amd64 or i386).

## Getting the sources

µEMACS source code is available on [github](https://github.com/rfivet/uemacs) and mirrored
at [git.sdf.org](https://git.sdf.org/rfivet/uemacs). You can either clone
the [git repository](https://github.com/rfivet/uemacs.git) or download a
[zip archive](https://github.com/rfivet/uemacs/archive/master.zip).

```
Move to working directory and clone:
% mkdir ~/Projects
% cd ~/Projects
% git clone https://github.com/rfivet/uemacs.git
```

## Building
```
% cd ~/Projects/uemacs
% make depend
% make
```

If gmake is not set as the default make you will have to call it explicitly

```
% gmake depend
% gmake
```

## Testing
Start the editor:

```
% ./ue
```

To leave the editor type CTL-X CTL-C

Execute a sample script:

```
% ./ue -x screensize.cmd
```

![Executing script screensize.cmd](https://warehouse.motd.org/wp-content/uploads/2020/10/ue_screensize.png)
___
© 2020-2021 Renaud Fivet
