LMG (Linux Memory Grabber):
==========================
A script for dumping Linux memory and creating Volatility(TM) profiles.
Hal Pomeranz (hal@deer-run.com), 2014-03-29

THANKS!
=======

>If I have seen further it is by standing on the shoulders of giants.
      Issac Newton

There are a lot of people who deserve thanks for making this simple
little tool possible:

* **Joe Sylve** for his work on LiME

* The entire **Volatility**(TM) development team for their ongoing work.
   I'd like to particularly recognize Andrew Case who answered a number
   of pesky questions from me during development of my tool.

* **David Anderson** for his ongoing support of libdwarf and dwarfdump

* **Matt Suiche** from MoonSols.  When I was putting my tool together,
   my design goal was "make it as easy to use as DumpIt" (if you need
   to capture Windows memory, I know of no easier to use tool).
   So thanks for the inspiration, Matt!

The community is better for all of these efforts.  I have chosen to make
my tool available under the Creative Commons "Attribution" License (CC BY),
in order to make it as widely available as possible.

ABOUT THE TOOL
==============

To analyze Linux memory, you first need to be able to capture Linux
memory.  Joe Sylve's Linux Memory Extractor (LiME) is excellent for 
this, but you need to have a LiME module compiled for the kernel of
the system where you want to grab RAM.

Volatility(TM) is great at analyzing Linux memory images.  But it needs
a profile that matches the system where the memory was captured.
Building a profile means compiling a C program on the appropriate system
and using dwarfdump to get the addresses of important kernel data structures.
You also need a copy of the System.map file from the /boot directory.

Now if you happen to have a duplicate of your target system, you can
build LiME and compile the Volatility(TM) profile on the clone and
use them to capture and analyze memory from your target.  But there are
many situations where a duplicate of your target system is not available.
So you may have to compile LiME and build your Volatility(TM) profile
on your target machine.

And this is not for the faint of heart.  There are a number of steps,
and some fairly low-level Linux commands involved.  My goal was to
create a package that could be installed (by an expert) on a thumb drive
and distributed to agents in the field.  The user of the thumb drive
should be able to plug the thumb drive in, run a single command, 
and successfully acquire a memory image of the target machine and
a working Volatility(TM) profile.  The result is my lmg (Linux Memory
Grabber) script.

ON FORENSIC PURITY
==================

If you're a stickler for forensic purity, this is probably not the
tool for you.  Let's discuss some of the ways in which my tool interacts
with the target system:

Removable Media -- The tool is designed to be run from a portable USB
  device such as a thumb drive.  You are going to be plugging a writable
  device into your target system, where it could potentially be targeted
  by malicious users or malware on the system.  The act of plugging the
  device into the system is going to change the state of the machine
  (e.g., create log entries, mtab entries, etc).  If the device is not 
  auto-mounted by the operating system, the user must manually mount the 
  device via a root shell.

####Compilation
  lmg builds a LiME kernel module for the system.
  Creating a Volatility(TM) profile also involves compiling code on
  the target machine.  So gcc will be executed, header files read,
  libraries linked, etc.  lmg tries to minimize impact on the file
  system of the target machine by setting TMPDIR to a directory on
  the USB device lmg runs from.  This means that intermediate files
  created by the compiler will be written to the thumb drive rather
  than the local file system of the target machine.

####Dependencies
  In order to compile kernel code on Linux, the target
  machine needs a working development environment with gcc, make, etc 
  and all of the appropriate include files and shared libraries.
  And in particular, the kernel header files need to be present on
  the local machine.  These dependencies may not exist on the target
  system.  In this case, the user is faced with the choice of installing
  the appropriate dependencies (if possible) or being unable to
  acquire memory from the target.

####Malware 
  lmg uses bash, gcc, and a host of other programs from
  the target machine.  If the system has been compromised, the applications
  lmg uses may not be trustworthy.  A more complete solution would be
  to create a secure execution environment for lmg on the portable USB
  device, but was beyond the scope of this initial proof of concept.

####Memory
  All of the commands being run will cause the memory of the
  target system to change.  The act of capturing RAM will always create
  artifacts, but in this case there is extensive compilation, file system
  access, etc in addition to running a RAM dumper.

