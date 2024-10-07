---
title: PA PA2 - Instruction Set Implementation & KLIB
date: 2024-10-03 01:37
categories: 技术
tags:
    - c
excerpt: Implementing specific details of the RISC-V instruction set
---

> [!WARNING]
> If someone is reading this blog, please be aware that the writer **did not** consider the experience of the other readers.
> After all, the most important part is about writing things down for better memorization.

## Bit Operation Macros

-   Bit operations are usually not very readable to human, so I decided to write down the explanations that I come up with here.

### _BITS_ && _BITMASK_

```c
#define BITMASK(bits) ((1ull << (bits)) - 1)

#define BITS(x, hi, lo)                                                        \
    (((x) >> (lo)) & BITMASK((hi) - (lo) + 1)) // similar to x[hi:lo] in verilog
```

-   These two macros are used to extract some of the bits from a binary number.
-   Specifically, they are used to extract the 'immediate' from the instructions.

---

-   First we take a look at `BITMASK`:

    -   This macro creates a bitmask of a specified number of bits.
    -   For example, `BITMASK(3)` would create a bitmask of 3 bits: `0000...0111`, which is the number `7`.
    -   The macro works by shifting the number `1` to the left by bits positions and then subtracting `1` from the result:

        -   `1` -> `0000...1000` (shift `1` to the left by 3 bits)
        -   `0000...1000` -> `0000...0111` (subtract `1` from the result)

    -   _NB: `1ull` stands for 'unsigned long long integer'. The suffix 'ull' here is used to make the number `1` 64-bit_

-   Then `BITS`:

    -   This macro extracts bits from position `lo` to position `hi` from the value `x`.
    -   It does it by performing several bit operations:

        -   First it shifts `x` to the right by `lo` positions, which effectively removes the bits lower than the bit on `lo` position.
        -   Second it masks `x` with the bitmask created by the macro explained above to wipe out all values higher than position `hi` of the original `x`.

### _SEXT_

```c
#define SEXT(x, len)                                                           \
    ({                                                                         \
        struct {                                                               \
            int64_t n : len;                                                   \
        } __x = {.n = x};                                                      \
        (uint64_t) __x.n;                                                      \
    })
```

-   `SEXT` stands for **Sign Extension**, which is the process of extending the sign bit of a binary number when moving it to a larger bit width.

-   In the macro, the expression uses a bit field within a struct to define a signed integer of length `len` bits. The bit field ensures that the value of `x` is treated as a signed integer of that specific length.
-   The struct with the bit field automatically sign-extends the value when it is assigned to the `int64_t` member. The result is then cast back to `uint64_t` to maintain the full precision of the extended value.

---

-   The reason why sign extension is needed is that, the offset(immediate) is a signed value and **it's length is less than 64 bits**(well of course since the length of the whole instruction is merely 32 bits). If we directly store it in a 64-bit variable, **the higher bits are filled with zeros by default**.

    -   Problem that sign extension resolves:
    -   If we have a negative offset(immediate) presented as **two's complement**, which means that it's higher bits should all be `1`.
    -   Now if we store it directly into a 64-bit variable, the higher bits are becoming `0`s, thus making the variable a positive one.
    -   Example:

        -   **Original 8-bit Signed Value**: Let's say you have the binary value `11111111` (which is `-1` in two's complement notation when interpreted as a 8-bit signed integer).
        -   **Direct Storage Without Sign Extension**: If you store this 8-bit value directly into a 32-bit variable without sign extension, it would look like this: `00000000 00000000 00000000 11111111` (which is `255` in decimal, a positive value).

## Casts and Conventions: Ensuring Proper Sign Extension in C

-   In the implementation of instruction `mulh`, I encountered a suttle issue deeply rooted in C's type conversion conventions.

