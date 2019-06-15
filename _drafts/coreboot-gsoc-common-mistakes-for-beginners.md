---
layout: post
title: [GSoC] Common Mistakes for Beginners 
date: 2019-06-15 9:00 +0900
categories: coreboot
---

Hello everyone. I am Asami, and a student for this year's GSoC project. [My project](https://summerofcode.withgoogle.com/projects/#5148970366533632) is adding a new mainboard, QEMU/AArch64, to make it easier for coreboot developers to support new boards. I've already written a small patch to enable building a sample program with libpayload for ARM architecture. Also, I've read the implementation of coreboot (main code path) for ARM and QEMU (qemu/hw/arm/vexpress.c). Now, I just created a new CL for my main project and I started to read implementation of target machine for AArch64 (qemu/hw/arm/virt.c).

In this article, I'm going to talk about my mistakes when I develop coreboot. I hope it helps for beginners of coreboot development. The target board is QEMU/ARM and the CPU is ARMv7.


## "ERROR: Ramstage region _postram_cbfs_cache overlapped by: fallback/payload"
I faced this error when I built coreboot.rom for QEMU/ARM with payloads/coreinfo. The cause is that coreinfo doesn't support ARM architecture and then the payload is compiled as a 32-bit x86. 

Make sure your payload is your target architecture. You need to use other executable files instead of payloads/coreinfo when you want to use ARM architecture. We provide payloads/libpayload which is a libc for coreboot and you can use it.

The details of what I did are:
$ make menuconfig
  select 'Mainboard' menu
    Beside 'Mainboard vendor' should be '(Emulation)'
    Beside 'Mainboard model' should be 'QEMU armv7 (vexpress-a9)'

  select 'Payload' menu
  select 'Add a Payload'
    choose 'An Elf executable payload'
  select 'Payload path and filename'
    enter 'payloads/coreinfo/build/coreinfo.elf'
$ make
…..(omitted).....
W: Written area will abut bottom of target region: any unused space will keep its current contents
CBFS fallback/romstage
CBFS fallback/ramstage
CBFS config
CBFS revision
CBFS fallback/payload
INFO: Performing operation on 'COREBOOT' region...
ERROR: Ramstage region _postram_cbfs_cache overlapped by: fallback/payload
Makefile.inc:1171: recipe for target 'check-ramstage-overlaps' failed
make: *** [check-ramstage-overlaps] Error 1


## "ERROR: undefined reference to '_ttb'" and "ERROR: undefined reference to '_ettb'"
This errors might happen when you build coreboot.rom by `make` at root directory. In this case, You need to add TTB() at your memleyout.ld.

TTB is a translation table base address for MMU. TTBR0/TTBR1 (TTB register) holds the start point of TTB. We can put TTB anywhere in a memory when we store the address at TTBR.

According to the "ARM Architecture Reference Manual ARMv7-A and ARMv7-R edition", the difference between TTBR0 and TTBR1 is:
> (B3-1345) When a PL1&0 stage 1 MMU is enabled, TTBR0 is always used. If TTBR1 is also used then:
> * TTBR1 is used for the top part of the input address range
> * TTBR0 is used for the bottom part of the input address range

> (B4-1724) TTBCR determines which of the Translation Table Base Registers, TTBR0 or TTBR1, defines the base address for a translation table walk required for the stage 1 translation of a memory access from any mode other than Hyp mode.

TTBR0 is bacically used for user processes and TTBR1 is used for kernel. However, Linux kernel uses only TTBR0 to reduce the time for context switch. (I just heard that Linux kernel starts to use TTBR1 because of security reasons such as Meltdown and Spectre.)

In coreboot, mmu_init() sets TTBR registers in arch/arm/armv7/mmu.c.


## TODO REMOVE
## Payload is not your target architecture
Coreinfo is x86 only and is compiled using the i386-gcc.
Trying to use it on arm machine causes this issue.

I made a mistake to compile payload as 32-bit x86 architecture.
So, I created an executable elf file for 32-bit ARM and added coreboot.rom as a payload.
However, "Payload not loaded." happens when I execute on qemu-system-arm.
I'm investigating why I cannot execute coreboot.rom with an ARM elf file and this issue should be closed after solve it.