All of that being said, lmg is a very convenient tool for allowing
less-skilled agents to capture useful memory analysis data from
target systems.

###Note 
LMG will look for an already existing LiME module on the
USB device that matches the kernel version and processor architecture
of the target machine.  If found, lmg will not bother to recompile.
Similarly, you may choose to not have lmg create the Volatility(TM)
profile for the target in order to minimize the impact on the target system.

USING LMG
=========

First, prepare a thumb drive according to the instructions in the
INSTALL document provided with lmg.

When you wish to acquire RAM, plug the thumb drive into your target
system.  On most Linux systems, new USB devices will get automatically
mounted under `/media`.  Let's assume yours ends up under `/media/LMG`.

Now, as root, run `/media/LMG/lmg`.  This is interactive mode and
the user will be prompted for confirmation before lmg builds a LiME
module for the system and/or creates a Volatility(TM) profile.
If you don't want to be prompted, use `/media/LMG/lmg -y`.

Everything else is automated.  After the script runs, you will have
a new directory on the thumb drive named 

   `.../capture/<hostname>-YYYY-MM-DD_hh.mm.ss`

In this directory you will find:
```
   <hostname>-YYYY-MM-DD_hh.mm.ss-memory.lime  -- the RAM capture
   <hostname>-YYYY-MM-DD_hh.mm.ss-profile.zip  -- Volatility(TM) profile
   <hostname>-YYYY-MM-DD_hh.mm.ss-bash         -- copy of target's /bin/bash
```

The copy of `/bin/bash` is helpful for determining the address of the shell 
history data structure in the memory of bash processes in the memory capture.
[See](http://code.google.com/p/volatility/wiki/LinuxCommandReference23#linux_bash)
for further details on how to use this executable.


USAGE EXAMPLE
=============

Here is an example of using the lmg tool, which includes using Volatility(TM)
directly off the thumb drive to analyze the captured image.  On my test
machine, the thumb drive was at `/dev/sdb` and it was not auto-mounted by
my operating system.  So I did everything manually.

1) Getting root and mounting the thumb drive
--------------------------------------------
```
caribou$ sudo -s
[sudo] password for hal: 
caribou# mkdir -p /mnt/usb
caribou# mount /dev/sdb1 /mnt/usb
```

2) Running LMG
--------------
```
caribou# /mnt/usb/lmg -y
make -C /lib/modules/3.2.0-41-generic/build M=/mnt/usb/lime/src modules
make[1]: Entering directory `/usr/src/linux-headers-3.2.0-41-generic'
  CC [M]  /mnt/usb/lime/src/tcp.o
  CC [M]  /mnt/usb/lime/src/disk.o
  CC [M]  /mnt/usb/lime/src/main.o
  LD [M]  /mnt/usb/lime/src/lime.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /mnt/usb/lime/src/lime.mod.o
  LD [M]  /mnt/usb/lime/src/lime.ko
make[1]: Leaving directory `/usr/src/linux-headers-3.2.0-41-generic'
strip --strip-unneeded lime.ko
mv lime.ko lime-3.2.0-41-generic-x86_64.ko
make tidy
make[1]: Entering directory `/mnt/usb/lime/src'
rm -f *.o *.mod.c Module.symvers Module.markers modules.order \.*.o.cmd \.*.ko.cmd \.*.o.d
rm -rf \.tmp_versions
make[1]: Leaving directory `/mnt/usb/lime/src'
Dumping memory in "lime" format to /mnt/usb/capture/caribou-2014-03-29_12.06.01
This could take a while...Done! Cleaning up...Done!
Grabbing a copy of /bin/bash...Done!
make -C //lib/modules/3.2.0-41-generic/build CONFIG_DEBUG_INFO=y M=/mnt/usb/volatility-2.3.1/tools/linux modules
make[1]: Entering directory `/usr/src/linux-headers-3.2.0-41-generic'
  CC [M]  /mnt/usb/volatility-2.3.1/tools/linux/module.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /mnt/usb/volatility-2.3.1/tools/linux/module.mod.o
  LD [M]  /mnt/usb/volatility-2.3.1/tools/linux/module.ko