-   RISC-V manual defines `mulh` as follows:

    > MUL performs an XLEN-bit×XLEN-bit multiplication of rs1 by rs2 and places the lower XLEN bits
    > in the destination register. MULH, MULHU, and MULHSU perform the same multiplication but return the upper XLEN bits of the full 2×XLEN-bit product, for signed×signed, unsigned×unsigned,
    > and signed rs1×unsigned rs2 multiplication, respectively.

-   Essentially, it's just an extended version `mul`, designed for caculating larger numbers with lower precision.

---

-   My first implementation looks like this:

    ```c
    INSTPAT(...);
    INSTPAT("0000001 ????? ????? 001 ????? 01100 11", mulh, R,
            R(rd) = ((int64_t)src1 * (int64_t)src2) >> 32);
    INSTPAT(...);
    ```

-   During the test of `mul-longlong.c`, I used difftest to locate the problem is at `pc = 0x800000a0`, which is:

    ```plaintext
    800000a0:	02fc17b3          	mulh	a5,s8,a5
    ```

-   The difftest showed a discrepancy:

    ```plaintext
    [src/isa/riscv32/difftest/dut.c:23 isa_difftest_checkregs] a5 is different after executing instruction at pc = 0x800000a0, right = 0x19d29ab9, wrong = 0x7736200d, diff = 0x6ee4bab4
    ```

---

-   At first glance, the logic written in `inst.c` seemed correct:

    1.  convert the value stored in the two registers to signed intergers.
    2.  multiply them together, then shift right for 32 bits to get the upper 32 bits, and then store the result in the destination register.

-   It turns out that the problem rooted in the conversion process.

---

-   In C, when converting a smaller signed type to a larger signed type, for example, casting a 32-bit signed integer(`int32_t`) to a 64-bit signed integer(`int64_t`), C correctly handles the sign extension, preserving the value after the conversion.
-   The same thing happens when we convert an unsigned type to a same size signed type, C will do the sign extension automatically as well.
-   But, if we convert a smaller unsigned type, to a larger signed type, C does not automatically perform sign extension, which leads to the problem that I was facing with.

-   Consider this simple example program:

    ```c
    #include <stdint.h>
    #include <stdio.h>

    int main() {
        // Max value for 32-bit unsigned integer
        uint32_t u32 = 0xaeb1c2aa;
        // Cast to a 64-bit signed integer
        int64_t s64 = (int64_t)u32;

        printf("u32:  %u\n", u32);
        printf("s32:  %d\n", (int32_t)u32);
        printf("directly converted s64:  %ld\n", s64);

        // Max value for 32-bit unsigned integer
        uint32_t u32_2 = 0xaeb1c2aa;
        // Cast to a 64-bit signed integer with sign extension
        int64_t s64_2 = (int64_t)(int32_t)u32;
        printf("expected s64:  %ld\n", s64_2);

        return 0;
    }
    ```

    output:

    ```plaintext
    u32:  2930885290
    s32:  -1364082006
    directly converted s64:  2930885290
    expected s64:  -1364082006
    ```

-   From the example above we can see that: **when directly convert a 32-bit unsigned integer to a 64-bit signed integer, C does not automatically perform sign extension.**
-   That is the reason why my first implementation for instruction `mulh` was incorrect: **it did not conduct sign conversion**.

---

-   The solution is simple, first convert the value into a 32-bit signed int, then convert it into a 64-bit signed one:

    ```c
    INSTPAT("0000001 ????? ????? 001 ????? 01100 11", mulh, R,
            R(rd) = ((int64_t)(int32_t)src1 * (int64_t)(int32_t)src2) >> 32);
    ```

## Cross Compilation Env

-   When testing the implemented instructions, we would need to run tests in `am-kernels/tests/cpu-tests/tests/`.
-   The execution command is(take `dummy.c` as example):

    ```bash
    make ARCH=$ISA-nemu ALL=dummy run
    ```

-   However, my machine(cpu) is of x86 architecture, which means that it can not compile source code to riscv32 executables directly.
-   We would need the following packages to perform the compilation:

    ```bash
    sudo pacman -Sy riscv64-linux-gnu-gcc
    ```