arm-eabi-gcc: error: unrecognized command line option '-m32'; did you mean '-mbe32'?

The main problem you face here is that coreinfo is compiled with i386 gcc, causing the overlap check to fail. It looks like coreinfo assumes x86.
The problem was a payload is a binary file for i386!!!!!

I need to update ARCH=arm at payloads/coreinfo/Makefile.


## Fails to build the sample program with libpayload 
We provide payloads/libpayload which is a libc for coreboot and also have sample program for it. However, 'make' got failed with the following errors:

/usr/bin/ld: cannot represent machine `arm'

The reason why this problem happens is Makefile in payloads/libpayload/sample is olddated. So I created a CL to update currest archtectures coreboot support.

https://review.coreboot.org/c/coreboot/+/33287


## "Payload not loaded"
## SELF segment doesn't target RAM and "Payload not loaded"

"Payload not loaded" happens when the load address of a payload is wrong. The load address should be placed the RAM place where anyone can use. You can define the load address via CONFIG_LP_BASE_ADDRESS if you use libpayload.

I created a CL for sample configuration.
https://review.coreboot.org/c/coreboot/+/33287



“SELF segment doesn’t target RAM:” is caused by if (prog_entry(payload) == NULL) {printk(“SELF…..”)} at ./src/lib/prog_loaders.c. prog_entry() is defined at ./src/arch/arm/boot.c.

prog_entry() calls the entry point, which is a function pointer, in struct prog defined at ./src/include/program_loading.h.

The reason why "Payload not loaded" happens is because the configuration of libpayload was not enough.
The load address of a payloa be placed the RAM place where anyone can use. You can define the load address by CONFIG_LP_BASE_ADDRESS in the libpayload directory.

The memory inside RAM can be used for payloads, so I update the base address of a sample program at payloads/libpayload/sample to 0x62030000 which is empty and in RAMSTAGE.


$ cat configs/config.emulation-qemu-arm 
CONFIG_LP_ARCH_ARM=y
CONFIG_LP_STACK_SIZE=64000
CONFIG_LP_BASE_ADDRESS=0x62030000
CONFIG_LP_TINYCURSES=y
CONFIG_LP_8250_SERIAL_CONSOLE=y
CONFIG_LP_USB=y
CONFIG_LP_USB_OHCI=y
CONFIG_LP_USB_EHCI=y
CONFIG_LP_USB_XHCI=y

Whole operation for building coreboot.rom with sample payload are:

Create libc for payload and cross compiler environment.
// At coreboot/payloads/libpayload/
$ make distclean // Always needs when switching a mainboard.
$ cp configs/config.emulation-qemu-arm configs/defconfig
$ make defconfig
$ make
$ make install

reate hello.elf as a sample payload.
// At coreboot/payloads/libpayload/sample
$ make

Create coreboot.rom with a sample payload.
// At coreboot/
$ make distclean // Always needs when switching a mainboard.
$ make menuconfig // or make defconfig
  Select payload “payloads/libpayload/sample/hello.elf”
  $ make

## Make sure to do 'make distclean' before switching your board target
'make distclean' removes build artifacts and config files. The default archtecture in coreboot is x86, so you need to do 'make distclean' when you want to use other architectures.

## Fails to update an existing CL on Gerrit 
Gerrit is a code review tool used in coreboot project. I'm familiar with GitHub so I thought the operations of Gerrit are almost same with the operations of GitHub, but it weren't.

On GitHub, developers can create a commit for each update. On the other hand, developers using Gerrit need to amend their commit until it will be merged.

Commands to create a new CL are almost same with the operations of GitHub:
$ git add <target files>
$ git commit -s
$ git push

Commands to update an existing CL are slightly different:
$ git add <target files>
$ git commit --amend --no-edit
Make sure to leave the "Change-Id" line of the commit message as is.

## Launching QEMU from GDB
This problem is not solved yet. Debugging coreboot on QEMU is tricky.

