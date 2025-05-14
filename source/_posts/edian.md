---
title: 记一次小端序内存的实际体现
date: 2025-05-14 22:03:05
categories: 笔记
tags:
    - c
excerpt: 小端序机器内存变量存储字节序的一种体现
---

## 起因

- 记得去年在完成 NJU PA 的时候，遇见一位群友在读取 elf 文件并校验魔数的时候遇到的问题：他读取出的魔数是字节颠倒的。

- 他读出的魔数是 `46 4c 45 7f`，但实际上应当为 `7f 45 4c 46`。从表面上看，就好像是因为小端序机器中内存的存储方式一样，以字节为单位颠倒过来。

- 我当时仔细检查他的代码后发现，他使用 `fread` 函数的时候出现了错误，填反了 `size` 和 `n_count` 这两个参数：

    ```c
    uint32_t ident;
    FILE *fp = fopen("./a.out", "rb");

    fread(&ident, 1, 4, fp);
    ```

- 我想当然的认为，这就是导致最后字节顺序倒转的原因。

- 但实际上当时我们都没有尝试调转这两个参数的位置后，输出的内容会不会恢复正常——那位朋友后来得知了可以 `#include <elf.h>` 并且使用其中定义好的数据结构承接 elf 文件中的内容，也就不需要这种方法了。

    ```c
    ElfW(Ehdr) ELF_header;
    fread(&ELF_header, sizeof(ELF_header), 1, file);
    // check file ident
    if (memcmp(ELF_header.e_ident, ELFMAG, SELFMAG) != 0)
        Assert(0, ANSI_FMT("Not an ELF file: %s.", ANSI_FG_RED), elfFile);
    ```

## 实际原因

- 时隔半年我看到考研群里大伙在讨论大小端序，突然想起来上面这位朋友遇到的问题，便想着重现一下并且和群友们分享。但是我突然发现我始终无法获取一个相反的文件内容，不管 `size` 是不是 1。

- 询问大模型后我才恍然大悟，原来真正的问题并不在于 `fread`，而是用来存放读取到的内容的变量类型： `uint32_t`。

- 真正的原因在于从文件中读取的内容被存在了一个 `uint32_t` 类型的变量里，而这类变量在小端序机器的内存中存放的方式正是字节颠倒的。此时使用 `%x` 作为占位符将其输出时，就会得到一个字节颠倒的结果。

- 下面这两个函数可以很好的展示这一差异：

    ```c
    void read_elf_uint() {
        uint32_t ident;
        FILE *fp = fopen("./a.out", "rb");

        fread(&ident, 4, 1, fp);

        printf("reversed: 0x%x\n", ident);

        fclose(fp);
    }

    void read_elf() {
        unsigned char ident[4];
        memset(ident, 0, 4);
        FILE *fp = fopen("./a.out", "rb");

        fread(ident, 4, 1, fp);

        printf("correct: 0x");
        for (int i = 0; i < 4; i++) {
            printf("%x", ident[i]);
        }
        printf("\n");

        fclose(fp);
    }
    ```

## 感慨

- C 真是门艰深又麻烦的语言，为了灵活性导致了非常多隐形的复杂度。
