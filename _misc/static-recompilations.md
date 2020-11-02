---
layout: post
title:  "Some interesting {de/re}compilation projects"
comments: true
---

### Commodore BASIC [blogpost](https://www.pagetable.com/?p=48), [repo](https://github.com/mist64/cbmbasic), [diploma thesis](http://softpear.sourceforge.net/down/steil-recompilation.pdf)

- original 1977, C64 ROM, 6502 8-bit
- custom disassembler to LLVM IR (not published)
- IR translated to C (no reconstruction of higher-level structure), compiled natively
- original program has very clean platform interface, no interrupts etc.

### NESgen [paper](https://github.com/Xenomega/NESgen/blob/master/NESgen/NESgen%20Documentation.docx) [repo](https://github.com/Xenomega/NESgen)

- very limited production ROM support

### Albion, X-Com: UFO Defense (UFO: Enemy Unknown), X-Com: Terror from the Deep, Warcraft: Orcs & Humans [repo](https://github.com/M-HT/SR)

- originals released 1994-1995, target: DOS, x86 ?-bit
- disassembler based on udis86
- generate x86 or ARM assembly (ARM back-end is 8600 lines)

### Frontier: First Encounters [repo](https://github.com/videogamepreservation/jjffe), [archived website](https://web.archive.org/web/20110303051207/http://jaj22.org.uk/jjffe/)

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

### NFSIISE [repo](https://github.com/zaps166/NFSIISE-CPP)

- original released 1997, target: Windows x86 32-bit
- auto-translation similar to StarCraft below
- tools published anywhere?

### StarCraft for Pandora [repo](https://pyra-handheld.com/boards/threads/starcraft.73844/)

- original released 1998, target: Windows 95, PE, x86 32-bit
- [custom IDA plugin](https://github.com/notaz/ia32rtools/blob/master/ida/saveasm/saveasm.cpp) to dump assembly
- [custom decompiler](https://github.com/notaz/ia32rtools/blob/master/tools/translate.c) that emits compilable low-level C representation of the assembly code

[example C code](https://pyra-handheld.com/boards/threads/starcraft.73844/page-3#post-1292054):
```
int sub_401310(int a1, int a2)
{
u32 eax = (u32)a1;
u32 ecx;
u32 edx;
u32 esi;
u32 edi;

if (eax != 0)
goto loc_40131D;
eax = (u32)a2; // arg_0
eax += 4;

loc_40131D:
esi = *(u32 *)(eax);
if (esi == 0)
goto loc_401351;
edx = *(u32 *)(eax+4);
if ((s32)edx > 0)
goto loc_40132F;
edx = ~edx;
goto loc_40133A;

loc_40132F:
edi = *(u32 *)(esi+4);
ecx = eax;
ecx -= edi;
edx += ecx;

loc_40133A:
*(u32 *)(edx) = esi;
ecx = *(u32 *)(eax);
edx = *(u32 *)(eax+4);
*(u32 *)(ecx+4) = edx;
*(u32 *)(eax) = 0;
*(u32 *)(eax+4) = 0;

loc_401351:
return eax;
}
```

### pokeruby [repo](https://github.com/pret/pokeruby)

- original released 2002, target: Game Boy Advance, flat ROM image, ARM7TDMI 32-bit
- unclear how initial disassembly was obtained
- exact toolchain is known
- functions are manually rewritten in C to generate identical binary code

### goldeneye\_src [repo](https://gitlab.com/kholdfuzion/goldeneye_src)

### n64decomp/majora [repo](https://github.com/n64decomp/majora)
### n64decomp/oot [repo](https://github.com/n64decomp/oot)
### n64decomp/sm64 [repo](https://github.com/n64decomp/sm64)

- unclear how initial disassembly was obtained
  - majora has a [custom annotating disassembler](https://github.com/n64decomp/majora/blob/1b335a770b90e7bc43c02776f2f3afb2402e254a/tools/disasm.py) written from scratch by [Rozelette](https://github.com/Rozelette)
- exact toolchain + SDK is known
- functions are manually rewritten in C to generate identical binary code
- useful tools: https://github.com/simonlindholm/asm-differ https://github.com/simonlindholm/decomp-permuter


