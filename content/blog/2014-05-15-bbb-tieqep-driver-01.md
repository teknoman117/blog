+++
title = "TI eQEP Driver Tutorial"
[taxonomies]
tags = ["beagleboard", "old"]
+++

About a year ago I wrote a driver for the enhanced quadrature encoder pulse decoder unit (eQEP) resident in the TI AM33xx series system on a chips.  The unit allows one to attach a quadrature encoder to the SoC directly and have it handle counting the encoder ticks, versus needing an external controller such as an Arduino.  Both the Beaglebone and Beaglebone Black have 3 eQEP units, but unfortunately only eQEP1 and eQEP2 are broken out on the original Beaglebone.  eQEP 0 can be used without conflicts with built in hardware on the Beaglebone Black, but due to multiplexing, eQEP 1 can only be enabled when HDMI is disabled and eQEP 2 requires either the HDMI or the eMMC to be disabled.  Previously, the only interface with the driver was via the sysfs interface, making writing software around the driver fairly simple, but the textual interface was rather slow.  After multiple requests to add a device node based interface I figured i might as well go and do it.  In order to clean things up, I’m refactoring the driver, and since this is one of the rare examples of a practical kernel module I’ve decided to take the opportunity and put together a tutorial on writing a kernel module.

For simplicity’s sake, I’m going to assume you have an ARM cross compiler installed.  If you are running on Ubuntu, the arm compilers are in the standard repositories (arm-linux-gnueabihf-gcc/g++), otherwise you can grab the Linaro toolchain, which is what I use.  You’ll also need to compile the kernel so we can build modules against it.  The [kernel](https://github.com/RobertCNelson/linux-dev) used by the majority of the community for Ubuntu and Debian is Robert C Nelson’s.  I’m using the am33x-v3.8 branch, which at the time of writing builds the v3.8.13-bone52 kernel.  There are a plethora of tutorials on building the kernel, my favorite of which is [Derrek Molly’s](https://www.youtube.com/watch?v=HJ9nUqYMjqs).  This video is a bit dated, but if you replace “git checkout am33x-v3.2 -b am33x-v3.2” with “git checkout am33x-v3.8 -b am33x-v3.8” the instructions are practically identical.

We need to setup some boilerplate code at first.

```c
#include <linux/module.h>
#include <linux/printk.h>
#include <linux/types.h>
 
// Include the configured options for the kernel
#include <generated/autoconf.h>
 
// Called when the module is loaded into the kernel
static int __init eqep_init(void)
{
    printk(KERN_INFO "[TIeQEP] Module Loaded");
 
    // Successfully initialized
    return 0;
}
 
// Called when the module is removed from the kernel
static void __exit eqep_exit(void)
{
    printk(KERN_INFO "[TIeQEP] Module Exited");
}
 
// Tell the compiler which functions are init and exit
module_init(eqep_init);
module_exit(eqep_exit);
 
// Module information
MODULE_DESCRIPTION("TI eQEP driver");
MODULE_AUTHOR("Nathaniel R. Lewis");
MODULE_LICENSE("GPL");
```

At the core of every kernel module are two methods – init and exit.  Init is called when the module is loaded into the kernel and exit is called when the module is removed.  For testing purposes, we’ve just stuck printk calls (print kernel) into those methods.  module_init() and module_exit() are macros which setup the passed function as the corresponding kernel module method.

In order to load this into the kernel, we have to compile it first.  Writing a makefile for a kernel module is a little different than most,  as we technically need to build in the kernel directory.  After building the kernel as mentioned above, the actual kernel lives in linux-dev/KERNEL.

```Makefile
obj-m = tieqep.o
KDIR := /home/nathaniel/Programming/linux-dev/KERNEL
ccflags-y = -I/home/nathaniel/Programming/linux-dev/KERNEL
 
all:
    make -C $(KDIR) M=$(shell pwd) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- modules
 
clean:
    make -C $(KDIR) M=$(shell pwd) clean
```

Replace “/home/nathaniel/Programming/linux-dev/KERNEL” with where you kernel was built.  Also, this make file assumes the name of the kernel module is tieqep.c.  After running make, you will get a file named tieqep.ko.  Boot up your beaglebone and copy this file to it.  To insert to kernel module, run “sudo insmod tieqep.ko” and to remove it run “sudo rmmod tieqep.ko”.  Here is the log from my beaglebone.

```
debian@arm:~$ ls
bin tieqep.ko
debian@arm:~$ sudo insmod tieqep.ko
[sudo] password for debian:
[ 43.064141] [TIeQEP] Module Loaded
debian@arm:~$ dmesg | tail
[ 14.464018] hub 2-0:1.0: TT requires at most 8 FS bit times (666 ns)
[ 14.464028] hub 2-0:1.0: power on to power good time: 10ms
[ 14.464068] hub 2-0:1.0: local power source is good
[ 14.464160] hub 2-0:1.0: enabling power on all ports
[ 14.564117] hub 2-0:1.0: state 7 ports 1 chg 0000 evt 0000
[ 14.564178] hub 2-0:1.0: hub_suspend
[ 14.564206] usb usb2: bus auto-suspend, wakeup 1
[ 14.636179] CAUTION: musb: Babble Interrupt Occurred
[ 14.755082] gadget: high-speed config #1: Multifunction with RNDIS
[ 43.064141] [TIeQEP] Module Loaded
debian@arm:~$ sudo rmmod tieqep.ko
[ 52.082653] [TIeQEP] Module Exited
debian@arm:~$ dmesg | tail
[ 14.464028] hub 2-0:1.0: power on to power good time: 10ms
[ 14.464068] hub 2-0:1.0: local power source is good
[ 14.464160] hub 2-0:1.0: enabling power on all ports
[ 14.564117] hub 2-0:1.0: state 7 ports 1 chg 0000 evt 0000
[ 14.564178] hub 2-0:1.0: hub_suspend
[ 14.564206] usb usb2: bus auto-suspend, wakeup 1
[ 14.636179] CAUTION: musb: Babble Interrupt Occurred
[ 14.755082] gadget: high-speed config #1: Multifunction with RNDIS
[ 43.064141] [TIeQEP] Module Loaded
[ 52.082653] [TIeQEP] Module Exited
debian@arm:~$
```

In the next post I’ll go over the device tree and writing the platform device driver component.
