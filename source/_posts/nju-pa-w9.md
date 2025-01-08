---
title: PA W9 - Linking & Loading
date: 2024-12-20 11:00:32
categories: 笔记
tags:
    - pa
    - c
excerpt: Notes taken for class W9 of NJU PA
---

{% notel orange fa-triangle-exclamation Warning %}
If someone is reading this blog, please be aware that the writer **DID NOT** consider the experience of the other readers.
After all, the most important thing is about writing things down for better memorization.
{% endnotel %}

## Compilation Process

-   We've already seen this before in class 1:

    ```plaintext
    .c --precompile--> .i --compile--> .s --assemble--> .o --link--> .out
    ```

-   Today's lesson would be focusing on the process where the object file(.o) being transformed into an executable(.out).

## Static Linking

### Introduction

-   The process of linking is basically connecting all of the required parts of a program together.
-   Say we are using a function from another file in the `main.c`, the compiler would not know which address to jump to unless we link the other file with `main.c` together.

---

-   For example, if we have these three source files:

    ```c
    // a.c
    int foo (int a, int b) {
        return a + b;
    }

    // b.c
    int x = 100, y = 200;

    // main.c
    extern int x, y;
    int foo(int a, int b);
    int main() {
        foo(x, y);
    }
    ```

-   And then compile them separately, we would get three object files: `a.o`, `b.o` and `main.o`.
-   Then we will have to link them together to get the final expected executable file:

    ```c
    gcc -static a.o b.o main.o
    ```

    _Note: the flag `-static` is to explicitly tell gcc to use static linking, as modern versions of gcc would use dynamic linking by default._

-   The linking process would fail if we are missing one of the object files:

    ```bash
     gcc -static b.o main.o
    /usr/bin/ld: main.o: in function `main':
    main.c:(.text.startup+0x11): undefined reference to `foo'
    collect2: error: ld returned 1 exit status
    ```

-   Without linking, the compiler would never know where the definitions of `foo` or `x` `y` are.
-   But How exactly does the compiler do this?

### Know How

-   Turns out the object files compiled from source files are a type of ELF file called **Relocatable File**. This type of file contains code and data, and could be used for linking.

| ELF type           | Explanation                                                                                                       | Example(s)                       |
| ------------------ | ----------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| Relocatable File   | Object files that can be combined with other object files during linking to create an executable or shared object | `.o`, `.obj`                     |
| Executable File    | Files that contain a program ready to be executed directly by the system                                          | `.exe`, regular Linux programs   |
| Shared Object File | Libraries that can be loaded and linked dynamically at runtim by multiple programs                                | `.so`, `.dll`                    |
| Core Dump File     | Files containing a snapshot of a program's memory when it crashes or terminates abnormally                        | `core.1234` (`1234` for the PID) |

-   Part of the ELF structure looks like this:

| Section         | Explanation                                                                      |
| --------------- | -------------------------------------------------------------------------------- |
| ELF_Header      | Contains ident_num, version, machine type, entry point, etc.                     |
| .text section   | Contains executable code/instructions of the program in machine code format      |
| .rodata section | Contains read-only data like string literals, constants and static lookup tables |
| .data section   | Contains initialized global and static variables that can be modified at runtime |
| .bss section    | Contains uninitialized global and static variables (zeroed at program start)     |
| ...             | ...                                                                              |

_Note: modern compiler would put variables initialized with value `0` into .bss section rather than .data section by default_
_Note: We can use `objdump -h` to see those sections' info._

