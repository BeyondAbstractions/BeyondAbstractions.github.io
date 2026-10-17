---
date: 2026-10-17T05:18:10Z
description: ""
# image: ""
lastmod: 2026-10-17
showTableOfContents: false
tags: ["Silicon Bringup", "Validation", "System Software Integration"]
title: "Minimal Distro"
type: "post"
draft: false
---

# WIP: Assembling a Minimal Distro

## The Why?

There are many classes of computing devices, ranging from small handheld mobile devices to servers, that are said to run "Linux." However, the phrase "It runs Linux" is often thrown around without clearly defining what it means. Does it mean that the device runs just the Linux kernel, or does it mean something more, such as a Linux distribution (whatever that means)?

When you ask developers what operating system they use, they often reply with "Ubuntu," "Fedora," "Windows," etc. Many, if not most, use terms such as "Ubuntu" and "Linux" interchangeably. While doing so is convenient in everyday conversation because it abstracts away much of the underlying complexity, from a software perspective, these terms cannot be considered interchangeable. In software literature, they refer to different entities. In fact, one is a composition that includes the other.

An operating system is a composition of many pieces of software that abstracts the underlying hardware (and software, more on this later) to provide an environment in which workloads and applications can run. Software is, after all, a creative art.

Does this mean that one can create their own composition and call it an operating system? Yes! But wouldn't doing so be a Herculean task? Not necessarily. How come?

Consider an analogy: assembling LEGO pieces is not the same as manufacturing the LEGO pieces themselves. You do not need to manufacture every individual system component in order to put existing components together in different ways and create something that gives life to hardware.

If you had an inventory in which you could store software components in a consumable form, you could assemble them together, provided that each component spoke the language required by the others. It sounds too good to be true, right? It isn't easy, but it definitely isn't as hard as it might initially seem.

But why would I create my own composition? Because I can.

Jokes apart, because doing so allows one to see the structural boundaries and interfaces hidden within the blob called "Linux," "Ubuntu," or an "Operating System" that we have been throwing around as a single entity.

The end result is that you walk away understanding that interfaces and software boundaries come in various shapes, sizes, colors, and cultures. An interface does not always have to look like a man-page-documented system call such as `open`. Interfaces exist everywhere within a system, and they take many different forms.

More importantly, by putting these pieces together yourself, you begin to understand what actually allows your system to tick. You also begin to understand which components of the composition can be replaced with alternatives should you choose to do so.

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

Those nuances are often left out, reducing the exercise to a toy project at best.

## Understanding the Boot Process

## Configuring and Building the Linux Kernel

## Creating the Composition of the initramfs

## Creating the Composition of the rootfs

## Storage Layout on the USB Drive

## Installing GRUB

## Flashing the Software Image

## Booting the Hardware with the New "Distro"

## Transitioning from initramfs to rootfs

## Bring-Up Complete

## What Next?