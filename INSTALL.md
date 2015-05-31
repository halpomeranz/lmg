Installing lmg On A Thumb Drive
===============================
Hal Pomeranz (hal@deer-run.com), 2014-03-29

###Step 1 - Thumb Drive Setup:

BE VERY CAREFUL HERE.  There is a large potential for data loss!

This example assumes you're using a 64G thumb drive, and you're following
these instructions on a Linux system with a single hard disk.

Plug the thumb drive into a spare USB port.  If you run `fdisk -l`
as root, you should see output like:
```
   Disk /dev/sda: 160.0 GB, 160041885696 bytes
   255 heads, 63 sectors/track, 19457 cylinders, total 312581808 sectors
   Units = sectors of 1 * 512 = 512 bytes
   Sector size (logical/physical): 512 bytes / 512 bytes
   I/O size (minimum/optimal): 512 bytes / 512 bytes
   Disk identifier: 0x0002a430

      Device Boot      Start         End      Blocks   Id  System
   /dev/sda1   *        2048      499711      248832   83  Linux
   /dev/sda2          501758   312580095   156039169    5  Extended
   /dev/sda5          501760   312580095   156039168   83  Linux

   Disk /dev/sdb: 61.8 GB, 61847109632 bytes
   72 heads, 8 sectors/track, 209713 cylinders, total 120795136 sectors
   Units = sectors of 1 * 512 = 512 bytes
   Sector size (logical/physical): 512 bytes / 512 bytes
   I/O size (minimum/optimal): 512 bytes / 512 bytes
   Disk identifier: 0x396a32a9

      Device Boot      Start         End      Blocks   Id  System
   /dev/sdb1            8064   120795135    60393536    c  W95 FAT32 (LBA)
```
`/dev/sda` is your primary hard drive, and `/dev/sdb` is the thumb drive 
we're going to use.  If you are not able to identify your thumb drive,
DO NOT PROCEED.  You will invariably end up destroying important data
and we will both be sad.  I am going to use `/dev/sdb` as the thumb drive
in the examples below.  BE CAREFUL AS YOU CUT AND PASTE.

Normally, I format my thumb drives as EXT4 file systems-- they're fast,
but can only be read on Linux systems.  If you want to be able to see
the captured memory images on other platforms, then format your thumb
drive as NTFS.  But the Linux NTFS-3g drivers have much slower performance
than EXT4.  Your call.

For EXT4 format:
```
   parted /dev/sdb mklabel msdos
   parted /dev/sdb mkpart primary ext4 1 62G
   mkfs -t ext4 /dev/sdb1
```
For NTFS:
```
   parted /dev/sdb mklabel msdos
   parted /dev/sdb mkpart primary ntfs 1 62G
   mkntfs -Q /dev/sdb1
```
Finally, mount the thumb drive so that we can install files on it.
I'm going to use /mnt/usb in all of the examples below, so I recommend
following these steps exactly.
```
   mkdir -p /mnt/usb
   mount /dev/sdb1 /mnt/usb
```

###Step 2 - Copy lmg Script

Copy the lmg script to the top-level directory of your thumb drive.
lmg uses the directory that it's run from as the starting point to
find all of the other files it needs to operate, so the location of
the script is important.
```
   cp /some/path/to/lmg /mnt/usb
   chmod +x /mnt/usb/lmg
```

###Step 3 - Install LiME Source Code

LiME is the kernel module lmg uses to dump memory from target Linux systems.  
Download the [souce code](http://code.google.com/p/lime-forensics/)

The lmg script will be compiling the LiME kernel module on the thumb drive 
as part of the acquisition process.  So we need to get the LiME source onto
our thumb drive:
```
   mkdir /mnt/usb/lime
   cd /mnt/usb/lime
   tar zxf ~/Downloads/lime-forensics-*.tar.gz
```
I've included a file called lime-Makefile.patch with the lmg distribution
that makes a minor tweak to the way LiME is built.  With my patch applied,
the resulting kernel module will include the CPU architecture ("i686" or
"x86_64") as part of the module name.  This makes life better when you're
dealing with both 32-bit and 64-bit systems.  YOU MUST APPLY MY PATCH or
lmg will be unable to find the appropriate kernel module for your system.

To apply the patch:
```
   cd /mnt/usb/lime/src
   patch < /some/path/to/lime-Makefile.patch
```

###Step 3 - Install (Statically-Linked) dwarfdump

**NOTE**: If you do not want lmg to automatically build Volatility(TM)
profiles for you, then you can stop right here.  lmg + LiME is enough
to capture RAM dumps from Linux systems.

But assuming you want to build Volatility(TM) profiles on the fly,
you're going to need the dwarfdump program.  And if you don't want
to be dependent on shared libraries, you're going to want to build
it statically-linked.  To save everybody some hassle, I'm including
statically-linked 32-bit and 64-bit versions of dwarfdump with lmg.
To install them onto your thumb drive:
```
   cd /mnt/usb
   tar zxf /some/path/to/static-dwarfdump.tgz
```
If you want to try to build these yourself, read on...

There's an issue around of which version of dwarfdump you want to use.
Various users have reported problems using newer versions of dwarfdump
to create Volatility(TM) profiles.  For further [details](http://code.google.com/p/volatility/issues/detail?id=260)
Per Andrew Case's comment in the thread, it appears that the v20120410
has been tested, so I recommend going with that.  The code can be found [here](http://www.prevanders.net/libdwarf-20120410.tar.gz)


To build dwarfdump statically linked, you'll need static versions of
the elfutils libraries for your architecture.  This worked for me on
my CentOS (Red Hat) system:
```
   yum install elfutils-devel-static elfutils-libelf-devel-static
```
Now download libdwarf, unpack the tar archive, and cd into the resulting
directory.  To build statically-linked executables:
```
   export LDFLAGS='-static'
   ./BLDLIBDWARF
```
Once you've built dwarfdump, copy it to the thumb drive along with the
dwarfdump.conf file.  Note that the binary must be installed using the
program name "dwarfdump-$(uname -m)" (e.g., "dwarfdump-i686") or lmg
won't find it.  The commands below will do the right thing:
```
   mkdir /mnt/usb/dwarfdump
   cp dwarfdump2/dwarfdump /mnt/usb/dwarfdump/dwarfdump-$(uname -m)
   cp dwarfdump2/dwarfdump.conf /mnt/usb/dwarfdump
```
You'll need to repeat the above steps on both a 32-bit and a 64-bit
system if you want to support both architectures.


###Step 4 - Install Volatility(TM)

Technically, building a Volatility(TM) profile for Linux only requires
the files under `.../volatility-*/tools/linux`.  But I prefer to install
the entire Volatility(TM) distro on my thumb drive.  That way I know
I'll always have a copy of Volatility(TM) handy when I want to do my
analysis.

You can grab a copy of the latest Volatility(TM) release from
```
   http://code.google.com/p/volatility/downloads/list 
```
After you've downloaded the source code, install it on the thumb drive:
```
   cd /mnt/usb
   tar zxf ~/Downloads/volatility-*.tar.gz
```
ALTERNATIVELY, grab the current source code tree using the instructions 
found at:
```
   http://code.google.com/p/volatility/source/checkout
```
Installing on the thumb drive might look like:
```
   cd /mnt/usb
   svn checkout http://volatility.googlecode.com/svn/trunk/ volatility-read-only
```
lmg doesn't really care about the exact name of the directory Volatility(TM)
gets installed in.  It will find the appropriate files automatically.
