---
title: PA W2 - C Programming Language Basic
date: 2024-06-15 10:57:09
categories: 技术
tags:
  - c
  - gcc
excerpt: Notes taken for class W2 of NJU PA
---

> [!WARNING]
> If someone is reading this blog, please be aware that the writer **did not** consider the experience of the other readers.
> After all, the most important part is about writing things down for better memorization.

## C programming language used in PA

-   How to generate an executable file from a C source file?

```plaintext
.c --precompile--> .i --compile--> .s --assemble--> .o --link--> .out
```

## Precompile

-   What are precompile behavior?

-   For example, these are precompile behavior:

```c
#include
#define
#
##
#ifdef
```

### _include_

-   Normally, head files do not contain the definition functions, instead, they include the declaration of the functions.(and some variables)

-   the declaration of the standard library function `printf`:
    ```c
    int printf(const char *restrict format, ...);
    ```
-   if we declare this `printf` function before we invoke it, the program will run well even without including `stdio.h`:
    ```c
    int printf(const char *restrict format, ...);

    int main(int argc, char *argv[]) {
        printf("%d\n", a);
        return 0;
    }
    ```
-   Thus, including head files are essentially **_copying the content inside the head files to the source code_**.

---

-   What's the differences between these two lines of code?
    ```c
    #include "stdio.h"
    #include <stdio.h>
    ```
-   The answer is: They are actually the same, except that they would search **different path** for the required head file.
-   We can see the search list by adding `--verbose` flag to gcc:

![06-15-1.png](https://s2.loli.net/2024/06/15/PY5X8Gf9verjClk.png)

-   We can also add search path manualy by adding telling gcc the path with `-I` flag:
    ```bash
    gcc --verbose test1.c -I /home/last/Coding/Cpp/test
    ```

![06-15-2.png](https://s2.loli.net/2024/06/15/OFwUgleymXWZvAo.png)

-   We can see that `/home/last/Coding/Cpp/test/` is added to the search list. (as the first one)

### _#if_

-   What would this piece of code output?

```c
#include <stdio.h>

int main(int argc, char *argv[]) {
#if aa == bb
    printf("Yes\n");
#else
    printf("No\n");
#endif
    return 0;
}
```

-   The answer is `Yes`.

-   But why? What exactly is `aa == bb` ?
-   Actually, `aa` and `bb` are _marcos_. As shown in the code, these two macros are both not defined, thus making them equal as they are both `null`.

-   After precompilation is done, the code in the false branch of the if statement will be dropped.

### _#define_

-   This statement is for defining _macros_. Macros will be expand when precompiling, which is basically copying and pasting again, just like the head file.

-   How to destroy an OJ? LMAO

![06-15-4.png](https://s2.loli.net/2024/06/15/LvEUfKjIw86oQzd.png)

-   From the picture above we can see that, macros can call(rely) on each other. It would be pretty common to see multiple macros calling each other in the source code of PA.

-   Macros can also be used to hide some keywords from inspection.

```cpp
#define A sys ## tem

int main() {
    A("echo Hello\n");
}
```

---

#### X-macro

```c
#define NAMES(X) \
    X(Tom) X(Jerry) X(Tyke) X(Spike)

int main() {
    #define PRINT(x) puts("Hello, " #x "!");
    NAMES(PRINT);
}
```

-   The output will be like:

```plaintext
Hello, Tom!
Hello, Jerry!
Hello, Tyke!
Hello, Spike!
```

-   We can see that `NAMES(PRINT)` calls the function `PRINT` for four times with different parameters each time:

```plaintext
NAMES(PRINT) --> PRINT(Tom) PRINT(Jerry) PRINT(Tyke) PRINT(Spike)
```

---

-   From above we can see that macro provides a way for the programmer to define some sort of customized 'function' or 'variable', which is called _**meta-programming**_.
-   It is very flexible and easy to use, but at the same time, it will reduce the readability of the code to some extend.

## Compile && Assemble

-   This step is mainly about translating the C programming language to the assembly language.
-   For example, a simple function like:

```c
int foo(int n) {
    int sum = 0;
    for (int i = 0; i <= n; i++) {
        sum += i;
    }

    return sum;
}
```

```bash
gcc --assemble 4test-1.c
```

will be translate to:

```asm
	.file	"4test-1.c"
	.text
	.globl	foo
	.type	foo, @function
foo:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	movl	%edi, -20(%rbp)
	movl	$0, -8(%rbp)
	movl	$0, -4(%rbp)
	jmp	.L2
.L3:
	movl	-4(%rbp), %eax
	addl	%eax, -8(%rbp)
	addl	$1, -4(%rbp)
.L2:
	movl	-4(%rbp), %eax
	cmpl	-20(%rbp), %eax
	jle	.L3
	movl	-8(%rbp), %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE0:
	.size	foo, .-foo
	.ident	"GCC: (GNU) 14.1.1 20240522"
	.section	.note.GNU-stack,"",@progbits
```

-   This rough file is hard to understand. But if we remove the parts which are unrelated to the core logic of the program, it will become this:

```asm
foo:
	movl	%edi, -20(%rbp)
	movl	$0, -8(%rbp)
	movl	$0, -4(%rbp)
	jmp	.L2
.L3:
	movl	-4(%rbp), %eax
	addl	%eax, -8(%rbp)
	addl	$1, -4(%rbp)
.L2:
	movl	-4(%rbp), %eax
	cmpl	-20(%rbp), %eax
	jle	.L3
	movl	-8(%rbp), %eax

	ret
```

-   Still hard to read for human, lets replace all of the variables with their names in the source code:

```plaintext
foo:
	movl	%edi, n
	movl	$0, sum
	movl	$0, i
	jmp	.L2
.L3:
	movl	i, tmp
	addl	tmp, sum
	addl	$1, i
.L2:
	movl	i, tmp
	cmpl	n, tmp
	jle	.L3
	movl	sum, tmp

	ret
```

-   Then, make it a pseudocode:

```plaintext
foo:
    n = ARG-1
    sum = 0
    i = 1
    goto .L2
.L3:
    tmp = i
    sum += tmp
    i += 1
.L2:
    tmp = i
    compare (n, tmp)
    if(<=)  goto .L3
    RETURN-VAL = sum

    ret
```

## Link (Static Link)

-   The file containing the assembly language translated from the `foo` function is not able to run yet, since it's missing a `main` function, the entry point.
-   Let's quickly define a `main` function in a separate file.

4test-2.c:

```c
#include "4test-1.c"
#include <stdio.h>

int foo(int n);

int main(int argc, char *argv[]) {
    printf("%d\n", foo(100));
    return 0;
}
```

-   AWAK, this source file will also be translated to assembly language by gcc. After that, gcc will link the two .o file together, producing the final `a.out` executable file.

## From a Memory Perspective

-   Why would this code report a segmentation fault?

```c
int main(int argc, char* argv[]) {
    int *p = (void *) 1;
    *p = 1; // Segmentation fault
}
```

-   From the code we can see that, this program is trying to write value `1` to a special memory address.
-   The thing is that, the memory block which the program can read or write is limited, mostly the part which allocated to the program by the OS.
-   This program is trying to write stuff into a memory address which **does not** 'belong' to it, which causes a segmentation fault.

### Understanding Pointer

-   Let's explore the following code:

```c
#include <assert.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
    int (*f)(int, char *[]) = main;
    if (argc != 0) {
        // clang-format off
        char ***a   = &argv;
        char *first = argv[0];
        char ch     = argv[0][0];
        // clang-format on

        printf("arg = \"%s\"; ch = '%c'\n", first, ch);
        assert(***a == ch);
        f(argc - 1, argv + 1);
    }

    return 0;
}
```

-   It might be complicated when taking a first look, so take another look on this picture, which analyze the 'pointing' relationship between these pointers and variables:

![06-15-5.png](https://s2.loli.net/2024/06/15/xhqTLAVC5WY8r1f.png)

-   From the picture we can clearly see that, pointer `a` is pointing to the address which stores an address pointing to an address pointing to the first element of `argv`.
-   `first` is pointing to the first character of the first element of `argv`, which is a string, AKA, a character array.
-   `ch` is the first character of the first element of `argv`, which is ...

---

## End of the First Day...
