---
title: PA W3 - C Programming Language Practice
date: 2024-06-16 10:49:08
categories: 笔记
tags:
    - pa
    - c
excerpt: Notes taken for class W3 of NJU PA
---

{% notel orange fa-triangle-exclamation Warning %}
If someone is reading this blog, please be aware that the writer **did not** consider the experience of the other readers.
After all, the most important part is about writing things down for better memorization.
{% endnotel %}

## Readability

-   Take a look at this function declaration:

    ```c
    void (*signal (int sig, void (*func)(int)))(int);
    ```

-   What is this? It looks like a piece of concentrated ~~shit~~. Well, if you dig deep into it, you will find that it is:

    -   a function called `signal`
    -   the parameters of `signal` are:
        -   an `int` variable called `sig`
        -   a function pointer called `func` which points to a function which takes in an `int` variable and returns void
    -   the return value of `signal` is another function pointer which points to another function which takes in an `int` variable and returns void

|

-   Very creepy code style isn't it? How about changing it to a more readable form:

    ```c
    typedef void (*sighandler_t)(int);
    sighandler_t signal(int, sighandler_t);
    ```

-   Now we can clearly see the meaning of this declaration.

---

-   From the example above, we can draw two conclusions:

    -   The readability of the code does not depend on the length of it. Short code can also have bad readability.
    -   Code with bad readability is difficult to understand, thus difficult to maintain and expand, not to mention debugging.

-   Therefore, it is important to write code with good readability.

## State Machine

