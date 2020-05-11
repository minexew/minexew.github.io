---
layout: post
title:  "TempleOS programs in Linux user-space, part 3: Stranger in a strange land"
comments: true
---

_[Last time](../../03/29/templeos-loader-part2.html), we took apart the kernel and looked at some binary formats. We will now put our gained knowledge to good use and build something that can be run._

# Application Binary Interface

Before we can bring our blessed code into the impure Linux-land, we need to talk a bit about how programs fit within the operating system. 

An Application Binary Interface (ABI) encompasses the following (and more):
 - function calling conventions
 - operating system interface and process initialization
 - executable file formats
 - dynamic linking model
 - program relocation
 - exception handling

To bridge the two worlds, we need to adapt all relevant aspects of the ABI. However, in this post we will focus only on the most essential parts to get off the ground, ignoring floating-point concerns, vararg functions and exceptions, for example.

## TempleOS

In the interest of simplicity, TempleOS doesn't attempt to adhere to any pre-existing standard. This makes it interesting from a study perspective, because there is not much need for legacy cruft (_ahem_, aside from the x86-64 architecture itself). Programs are preferably compiled Just-in-Time, but Ahead-of-Time compilation is also possible and will be our focus from now on. The BIN format that is used for AOT compilation has been described in the previous post, but we will quickly recap the most important points:

