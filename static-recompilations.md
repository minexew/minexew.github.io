---
layout: post
title:  "Some interesting static recompilation projects"
comments: true
---

### Commodore BASIC [blogpost](https://www.pagetable.com/?p=48), [repo](https://github.com/mist64/cbmbasic), [diploma thesis](http://softpear.sourceforge.net/down/steil-recompilation.pdf)

- original 1977, C64 ROM, 6502 8-bit
- custom disassembler to LLVM IR (not published)
- IR translated to C (no reconstruction of higher-level structure), compiled natively
- original program has very clean platform interface, no interrupts etc.

### Albion, X-Com: UFO Defense (UFO: Enemy Unknown), X-Com: Terror from the Deep, Warcraft: Orcs & Humans [repo](https://github.com/M-HT/SR)

- originals released 1994-1995, target: DOS, x86 ?-bit
- disassembler based on udis86
- generate x86 or ARM assembly (ARM back-end is 8600 lines)

### Devilution [repo](https://github.com/diasurgical/devilution)

- original release 1996, target: Windows 95, x86 32-bit
- debug executable available -> symbol, variable and file names known
- hex-rays
- known compiler
- custom IDA scripting

### Syndicate Wars Port [website](http://swars.vexillium.org)

- original released 1996, target: DOS, x86 ?-bit
- custom disassembler (not published)
- no attempt at supporting non-x86

### StarCraft for Pandora [repo](https://pyra-handheld.com/boards/threads/starcraft.73844/)

- original released 1998, target: Windows 95, PE, x86 32-bit
- [custom IDA plugin](https://github.com/notaz/ia32rtools/blob/master/ida/saveasm/saveasm.cpp) to dump assembly
- [custom decompiler](https://github.com/notaz/ia32rtools/blob/master/tools/translate.c) that emits compilable low-level C representation of the assembly code

### pokeruby [repo](https://github.com/pret/pokeruby)

- original released 2002, target: Game Boy Advance, flat ROM image, ARM7TDMI 32-bit
- unclear how initial disassembly was obtained
- exact toolchain is known
- functions are manually rewritten in C to generate identical binary code