-   Take a look at this picture below:

    ![06-16-1.png](https://s2.loli.net/2024/06/16/2UzgvIuOQdotaNL.png)

-   The basic logic of the changes happen to `X` and `Y` can be illustrated as follow:

    ```c
    int X = 0, Y = 0;
    int X1 = 0, Y1 = 0;

    while (1) {
        X1 = (!X && Y) || (X && !Y);
        Y1 = !Y;

        X = X1; Y = Y1;
    }
    ```

-   What if we want to add another digit `Z`?

    ```c
    int X = 0, Y = 0, Z = 0; // updated
    int X1 = 0, Y1 = 0, Z1 = 0; // updated

    while (1) {
        X1 = (!X && Y) || (X && !Y);
        Y1 = !Y;
        Z1 = ...; // updated

        X = X1; Y = Y1; Z = Z1; // updated
    }
    ```

-   We can see that there are multiple places needs to be modified, which would be a pretty nasty situation.
-   Consider improving the initial code to this:

    ```c
    #include <stdio.h>
    #include <unistd.h>

    #define FORALL_REGS(_) _(X) _(Y)
    #define LOGIC                                                                  \
        X1 = (!X && Y) || (X && !Y);                                               \
        Y1 = !Y;
    #define DEFINE(X) static int X, X##1;
    #define UPDATE(X) X = X##1;
    #define PRINT(X) printf(#X " = %d; ", X);

    int main(int argc, char *argv[]) {
        FORALL_REGS(DEFINE);
        while (1) {
            FORALL_REGS(PRINT);
            putchar('\n');
            sleep(1);
            LOGIC;
            FORALL_REGS(UPDATE);
        }
        return 0;
    }
    ```

-   Now, if we want to add another digit `Z`, and would like these three digits perform a loop from `000` to `111` by adding up itself by 1, we can modify the code to this:

    ```c
    #include <stdio.h>
    #include <unistd.h>

    #define FORALL_REGS(_) _(X) _(Y) _(Z) // updated
    #define LOGIC                                                                  \
        X1 = (!X && Y && Z) || (X && (!(Y && Z)));                                 \
        Y1 = (!Y && Z) || (Y && !Z);                                               \
        Z1 = !Z; // updated
    #define DEFINE(X) static int X, X##1;
    #define UPDATE(X) X = X##1;
    #define PRINT(X) printf(#X " = %d; ", X);

    int main(int argc, char *argv[]) {
        FORALL_REGS(DEFINE);
        while (1) {
            FORALL_REGS(PRINT);
            putchar('\n');
            sleep(1);
            LOGIC;
            FORALL_REGS(UPDATE);
        }
        return 0;
    }
    ```

-   Noted that only macro `FORALL_REGS` and `LOGIC` are changed, the actual code remains the same. Don't tell me that you think the update of logic is too complicated, it would be the same complicated if you stick to the original code, plus other changes around the source code.

## Simulating an Operating System(OS)

-   Remember that, all computer system is a kind of state machine. So base on this simple 3-digit state machine shown above, we can expand it even to a real operating system(theoratically lol).

-   The whole thing about the 'big loop' which the cpu is processing can be simplified to three steps:

    -   **fetch**: read instruction from `M[R[PC]]`
    -   **decode**: parse the instruction according to the instruction set
    -   **execute**: execute the instruction, and the write the result to register/memory

-   These three steps are being gone through all the time when the OS is running.

### Simulating Storage

-   Suppose we have a computer which contains 4 registers and 16 bytes of memory. We can simulate them as the following code present:

    ```c
    #include <stdint.h>

    #define NREG 4  // 4 registers, 1 byte per register
    #define NMEM 16 // 16 bytes of memory

    typedef uint8_t u8;

    u8 pc = 0;
    u8 R[NREG];         // register
    u8 M[NMEM] = {...}; // memory
    ```

-   Is there a better way to simulate this scenario? Like adding names for each register?

    ```c
    enum { RA, R1, ..., PC };
    u8 R[] = {
        [RA] = 0,
        [R1] = 0,
        ...
        [PC] = init_pc,
    }
    #define pc (R[PC])
    #define NREG (sizeof(R) / sizeof(u8))
    ```

-   This time it looks great, but if the size of the register changes, we would still have to change the `typedef` for `u8`.
-   Here's serveral ways to improve this issue:

    ```c
    // breaks when adding a register
    #define NREG 5

    // breaks when changing register size
    #define nreg (sizeof(r) / sizeof(u8))

    // never breaks, but needs the definition of `r`
    #define nreg (sizeof(r) / sizeof(r[0]))

    // even better
    // does not need anything, even the calculation
    enum { ra, ..., pc, nreg }
    ```

### Simulating Instruction

-   Consider the following register and memory:
    -   **register**: _PC, R0(RA), R1, R2, R3 (8-bit)_
    -   **memory**: _16 bytes_
-   And this simple table as an instruction list.

    ![06-16-2.png](https://s2.loli.net/2024/06/16/AeB136FlfpLZSsC.png)

-   For example, instruction `0000 1100` means `mov r3 r0`, which means move the value stored in the `R3` register to `R0` register.

-   How can we perfrom this translation from the instruction to action in an actual C program? How to solve the idex issue? Well, please refer to the code below:

    ```c
        void idex() {
            if ((M[pc] >> 4) == 0) {
                // mov instruction, mov rt rs
                R[(M[pc] >> 2) & 3] = R[M[pc] & 3];
                pc++;
            } else if ((M[pc] >> 4) == 1) {
                // add instruction, add rt rs
                R[(M[pc] >> 2) & 3] += R[M[pc] & 3];
                pc++;
            } else if ((M[pc] >> 4) == 14) {
                // load instruction, put value in specified memory address(addr) to r0
                R[0] = M[M[pc] & 0xf];
                pc++;
            } else if ((M[pc] >> 4) == 15) {
                // store instruction, put r0 to specified memory address(addr)
                M[M[pc] & 0xf] = R[0];
                pc++;
            }
        }

        int main() {
            while (!is_halt(M[pc])) {
                idex();
            }
        }
    ```

-   This code is a little hard to understand when reading, cuz it uses a lot of decimal numers.
    -   For example, `(M[pc] >> 4) == 14`. This `14` at the right of the equal sign, actually means the binary number `1110`, which represents the instruction `load`.
-   What's more, it's using 'index' to find the required register at a high frequency, causing a lot of middle brackets in the code, which also reduces the readability.

-   Let's improve the code above into this form:

    ```c
    void index() {
        u8 inst = M[pc++];
        u8 op = inst >> 4; // the actual instruction code

        if (op == 0x0 || op == 0x1) {
            int rt = (inst >> 2) & 3, rs = (inst & 3);
            // mov
            if (op == 0x0) { R[rt] = R[rs]; }
            // add
            else if (op == 0x1) { R[rt] += R[rs]; };
        }
        if (op == 0xe || op == 0xf) {
            int addr = inst & 0xf;
            // load
            if (op == 0xe) { R[0] = M[addr]; }
            // store
            else if (op == 0xf) { M[addr] = R[0]; }
        }
    }
    ```

-   Now this piece of code is a lot more cleaner and more readable, the instructions are written in hexadecimal, and the register's index are presented with their actual name.
-   But if there are more instructions being added to the os, it would grow in a horrible speed. So why not improve it this way:

    ```c
    typedef union inst {
        struct { u8 rs: 2, rt: 2, op: 4;} rtype;
        struct { u8 addr: 4,      op: 4;} mtype;
    } inst_t;
    #define RTYPE(i) u8 rt = (i)->rtype.rt, rs = (i)->rtype.rs;
    #define MTYPE(i) u8 addr = (i)->mtype.addr;
    #define RA ...

    void idex() {
        inst_t *cur = (inst_t *)&M[pc]; // current instruction
        switch (cur->rtype.op) {
            case 0b0000: { RTYPE(cur); R[rt] = R[rs]  ; pc++; break; }
            case 0b0001: { RTYPE(cur); R[rt] += R[rs] ; pc++; break; }
            case 0b1110: { MTYPE(cur); R[RA] = M[addr]; pc++; break; }
            case 0b1111: { MTYPE(cur); R[addr] = R[RA]; pc++; break; }
            default: panic("invalid instruction at PC = %x", pc);
        }
    }
    ```

    -   NB: for syntax of the code defining `inst`, please refer to [Bit Fields in C](https://www.geeksforgeeks.org/bit-fields-c/).

-   Now this is a really elegant and beautiful way to handle the instructions.

---

-   Though the way we handle instructions above is rather elegant and beautiful, the way used in the current version of PA is still quite different:

    `decode.h` : _line 105_

    ```c
    // --- pattern matching wrappers for decode ---
    #define INSTPAT(pattern, ...)                                                  \
        do {                                                                       \
            uint64_t key, mask, shift;                                             \
            pattern_decode(pattern, STRLEN(pattern), &key, &mask, &shift);         \
            if ((((uint64_t)INSTPAT_INST(s) >> shift) & mask) == key) {            \
                INSTPAT_MATCH(s, ##__VA_ARGS__);                                   \
                goto *(__instpat_end);                                             \
            }                                                                      \
        } while (0)
    ```

    `inst.c` : _line 75_

    ```c
    static int decode_exec(Decode *s) {
        int rd = 0;
        word_t src1 = 0, src2 = 0, imm = 0;
        s->dnpc = s->snpc;

    #define INSTPAT_INST(s) ((s)->isa.inst.val)
    #define INSTPAT_MATCH(s, name, type, ... /* execute body */)                   \
        {                                                                          \
            decode_operand(s, &rd, &src1, &src2, &imm, concat(TYPE_, type));       \
            __VA_ARGS__;                                                           \
        }

        INSTPAT_START();
        INSTPAT("??????? ????? ????? ??? ????? 00101 11", auipc, U,
                R(rd) = s->pc + imm);
        INSTPAT("??????? ????? ????? 100 ????? 00000 11", lbu, I,
                R(rd) = Mr(src1 + imm, 1));
        INSTPAT("??????? ????? ????? 000 ????? 01000 11", sb, S,
                Mw(src1 + imm, 1, src2));

        INSTPAT("0000000 00001 00000 000 00000 11100 11", ebreak, N,
                NEMUTRAP(s->pc, R(10))); // R(10) is $a0
        INSTPAT("??????? ????? ????? ??? ????? ????? ??", inv, N, INV(s->pc));
        INSTPAT_END();

        R(0) = 0; // reset $zero to 0

        return 0;
    }
    ```

---

## End of the Second Day.
