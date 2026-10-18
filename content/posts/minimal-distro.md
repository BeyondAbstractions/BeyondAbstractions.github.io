---
date: 2026-10-17T05:18:10Z
description: ""
# image: ""
lastmod: 2026-10-17
showTableOfContents: false
tags: ["Silicon Bringup", "Validation", "System Software Integration"]
title: "WIP: Minimal Distro"
type: "post"
draft: true
---

# WIP: Assembling a Minimal Distro

## The Why?

There are many classes of computing devices, ranging from small handheld mobile devices to servers, that are said to run "Linux." However, the phrase "It runs Linux" is often used without clearly defining what it means. Does it mean that the device runs just the Linux kernel, or does it mean something more, such as a Linux distribution (whatever that means)?

When you ask developers what operating system they use, they often reply with "Ubuntu," "Fedora," "Windows," etc. Many, if not most, use terms such as "Ubuntu" and "Linux" interchangeably. While doing so is convenient in everyday conversation because it abstracts away much of the underlying complexity, from a software perspective, these terms cannot be considered interchangeable. In software literature, they refer to different entities. In fact, one is a composition that includes the other.

An operating system is a composition of many pieces of software that abstracts the underlying hardware (and software, more on this later) to provide an environment in which workloads and applications can run. Software is, after all, a creative art.

Does this mean that one can create their own composition and call it an operating system? Yes! But wouldn't doing so be a Herculean task? Not necessarily. How come?

Consider an analogy: assembling LEGO pieces is not the same as manufacturing the LEGO pieces themselves. You do not need to manufacture every individual system component in order to put existing components together in different ways and create something that gives life to hardware.

If you had an inventory in which you could store software components in a consumable form, you could assemble them together, provided that each component spoke the language required by the others. It sounds too good to be true, right? It isn't easy, but it definitely isn't as hard as it might initially seem.

But why would one create thier own composition? Because they can.

Jokes apart, because doing so allows one to see the structural boundaries and interfaces hidden within the blob called "Linux," "Ubuntu," or an "Operating System" that we have been throwing around as a single entity. It allows you to build a functioning operating system tailored to your
needs.

The end result is that you walk away understanding that interfaces and software boundaries come in various shapes, sizes, colors, and cultures. An interface does not always have to look like a man-page-documented system call such as `open`. Interfaces exist everywhere within a system, and they take many different forms.

More importantly, by putting these pieces together yourself, you begin to understand what actually allows your system to tick. You also begin to understand which components of the composition can be replaced with alternatives should you choose to do so. If a system component speaks the
language expected by the interface it can replace the existing one.

Why replace them? Maybe because doing so adds business value. Maybe because it makes you look cool or tech-savvy. Maybe it simply boosts your ego to know that you can conquer something that initially appeared complex. Whatever the reason may be, you walk away with a newfound understanding of which aspects of the composition are optional, which are necessary, and which are replaceable when creating your own "operating system."

There are, of course, other projects such as "Linux From Scratch," "Yocto," "Android," and "Buildroot" that allow you to create your own Linux-based operating system.

So, what is different here?

Those approaches generally involve building many of the components that eventually make up the system. The approach explored here is different: rather than manufacturing the LEGO pieces yourself, the goal is to consume software components that are already available in an integration-ready form from an existing software store and assemble them into a working system.

In other words, we are interested primarily in **composition rather than manufacturing**.

## What is a Linux Distribution?

A Linux distribution is a reproducible way of composing, configuring, and managing system software components to create a functioning Linux-based operating system. It defines how these components are packaged, integrated, configured, updated, secured, and maintained throughout the lifecycle of the system.

**Note:** For the purposes of this exercise, we are going to leave out the reproducibility aspect.

## Requirements

* A USB drive with at least 16 GB of storage
* An x86-64 computer or laptop running Ubuntu 24.04
* Access to the computer's BIOS/UEFI settings

## Why Actual Hardware?

With alternatives like QEMU, it is easy to test an operating system in a virtualized environment. The problem, however, is that you do not get to see many of the aspects that force you to think about how the current composition and configuration will actually run on real hardware.

Those nuances are often left out, reducing the exercise to a toy project at best. Doing
it on real harsware makes things real/authentic as it can get.

## Understanding the Boot Process

![Linux boot flow](/images/posts/minimal-distro/boot-flow-component.png)

When you turn on your computer, a couple of complex processes run before
the actual "operating system" can even start. The first program that runs
is the BIOS/UEFI firmware. UEFI then loads a UEFI application, which, for the
purpose of this exercise, is GRUB. GRUB then loads the Linux kernel and the
initial RAM disk.

Now, why this particular chain of control flow?

Recall the discussion about replaceable system components: as long as
they speak the same language, your UEFI application need not be GRUB.
It could be systemd-boot. Each component of the boot chain is replaceable.
Arranging the components around well-defined interfaces provides greater
flexibility compared to tightly coupling them together.

GRUB loads the Linux kernel and the compressed initial RAM disk into memory
and launches the kernel. The kernel then unpacks the initial RAM disk and
uses it as its initial root file system. A complete file system residing
entirely in volatile RAM!

