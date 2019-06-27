---
layout: post
title:  "Running C programs on CRIS"
date:   2019-06-26 00:00:00 +0000
category: Hacking
tags: [Reversing]
---

[Zeus WPI](https://zeus.gent) recently got some old IP cameras: we got
8 AXIS 210A network camera's. These can be used to monitor things, but in this
blog post we will be using them as a "computer" to run arbitrary programs on instead
of only the application made by the vendor.

The camera has a web interface where you can edit files. The file `/etc/inittab`
contains a line that is by default commented out that starts a telnet server at boot.
After enabling this service, some Linux commands can be executed on the camera.
Unfortunately, the camera only has a limited set of commands (most of them included
in a busybox multi-call binary). The camera also runs an ftp server.

The file `/proc/cpuinfo` contains information about the processor:
in this case it was rather non-standard: ETRAX 100LX running the CRIS instruction set.

```
$ cat /proc/cpuinfo
processor       : 0
cpu             : CRIS
cpu revision    : 11
cpu model       : ETRAX 100LX v2
cache size      : 8 kB
fpu             : no
mmu             : yes
mmu DMA bug     : no
ethernet        : 10/100 Mbps
token ring      : no
scsi            : yes
ata             : yes
usb             : yes
bogomips        : 99.32
$ uname -a
Linux axis-00408c7eee9f 2.6.17 #1 Mon Jan 22 11:35:26 CET 2007 cris unknown
```

After some research, a toolchain was found on [the vendors ftp server](ftp://ftp.axis.se/pub/axis/tools/cris/compiler-kit/). Unfortunately, the toolchain itself didn't compile on my system.
This was not unexpected, since the last time the files on the server were updated
was in 2007. Luckily, there were also prebuilt packages of the compiler for Debian
(32-bits Debian, that is). These were installed on a virtual machine downloaded from [osboxes.org](https://osboxes.org).

After installing the `.deb` file, there are now some binaries having a `cris-`
prefix in `/usr/local/cris/bin/`. This includes a compiler `cris-gcc`, but it doesn't
work in the default configuration: the camera uses `libuClibc-0.9.27.so` as its libc,
but the compiler uses `glibc`. This could not be fixed by just linking against the
`libuClibc-0.9.27.so` from the device, because the linker would give errors like:
`error adding symbols: File in wrong format`. This was a dead end, and was not solved.

## Writing our own syscalls

An approach of writing C without libc was then tried: this can be done with
the following GCC flags:

```
cris-gcc -g -mcpu=v10 -mno-side-effects -mcc-init -static -nostdlib main.c -Wl,--entry=_main -o /tmp/compiled
```

This in itself is of course useless without a way provide input to the program and
have the program provide output. To combat this, some of the Linux kernel headers
were copied, modified and compiled with the binary. Specifically, the file `asm/unistd.h`
was included since it had some Linux system call assembly macro's.

Adding the syscalls enabled the program to perform IO, and enabled the porting of
[RCPU](https://github.com/redfast00/RCPU) (RCPU is a fantasy CPU emulator) to
the CRIS architecture. There was already a brainfuck interpreter for RCPU, so now
there's indirectly a brainfuck interpreter for CRIS.

The code and some scripts to make deploying easier can be found on [this GitHub repository](https://github.com/redfast00/RCPU_cris).


The toolchain can be downloaded from the site, or if it goes down, from the [Internet archive](https://web.archive.org/web/20190627200033/http://ftp.axis.se/pub/axis/tools/cris/compiler-kit/cris-dist_1.64-1_i386.deb)

![Yo dawg, I heard you liked obscure instruction sets/so I put an obscure instruction set in your obscure instruction set in your obscure instruction set, so you can cry when you debug](/assets/images/meme_cris.jpg)
