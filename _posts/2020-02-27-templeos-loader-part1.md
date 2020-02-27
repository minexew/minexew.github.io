---
layout: post
title:  "TempleOS programs in Linux user-space, part 1: Motivation"
comments: true
---

For an eternity, [Shrine](https://github.com/minexew/Shrine) has suffered from a fragile, clunky build process. Perhaps this is a natural result of the impedance mismatch between the isolationist nature of TempleOS and a desire to publish ISOs online. I also placed great emphasis on making my own changes clearly distinguished from the original code, which further complicated the build process and even the structure of the repository.
But what if we could bring these two worlds closer together? Indeed, there are several projects aiming for that exact goal: Shrine’s own *Mfa* (minimalist file access) allows copying files between a TempleOS VM (guest) and the host system. *HGBD* is a more advanced approach that makes use of shared memory. There is also the classic approach of mounting a second hard drive to the VM (with a FAT32 filesystem) and extracting/inserting data after the VM is shut down. All of these can be used to build a mostly automated pipeline, but a VM is still necessarily involved, there may be timing-related pitfalls, and the whole thing is a tangled mess.

This could be potentially solved if we were able to compose the distribution ISO directly from the host operating system. This would roughly entail:

- Compiling the kernel
- Compiling the compiler
- Compiling the bootloader
- Creating a RedSea disk image

Let’s take it from the end: the RedSea format is straightforward enough (on purpose!), surely it would not take more than an afternoon to implement a program in Python to create a RedSea image from the contents of a directory. Right after that, things get very tricky, though.
Over the years, there have been more than one attempts at producing a native HolyC compiler for Linux:

- [HolyC-for-Linux](https://github.com/jamesalbert/HolyC-for-Linux) is written in Python and translates HolyC to C, for compilation with a native toolchain
- [Tabernacle](https://github.com/nrootconauto/Tabernacle) seems to be composed of a direct translation of HolyC to C, and a planned LLVM back-end

Needless to say, neither of these got far. It is easy to underestimate the tremendous effort required to produce even a simple toolchain from scratch. And still, this is only one part of the equation. Do not forget that the TempleOS source code includes a considerable amount of both inline and stand-alone assembly.

Clearly, to have any chance of achieving real-world usability in a reasonable timeframe, the chosen approach must maximize re-use of existing code -- ideally, the entirety of the compiler written by Terry Davis himself!

A TempleOS installation always includes two copies of the compiler -- source and binary. This is necessary to avoid a chicken-and-egg problem. A large part of TempleOS is compiled just-in-time, after the operating system kernel is loaded on boot. This includes Adam, a rough equivalent of a UNIX *init* process. But how to compile the compiler? A possible approach would be to write the compiler in a minimalist language that can have its compiler included directly in the kernel. Terry decided for a different way; in a running TempleOS system, the compiler is built as a relocatable binary image. The kernel can load this image on boot, resolve imports, and ingest exported functions into the symbol hash table (in fact, a large part of the kernel is loaded in the very same way).

Now, who says that it has to be the TempleOS kernel who loads the image, resolves the imports, and jumps to the main body of code? Could we not implement all of the operating system API, and just use the compiler binary as is? Well, yes, and this road has been taken many times before, perhaps most famously by the Wine project, which reimplements the Windows API in order to allow running unmodified Windows programs on other operating systems. In case of TempleOS, this is somewhat complicated by the fact that there is no clearly separated "public" kernel API, and by the fact that the semantics of many functions are vague, at best (the implementation *is* the specification)
There are other concerns such as mismatch in ABI and calling conventions, which we will conveniently skip for the moment. Nevertheless, this goal is certainly achievable; in fact, a proof-of-concept [has been demonstrated](https://www.reddit.com/r/TempleOS_Official/comments/f9mtah/running_native_temple_os_binaries_on_linux_and/) very recently.

The project above is interesting for another reason: it does not re-implement all kernel APIs from scratch. It re-uses the compiled kernel as a library of functions, and exposes those to the program it loads. This begs a question: since we are able to load the kernel, and since the kernel is able to execute TempleOS binaries, why not take the whole kernel and run _that_?

And indeed, that is exactly what this series will explore.