---

-   Though still some errors may occur:

    {% folding orange::Possible Compilation Error %}

    1. `wordsize.h` :

        ```bash
        /usr/riscv64-linux-gnu/include/bits/wordsize.h:28:3: error: #error "rv32i-based targets are not supported"
        ```

        - **solution**: modify `/usr/riscv64-linux-gnu/include/bits/wordsize.h` :

        ![diff_1](https://s2.loli.net/2024/09/11/TLHS93vWhu4dXtG.png)

    2. `stubs.h` :

        ```bash
        /usr/riscv64-linux-gnu/include/gnu/stubs.h:8:11: fatal error: gnu/stubs-ilp32.h: No such file or directory
        ```

        - **solution**: modify `/usr/riscv64-linux-gnu/include/gnu/stubs.h` :

        ![diff_2](https://s2.loli.net/2024/09/11/21xQnpzPo796sL4.png)

    {% endfolding %}

## Weird `int` parameter of `memset`

-   When implementing our own lib utils, I encountered a little problem in `memset`.

-   The first version of my `memset` looks like this:

    ```c
    void *memset(void *s, int c, size_t n) {
        size_t *p = s;
        for (size_t i = 0; i < n; i++) {
            *p = c;
            p++;
        }

        return s;
    }
    ```

-   The problem is that, the size of the memory which would be set by this function is of unit **byte**. However, here in my implemtation, the unit is **int**.

-   The correct version is:

    ```c
    void *memset(void *s, int c, size_t n) {
        unsigned char *p = s;
        for (size_t i = 0; i < n; i++) {
            *p = (unsigned char)c;
            p++;
        }

        return s;
    }
    ```

-   We use `unsigned char` to represent a **byte** in memory, as it's size is exactly one byte.
-   As for the reason of using `unsigned`, it is because of that we want to eliminate the ambiguity of the various implemetation of signed char.

## Parsing ELF

-   Parsing ELF files is the most difficult part in `ftrace` development. I knew nothing about the elf file before reading the man page of `elf` for two afternoons.
-   Here I recorded some of the most complicated part of the development for future review.

---

Wikipedia:

> In computing, the **Executable and Linkable Format**(**ELF**, formerly named Extensible Linking Format) is a common standard file format for executable files, object code, shared libraries, and core dumps.

-   We can use `readelf -a` to extract the content stored in an ELF file.
-   Take a `hello.c` as an example:

{% folding blue::readelf %}

    ````bash
     readelf -a hello
    ELF Header:
    Magic: 7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
    Class: ELF64
    Data: 2's complement, little endian
    Version: 1 (current)
    OS/ABI: UNIX - System V
    ABI Version: 0
    Type: DYN (Position-Independent Executable file)
    Machine: Advanced Micro Devices X86-64
    Version: 0x1
    Entry point address: 0x1040
    Start of program headers: 64 (bytes into file)
    Start of section headers: 13552 (bytes into file)
    Flags: 0x0
    Size of this header: 64 (bytes)
    Size of program headers: 56 (bytes)
    Number of program headers: 13
    Size of section headers: 64 (bytes)
    Number of section headers: 30
    Section header string table index: 29

    Section Headers:
    [Nr] Name              Type             Address           Offset
        Size              EntSize          Flags  Link  Info  Align
    [ 0]                   NULL             0000000000000000  00000000
        0000000000000000  0000000000000000           0     0     0
    [ 1] .interp           PROGBITS         0000000000000318  00000318
        000000000000001c  0000000000000000   A       0     0     1
    [ 2] .note.gnu.pr[...] NOTE             0000000000000338  00000338
        0000000000000040  0000000000000000   A       0     0     8
    [ 3] .note.gnu.bu[...] NOTE             0000000000000378  00000378
        0000000000000024  0000000000000000   A       0     0     4
    [ 4] .note.ABI-tag     NOTE             000000000000039c  0000039c
        0000000000000020  0000000000000000   A       0     0     4
    [ 5] .gnu.hash         GNU_HASH         00000000000003c0  000003c0
        000000000000001c  0000000000000000   A       6     0     8
    [ 6] .dynsym           DYNSYM           00000000000003e0  000003e0
        00000000000000a8  0000000000000018   A       7     1     8
    [ 7] .dynstr           STRTAB           0000000000000488  00000488
        000000000000008d  0000000000000000   A       0     0     1
    [ 8] .gnu.version      VERSYM           0000000000000516  00000516
        000000000000000e  0000000000000002   A       6     0     2
    [ 9] .gnu.version_r    VERNEED          0000000000000528  00000528
        0000000000000030  0000000000000000   A       7     1     8
    [10] .rela.dyn         RELA             0000000000000558  00000558
        00000000000000c0  0000000000000018   A       6     0     8
    [11] .rela.plt         RELA             0000000000000618  00000618
        0000000000000018  0000000000000018  AI       6    23     8
    [12] .init             PROGBITS         0000000000001000  00001000
        000000000000001b  0000000000000000  AX       0     0     4
    [13] .plt              PROGBITS         0000000000001020  00001020
        0000000000000020  0000000000000010  AX       0     0     16
    [14] .text             PROGBITS         0000000000001040  00001040
        000000000000012f  0000000000000000  AX       0     0     16
    [15] .fini             PROGBITS         0000000000001170  00001170
        000000000000000d  0000000000000000  AX       0     0     4
    [16] .rodata           PROGBITS         0000000000002000  00002000
        000000000000000f  0000000000000000   A       0     0     4
    [17] .eh_frame_hdr     PROGBITS         0000000000002010  00002010
        000000000000002c  0000000000000000   A       0     0     4
    [18] .eh_frame         PROGBITS         0000000000002040  00002040
        000000000000009c  0000000000000000   A       0     0     8
    [19] .init_array       INIT_ARRAY       0000000000003dd0  00002dd0
        0000000000000008  0000000000000008  WA       0     0     8
    [20] .fini_array       FINI_ARRAY       0000000000003dd8  00002dd8
        0000000000000008  0000000000000008  WA       0     0     8
    [21] .dynamic          DYNAMIC          0000000000003de0  00002de0
        00000000000001e0  0000000000000010  WA       7     0     8
    [22] .got              PROGBITS         0000000000003fc0  00002fc0
        0000000000000028  0000000000000008  WA       0     0     8
    [23] .got.plt          PROGBITS         0000000000003fe8  00002fe8
        0000000000000020  0000000000000008  WA       0     0     8
    [24] .data             PROGBITS         0000000000004008  00003008
        0000000000000010  0000000000000000  WA       0     0     8
    [25] .bss              NOBITS           0000000000004018  00003018
        0000000000000008  0000000000000000  WA       0     0     1
    [26] .comment          PROGBITS         0000000000000000  00003018
        0000000000000036  0000000000000001  MS       0     0     1
    [27] .symtab           SYMTAB           0000000000000000  00003050
        0000000000000258  0000000000000018          28     6     8
    [28] .strtab           STRTAB           0000000000000000  000032a8
        000000000000012d  0000000000000000           0     0     1
    [29] .shstrtab         STRTAB           0000000000000000  000033d5
        0000000000000116  0000000000000000           0     0     1
    Key to Flags:
    W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
    L (link order), O (extra OS processing required), G (group), T (TLS),
    C (compressed), x (unknown), o (OS specific), E (exclude),
    D (mbind), l (large), p (processor specific)

    There are no section groups in this file.

    Program Headers:
    Type           Offset             VirtAddr           PhysAddr
                    FileSiz            MemSiz              Flags  Align
    PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                    0x00000000000002d8 0x00000000000002d8  R      0x8
    INTERP         0x0000000000000318 0x0000000000000318 0x0000000000000318
                    0x000000000000001c 0x000000000000001c  R      0x1
        [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
    LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                    0x0000000000000630 0x0000000000000630  R      0x1000
    LOAD           0x0000000000001000 0x0000000000001000 0x0000000000001000
                    0x000000000000017d 0x000000000000017d  R E    0x1000
    LOAD           0x0000000000002000 0x0000000000002000 0x0000000000002000
                    0x00000000000000dc 0x00000000000000dc  R      0x1000
    LOAD           0x0000000000002dd0 0x0000000000003dd0 0x0000000000003dd0
                    0x0000000000000248 0x0000000000000250  RW     0x1000
    DYNAMIC        0x0000000000002de0 0x0000000000003de0 0x0000000000003de0
                    0x00000000000001e0 0x00000000000001e0  RW     0x8
    NOTE           0x0000000000000338 0x0000000000000338 0x0000000000000338
                    0x0000000000000040 0x0000000000000040  R      0x8
    NOTE           0x0000000000000378 0x0000000000000378 0x0000000000000378
                    0x0000000000000044 0x0000000000000044  R      0x4
    GNU_PROPERTY   0x0000000000000338 0x0000000000000338 0x0000000000000338
                    0x0000000000000040 0x0000000000000040  R      0x8
    GNU_EH_FRAME   0x0000000000002010 0x0000000000002010 0x0000000000002010
                    0x000000000000002c 0x000000000000002c  R      0x4
    GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                    0x0000000000000000 0x0000000000000000  RW     0x10
    GNU_RELRO      0x0000000000002dd0 0x0000000000003dd0 0x0000000000003dd0
                    0x0000000000000230 0x0000000000000230  R      0x1

    Section to Segment mapping:
    Segment Sections...
    00
    01     .interp
    02     .interp .note.gnu.property .note.gnu.build-id .note.ABI-tag .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt
    03     .init .plt .text .fini
    04     .rodata .eh_frame_hdr .eh_frame
    05     .init_array .fini_array .dynamic .got .got.plt .data .bss
    06     .dynamic
    07     .note.gnu.property
    08     .note.gnu.build-id .note.ABI-tag
    09     .note.gnu.property
    10     .eh_frame_hdr
    11
    12     .init_array .fini_array .dynamic .got

    Dynamic section at offset 0x2de0 contains 26 entries:
    Tag        Type                         Name/Value
    0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
    0x000000000000000c (INIT)               0x1000
    0x000000000000000d (FINI)               0x1170
    0x0000000000000019 (INIT_ARRAY)         0x3dd0
    0x000000000000001b (INIT_ARRAYSZ)       8 (bytes)
    0x000000000000001a (FINI_ARRAY)         0x3dd8
    0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
    0x000000006ffffef5 (GNU_HASH)           0x3c0
    0x0000000000000005 (STRTAB)             0x488
    0x0000000000000006 (SYMTAB)             0x3e0
    0x000000000000000a (STRSZ)              141 (bytes)
    0x000000000000000b (SYMENT)             24 (bytes)
    0x0000000000000015 (DEBUG)              0x0
    0x0000000000000003 (PLTGOT)             0x3fe8
    0x0000000000000002 (PLTRELSZ)           24 (bytes)
    0x0000000000000014 (PLTREL)             RELA
    0x0000000000000017 (JMPREL)             0x618
    0x0000000000000007 (RELA)               0x558
    0x0000000000000008 (RELASZ)             192 (bytes)
    0x0000000000000009 (RELAENT)            24 (bytes)
    0x000000006ffffffb (FLAGS_1)            Flags: PIE
    0x000000006ffffffe (VERNEED)            0x528
    0x000000006fffffff (VERNEEDNUM)         1
    0x000000006ffffff0 (VERSYM)             0x516
    0x000000006ffffff9 (RELACOUNT)          3
    0x0000000000000000 (NULL)               0x0

    Relocation section '.rela.dyn' at offset 0x558 contains 8 entries:
    Offset          Info           Type           Sym. Value    Sym. Name + Addend
    000000003dd0  000000000008 R_X86_64_RELATIVE                    1130
    000000003dd8  000000000008 R_X86_64_RELATIVE                    10e0
    000000004010  000000000008 R_X86_64_RELATIVE                    4010
    000000003fc0  000100000006 R_X86_64_GLOB_DAT 0000000000000000 __libc_start_main@GLIBC_2.34 + 0
    000000003fc8  000200000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTM[...] + 0
    000000003fd0  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
    000000003fd8  000500000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCl[...] + 0
    000000003fe0  000600000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0

    Relocation section '.rela.plt' at offset 0x618 contains 1 entry:
    Offset          Info           Type           Sym. Value    Sym. Name + Addend
    000000004000  000300000007 R_X86_64_JUMP_SLO 0000000000000000 puts@GLIBC_2.2.5 + 0
    No processor specific unwind information to decode

    Symbol table '.dynsym' contains 7 entries:
    Num:    Value          Size Type    Bind   Vis      Ndx Name
        0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
        1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _[...]@GLIBC_2.34 (2)
        2: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterT[...]
        3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (3)
        4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
        5: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMC[...]
        6: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND [...]@GLIBC_2.2.5 (3)

    Symbol table '.symtab' contains 25 entries:
    Num:    Value          Size Type    Bind   Vis      Ndx Name
        0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
        1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS hello.c
        2: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS
        3: 0000000000003de0     0 OBJECT  LOCAL  DEFAULT   21 _DYNAMIC
        4: 0000000000002010     0 NOTYPE  LOCAL  DEFAULT   17 __GNU_EH_FRAME_HDR
        5: 0000000000003fe8     0 OBJECT  LOCAL  DEFAULT   23 _GLOBAL_OFFSET_TABLE_
        6: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_mai[...]
        7: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterT[...]
        8: 0000000000004008     0 NOTYPE  WEAK   DEFAULT   24 data_start
        9: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5
        10: 0000000000004018     0 NOTYPE  GLOBAL DEFAULT   24 _edata
        11: 0000000000001170     0 FUNC    GLOBAL HIDDEN    15 _fini
        12: 0000000000001139    22 FUNC    GLOBAL DEFAULT   14 hello
        13: 0000000000004008     0 NOTYPE  GLOBAL DEFAULT   24 __data_start
        14: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
        15: 0000000000004010     0 OBJECT  GLOBAL HIDDEN    24 __dso_handle
        16: 0000000000002000     4 OBJECT  GLOBAL DEFAULT   16 _IO_stdin_used
        17: 0000000000004020     0 NOTYPE  GLOBAL DEFAULT   25 _end
        18: 0000000000001040    38 FUNC    GLOBAL DEFAULT   14 _start
        19: 0000000000004018     0 NOTYPE  GLOBAL DEFAULT   25 __bss_start
        20: 000000000000114f    32 FUNC    GLOBAL DEFAULT   14 main
        21: 0000000000004018     0 OBJECT  GLOBAL HIDDEN    24 __TMC_END__
        22: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMC[...]
        23: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@G[...]
        24: 0000000000001000     0 FUNC    GLOBAL HIDDEN    12 _init

    Version symbols section '.gnu.version' contains 7 entries:
    Addr: 0x0000000000000516  Offset: 0x00000516  Link: 6 (.dynsym)
    000:   0 (*local*)       2 (GLIBC_2.34)    1 (*global*)      3 (GLIBC_2.2.5)
    004:   1 (*global*)      1 (*global*)      3 (GLIBC_2.2.5)

    Version needs section '.gnu.version_r' contains 1 entry:
    Addr: 0x0000000000000528  Offset: 0x00000528  Link: 7 (.dynstr)
    000000: Version: 1  File: libc.so.6  Cnt: 2
    0x0010:   Name: GLIBC_2.2.5  Flags: none  Version: 3
    0x0020:   Name: GLIBC_2.34  Flags: none  Version: 2

    Displaying notes found in: .note.gnu.property
    Owner                Data size 	Description
    GNU                  0x00000030	NT_GNU_PROPERTY_TYPE_0
        Properties: x86 ISA needed: x86-64-baseline
        x86 feature used: x86
        x86 ISA used:

    Displaying notes found in: .note.gnu.build-id
    Owner                Data size 	Description
    GNU                  0x00000014	NT_GNU_BUILD_ID (unique build ID bitstring)
        Build ID: 7e623b7c81161e12006d8a813799940eaafd1901

    Displaying notes found in: .note.ABI-tag
    Owner                Data size 	Description
    GNU                  0x00000010	NT_GNU_ABI_TAG (ABI version tag)
        OS: Linux, ABI: 4.4.0
    ```

{% endfolding %}

-   There're a lot of information shown here. Let's just focus on what we need: **function symbols**.

    ```plaintext
    Symbol table '.symtab' contains 25 entries:
    Num:   Value             Size Type    Bind   Vis      Ndx Name
        0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
        1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS hello.c
        2: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS
        3: 0000000000003de0     0 OBJECT  LOCAL  DEFAULT   21 _DYNAMIC
        4: 0000000000002010     0 NOTYPE  LOCAL  DEFAULT   17 __GNU_EH_FRAME_HDR
        5: 0000000000003fe8     0 OBJECT  LOCAL  DEFAULT   23 _GLOBAL_OFFSET_TABLE_
        6: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_mai[...]
        7: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterT[...]
        8: 0000000000004008     0 NOTYPE  WEAK   DEFAULT   24 data_start
        9: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5
       10: 0000000000004018     0 NOTYPE  GLOBAL DEFAULT   24 _edata
       11: 0000000000001170     0 FUNC    GLOBAL HIDDEN    15 _fini
       12: 0000000000001139    22 FUNC    GLOBAL DEFAULT   14 hello
       13: 0000000000004008     0 NOTYPE  GLOBAL DEFAULT   24 __data_start
       14: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
       15: 0000000000004010     0 OBJECT  GLOBAL HIDDEN    24 __dso_handle
       16: 0000000000002000     4 OBJECT  GLOBAL DEFAULT   16 _IO_stdin_used
       17: 0000000000004020     0 NOTYPE  GLOBAL DEFAULT   25 _end
       18: 0000000000001040    38 FUNC    GLOBAL DEFAULT   14 _start
       19: 0000000000004018     0 NOTYPE  GLOBAL DEFAULT   25 __bss_start
       20: 000000000000114f    32 FUNC    GLOBAL DEFAULT   14 main
       21: 0000000000004018     0 OBJECT  GLOBAL HIDDEN    24 __TMC_END__
       22: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMC[...]
       23: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@G[...]
       24: 0000000000001000     0 FUNC    GLOBAL HIDDEN    12 _init
    ```

-   From the above _symbol table_ we can see that, among all of the symbols, those with type `FUNC` should be the ones that we want for ftrace.
-   However, it's not that simple.

    -   It's not easy to locate and read the _symbol table_ from an ELF file since it's in binary.
    -   The `NAME` in the _symbol table_ is actually an **offset** used to search from _string table_.

-   To resolve these two issues, we will need to take a look at the structure of a common ELF file.

### ELF File Structure

-   Basically, a common ELF file contains an **ELF header**, which includes information about itself, such as:

    -   `e_ident`: a magic number identifying the file as an ELF file
    -   `e_shoff`: section header's offset
    -   `e_shnum`: count of section headers
    -   `e_shstrndx`: index of the names' section in the table

-   Those variables are the guides for us to extract useful content from the ELF file.
-   Other parts of ELF file, such as _section header table_, _string table_ and _section name string table_, can be found using these offsets extracted from **ELF header**.

-   But how exactly should we read them from an ELF file? The answer is, with the help of `<elf.h>`.

### `<elf.h>`

`man 5 elf` :

> The header file <elf.h> defines the format of ELF executable binary files. Amongst these files are normal executable files, relocatable object files, core files, and shared objects.

> An executable file using the ELF file format cosists of an ELF header, followed by a program header table or a section header table, or both. The ELF header is always at offset zero of the file. The program header table and the section header table's offset in the file are defined in the ELF header. The two tables describe the rest of the particularities of the file.

-   This header file defines a lot of types(structs) to help users read from the binary ELF files.
-   For example, a struct `Elf32_Ehdr` of ELF header:

    ```c
    typedef struct {
        unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
        Elf32_Half	e_type;			/* Object file type */
        Elf32_Half	e_machine;		/* Architecture */
        Elf32_Word	e_version;		/* Object file version */
        Elf32_Addr	e_entry;		/* Entry point virtual address */
        Elf32_Off	e_phoff;		/* Program header table file offset */
        Elf32_Off	e_shoff;		/* Section header table file offset */
        Elf32_Word	e_flags;		/* Processor-specific flags */
        Elf32_Half	e_ehsize;		/* ELF header size in bytes */
        Elf32_Half	e_phentsize;		/* Program header table entry size */
        Elf32_Half	e_phnum;		/* Program header table entry count */
        Elf32_Half	e_shentsize;		/* Section header table entry size */
        Elf32_Half	e_shnum;		/* Section header table entry count */
        Elf32_Half	e_shstrndx;		/* Section header string table index */
    } Elf32_Ehdr;
    ```

-   We can use it to define a variable and store the content read from the file to it:

    ```c
    // read the whole ELF file
    ElfW(Ehdr) ELF_header;
    fread(&ELF_header, sizeof(ELF_header), 1, file);
    ```

    -   BTW, there's a macro `ElfW` used here:

        ```c
        #if defined(CONFIG_RV64)
        #define ElfW(type) Elf64_##type
        #else
        #define ElfW(type) Elf32_##type
        #endif /* if defined (CONFIG_RV64) */
        ```

    -   I have to say, that, macro is kinda useful. While I disliked it when first encountered it.

-   We could then use the `e_shoff` from the ELF header to locate the position of section headers:

    ```c
    // read section headers
    size_t sh_entry_size = sizeof(ElfW(Shdr));
    ElfW(Shdr) *shdrs = malloc(ELF_header.e_shnum * sh_entry_size);
    Assert(shdrs, ANSI_FMT("Failed to allocate memory for section headers\n",
                           ANSI_FG_RED));

    fseek(file, ELF_header.e_shoff, SEEK_SET);
    fread(shdrs, sh_entry_size, ELF_header.e_shnum, file);
    ```

-   And so on.

## Missing Symbols in ELF

-   Sometimes `gcc` automatically inlines functions written in the source file.
-   In this case, they are not shown in the symbol table as there's no such symbol after pre-compilation.

-   We can use an attribute to stop `gcc` from inlining these functions:

    ```c
    int is_prime(int n) __attribute__((noinline));
    ```

## _use-after-free_ Error

```c
FuncSym func = {
    .name = string_table[symbol_table[i].st_name],
    .value = symbol_table[i].st_value,
    .size = symbol_table[i].st_size,
};

...

free(string_table);
```

-   I ran into this problem as the code shown above. This is the first time I encounter with a use-after-free error, so it took me a while to figure out what's happening.
-   The tricky part of this bug is that, there're no errors reported by compiler or the program itself, it just caused the output info from the function table to be random characters.

-   Solution: use `strdup` to duplicate a string from the `string_table`.

    ```c
    FuncSym func = {
        .name = strdup(&string_table[symbol_table[i].st_name]),
        .value = symbol_table[i].st_value,
        .size = symbol_table[i].st_size,
    };
    ```

## DiffTest

### `device-tree-compiler` on ArchLinux

-   `device-tree-compiler` is a package required for doing difftest on `spike`.
-   This pacakge is named `dtc` on Arch Linux official repo.

    ```bash
    sudo pacman -Sy dtc
    ```

## To Be Continued...
