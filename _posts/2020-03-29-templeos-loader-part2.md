---
layout: post
title:  "TempleOS programs in Linux user-space, part 2: Anatomy of a kernel"
comments: true
---

_[Last time](../../02/27/templeos-loader-part1.html), we discussed why it might be desirable to run TempleOS on Linux in some form other than a full-blown virtual machine, and we teased some possible approaches. In the end, we commited to finding out whether it would be possible to run the standard kernel as a user-space program. Today, we will see what we are up against._

Being at the heart of TempleOS, the kernel, consisting of about 22 000 lines of source code, has several crucial responsibilities:

- Initialization and interfacing with hardware
- Memory management
- I/O services, including file management
- Task management and multi-tasking
- Providing a standard library of functions for compression, date/time handling, maths and string operations

The last bullet point demonstrates how closely TempleOS programs are coupled to the kernel, providing a very coherent experience for the programmer on the one hand, but a very difficult emulation target on the other hand. (Note the contrast with the Linux world for example, where the user space C runtime library interfaces with the kernel through a limited set of system calls. This architecture is what allows Microsoft’s [WSL](https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux#WSL_1) to work without any "real" Linux kernel).

## Genesis

Since UEFI booting is not supported, the kernel is always born in 16-bit mode, with only one CPU core up and running. A compact 16-bit bootloader loads the entire binary kernel image at once, which limits its size to 640 KiB (and requires that the image is stored in one continuous block on disk). From there, roughly the following happens, in order:

#### KStart16.HC

1. VGA graphics is initialized via a [BIOS call](http://stanislavs.org/helppc/int_10-0.html) (if enabled at kernel compile time) [[L84](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KStart16.HC#L84)]
2. Memory map is discovered, [A20 gate](https://en.wikipedia.org/wiki/A20_line) configured [[L81](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KStart16.HC#L91)]
3. PCI buses are enumerated [[L147](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KStart16.HC#L147)]
4. The load segment of the kernel is determined and stored into `MEM_BOOT_BASE` [[L174](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KStart16.HC#L174)]
5. A pointer to the kernel's `patch_table_offset` is saved (this will come in handy later) [[L181](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KStart16.HC#L181)]
6. A minimal page table is set up and the processor is switched to [32-bit protected mode](https://wiki.osdev.org/Protected_Mode) [[L186](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KStart16.HC#L186)]

#### KStart32.HC

1. (Boring: 32-bit control & segment registers are initialized for the first time) [[L75](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KStart32.HC#L75)]
2. The kernel is [relocated](https://en.wikipedia.org/wiki/Relocation_(computing)) --- it must be ensured that any absolute addresses in the code correspond to the actual kernel load address. [[L95](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KStart32.HC#L95)]
3. More "boring" x86 initialization (`SYS_FIND_PCIBIOS_SERVICE_DIR`, `SYS_INIT_16MEG_SYS_CODE_BP`) [[L109+](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KStart32.HC#L109)]
4. Finally, we enter 64-bit mode via _[SYS_ENTER_LONG_MODE](https://github.com/cia-foundation/TempleOS/blob/09f344d2a97bad5e37e3c6b657360e16d72a80e1/Kernel/KStart64.HC#L44)_

#### KStart64.HC

1. x87 FPU is enabled [[L5](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KStart64.HC#L5)]
2. Internal kernel structures are prepared (CPUStruct, Adam HeapCtrl, Adam stack) [[L9+](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KStart64.HC#L9)]
3. Adam task (aka init process) is established [[L34](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KStart64.HC#L34)]
4. We jump to _KMain_

#### KMain.HC

1. Enter [KMain](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KMain.HC#L135) --- the first HolyC code to run after boot
2. More internal structures are set up, but more importantly:
3. The kernel symbols are loaded and dynamic linking is performed via [LoadKernel](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KLoad.HC#L240). More on this later!
4. Data structures for graphics are constructed
5. The white-on-black [boot-time version banner](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KMain.HC#L160) is printed now (not to be confused with the [other banner](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/HomeSys.HC#L38), printed by HomeSys after boot)
6. Hardware timers are configured, memory size is checked against the requirement of 512 MiB
7. Interrupts are enabled [[L167+](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KMain.HC#L167)]
8. Block devices (disks, optical drives) are initialized and file systems mounted [[L175](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KMain.HC#L175)]
9. The rest of the CPU cores are brought up [[L199](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KMain.HC#L199)]. Note that while TempleOS suports a large number of cores, it is [_not_ SMP](https://templeos.holyc.xyz/Wb/Doc/MultiCore.html).
10. Keyboard & mouse input is initialized [[L203](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KMain.HC#L203)]
11. The HolyC compiler binary is loaded. [[L210](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KMain.HC#L210)] The kernel contains direct calls to compiler functions like ExeFile. How? The kernel is built against the Compiler headers, and these calls are only resolved at this point.
12. StartOS.HC is compiled and run [just-in-time](https://templeos.holyc.xyz/Wb/Doc/Glossary.html#l221) in context of the [Adam task](https://templeos.holyc.xyz/Wb/Doc/Glossary.html#l171) [[L216](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KMain.HC#L216)]


#### StartOS.HC

_(Now we are no longer in the [AOT-compiled](https://templeos.holyc.xyz/Wb/Doc/Glossary.html#l208) kernel, but for completeness, it is useful to understand the final steps of the start-up)_

1. The JIT compilation context starts out as a tabula rasa. No function prototypes or global variables are known, only the basic built-in types.
2. Therefore, kernel headers must be parsed to gain access to its public functions, structures and exported variables. Since Adam is the [father of all tasks](https://templeos.holyc.xyz/Wb/Doc/GuideLines.html#l26), this environment will be inherited by all user programs. [[L8](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/StartOS.HC#L8)]
3. The Adam environment is loaded [[L18](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/StartOS.HC#L18)]
4. We proceed to build up the user space, opening a DolDoc window, starting up the window manager, and applying user customizations via [MakeHome.HC](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/MakeHome.HC)
5. The OS is now ready to use. The kernel keeps running behind the scenes --- handling interrupts, some timed events, and, of course, function calls from "user space"

## Memory map

The [virtual memory model](https://compas.cs.stonybrook.edu/~nhonarmand/courses/fa17/cse306/slides/07-x86_vm.pdf) model in x86 is super complex. And TempleOS uses none of it! In fact, the memory map is as simple as could ever be: virtual address space and physical address space are identity-mapped. The only things obstructing full access to the computer's RAM are some BIOS areas and memory-mapped devices.

{% include figure-seamless.html alt="" src="2020-03-29-templeos-loader-part2/templeos-memory.svg" caption="An almost 1-to-1 mapping. Even address 0x0 is a valid memory location!" %}

By default, memory access is cached. For accessing hardware, this is usually undesirable, so an _uncached alias_ is available, which allows access to all of the same memory locations, but bypassing the cache entirely. (Keep in mind that the 64-bit address space is much larger than all the RAM you could ever have, so there is no point in trying to be economical)

Since the use of the uncached alias is so niche, from now on we shall pretend it doesn't exists.

{% include figure-seamless.html alt="" src="2020-03-29-templeos-loader-part2/templeos-memory-2.svg" caption="Code and data memory" %}

It is also worth noting that to reduce binary program size, TempleOS decidedly allows code to reside only in the lower 2 GiB of memory. (the technical reason is that this makes it possible to use 32-bit relative jumps everywhere)

There are such concepts as [code heap](https://templeos.holyc.xyz/Wb/Doc/Glossary.html#l185) and data heap, and the boundary is not always at 2 GiB, but the details are not terribly important. More details can be found in [MemOverview.DD](https://templeos.holyc.xyz/Wb/Doc/MemOverview.html).

## Kernel dynamic linking

Interestingly, the kernel is not compiled as a flat binary. The BIN format, which is also used for the AOT-compiled compiler, has a level of complexity comparable to [16-bit Windows Executables](https://en.wikipedia.org/wiki/New_Executable).

{% include figure-seamless.html alt="" src="2020-03-29-templeos-loader-part2/templeos-bin.svg" caption="Structure of a BIN file" %}

Dynamic linking is used even to resolve some of the function calls internal to the kernel.

One way to explore BIN files is a command-line tool called [bininfo](https://github.com/cia-foundation/bininfo). It generates a textual dump of the kernel's headers which can be found [here](https://github.com/cia-foundation/bininfo/blob/92e273972cb304828aef75aedd2fa48682080394/.github/expected/Kernel.txt). In there, you can see an import of `ExeFile` (a compiler function), as well as `SET_GS_BASE` provided by the kernel itself. Note that the BIN file doesn't give any hints about _where_ the imported functions come from!

## Task scheduling

Since everything shares a single address space, TempleOS [tasks](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KernelA.HH#L3271) (aka processes) are quite lightweight and context switching is very fast. Multi-tasking is cooperative --- a task runs until it give up the CPU by calling _[Yield](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/Sched.HC#L156)_. Running tasks are organized in a doubly-linked list, with flags specifying which tasks are ready to run, and which are blocked, for example, waiting for external input.

## Conclusion

There is much more that could be said, but for today, our time's up. In part 3, we will finally do less talking, and more... doing. See you next month, class!

As a homework, you are invited to AOT-compile a simple TempleOS program and explain what is seen in the resulting BIN file headers.