Because the initial RAM disk ultimately occupies memory, keeping it small is
important, particularly on embedded systems where RAM may be limited. Every
unnecessary program, library, or kernel module included in the initial RAM
disk consumes additional storage and, once unpacked, additional memory.

For the purpose of this exercise, however, we will take the relatively
abundant memory available on a desktop computer for granted. A larger initial
RAM disk can also affect boot time because more data must be read from
persistent storage, loaded into RAM, and decompressed during boot.

When the kernel is launched by the bootloader (GRUB, in our case), it needs
a program called `init`, the first userspace program responsible for system
initialization. Everything else you see on a mainstream distribution, from
system services to the login screen and graphical user interface, is ultimately
started as part of the userspace initialization that begins with `init`.
Linux models processes as a tree, with the initial `init` process at its root.

This also means that your operating system need not use the `init` that came
with Ubuntu or Fedora. You can replace it with your own. We will create our
own `init` in this exercise.

Notice that there are two `init` programs in our boot process: one in the
initial RAM disk and another in the root file system. They can be the same
program or different programs. The kernel initially executes the `init` in
the initial RAM disk. That `init` prepares and mounts the real root file system
and eventually transfers control to the `init` in the real root file system.
The latter then becomes responsible for bringing up the rest of the operating
system.

The `init` program in the initial RAM disk also loads the required kernel
modules (`*.ko` files) containing the driver logic needed to access storage
devices that may not yet be accessible. In our case, this includes the drivers
required to access the USB mass-storage device containing our real root file
system. Once the storage device becomes accessible, `init` can locate and
mount the real root file system and switch to it. Other drivers can also be
loaded if you want a GUI-driven RAM disk. We will not do that here.

The initial RAM disk provides a complete file system that lives in the
computer's RAM alongside the Linux kernel. However, programs running from
the initial RAM disk need not be aware that they originate from or are
executing from a RAM-backed file system. They can operate just as they would
if their files were stored on persistent secondary storage.

How is this possible?

Because programs consume interfaces rather than implementations. A userspace
program accesses files through system calls such as `open()`, `read()`, and
`write()`. Inside the kernel, the Virtual File System (VFS) provides a common
filesystem abstraction over different underlying filesystem implementations
and storage devices.

This is also why the same `init` binary can, in principle, run from both the
initial RAM disk and the real root file system. The program does not need to
know whether the files it accesses ultimately reside in RAM, on an NVMe drive,
on a hard disk, or on a USB drive. Those implementation details are largely
abstracted away behind the interfaces provided by the kernel.

At a high level, a computer consists of two fundamental resources: compute
and storage. So, where do GRUB, the Linux kernel, the initial RAM disk, and
the root file system live?

They reside on persistent storage, which, in our case, is the USB drive.
During boot, GRUB loads the Linux kernel and the initial RAM disk from this
persistent storage into RAM, from where they are used during the next stage
of the boot process. The BIOS/UEFI firmware itself is stored in non-volatile
flash memory on the computer's motherboard.

Now, you might have heard the term "silicon bring-up." It refers to the
process of bringing newly manufactured silicon to life by getting software
running on it for the first time. This involves progressively enabling and
debugging the hardware, firmware, boot chain, Linux kernel, device drivers,
initial RAM disk, and eventually the root file system.

The goal is to move step by step through the boot chain until the complete
software stack runs successfully on the new hardware. Before this happens,
the hardware is essentially just silicon. Software gives it behavior. In
that sense, software becomes the soul of the hardware.

So, what kinds of issues might one face during bring-up?

Kernel modules might fail to load from the initial RAM disk. Even after a
kernel module (`*.ko`) is successfully loaded, the driver logic contained
within it might fail to initialize or operate correctly. The kernel itself
might panic or crash midway through the initialization of the RAM disk.
Hardware might not be discovered, the root file system might fail to mount,
or `init` itself might fail, or boot times might increase significantly.
The list of things that can go wrong is practically endless.

## Set Up the Workspace

Create a workspace directory and switch into it:

```bash
mkdir ~/workspace
cd ~/workspace
```

Unless specified otherwise, all commands from this point onward assume that
your current working directory is `~/workspace`.


## Configuring and Building the Linux Kernel

But you said no building!

Yes, I lied. I said we wouldn't build everything, not that we wouldn't build
anything. Some components need to be custom-tailored to work with our system.
The Linux kernel is one of them.

Rather than starting with the upstream Linux kernel tree (the vanilla kernel),
we will use the kernel source provided by Ubuntu 24.04. Since our computer is
already running Ubuntu 24.04 successfully, this gives us a known-good baseline
for the hardware we are targeting.

The goal is to minimize configuration drift and, more importantly, reduce
the number of unknown variables while we build our system. Starting from a
kernel source and configuration close to the one already running on the
machine allows us to change one thing at a time.

## Creating the Composition of the initramfs

```bash
mkdir initramfs
sudo 
```

## Creating the Composition of the rootfs

## Storage Layout on the USB Drive

## Installing GRUB

## Flashing the Software Image

## Booting the Hardware with the New "Distro"

## Transitioning from initramfs to rootfs

## Bring-Up Complete

## What Next?