- binaries may import and export any number of symbols (functions and variables)
- binaries can be loaded at any address, by means of load-time [relocation](https://en.wikipedia.org/wiki/Relocation_(computing)) (as opposed to [position-independent code](https://en.wikipedia.org/wiki/Position-independent_code))
- each task has a hash table of loaded symbols (inherited by sub-tasks) that needs to be pre-filled to satisfy the imports of a BIN file
- code can be only loaded into the low 2 GiB of address space
- There is no _.bss_. You want a variable, you pay for it.
- There is no split between _.data_, _.rodata_ and _.text_ either. Everything is one big blob that is readable, writable and executable at the same time. A security nightmare, but also simple and straightforward.

The following calling convention is observed in HolyC code and when interacting with it:

 - arguments are passed on stack, last argument pushed first
 - called function may clobber arguments and is responsible for cleaning up the stack
 - the registers RBP, RSI, RDI and R10 to R15 must be [preserved](https://templeos.holyc.xyz/Wb/Doc/GuideLines.html#l167) by the called function; RAX, RBX, RCX, RDX, R8 and R9 are free to be clobbered

{% include figure-seamless.html alt="" src="templeos-loader/templeos-calling-convention.svg" caption="Comparison of the two calling conventions" %}

## Linux

Linux uses what is known as the _System V Application Binary Interface_ (commonly just _SysV ABI_). It is specified in the [x86-64 psABI document](https://github.com/hjl-tools/x86-psABI/wiki/X86-psABI).

SysV ABI uses an executable format called ELF. It is extremely versatile, and for this reason also quite complex and difficult to grasp at first. However, the fundamental principle is not different from other common formats -- namely, you have a big blob of code annotated with headers, sections and other data structures that help hook everything up, and sometimes reveal more than would be strictly needed just to run the program.

In terms of calling convention, the rules are as follows:

- arguments are passed (in order) in RDI, RSI, RDX, RCX, R8, R9; any further arguments are pushed to the stack, last argument pushed first
- called function may clobber arguments, including those passed on stack
- the stack pointer (RSP) must be aligned to 16 bytes when a function is being called
- the registers RBX, RBP and R12 to R15 must be [preserved](https://templeos.holyc.xyz/Wb/Doc/GuideLines.html#l167) by the called function; RAX, RCX, RDX, RSI, RDI and R8 to R11, as well as the x87 stack and MMX/SSE/AVX registers, are free to be clobbered

# Hello world

Now is a good time to bring out your homework from last time. In case you slacked it off, you can use the following program that we will compile, dissect, adapt and re-assemble together.

Our minimal program, _Example.HC_ looks like this:

```c++
PutS("Hello world\n");
```

That's it! Unlike _Print_, _PutS_ has a very straightforward calling convetion, so it's perfect for our simple example. We can compile the program like this:

```c++
Cmp("Example");
```

which will produce `Example.BIN.Z`.

## Dissection

Now we will look at the executable from a few different angles. I will use Linux for this, because it gives us a lot of tools for this job, which implies that first we have to somehow extract `Example.BIN` from the TempleOS system.

Having done that, let's start by simply hex-dumping the contents of the file, to see if there is anything obviously interesting.

    $ xxd Example.BIN
    00000000: eb1e 0000 544f 5342 ffff ffff ffff ff7f  ....TOSB........
    00000010: 3800 0000 0000 0000 6000 0000 0000 0000  8.......`.......
    00000020: 680b 0000 00e8 0000 0000 c348 656c 6c6f  h..........Hello
    00000030: 2077 6f72 6c64 0a00 1401 0000 0000 0100   world..........
    00000040: 0000 1900 0000 0000 0806 0000 0050 7574  .............Put
    00000050: 5300 0000 0000 0000 0000 0000 0000 0000  S...............

Swell. We can see the `TOSB` signature, our string, and also the name of the called function. Presumably, somewhere in the middle of all this is executable code.

Next, we make use of the [bininfo](https://github.com/cia-foundation/bininfo) tool that was introduced last time.

    $ bininfo.py Example.BIN
    BIN header:
        jmp                 [EB 1E]h
        alignment           1 byte(s)
        org                 7FFFFFFFFFFFFFFF (9223372036854775807)
        patch_table_offset  0000000000000038 (56)
        file_size           0000000000000060 (96)

    Patch table:
      entry IET_ABS_ADDR ""
        at        1h
      entry IET_MAIN ""
        main function @        0h
      entry IET_REL_I32 "PutS"
        at        6h

This tells us the following:

 - the binary has an implicit _main_ function starting at image offset 0 (recall that the image in a BIN file starts just after the [32-byte file header](https://github.com/cia-foundation/TempleOS/blob/1dd8859b7803355f41d75222d01ed42d5dda057f/Kernel/KernelA.HH#L384)). Implicit? Well, we didn't put our code in any function, but pretending that we did simplifies a lot of things for the compiler.
 - at address 01h in the image, a `IET_ABS_ADDR` relocation is applied. [For the binary loader](https://github.com/cia-foundation/TempleOS/blob/c26482bb6ad3f80106d28504ec5db3c6a360732c/Kernel/KLoad.HC#L95), this means:
   - load the 32-bit value at image offset 01h
   - to this value, add the address where the loaded image starts in memory
   - store the updated value back
 - at address 06h in the image, the `PutS` function is imported. We will discuss details of this patch type later.

Now that we have gathered some meta-data, let's look at the actual code. First we have to untangle it from the BIN file, though. Observe that the patch table starts at position 56 in the file, and the BIN header is 32 bytes in length. That leaves 24 bytes of program code in between. Let's extract it:

    dd skip=32 count=24 if=Example.BIN of=Example.text bs=1
    objdump -b binary -m i386:x86-64 -D -M intel Example.text

Some explanation is probably in order. _objdump_, which comes from _binutils_, is a versatile tool for analyzing and disassembling object files. It supports many mainstream formats, including ELF, PE and Mach-O, but unfortunately has no support for TempleOS BIN files. What it _can_ do, however, is to disassemble flat binaries, and this is good enough for us. The first 2 options set up this mode of operation:
 - `-b binary` tells objdump to treat the file as a flat binary, and
 - `-m i386:x86-64` specifies the CPU architecture

The following arguments tell objdump to `-D`isassemble, using `intel` syntax and finally the name of our image.

The output should look more or less as follows:

<div class="highlight"><pre class="highlight" style="font-size: 75%">
   0:	<span style="font-style: italic; opacity: 0.5">68 0b 00 00 00</span>       	<strong>push</strong>   0xb
   5:	<span style="font-style: italic; opacity: 0.5">e8 00 00 00 00</span>       	<strong>call</strong>   0xa
   a:	<span style="font-style: italic; opacity: 0.5">c3</span>                   	<strong>ret</strong>    
   b:	<span style="font-style: italic; opacity: 0.5">48</span>                   	rex.W
   c:	<span style="font-style: italic; opacity: 0.5">65 6c</span>                	gs ins BYTE PTR es:[rdi],dx
   e:	<span style="font-style: italic; opacity: 0.5">6c</span>                   	ins    BYTE PTR es:[rdi],dx
   f:	<span style="font-style: italic; opacity: 0.5">6f</span>                   	outs   dx,DWORD PTR ds:[rsi]
  10:	<span style="font-style: italic; opacity: 0.5">20 77 6f</span>             	and    BYTE PTR [rdi+0x6f],dh
  13:	<span style="font-style: italic; opacity: 0.5">72 6c</span>                	jb     0x81
  15:	<span style="font-style: italic; opacity: 0.5">64 0a 00</span>             	or     al,BYTE PTR fs:[rax]
</pre></div>

That's a bunch of code for a simple print! What is going on? In fact, only the first 3 instructions are real code (our main function), while the rest correspond to the "Hello world" string. (objdump has no way of distinguishing between code and interleaved data!)

In case you are not fluent in x86 assembly, a quick explanation of what the code does:

 - objdump doesn't know any better than to assume that our program starts at address 0. Since we don't know where the program will be loaded anyway, this is fine; at least, this way, all addresses correspond to offsets in the image.
 - The 32-bit value 0000_000Bh is zero-extended to 64 bits and pushed onto the stack. This is a pointer to our string.
 - A function at address 0Ah is called. Hold on, what the hell is going on here? Shouldn't we be calling `PutS`?
 - Finally, we return from the main function. This marks the end of the program.

Now the fun part: let's cross-reference the binary code with the patch table!

- The first entry was `IET_ABS_ADDR` at 01h -- which is in the middle of the first instruction. This _fixes up_ the value pushed onto the stack to ensure that it will point to our string, regardless of where in memory the program was loaded.
- The `IET_MAIN` entry is trivial, since our program only has one function anyways.
- On the other hand, `IET_REL_I32` is quite interesting. Remember the strange call instruction? As it turns out, the address 0Ah is bogus; the instruction is encoded as E8h plus a bunch of zeroes -- placeholders. E8h is a _32-bit relative call_ opcode. When executing this instruction, the CPU calculates the destination as follows:

        destination = pc + (int32) operand

    where _pc_ corresponds to the program counter after the whole instruction is fetched, which is the address just past the instruction (our 0Ah). So why does our program contain zeros, resulting in a non-sense destination? That's because the called function is not a part of the program -- it is linked _dynamically_. When the binary is loaded, the adress of the actual `PutS` function in memory will be taken, and the instruction will be patched to point to the right place. In order to ensure that `destination == &PutS`, the loader actually has to compute the expression

        operand = &PutS - pc

    or,

        operand = &PutS - (image_load_address + 06h + 4)

    Fun stuff!

## Transforming relocations

Where TempleOS does away with one universal table, ELF needs at least two structures: a _symbol table_ and a _relocation table_. The symbol table contains names of all imported and exported symbols; the relocation table contains relocations (duh), but the point is that relocation entries may also point to symbol table entries, to use symbol adresses in relocation calculations. Keep the psABI document at hand, because it also documents the applicable relocation types. To speak in concrete terms:

- `IET_ABS_ADDR` has an ELF equivalent in `R_X86_64_32`. The equation for this type of relocation is `S + A`, where _S_ stands for the address of a _symbol_ and _A_ stands for a constant _addend_. Since we are fixing up an image offset, _symbol_ has to point to the image start (and indeed, ELF lets us refer to _program sections_ through symbols) and _addend_ must be the un-fixed-up value that we found by disassembling the program. In ELF for x86-64, the addend is stored in the relocation table itself (see `Elf64_Rela`); the value of the placeholder in the code is not used.

    Note that by setting the addend to 0 and referring to the appropriate symbol table entry, `R_X86_64_32` can be also used to implement `IET_IMM_U32` (which stands for _import the absolute 32-bit address of a named symbol_)

- `IET_REL_I32` has an ELF equivalent in `R_X86_64_PC32`. The equation is `S + A - P`, with _P_ referring to the address being patched up. Recall that we need to achieve the calculation of `destination - (image_load_address + offset_in_image + 4)`. Since `P` corresponds exactly to `image_load_address + offset_in_image`, we can point `S` to the appropriate symbol table entry and set `A = -4`.

- for `IET_MAIN` there is nothing directly corresponding. For our purposes, we will make up a symbol name and create a symbol table entry to be able to refer to the program start (since in general, it does not have to be offset 0).

## Adapting calling conventions

How to deal with the discrepancy in the calling conventions? We could:

- modify the HolyC compiler to comply with the SysV ABI (a lot of work)
- modify the Linux-side compiler to observe the HolyC calling convention (a LOT of work)
- write all Linux-side code in assembly language (meh)
- cry and give up
- make sure that our HolyC code doesn't violate the SysV ABI constraints _too much_, and write translation _thunks_ that would allow functions to be called across the ABI boundary

Clearly, you know where this is going.

The task is somewhat complicated by the fact that we always need to know the number of arguments to write a correct thunk -- for example, while we could just speculatively copy up to N arguments from the stack into registers to call from HolyC to Linux, the calling code will expect the arguments to be popped from the stack by the called function. Our C function will not do this, so it has to be handled by the thunk. This is not a big deal when writing code by hand, but since we are lazy, we will want to auto-generate as much as possible. Whatever tool we end up using, somehow it will need to be aware of the function signatures -- something that is notably *not* encoded in the BIN file.

### A thunk from HolyC to Linux

Let's now try to write such a translation thunk. Presume that we have changed all symbols in HolyC code to add a distinguishing suffix; for example, `PutS` becomes `PutS$HolyC`. Now we want to write a function that will handle HolyC calls to `PutS` and dispatch them to a function written in C.

First of all, we have to save all registers that HolyC expects to be preserved.

    PutS$HolyC:
        push rsi
        push rdi
        push r10
        push r11

At entry, the return address is at the top of stack and the sole argument is just below it -- at the address pointed by _rsp + 8_. However, our register preservation effort has shifted the stack and the argument is now at _rsp + 40_. C code expects it in RDI (discussed above), so we need to _mov_ it there:

        mov rdi, qword ptr [rsp+40]

(If we had further arguments, we would continue with

        mov rsi, qword ptr [rsp+48]
        mov rdx, qword ptr [rsp+56]
        ...

but that is not the case for `PutS`)

Registers saved, arguments prepared... time to jump into the native function.

        call PutS

Now a lot of magic happens, but eventually, `PutS` should return back to the call site. What now?

We don't care about the return value, but if we did, HolyC and SysV conveniently agree on placing it in RAX. What we do need to do, though, is restore the previously saved registers:

        pop r11
        pop r10
        pop rdi
        pop rsi

And finally, we need to pop the argument(s) pushed by HolyC code, and return to it. Since we only have one argument, we will be dropping 8 bytes from the stack (*after* the return address)

        ret 8

And there we go! Writing a thunk in the opposite direction is conveniently left as an exercise for the reader.

### Converting object files

We have tackled the 2 main interfacing problems, and what is left now is writing a bunch of code to automate the process. Despair not, for it has been done already. Enter [bin2elf](https://github.com/cia-foundation/bininfo).

Invoking

    bin2elf.py -h

shows that there are several options that we can specify. We will only need some of them:

- HolyC import definitions: we will create a file named _ExampleImportDefs.HH_ with the following line in it:

  ```
  U0 PutS(U8 *st);
  ```

- HolyC export definitions: we will use this to expose the HolyC main function; in _ExampleExportDefs.HH_ put the following:

  ```
  U0 HCMain();
  ```

- thunks output file -- we need these to wire things up!
- input and output

The complete invocation will then look like this:

    bin2elf.py --import-defs ExampleImportDefs.HH \
               --export-defs ExampleExportDefs.HH \
               --export-main HCMain \
               --thunks-out Example.thunks.s \
               -o Example.o \
               Example.BIN

_Example.o_ is now a standard relocatable ELF file, and can be inspected using the usual tools (including objdump). However, due to a limitation of the library used by _bin2elf_, the file is in ELF32 format. This is perfectly fine for us, since TempleOS only permits code in the lower 2 GiB, but GCC, for example, will not accept ELF32 objects with 64-bit code. To remedy this, we need to do one more conversion:

    objcopy -I elf32-x86-64 -O elf64-x86-64 Example.o Example.o64

Now we have everything ready on the HolyC side. Let's write some C!

```c
#include <stdio.h>

void HCMain(void);

void PutS(const char* message) {
    printf("%s", message);
}

int main(int argc, char** argv) {
    HCMain();
}
```

(save as example.c, or whatever)

We have 3 components to put together:

- our converted HolyC binary
- the thunks generated by bin2elf (based on our import/export definitions)
- hand-written C code

Conveniently, we can compile & link everything together with a single line

    gcc -o example example.c Example.o64 Example.thunks.s

And if you feel lucky:

    $ ./example 
    Hello world

Congratulations, you are now a heretic!

More could be said, but we will end here for the time being.

As a homework, you are invited to find and report (or fix!) a bug in _bin2elf_.
