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

## Table of Contents

- [The Why?](#the-why)
- [What is a Linux Distribution?](#what-is-a-linux-distribution)
- [Requirements](#requirements)
- [Why Actual Hardware?](#why-actual-hardware)
- [Understanding the Boot Process](#understanding-the-boot-process)
- [Set Up the Workspace](#set-up-the-workspace)
- [Configuring and Building the Linux Kernel](#configuring-and-building-the-linux-kernel)
- [Creating the Composition of the initramfs](#creating-the-composition-of-the-initramfs)
- [Creating the Composition of the rootfs](#creating-the-composition-of-the-rootfs)
- [Storage Layout on the USB Drive](#storage-layout-on-the-usb-drive)
- [Installing GRUB](#installing-grub)
- [Flashing the Software Image](#flashing-the-software-image)
- [Booting the Hardware with the New "Distro"](#booting-the-hardware-with-the-new-distro)
- [Transitioning from initramfs to rootfs](#transitioning-from-initramfs-to-rootfs)
- [Bring-Up Complete](#bring-up-complete)
- [What Next?](#what-next)

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
it on real hardware makes things real/authentic as it can get.

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

| Component | Interface Consumed | Interface Provided | Nature/Shape of the Interface | Purpose of the Interface | Current Implementation | Alternate Implementation |
|:----------|:-------------------:|:------------------:|:------------------------------|:-------------------------|:-----------------------|:-------------------------|
| BIOS/UEFI | Power-on / Reset | UEFI Services API | A C-style firmware API that provides UEFI applications with services for accessing platform resources and peripherals. During early boot, these services allow a bootloader to discover and access storage devices containing the Linux kernel and initial RAM disk. | Provides a standardized environment in which firmware and boot applications can discover hardware and load the next stage of the boot process. | UEFI firmware supplied by the PC/laptop manufacturer | coreboot + UEFI payload such as EDK II/TianoCore |
| GRUB | UEFI Services API | GRUB shell, configuration, and Linux boot protocol | The GRUB shell and configuration provide human- and configuration-facing interfaces for controlling the boot process, including kernel command-line parameters and the locations of the kernel and initial RAM disk. A binary boot protocol defines the contract used to hand control to the Linux kernel. | Locates and loads the Linux kernel and initial RAM disk into memory, supplies the kernel command line and other boot information, and transfers execution to the kernel. | GRUB 2 | systemd-boot, U-Boot, or a custom UEFI boot application |
| Linux Kernel | Linux boot protocol | System calls, virtual filesystems, sockets, device nodes, Netlink, etc. | The kernel exposes several interfaces to userspace. System calls form the fundamental programmatic interface. Higher-level C library functions commonly wrap these system calls. Other kernel interfaces appear as files, device nodes, sockets, and message-based protocols. | Provides controlled access to kernel-managed resources such as processes, memory, filesystems, devices, networking, scheduling, and IPC. | Linux kernel | A custom kernel implementing the required boot protocol and userspace interfaces, or another compatible OS kernel |
| Initial RAM Disk | Linux kernel userspace interfaces | Initial userspace environment and `/init` | A small temporary root filesystem loaded into memory. The kernel mounts it as the initial root filesystem and executes `/init`. `/init` can ultimately be a shell script or an executable implemented using C, C++, Rust, systemd, BusyBox, or another suitable runtime. | Performs the userspace work required before the real root filesystem can be used, such as loading required kernel modules, discovering storage, assembling storage stacks, checking or repairing filesystems when appropriate, mounting the real root filesystem, and performing `switch_root`. | debootstrap-generated Ubuntu userspace with a custom `/init` shell script | BusyBox-based initramfs, systemd-based initramfs, dracut, or a custom statically linked `/init` |
| Root File System | Linux kernel filesystem/syscall interfaces | Userspace runtime environment | A persistent filesystem hierarchy containing executables, shared libraries, configuration, device-management infrastructure, system services, applications, and other userspace components. After the root transition, the kernel executes the real system's init process from this filesystem. | Provides the persistent userspace environment from which the system is initialized and brought to its normal operational state. | debootstrap-generated minimal Ubuntu rootfs on ext4 | Debian, Alpine, Buildroot, Yocto-generated rootfs, or a completely custom userspace |

From the table above, it is clear that many components of the system can be replaced as long as
the replacement continues to satisfy the interfaces expected by the components on either side.

For example, you could replace the Linux kernel with your own kernel, provided that it understands
the boot protocol used by the bootloader on that platform and provides the interfaces required by
the userspace software you intend to run.

If you do not require compatibility with Linux userspace at all, you could go even further and
replace the Linux kernel and its userspace environment entirely with an RTOS or your own operating
system, provided that it can be loaded and entered through the platform's boot mechanism.

Similarly, the init process in both the initial RAM disk and the root filesystem can be replaced
with much simpler alternatives to systemd. The initramfs `/init`, for example, could be a shell
script, a BusyBox-based init, or a custom executable, provided that it performs the initialization
required to reach and transition to the real root filesystem.

Likewise, the init process of the real root filesystem does not have to be systemd. It can be any
suitable executable capable of initializing and managing the userspace environment required by
that particular system.

From the above, it is clear that understanding the interfaces between system components, combined
with a moderate deep dive into a reference implementation of those interfaces, provides a good
understanding of the behaviour expected at each boundary. It also reveals how implementation
details are abstracted away, allowing components on either side of the interface to evolve or be
replaced independently.

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
cd ~/workspace
mkdir initramfs
sudo debootstrap --arch=amd64 --variant=minbase noble initramfs http://archive.ubuntu.com/ubuntu
```

The above command creates the initial composition of the filesystem that will later become our
initial RAM disk. `debootstrap` constructs this filesystem using software artifacts obtained from
the Ubuntu software repository. These artifacts are distributed in a consumable format called
Debian packages (`.deb`).

The resulting `initramfs/` directory therefore represents our initial selection of userspace
components. We can add, remove, or configure packages before eventually packaging this filesystem
into an initramfs image that can be loaded by the bootloader.

A key idea to understand before going further is the concept of **reproduction**.

Humans create other humans, computers are used to build other computers, and software is used to
build other software. Operating systems are no different. An existing Linux system can provide
the tools and environment required to construct another Linux system.

This is the fundamental idea behind `debootstrap`. Starting from an already functioning system,
we retrieve a minimal set of Ubuntu packages and use them to construct a new Ubuntu filesystem
hierarchy. The resulting system does not need to contain the same software as the host. It can
instead be a much smaller and highly customized composition containing only the components
required for its intended purpose.

In other words, an existing distribution provides the environment from which we can bootstrap a
new and customized instance of a distribution.

Now that we have a base filesystem, how do we go about customizing our newborn distribution?
 
Recollect that we are building this distribution for the same hardware architecture as the
distribution we are currently running. This gives us an interesting opportunity: we can actually
run programs from the new filesystem while continuing to use our existing system.
 
Every process has a view of what it considers to be its root directory (`/`). You can see the root
directory of your current shell through `/proc`:
 
```bash
ls -l /proc/self/root
```
 
Interesting. Does this mean that a process can have a different root directory from the rest of the
running system?
 
Yes!
 
Enter `chroot`.
 
`chroot` allows us to run a program with a different directory treated as its root directory (`/`),
without changing the root filesystem used by the rest of the running system.
 
For example, if our newly created filesystem is located at:
 
```text
~/workspace/initramfs/
```
 
we can enter it with:
 
```bash
sudo chroot ~/workspace/initramfs /bin/bash
```
 
From the perspective of the newly started shell, `~/workspace/initramfs` is now `/`.
 
Therefore, something that appears as:
 
```text
~/workspace/initramfs/etc
```
 
from the host system appears as:
 
```text
/etc
```
 
from inside the `chroot`.
 
We can now run tools from within this new filesystem and begin configuring it almost as though it
were a running system of its own.
 
An important distinction is that `chroot` only changes the process's view of the filesystem root.
It does not give the process a different kernel. Programs running inside the `chroot` are still
ordinary processes running on the host Linux kernel.
 
This shared-kernel property is exactly what makes `chroot` useful here. We can execute the programs
installed in our newly bootstrapped filesystem and configure the distribution before it has ever
booted on its own.

`chroot` exists both as a userspace program and as a system call. Looking through the man pages,
you will realize that the underlying mechanism is surprisingly simple. The `chroot` system call
changes the root directory associated with a process. Consequently, when a program inside the
`chroot` accesses an absolute path such as `/etc/passwd`, the kernel begins pathname resolution
from the process's new root directory rather than from the root of the original filesystem
hierarchy. For example, if the process was chrooted to `/home/user/initramfs`, `/etc/passwd` would
resolve to `/home/user/initramfs/etc/passwd` when viewed from the host. Conceptually, this may look
like the kernel is simply prepending the new root to every absolute path, but the actual mechanism
is that the process's starting point for absolute pathname resolution has been changed.

Again, because every process has the concept of a root directory (`/`) as part of its filesystem
view, that root can be changed on a per-process basis. This is the fundamental idea that makes
`chroot` possible: instead of changing the root filesystem of the entire running system, we can
change the root directory from which pathname resolution begins for a particular process and its
descendants.

Now, before we enter the `initramfs` using `chroot`, what program are we actually going to run
inside it? A shell, specifically an instance of Bash!

When we execute `chroot` and ask it to start `/bin/bash`, the `/bin/bash` binary is resolved relative
to the new root filesystem. Therefore, the Bash executable being started is the one contained
within our newly created filesystem, not the `/bin/bash` binary from the host root filesystem.

On a normally running root filesystem, the `mount` command will show several filesystems mounted
at different mount points. Some of these provide important kernel interfaces that userspace
programs expect to access. The most notable are `devtmpfs` at `/dev`, `sysfs` at `/sys`, and
`procfs` at `/proc`.

Recollect that `chroot` changes the root directory from which a process resolves absolute paths.
Consequently, a process started inside our new root will see the `/dev`, `/sys`, and `/proc`
directories belonging to the new filesystem hierarchy, but these directories do not automatically
contain the filesystems mounted at those locations on the host.

Linux allows an existing mount to be made visible at another location in the filesystem hierarchy
using a bind mount. We can therefore make the host's `/dev`, `/sys`, and `/proc` hierarchies
available at the corresponding locations inside our new root. Before starting Bash inside the
`chroot`, we bind mount these important kernel-facing filesystems into it. This gives programs
running inside the `chroot` access to devices through `/dev`, kernel and device information through
`/sys`, and process and kernel information through `/proc`.

And once we are done customizing the new root filesystem, we simply unmount the bind mounts that
we created. This removes the views of the host's `/dev`, `/sys`, and `/proc` filesystems from the
new root filesystem, returning it to a self-contained filesystem hierarchy ready for the next
stage of our build process.

```bash
# Make important kernel interfaces available inside the new root
sudo mount --bind /dev  initramfs/dev
sudo mount --bind /sys  initramfs/sys
sudo mount --bind /proc initramfs/proc

# Enter the new root filesystem
sudo chroot initramfs /bin/bash

# Install userspace utilities for managing kernel modules
apt install kmod

# List the baseline set of installed packages
dpkg -l
```

Installing packages into the `initramfs` generally requires entering the `chroot` environment so
that the package-management tools operate relative to the new root filesystem. Editing the contents
of the `initramfs`, however, only requires sufficient permissions, typically root permissions.

This means that tools already available on the host distribution can be used to create, modify,
copy, or remove files within the `initramfs` without installing those tools into the `initramfs`
itself. This is particularly useful when building a minimal system, as build-time tools do not
need to become part of the final filesystem, thereby conserving disk space and keeping the
resulting system small.

```bash
# Within the host shell, outside the initramfs chroot
cd ~/workspace

sudo su
cd initramfs

cat > init <<'__EOF__'
#!/bin/bash

export PS1='\u@initramfs-\h:\w\$ '

cd /
./setup.sh

exec /bin/bash
__EOF__

chmod +x init

cat > setup.sh <<'__EOF__'
#!/bin/bash

# Create mount points for kernel-provided filesystems
mkdir -p /proc /sys /dev /tmp

# Mount the major kernel interfaces
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
mount -t tmpfs tmpfs /tmp

# Create and mount the pseudo-terminal filesystem
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts

# Create and mount the POSIX message queue filesystem
mkdir -p /dev/mqueue
mount -t mqueue mqueue /dev/mqueue

# Create shared-memory storage
mkdir -p /dev/shm
mount -t tmpfs tmpfs /dev/shm
__EOF__

chmod +x setup.sh

```

You have now created your own `init` process. It sets up the minimal userspace environment required
for the initial RAM disk and then launches a shell.

The `exec /bin/sh` directive is particularly important. It does not create another process.
Instead, `exec` replaces the program currently running as the `init` process with `/bin/sh` while
retaining the same process ID. Since `/init` was started by the kernel as PID 1, the resulting
shell continues running as PID 1.

In other words, the shell script initially acts as `init`, performs the required setup, and then
transforms itself into the interactive shell through `exec`.

Now we install the kernel modules and the vDSO into the initial RAM filesystem from the host
machine. Please refer to the section
[Configuring and Building the Linux Kernel](#configuring-and-building-the-linux-kernel).

```bash
# Within the host shell, outside the initramfs chroot
cd ~/workspace/kernel-src

sudo su

export KBUILD_OUTPUT=TODO

# Install kernel modules into the initramfs
make INSTALL_MOD_STRIP=1 \
     INSTALL_MOD_PATH=/.../initramfs \
     modules_install

# Install the vDSO into the initramfs
make INSTALL_MOD_STRIP=1 \
     INSTALL_MOD_PATH=/.../initramfs \
     vdso_install
```

`INSTALL_MOD_PATH` specifies the filesystem into which the kernel modules are staged. Instead of
installing the modules into the host root filesystem, `modules_install` installs them under the
corresponding `/lib/modules/<kernel-version>/` hierarchy within our `initramfs`.

`INSTALL_MOD_STRIP=1` strips debugging information from the installed kernel modules, helping
reduce the size of the resulting initial RAM disk.

Similarly, `vdso_install` installs the kernel-provided vDSO artifacts into the new filesystem
rather than into the host filesystem.

The initial RAM disk is a `cpio` archive. We need to take the directory tree under `~/workspace/initramfs` and convert it into a `cpio` archive.

Before creating the archive, we need to unmount `/dev`, `/sys`, and `/proc` from the initramfs directory tree. We do not want the host's `/dev`, `/sys`, and `/proc` to become part of the archive. These mount points will later be populated with the appropriate filesystems by `setup.sh`, which is run by `init` from within the initial RAM disk.

```bash
sudo umount ~/workspace/initramfs/dev
sudo umount ~/workspace/initramfs/sys
sudo umount ~/workspace/initramfs/proc

# The ramdisk is located at ~/workspace/initramfs.cpio.gz
cd ~/workspace/initramfs
sudo su
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
```

The disadvantage of `debootstrap`, compared with more complete build systems(buildroot, yocto, soong),
is that it does not build the userspace components from source. Instead, it assembles the new filesystem from
pre-built Debian packages (`.deb`) obtained from a package repository. As a result, you have
considerably less opportunity for build-time customization of the individual userspace components,
such as changing compile-time options, applying source patches, or enabling and disabling features.

Another consideration is the target architecture. The simplest `debootstrap` workflow assumes that
the binaries placed in the new filesystem can be executed by the host while constructing and
customizing it. Therefore, building for the same architecture as the host is straightforward.
Cross-architecture bootstrapping is possible, but requires additional mechanisms, such as
cross-architecture emulation, if binaries from the target filesystem need to be executed on the
host which is mostly the case other than toy distros like the one here.

## Creating the Composition of the rootfs

```bash
cd ~/workspace
mkdir rootfs

# Create a minimal Ubuntu root filesystem
sudo debootstrap --arch=amd64 --variant=minbase \
    noble rootfs http://archive.ubuntu.com/ubuntu

# Make important kernel interfaces available inside the new root
sudo mount --bind /dev  rootfs/dev
sudo mount --bind /sys  rootfs/sys
sudo mount --bind /proc rootfs/proc

# Enter the new root filesystem
sudo chroot rootfs /bin/bash

# Install userspace utilities for managing kernel modules
apt install kmod

# Exit the chroot and return to the host
exit

# From the host shell, outside the rootfs chroot
cd ~/workspace

sudo su
cd rootfs

# Copy the initial RAM disk created earlier into /boot
mkdir -p boot
cp ../initramfs.cpio.gz boot/

# Create the init process for the rootfs
cat > init <<'__EOF__'
#!/bin/bash

export PS1='\u@rootfs-\h:\w\$ '

cd /
./setup.sh

exec /bin/bash
__EOF__

chmod +x init

# Create the rootfs setup script
cat > setup.sh <<'__EOF__'
#!/bin/bash

# TODO: Add the required mount operations here.

__EOF__

chmod +x setup.sh

# Move from ~/workspace/rootfs to ~/workspace/kernel-src
cd ../kernel-src

export KBUILD_OUTPUT=TODO

# Install the kernel modules into the rootfs
make INSTALL_MOD_STRIP=1 \
     INSTALL_MOD_PATH=/absolute/path/to/workspace/rootfs \
     modules_install

# Install the vDSO into the rootfs
make INSTALL_MOD_STRIP=1 \
     INSTALL_MOD_PATH=/absolute/path/to/workspace/rootfs \
     vdso_install

# Install the kernel image into rootfs/boot
make INSTALL_PATH=/absolute/path/to/workspace/rootfs/boot \
     install

cd ~/workspace

# Create a 4 GiB image file
truncate -s 4G rootfs.ext4

# Create an ext4 filesystem and populate it with the contents of rootfs/
sudo mkfs.ext4 -F -d rootfs rootfs.ext4

```

The rootfs is similar in composition to the initial RAM disk. However, observe that the rootfs itself contains the compressed initial RAM disk image. This raises an interesting question: how can the initial RAM disk be accessed if it resides inside the rootfs, when the drivers required to access the USB storage containing the rootfs are themselves inside the initial RAM disk?

In other words, you need the storage drivers (`*.ko` kernel modules loaded using `modprobe`) from the initial RAM disk to access the rootfs, but the initial RAM disk image is stored inside the rootfs. You seemingly need the rootfs to access the initial RAM disk, while simultaneously needing the initial RAM disk to access the rootfs. A classic chicken-and-egg problem?

Not really. It turns out that GRUB is quite powerful. GRUB has its own filesystem support and can understand the `ext4` filesystem independently of the Linux kernel. Using its configuration, GRUB locates the initial RAM disk image within the filesystem, reads it from the USB storage, and loads it into memory.

GRUB then loads the Linux kernel and provides the information necessary for the kernel to locate the initial RAM disk in memory as part of the bootloader-to-kernel boot protocol. The kernel can therefore access the initial RAM disk before it has mounted or accessed the rootfs itself.

The bootstrapping sequence therefore looks like this:

`GRUB → reads ext4 → loads kernel + initramfs into memory → kernel starts → initramfs provides storage drivers → kernel gains access to USB storage → rootfs becomes accessible`

This again demonstrates how a well-defined interface allows multiple components to work with the same underlying implementation. The `ext4` filesystem can be accessed by both the Linux kernel and GRUB because `ext4` has a well-defined on-disk format. This format specifies how filesystem structures and data are organized within a linear sequence of bytes on a storage device.

As long as a component understands this format, it can interpret the bytes and navigate the filesystem. The Linux kernel provides one implementation of this logic, while GRUB provides another. Both implementations operate on the same `ext4` filesystem because they understand the same on-disk representation.

How, then, is it possible for UEFI/BIOS to access the USB storage medium, parse the bytes stored on it, and discover that a UEFI application exists there?

It turns out that the firmware has built-in support for accessing certain storage devices, including USB mass-storage devices. Without such support, the firmware would not be able to boot from USB storage in the first place.

In the case of UEFI, this storage access is exposed to UEFI applications through UEFI services and protocols. GRUB, while running as a UEFI application, can use these firmware-provided interfaces to access the USB storage device. UEFI also understands the filesystem used by the EFI System Partition, typically FAT, allowing it to locate and load the GRUB UEFI executable.

Once GRUB starts executing, it can use UEFI-provided services for low-level access to the storage device while using its own filesystem implementations, such as its `ext4` module, to interpret the filesystem containing the kernel and initial RAM disk. GRUB can then locate the kernel and initial RAM disk, read their bytes from storage, load them into memory, and finally hand control over to the Linux kernel.

This gives us another example of interfaces being layered on top of one another:

`USB storage hardware → firmware USB/storage support → UEFI protocols → GRUB → GRUB filesystem implementation → kernel + initramfs`

Also notice that you are converting the directory structure of the rootfs into an arrangement of bytes within a file called `rootfs.img`, essentially serializing the filesystem.

Again, this is possible because the userspace tool performing this operation implements the `ext4` on-disk format. It understands how directories, files, permissions, ownership, and other filesystem metadata must be represented as `ext4` filesystem structures within a linear sequence of bytes.

The resulting `rootfs.img` is therefore no longer merely a representation of a directory tree. It is an actual `ext4` filesystem encoded within a regular file, which can later be written directly into a partition on the USB storage.

Is it necessary to store the initial RAM disk within the rootfs? No.

Many desktop distributions store the kernel and initial RAM disk alongside other boot-related files in `/boot`. This makes it convenient for utilities provided by the distribution to generate or update the initial RAM disk whenever the kernel, kernel modules, or other boot-critical components change.

In production embedded systems, however, the kernel and initial RAM disk are often treated as boot artifacts and may be placed in dedicated boot partitions or packaged into bootable images, depending on the platform. Separating these artifacts from the rootfs allows the early-boot environment and the rootfs to be managed and updated independently. This is particularly useful when implementing robust OTA update and recovery strategies.

For example, if an OTA update to the rootfs fails, a known-good early-boot environment can still be available. The system can boot into this environment, provide access to a recovery shell or recovery application, and allow the rootfs to be inspected, repaired, rolled back, or otherwise recovered. It also provides an environment from which failures occurring during the next stage of boot can be triaged and debugged.

Do desktop systems not face the same problem? They do. The difference is primarily in how the system is designed and updated. On a typical desktop distribution, the kernel, initial RAM disk, bootloader configuration, and rootfs are usually managed together by the distribution's package-management and boot infrastructure. Embedded systems, particularly those designed for reliable OTA updates, are more likely to explicitly separate these components and introduce mechanisms such as redundant boot slots or dedicated recovery environments.

## Storage Layout on the USB Drive

Now that we have the contents of the rootfs, it is time to look at how to structure the USB storage.

If you imagine USB storage as a linear array of persistent bytes that you can read from and write to, it is simply too large to manage in its raw form. You need a way to compartmentalize it. Each such compartment is called a partition, and it represents a range of bytes whose contents can be organized according to the structure and policies of a particular file system.

We need to create two such partitions: an EFI System Partition (ESP) and a rootfs partition. The ESP is FAT32 based and rootfs
is ext4 based.

## Installing GRUB

```bash
sudo grub-install --target=x86_64-efi --efi-directory=... --boot-directory=... --removable --no-nvram --recheck
```

## Flashing the Software Image

Now that we have organized the storage on the USB drive, we need to transfer the contents of `rootfs.img` into the second partition of the USB drive.

On my machine, this partition is `/dev/sdb2`. Discover the corresponding partition on your machine by exploring `/dev` or, preferably, using `lsblk`.

```bash
lsblk

sudo dd if=rootfs.img of=/dev/sdb2 status=progress conv=fsync
```

Be extremely careful when selecting the output device (`of`). Writing to the wrong partition with `dd` will overwrite its contents.

## Booting the Hardware with the New "Distro"

Insert the USB drive into your computer and press the appropriate hotkey during startup to enter the UEFI/BIOS settings.

Disable Secure Boot if the kernel and bootloader you built are not appropriately signed, and configure the boot order. You should typically see a boot entry corresponding to the USB storage if its EFI System Partition (ESP) has been set up correctly. Make the USB storage the first boot medium, save the configuration, and exit.

The firmware should now discover the UEFI bootloader on the ESP and launch GRUB.

## Transitioning from initramfs to rootfs

You should see the GRUB menu appear. Select `Debug Linux`.

The Linux kernel should start, load the initial RAM disk, execute its `init` process, and eventually present a shell on the console.

At this point, the kernel may not yet have access to the partition containing the rootfs. You need to load the required kernel modules using `modprobe` so that the kernel can discover the USB storage device and its partitions.

Once `/dev/sdb2` (or the corresponding device on your machine) becomes available, mount the rootfs and transition from the initramfs environment to the real root filesystem. The `init` process from the rootfs can then take over the next stage of system initialization.

## Bring-Up Complete

## What Next?