-   For the structure of an ELF file, also see [this chapter](https://blog.imlast.top/2024/10/02/nju-pa-2/#Parsing-ELF).

---

-   In the linking process, the compiler would join the different parts of those ELF files together to form a bigger file as the executable file. The differenct sections of those object files are merged.
-   This process includes two main phases: **Symbol Resolution** and **Relocation**.

#### Symbol Resolution

-   When source files are compiled into object files separately, the name of the functions and variables declared in the file are stored as **symbols**.
-   Each object file has a symbol table containing:

    -   Defined symbols (functions/variables defined in the source file being compiled)
    -   undefined symbols (functions/variables defined elsewhere)

-   During linking, the linker would scan all of the object files to match the defined and undefined symbols as much as possible.

-   In the example above, we have function `foo` and variable `x` & `y` defined in other source files other than `main.c`. It is the linker which put them together to form a complete executable file that include all of those symbols and their definitions.

#### Relocation

-   Initially, each object files's code is written assuming it starts at address 0.
-   The linker would:

    1.  Assign final memory locations to all sections(.text, .data, etc)
    2.  Adjust all memory references to use these new addresses
    3.  Update the machine code with correct addresses

-   Take the example above:

    ```bash
     objdump -d main.o

    main.o:     file format elf64-x86-64


    Disassembly of section .text.startup.main:

    0000000000000000 <main>:
    0:    48 83 ec 08             sub    $0x8,%rsp
    4:    8b 35 00 00 00 00       mov    0x0(%rip),%esi        # a <main+0xa>
    a:    8b 3d 00 00 00 00       mov    0x0(%rip),%edi        # 10 <main+0x10>
    10:   e8 00 00 00 00          call   15 <main+0x15>
    15:   31 c0                   xor    %eax,%eax
    17:   48 83 c4 08             add    $0x8,%rsp
    1b:   c3                      ret
    ```

    -   Notice the zeros in the address references:

        ```plaintext
        10:   e8 00 00 00 00          call   15 <main+0x15>
        ```

    -   This is where the relocation placeholder is. This blank address will be filled in when linking happens.

    -   Also, notice the instructions before `call`:

        ```plaintext
        4:    8b 35 00 00 00 00       mov    0x0(%rip),%esi        # a <main+0xa>
        a:    8b 3d 00 00 00 00       mov    0x0(%rip),%edi        # 10 <main+0x10>
        ```

    -   These are actually calling to variable `x` and `y`. As you can see, they are also placeholders like the `foo` one mentioned above.

    -   Inside `main.o` lies a table(Relocatable Section) indicating the symbols relocated from other elf files.
    -   `readelf -a main.o` :

        ```plaintext
        Relocation section '.rela.text.startup.main' at offset 0x198 contains 3 entries:
        Offset          Info           Type           Sym. Value      Sym. Name + Addend
        000000000006  000400000002 R_X86_64_PC32     0000000000000000 y - 4
        00000000000c  000500000002 R_X86_64_PC32     0000000000000000 x - 4
        000000000011  000600000004 R_X86_64_PLT32    0000000000000000 foo - 4
        ```

    -   The reason why the address of `foo` needs to be substracted by 4 is that, the offset encoded in `call` instruction is calculated relative to the address of the next instruction, which is 4 bytes ahead of the current instruction on x86-64.

    {% folding green::Fun_Fact %}

    -   This `-4` adjustment is a platform-related design -- pc always points to the next instruction in x86, which is likely intended to make shift for the poor hardware in the age of 1970s.
    -   On modern platforms like Risc-V _(some ARM modes as execption)_, pc will always point to the current executing instruction, so there's no need of `-4` adjustment.

        {% endfolding %}

---

{% notel blue fa-circle-question Whimsy %}

1. Is it possible to 'hack' someone's machine by tamperring this relocation process? Hijack the program from executing the original function and redirect it to execute another injected function? _(Turns out this way of 'hacking' has real-world example, which is GOT overwrite.)_
2. Is it the way of counting the usage(benchmark) of a certain function? Relocating it to a wrapper function which marks the usage and the time consumed and then jump to the intended function? _(It seems that there are better options.)_
   {% endnotel %}

---

-   Also, we can use `nm` to take a look at the symbol table stored inside `main.o`:

    ```bash
     nm main.o
                     U foo
    0000000000000000 T main
                     U x
                     U y
    ```

{% notel blue fa-circle-exclamation Note: %}
`a.out` compiled/linked with `-static` flag would contain symbols from `glibc` as static linkage would link `glibc` by default.
To get an `a.out` executable with a clean symbol table, one should remove the `-static` flag from the compilation command.
{% endnotel %}

### What Else?

-   Actually, `gcc` does not only link those code/data we've written in that three source files. It also links a lot of other standard libraries to ensure the proper execution of the program.
-   Take a look at the verbose information output by `gcc a.o b.o main.o -Wl,--verbose` and you'll see that gcc uses `ld` to link a lot of files including `a.o`, `b.o` and `main.o`.
-   As a result, linking them only using ld won't be enough: `ld a.o b.o main.o` would output an `a.out` that would trigger a _Segmentation Fault_.

## Dynamic Linking

-   Static linking would create a large executable file as it includes the entire code of the used libraries. When it comes to larger project which links a certain amount of libraries, the size of the executable may become significantly large, making it impractical for deployment or storage.
-   Dynamic linking was invented to resolve this issue: The system loads the requried shared libraries into memory(if not already loaded) when the executable is run and resolves the symbols.

-   Running executables compiled and linked with dynamic linking strategy is actually passing it to a dynamic linker.
-   We can use `readelf -l a.out | grep interpreter` to get the dynamic linker used for executing:

    ```bash
     readelf -l a.out | grep interpreter
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
    ```

---

-   There's a lot more about dynamic linking not mentioned in this class, as it's too complicated and has too much details.