make[1]: Leaving directory `/usr/src/linux-headers-3.2.0-41-generic'
dwarfdump -di module.ko > module.dwarf
make -C //lib/modules/3.2.0-41-generic/build M=/mnt/usb/volatility-2.3.1/tools/linux clean
make[1]: Entering directory `/usr/src/linux-headers-3.2.0-41-generic'
  CLEAN   /mnt/usb/volatility-2.3.1/tools/linux/.tmp_versions
  CLEAN   /mnt/usb/volatility-2.3.1/tools/linux/Module.symvers
make[1]: Leaving directory `/usr/src/linux-headers-3.2.0-41-generic'
  adding: module.dwarf (deflated 90%)
  adding: boot/System.map-3.2.0-41-generic (deflated 79%)
caribou# ls /mnt/usb/capture/caribou-2014-03-29_12.06.01/
caribou-2014-03-29_12.06.01-bash
caribou-2014-03-29_12.06.01-memory.lime
caribou-2014-03-29_12.06.01-profile.zip
```
3) Check for the new profile
----------------------------

```
caribou# export VOLATILITY_PLUGINS=/mnt/usb/capture/caribou-2014-03-29_12.06.01
caribou# /mnt/usb/volatility-2.3.1/vol.py --info | grep Linux
Volatility Foundation Volatility Framework 2.3.1
linux_banner            - Prints the Linux banner information
linux_yarascan          - A shell in the Linux memory image
Linuxcaribou-2014-03-29_12_06_01-profilex64 - A Profile for Linux caribou-2014-03-29_12.06.01-profile x64
```


4) Choose the new profile and memory capture, run linux_pslist to test
----------------------------------------------------------------------
```
caribou# export VOLATILITY_PROFILE=Linuxcaribou-2014-03-29_12_06_01-profilex64
caribou# export VOLATILITY_LOCATION=file:///mnt/usb/capture/caribou-2014-03-29_12.06.01/caribou-2014-03-29_12.06.01-memory.lime
caribou# /mnt/usb/volatility-2.3.1/vol.py linux_pslist
Volatility Foundation Volatility Framework 2.3.1

Offset             Name                 Pid             Uid             Gid    DTB                Start Time
------------------ -------------------- --------------- --------------- ------ ------------------ ----------
0xffff88022e0e8000 init                 1               0               0      0x0000000228f73000 2014-03-29 14:10:23 UTC+0000
0xffff88022e0e9700 kthreadd             2               0               0      ------------------ 2014-03-29 14:10:23 UTC+0000
0xffff88022e0eae00 ksoftirqd/0          3               0               0      ------------------ 2014-03-29 14:10:23 UTC+0000
[... more output not shown ...]
```

5) Use the captured copy of /bin/bash to dump shell history with linux_bash
---------------------------------------------------------------------------
```
caribou# gdb /mnt/usb/capture/caribou-2014-03-29_12.06.01/caribou-2014-03-29_12.06.01-bash 
GNU gdb (Ubuntu/Linaro 7.4-2012.04-0ubuntu2.1) 7.4-2012.04
Copyright (C) 2012 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
For bug reporting instructions, please see:
<http://bugs.launchpad.net/gdb-linaro/>...
Reading symbols from /mnt/usb/capture/caribou-2014-03-29_12.06.01/caribou-2014-03-29_12.06.01-bash...(no debugging symbols found)...done.
(gdb) disass history_list
Dump of assembler code for function history_list:
   0x00000000004a53f0 <+0>:	mov    0x2490c9(%rip),%rax        # 0x6ee4c0
   0x00000000004a53f7 <+7>:	retq   
End of assembler dump.
(gdb) quit
caribou# vol.py linux_bash -H 0x6ee4c0 -P
Volatility Foundation Volatility Framework 2.3.1
Pid      Name                 Command Time                   Command
    -------- -------------------- ------------------------------ -------
    2604 bash                 2014-03-29 14:11:17 UTC+0000   cat workshop-outline 
    2604 bash                 2014-03-29 14:11:17 UTC+0000   sigfind -b 4096 006D6C6F6361 /dev/mapper/RD-var
```
[... more output not shown ...]
