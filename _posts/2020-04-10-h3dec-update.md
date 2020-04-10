---
layout: post
title:  "On iterative reassembleable disassembly and the Heroes III decompilation"
comments: true
---

Generally, reverse engineering tools aim to produce output that can be well-understood by _humans_. This makes perfect sense -- if you are in the business of analyzing malware, when a new sample arrives you need to get an idea of which vulnerabilities it is taking advantage, how it is controlled, how it spreads, etc. Similarly, if your hobby (or business) is cracking software license protection, you will want to understand the general flow and logic of the program, in order to zoom into the relevant part of the code.

As noble as these aims are, ours is different. We need to convert the executable in question into a representation that is understood, first and foremost, by a _machine_.

The reasons have been hinted at in the [previous post](../../../2018/12/28/h3dec.html): while the goal is to decompile the game as much as possible to gain high-level understanding and to answer [this Stack Overflow question](https://gaming.stackexchange.com/questions/13909/how-does-heroes-of-might-magic-3-choose-which-music-to-play-at-the-beginning-o), in order to facilitate changes it is necessary to be able to reproduce a working executable. This process is called _re-assembleable disassembly_ and is generally considered [_really hard to do_](https://reverseengineering.stackexchange.com/a/3803).

Naturally, our end-goal is even higher, and it could be called _recompilable decompilation_. This should be even more difficult -- perhaps impossible -- and I am not aware of a freely licensed toolkit for automating this task (for an example of a project that does it manually, see the impressive [pokeruby](https://github.com/pret/pokeruby)).

Another example is [Project Ironfist](https://github.com/jkoppel/project-ironfist/), an extensive mod of Heroes II. Their workflow is based on IDA, the Hex-Rays decompiler, [a bunch](https://github.com/jkoppel/project-ironfist/tree/master/tools/Revitalize/to_masm) of [custom](https://github.com/jkoppel/project-ironfist/tree/master/tools/Revitalize/Referee) [scripts](https://github.com/jkoppel/project-ironfist/tree/master/tools/Revitalize/seance) and a few years of manual work. Definitely cool, but not quite what we are aiming for.

## The rough idea

To make the task easier, we shall introduce some assumptions:

- the code is compiled from a structured language (C/C++)
- the code can be compartmentalized into self-contained _functions_ that only interface through
  - a limited number of well-defined calling conventions (such as `cdecl`, `stdcall`, `fastcall` or `thiscall`)
  - global variables in the executable's DATA/BSS sections
- the code is _sane_, for example it does not invoke CPU exceptions on purpose
- the program is a _good citizen_ in the host operating system (Windows), e.g. observing the rules of memory allocation and the OS APIs
- the executable is _honest_ and complete: no self-modifying code, no runtime-generated code, no executable packing
- perhaps most importantly: the flow of the program is _linear_ and only diverted by jumps/calls in the program (hardware interrupts are a bane of static recompilation on bare-metal platforms, such as the NES)

These are some bold claims, and will probably not hold for 100% of the code (e.g. hand-written assembly routines might not have the structure as compiled C code). The important observation to note here is that violations, if they are limited to a small subset of the code, will only degrade the decompilation _locally_ in those parts. They should not hamper the overall understanding of the program, as long as at least the flow linearity assumption holds.

We can proceed iteratively, decompiling one function at a time. In terms of decompilers, we will need one that produces output that can be turned into valid C with a few #defines and substituions -- at present, the top two contenders seem to be Hex-Rays (sadly proprietary) and Ghidra.

While there have been significant advances in decompilation of logic, data structures still have to be defined ([manually](https://www.hex-rays.com/products/ida/support/tutorials/structs/). Therefore, it seems that there is a big opportunity for novel research in this area.

The decompilation is an ambitious undertaking, but we will start with plain simple disassembly, which can already tell us a lot about the structure of the code.

## Probably correct

Besides receiving a C&D notice, perhaps the worst nightmare of a decompilation project is to introduce a subtle bug somewhere along the way, which only shows up much later, by which time it is nearly impossible to track down. How to prove that our disassembly is good? Despite the recent progress in symbolic execution and dynamic analysis, in absence of exhaustive unit tests the only reliable way is _proof by equivalence_. This implies that our recreated source code can be used to construct an executable identical to the original, down to the last bit. This is feasible for a disassembly, and somewhere between ridiculously difficult and impossible for a high-level decompilation. (Among others, it implies tracking down the exact toolchain used originally. And as usual, [pokeruby](https://github.com/pret/pokeruby) does it.)

At least for now, we will take this route.

## Taking apart an executable

The executable in question (_Heroes3.exe, 2 732 032 bytes, SHA-256 057c9d88_) is a 32-bit Windows Portable Executable; in principle, ELF files are not that different, so much of this section applies to those as well. If you are interested in details of the PE format, check out [this fantastic write-up](https://blog.kowalczyk.info/articles/pefileformat.html) -- we will only cover what is most important for this project.

{% include figure-seamless.html alt="" src="2020-04-10-h3dec-update/Heroes3-exe-layout.svg" caption="Layout of the executable (not to scale). The <strong>data</strong> section takes up 315 KiB in memory, but only 208 KiB of the data is defined in the file. Where does the rest come from? It corresponds to what is usually referred to as the BSS section, and it will be initialized to zeros by the Windows loader." %}

Conveniently, there are no gaps inbetween sections, or extra data at the end. In addition, no kind of compression is applied to the executable. We can therefore focus entirely on the header and the 4 sections. 

**The PE header** is not just a simple structure, in fact it is a tree of many smaller structures. Since it is small and does not contain code (hopefully!), we care about extracting important information from it (memory layout, imported functions, address of entry point), but not so much about re-creating it from source code.

**.text** contains the program code itself, and is of utmost importance. The PE header tells us that this section is loaded at a virtual address of **0x401000**. Aside from x86 instructions, it also contains padding between functions and constants that need to be accessed by the program ([note](https://stackoverflow.com/a/56891142/2524350)). We will attempt to disassemble as much of the code as we can, and produce one or more _.s_ files (we will call these "clusters") for GNU assembler. Labels shall be generated and used for all cross-references, instead of hard-coded addresses (this process is also known as _symbolization_).

**.rdata** contains statically-allocated constant variables; in C, those would be *const* global variables, or *static const* scoped variables. We could extract this as a binary blob, but since code refers to variables located here -- and the converse --, it makes sense to also produce a _.s_ file for this section.

**.data** contains statically-allocated non-constant variables with initial values assigned. The uninitialized part of _.data_, _BSS_, contains variables which are initialized to zero. We will produce a _.s_ file with the same justification as we do for **.rdata**

**.rsrc** contains Windows resources (not to be confused with game resources), such as the icon of the executable. We don't care about any of these. There are no cross-references between **.rsrc** and the rest of the executable, so we can happily ignore it.

### Virtual addressing

To make matters a bit more complicated, offsets in the file do not directly correspond to adresses in memory after the process is loaded. To convert from a location in the file to the corresponding Virtual Memory Address -- VMA -- we have to go through the section table in the PE header, like this:

<div class="highlight"><pre class="highlight" style="font-size: 75%">
func FileOffsetToVMA(pe: PE, offset: intptr) -> (section: PE.Section?, vma: intptr?):
    for section in pe.sections do
        if offset ≥ section.PointerToRawData and offset < section.PointerToRawData + section.SizeOfRawData do
            if offset < section.PointerToRawData + section.VirtualSize do
                return (section, pe.ImageBase + section.VirtualAddress + (offset - section.PointerToRawData))
            else
                # Pointer lies in part of the section that is not loaded (*usually*, this would be just padding)
                # As such, there is no corresponding VMA
                return (section, nil)
            end if
        end if
    next

    return (nil, nil)
</pre></div>

As homework, you can implement the opposite. Keep in mind that `section.SizeOfRawData` can be less, equal, or greater than `section.VirtualSize`!

### jmp _start

Now, back to the code. As mentioned earlier, **.text** contains a mixture of instructions, data and padding. The instructions should be disassembled, the data should be defined as such, and the padding should be ideally replaced by assembler directives like _.align_, as long as they produce identical results. How do we tell what is what? This is a non-trivial problem, and a well-established RE tool like [radare2](https://github.com/radareorg/radare2) or [IDA](https://www.hex-rays.com/products/ida/) could be very helpful here. However, to increase the pedagogic value, we will propose a very simple (and very imperfect) algorithm of our own:

```
pending_seeds := set { VMA of the program entry point }
all_seeds := pending_seeds

while pending_seeds ≠ {} do
    VMA := remove_one_element_from(pending_seeds)

    loop
        instruction := disassemble_at(VMA)

        if instruction is "call" or instruction is any jump do
            if dest := instruction.immediate_destination_operand do
                if dest not in all_seeds do
                    all_seeds := all_seeds ∪ { VMA }
                    pending_seeds := pending_seeds ∪ { VMA }
                end if
            end if
        end if

        if instruction is "jmp" or instruction is "ret" do
            break
        end if
    next
next
```

This algorithm will discover all code _statically reachable_ (note: this might not be the proper term) from the entry point. The most obvious problem is that it will not discover code that is only reachable dynamically; in other words, any jump/call instructions with computed operands are simply ignored.

(Luckily, jumps on x86 are "easy". Compare to ARM, particularly in Thumb mode, where an absolute jump, for example, requires a sequence of `LDR reg, =address; B reg`)

Since for now we want to be very conservative and only attempt to disassemble what we are 100% sure is valid code, this approach will do. In the future, it would seem wise to use radare2's [extensive analysis features](https://radare.gitbooks.io/radare2book/analysis/code_analysis.html) instead.

### Computed jumps

Just for completeness, let's look at an example where we _fail_ to derive useful information:

<div class="highlight"><pre class="highlight" style="font-size: 75%">
<span style="color: #60a0b0; font-style: italic">/* chunk (CodeChunk .text 00402D90h..00402DA1h, 17 bytes) */</span>
<span style="color: #002070; font-weight: bold">l_402d90:</span>
    add         eax, <span style="color: #40a070">0xfffffc0f</span>                  <span style="color: #60a0b0; font-style: italic">/* 0x00402d90 05 0F FC FF FF */</span>
    cmp         eax, <span style="color: #40a070">6</span>                           <span style="color: #60a0b0; font-style: italic">/* 0x00402d95 83 F8 06 */</span>
    ja          l_402d55                         <span style="color: #60a0b0; font-style: italic">/* 0x00402d98 77 BB */</span>
    jmp         dword ptr [eax<span style="color: #666666">*</span><span style="color: #40a070">4</span> <span style="color: #666666">+</span> text_402e4c]  <span style="color: #60a0b0; font-style: italic">/* 0x00402d9a FF 24 85 4C 2E 40 00 */</span>
<span style="color: #60a0b0; font-style: italic">/* end chunk (CodeChunk .text 00402D90h..00402DA1h, 17 bytes) */</span>
</pre></div>

Why this particular example? You may notice that the memory address for the indirect jump is perfectly bounded by the previous two instructions (the `add`, on the other hand, is irrelevant). Some sort of symbolic execution, or even static pattern matching would be sufficient to discover all possible jump destinations here.

## The jungle of x86

The Intel x86 architecture, being very CISC-y, is probably the worst choice for this kind of research. There are many problems that we have to face:

#### Multiple encodings of the same instruction (absolute/relative; different operand sizes)

This one is not unique to x86, but certainly applies there. Take for example the [JMP](https://www.felixcloutier.com/x86/jmp) instruction. In practice, we can encounter these:
  - `JMP rel8`
  - `JMP rel32`
  - `JMP r/m32`

You might see the problem now: 3 different opcodes, but they will all disassemble to the same mnemonic and operand. People always point out that compilation is a lossy process, but with commonly used tools, **_the same is true for disassembly_**!

In practice, it is the assembler that chooses a suitable representation, based on the actual value of the operand. For instance, in the following snippet of code:

<div class="highlight"><pre class="highlight" style="font-size: 75%">
<span style="color: #002070; font-weight: bold">l_403d9a:</span>
    test        edx, edx                         <span style="color: #60a0b0; font-style: italic">/* 0x00403d9a 85 D2 */</span>
    jne         l_403da2                         <span style="color: #60a0b0; font-style: italic">/* 0x00403d9c 75 04 */</span>
    xor.s       eax, eax                         <span style="color: #60a0b0; font-style: italic">/* 0x00403d9e 33 C0 */</span>
    jmp         l_403da7                         <span style="color: #60a0b0; font-style: italic">/* 0x00403da0 EB 05 */</span>

<span style="color: #002070; font-weight: bold">l_403da2:</span>
    sub.s       eax, edx                         <span style="color: #60a0b0; font-style: italic">/* 0x00403da2 2B C2 */</span>
    sar         eax, <span style="color: #40a070">2</span>                           <span style="color: #60a0b0; font-style: italic">/* 0x00403da4 C1 F8 02 */</span>

<span style="color: #002070; font-weight: bold">l_403da7:</span>
    add.s       eax, ecx                         <span style="color: #60a0b0; font-style: italic">/* 0x00403da7 03 C1 */</span>
    test        eax, eax                         <span style="color: #60a0b0; font-style: italic">/* 0x00403da9 85 C0 */</span>
    mov         dword ptr [ebp <span style="color: #666666">-</span> <span style="color: #40a070">8</span>], eax         <span style="color: #60a0b0; font-style: italic">/* 0x00403dab 89 45 F8 */</span>
</pre></div>

The assembler is able to see that the jump target of `jmp l_403da7` is close enough to use a signed 8-bit relative offset, so it will (correctly) choose the `JMP rel8` encoding.

However, occasionally we encounter this:

<div class="highlight"><pre class="highlight" style="font-size: 75%">
    mov         ecx, dword ptr [bss_699604]      <span style="color: #60a0b0; font-style: italic">/* 0x004f81f5 8B 0D 04 96 69 00 */</span>
    test        ecx, ecx                         <span style="color: #60a0b0; font-style: italic">/* 0x004f81fb 85 C9 */</span>
    je          l_4f8210                         <span style="color: #60a0b0; font-style: italic">/* 0x004f81fd 74 11 */</span>
    mov         eax, <span style="color: #40a070">1</span>                           <span style="color: #60a0b0; font-style: italic">/* 0x004f81ff B8 01 00 00 00 */</span>
    sub.s       eax, edx                         <span style="color: #60a0b0; font-style: italic">/* 0x004f8204 2B C2 */</span>
    mov         dword ptr [bss_699618], eax      <span style="color: #60a0b0; font-style: italic">/* 0x004f8206 A3 18 96 69 00 */</span>
    jmp         func_4f8220                      <span style="color: #60a0b0; font-style: italic">/* 0x004f820b E9 10 00 00 00 */</span>

<span style="color: #002070; font-weight: bold">l_4f8210:</span>
    ret                                          <span style="color: #60a0b0; font-style: italic">/* 0x004f8210 C3 */</span>

    nop
    nop
    ...
    nop
    nop

<span style="color: #002070; font-weight: bold">func_4f8220:</span>
    push        ebp                              <span style="color: #60a0b0; font-style: italic">/* 0x004f8220 55 */</span>
</pre></div>

Here, for `jmp func_4f8220` our assembler will emit a short `EB 10` instruction, but that is _not_ what the exectuable says! (in this specific case we seem to be dealing with a [tail call](https://en.wikipedia.org/wiki/Tail_call#In_assembly) across compilation unit boundaries, which would explain the unusual encoding)

In terms of functionality, this would not hurt, but the shorter encoding will shift all following instructions, which will
1) prevent us from reproducing a bit-exact reproduction of the original executable
2) break code that was not disassembled, because offsets will change

After some experimentation with forcing the GNU assembler to emit a `JMP rel32` encoding, we gave up and simply replaced the disassembly with

<div class="highlight"><pre class="highlight" style="font-size: 75%">
    .byte    <span style="color: #40a070">0xe9</span>                              <span style="color: #60a0b0; font-style: italic">/* 0x004f820b E9 jmp rel32 0x4f8220 */</span>
    .byte    <span style="color: #40a070">0x10</span>                              <span style="color: #60a0b0; font-style: italic">/* 0x004f820c 10 jmp rel32 0x4f8220 */</span>
    .byte    <span style="color: #40a070">0x00</span>                              <span style="color: #60a0b0; font-style: italic">/* 0x004f820d 00 jmp rel32 0x4f8220 */</span>
    .byte    <span style="color: #40a070">0x00</span>                              <span style="color: #60a0b0; font-style: italic">/* 0x004f820e 00 jmp rel32 0x4f8220 */</span>
    .byte    <span style="color: #40a070">0x00</span>                              <span style="color: #60a0b0; font-style: italic">/* 0x004f820f 00 jmp rel32 0x4f8220 */</span>
</pre></div>

A possible solution, yet to be tested, would be to more closely emulate how the code was built originally, and split the cluster at address 0x4f8220. The assembler, not seeing the jump destination anymore, would then be forced to emit the long encoding.

#### Multiple encodings of the same instruction (swapped operands)

This one might be specific to the x86. The encoding of many functions includes a bit to _swap_ the operands. This can be useful if normally the operands do not come from the same domain (e.g. _reg_ and _reg OR memory_), however the bit is also valid when the operands _are_ in the same domain, which introduces another encoding ambiguity. For example, the following two encodings are functionally equivalent:

<div class="highlight"><pre class="highlight" style="font-size: 75%">
    mov       ebp, esp                           <span style="color: #60a0b0; font-style: italic">/* 89 E5 */</span>

    mov       ebp, esp                           <span style="color: #60a0b0; font-style: italic">/* 8B EC */</span>
</pre></div>

Again, it introduces an **OFCOPOTA** problem (Opportunity For Choice On Part Of The Assembler). Fortunately, the amazing GNU assembler has a syntax to disambiguate this ambiguity:

<div class="highlight"><pre class="highlight" style="font-size: 75%">
    mov       ebp, esp                           <span style="color: #60a0b0; font-style: italic">/* assembles to 89 E5 */</span>

    mov.s     ebp, esp                           <span style="color: #60a0b0; font-style: italic">/* assembles to 8B EC */</span>
</pre></div>

## Putting it back together

Earlier, we said that we didn't really want to mess with the PE header and with the resource section. But how are we supposed to reconstruct the executable, then? We can cop out by accepting yet another simplification of the problem (ignoring problems is my favorite way of solving them); for now, we will aim to produce a binary file that can be patched over the original executable, skipping the headers and only overwriting the executable code. Applying the patch should not change any bits in the executable, thus achieving our proof by equality. This lets us focus on the interesting part of the problem, which is the code, rather than having to mess with linker options to reconstruct a specific set of PE headers from scratch (which might prove impossible with standard tools due to juicy bits like the ["Rich" header](https://bytepointer.com/articles/the_microsoft_rich_header.htm)).

It seems fitting to use the mingw-w64 toolchain for this (the name is misleading; it targets both 32-bit and 64-bit Windows), although since we opted to ignore everything Windows-specific, plain GCC would do just fine as well.

We will use the GNU assembler (as) to assemble the disassembled clusters and the GNU linker (ld) to glue everything together into one big object file.

### Symbolize early and often

The GNU assembler seems to have one limitation worth mentioning: there is no way to force a fixed starting address when compiling a _.s_ file -- it always starts at 0 and expects to be relocated in place by the linker. This makes sense considering how the assembler is typically used, but the consequence for us is that **all** jumps have to be symbolized. Why? Consider this example:

<div class="highlight"><pre class="highlight" style="font-size: 75%">
<span style="color: #002070; font-weight: bold">l_401000:</span>
    mov eax, <span style="color: #40a070">1</span>
    ret

.align <span style="color: #40a070">16</span>

<span style="color: #002070; font-weight: bold">func_401010:</span>
    cmp ecx, <span style="color: #40a070">5</span>
    je <span style="color: #40a070">0x401000</span>   <span style="color: #60a0b0; font-style: italic">/* since the assembler does not know where we are in memory,</span>
<span style="color: #60a0b0; font-style: italic">                     it has to assume the worst, and emit a 32-bit absolute call</span>
<span style="color: #60a0b0; font-style: italic">                     (technically, a 32-bit relative call + an appropriate relocation)</span>
<span style="color: #60a0b0; font-style: italic">                   */</span>
</pre></div>

Instead, note the difference:

<div class="highlight"><pre class="highlight" style="font-size: 75%">
<span style="color: #002070; font-weight: bold">l_401000:</span>
    mov eax, <span style="color: #40a070">1</span>
    ret

.align <span style="color: #40a070">16</span>

<span style="color: #002070; font-weight: bold">func_401010:</span>
    cmp ecx, <span style="color: #40a070">5</span>
    je l_401000     <span style="color: #60a0b0; font-style: italic">/* since the assembler has seen the definition of l_401000 in this</span>
<span style="color: #60a0b0; font-style: italic">                        cluster, it knows that it can be reached by a rel8 encoding</span>
<span style="color: #60a0b0; font-style: italic">                     */</span>
</pre></div>

Just like there is no way to force a `rel32` encoding, there is also no way to force a `rel8` encoding. This, at least, makes sense, considering how ELF relocations are implemented -- there is simply not enough space to encode the value `0x401000` in the 8-bit operand.

This also means that we cannot split the clusters in arbitrary locations -- if we cut a function in half, the assembler will be forced to emit 32-bit jumps across the split boundary! In practice, we have achieved good results by splitting at x86 function prologues (`push ebp; mov ebp, esp`).

Another practical observation is that GNU ld seems to impose 4-byte alignment of objects, so clusters should be split at aligned addresses, otherwise the added padding will shift the rest of the file.

Of course, other toolchains than the GNU binutils can be used. These might have their own benefits and limitations; for example, NASM starts to choke when the cluster size exceeds about 64 KiB of compiled code. However, GNU assembler seems to be the only one supporting the `.s` suffix for alternate encoding of instruction operands.

## From theory to practice

All of the abovementioned can be done quite easily in Python, leveraging 2 existing libraries to do much of the otherwise tedious work:

  - [pefile](https://github.com/erocarrera/pefile) to parse the executable
  - [Capstone](https://www.capstone-engine.org/lang_python.html) to decode machine instructions

Capstone is a great disassembly framework, and while it is neither 100% complete, nor bug-free, it does a good enough job to be able to handle the few exceptions manually. Some custom plumbing is still needed; for example, AFAIU with Capstone there is no straightforward way to predict where the `.s` mnemonic suffix will be needed to preserve the encoding. It is possible to hardcode the byte sequences of all these cases, but a more WSNH way (Work Smart, Not Hard) is described below.

### Bash it until it works

Occasionally, the re-assembled code will differ from the original, or fail to assemble in the first place. Why, you ask? Well, what does it matter?! The point is that rather than discovering these problems one-by-one for each analyzed executable, we can use a feedback loop to understand the issue, attempt to fix it, and re-test -- in a fully automated way.

For example, after disassembling and re-assembling our executable, we can run _objdump -D_ for both the original and the replica. Then we scan both disassemblies line-by-line, and whenever there is a discrepancy, parse the text to understand what the problem is:

### Example 1

<div class="highlight"><pre class="highlight" style="font-size: 75%">
<span style="color: #702000; font-weight: bold">Error:</span> 004013D1: disassembly mismatch
<span style="color: #002070; font-weight: bold">EXPECT:</span>   4013d1:	8b ec                   mov    ebp, esp
<span style="color: #002070; font-weight: bold">GOT:</span>      4013d1:	89 e5                   mov    ebp, esp
</pre></div>

- **Symptom:** the instruction length is the same in both cases
- **Symptom:** the mnemonics and operands match

**Diagnosis:** easy, we've seen this one before -- an alternate encoding is required (`.s` suffix). But how to make the machine understand without next-generation AI?

A simple heuristic: we just go by the two symptoms described above. When we encounter a case matching this pattern, we add a record into our _knowledge database_, saying "_for 8B EC, try to add the .s suffix next time_". Indeed, our disassembler does exactly that, and the re-assembly result is observed; if the output is now as expected, the issue is considered resolved.

If, however, the problem persists, the sequence `8B EC` is marked **DND**: Do Not Disassemble. In the next pass, the disassembler will simply emit a sequence of `.byte` directives. That is not terribly helpful for reverse-engineering the code, but it allows us to move on in terms of the big picture.

### Example 2

<div class="highlight"><pre class="highlight" style="font-size: 75%">
<span style="color: #702000; font-weight: bold">Error:</span> 0040102D: instruction encoding mismatch
<span style="color: #002070; font-weight: bold">EXPECT:</span>   40102d:       0f 8e 46 00 00 00       jle    0x401079
<span style="color: #002070; font-weight: bold">GOT:</span>      40102d:       7e 3b                   jle    0x40106a
</pre></div>

- **Symptom:** the mnemonics match
- **Symptom:** the instruction is a jump
- **Symptom:** a longer encoding was expected
- **False symptom:** the operands are different

**Diagnosis:** we have also discussed this one earlier -- it is the case where a `rel32` encoding is required. For now, we mark it **DND**, as it occurs rarely enough to make us care.

Note the apparent symptom of difference in operands; this is simply the result of incorrect encoding of instructions _between_ the jump origin and destination. Since this problem easily becomes cyclic, it is best to (temporarily!) ignore it, and hope that eventually the code aligns.

Admittedly, we don't even need our "machine learning" to handle this case -- the needlessly long encoding can be detected at disassembly time.

### Example 3

<div class="highlight"><pre class="highlight" style="font-size: 75%">
<span style="color: #702000; font-weight: bold">Error:</span> 006234C1: disassembly mismatch
<span style="color: #002070; font-weight: bold">EXPECT:</span>   6234c1:	      8d 49 00             	lea    ecx,[ecx+0x0]
<span style="color: #002070; font-weight: bold">GOT:</span>      6234c1:	      8d 09                	lea    ecx,[ecx]
</pre></div>

- **Symptom:** the mnemonics match
- **Symptom:** the operands match in semantics, but not in syntax
- **Symptom:** the instruction lengths mismatch

**Diagnosis:** for whatever reason, an unnecessarily wasteful encoding was emitted in the original executable. Again, this happens rarely enough to justify making it a **DND**. At least for now.

### Conservative progression

To avoid hitting a huge brick wall at once, it is advisable to work very iteratively, always tracing only a few seeds at a time, and then attempting a disassembly. For parts of the code that have not been analyzed yet, we can simply emit a bunch of `.byte` directives. As soon as a re-assembly error results in different instruction length, the rest of the code becomes mis-aligned, so detecting and identifying subsequent discrepancies gets more difficult. When the disassembly fails, it can be re-tried as long as new information is being learned (= new entries being added to the knowledge database).

Eventually, either the re-assembly will finally pass and match, and then we can proceed to analyze more seeds from the backlog. Alternatively, we fail to discover any new hints and a manual intervention is needed (as a third possibility, the tool might get stuck looping forever, _thinking_ that it is discovering more and more new problems). Ideally, this manual intervention should lead to improvement of the tool, rather than having to enter hints specific to the executable being analyzed -- we teach the system _learn_ on its own.

## Results thus far

{% include figure-seamless.html alt="" src="2020-04-10-h3dec-update/Heroes3-reassembly.svg" caption="The re-assembly pipeline" %}

With the described approach, and only static analysis, we are able to reach and disassemble about 25 % of the **.text** section. That should be more than sufficient to try and explore possible next steps.

There have been some recent papers which might obsolete the entirety of this research, so we'll probably start with those. 

<!--
For one, PEFile can be replaced with [CLE]() which also supports ELF, Mach-O and flat binaries.
-->

## See also

- [Datalog Disassembly](https://arxiv.org/abs/1906.03969)
    - [ddisasm](https://github.com/GrammaTech/ddisasm)
- [Tricks to Reassemble Disassembly](https://gts3.org/2015/rasm.html)
- [Ramblr: Making Reassembly Great Again]()
- [Reassembleable Disassembling](https://www.usenix.org/system/files/conference/usenixsecurity15/sec15-paper-wang-shuai.pdf)
    - [Uroboros: Infrastructure for Reassembleable Disassembling and Transformation](https://github.com/s3team/uroboros)
- [angr](https://angr.io/)
- [Superset Disassembly: Statically Rewriting x86 Binaries Without Heuristics](https://web.cse.ohio-state.edu/~lin.3021/file/NDSS18a.pdf)
- [awesome-decompilation: A curated list of awesome decompilation resources and projects](https://github.com/nforest/awesome-decompilation)

## Special thanks

- [hilite.me](http://hilite.me/) (Settings used: language=c-objdump, style=friendly)
- [120385 Palette](https://lospec.com/palette-list/120385) by Lujain Hakmee
