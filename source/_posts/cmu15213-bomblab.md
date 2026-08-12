---
title: CMU 15-213 Bomblab
date: 2026-07-21 09:17:10
categories: 笔记
tags:
    - cmu15213
    - c
    - asm
excerpt: Fun puzzles in bomblab
---

## Bomblab

- This is the second lab of course CMU 15-213. I have to say that, unlike datalab, this lab is fairly 'logical' and fun to play with. The idea of looking into assembly code to get a fundamental understanding of how programs actually got compiled into machine level language is absolutely effective and genius in teaching.

- **Make sure you've solved the puzzles by yourself before checking my solutions!**

- If you find any mistakes in this blog, you're most welcome to leave a comment on it.

## Records

- Disassemble the executable file into assembly code by:
    ```bash
    objdump -D bomb > bomb.s
    ```

### Phase 1

- Obviously we gotta look into the assembly code of `phase_1`:

    ```asm
    0000000000400ee0 <phase_1>:
    400ee0:	48 83 ec 08          	sub    $0x8,%rsp
    400ee4:	be 00 24 40 00       	mov    $0x402400,%esi
    400ee9:	e8 4a 04 00 00       	call   401338 <strings_not_equal>
    400eee:	85 c0                	test   %eax,%eax
    400ef0:	74 05                	je     400ef7 <phase_1+0x17>
    400ef2:	e8 43 05 00 00       	call   40143a <explode_bomb>
    400ef7:	48 83 c4 08          	add    $0x8,%rsp
    400efb:	c3                   	ret
    ```

- Even more obvious we should take a look into function `<strings_not_equal>`:

    ```asm
    0000000000401338 <strings_not_equal>:
    401338:	41 54                	push   %r12
    40133a:	55                   	push   %rbp
    40133b:	53                   	push   %rbx
    40133c:	48 89 fb             	mov    %rdi,%rbx
    40133f:	48 89 f5             	mov    %rsi,%rbp
    401342:	e8 d4 ff ff ff       	call   40131b <string_length>
    401347:	41 89 c4             	mov    %eax,%r12d
    40134a:	48 89 ef             	mov    %rbp,%rdi
    40134d:	e8 c9 ff ff ff       	call   40131b <string_length>
    401352:	ba 01 00 00 00       	mov    $0x1,%edx
    401357:	41 39 c4             	cmp    %eax,%r12d
    40135a:	75 3f                	jne    40139b <strings_not_equal+0x63>
    40135c:	0f b6 03             	movzbl (%rbx),%eax
    40135f:	84 c0                	test   %al,%al
    401361:	74 25                	je     401388 <strings_not_equal+0x50>
    401363:	3a 45 00             	cmp    0x0(%rbp),%al
    401366:	74 0a                	je     401372 <strings_not_equal+0x3a>
    401368:	eb 25                	jmp    40138f <strings_not_equal+0x57>
    40136a:	3a 45 00             	cmp    0x0(%rbp),%al
    40136d:	0f 1f 00             	nopl   (%rax)
    401370:	75 24                	jne    401396 <strings_not_equal+0x5e>
    401372:	48 83 c3 01          	add    $0x1,%rbx
    401376:	48 83 c5 01          	add    $0x1,%rbp
    40137a:	0f b6 03             	movzbl (%rbx),%eax
    40137d:	84 c0                	test   %al,%al
    40137f:	75 e9                	jne    40136a <strings_not_equal+0x32>
    401381:	ba 00 00 00 00       	mov    $0x0,%edx
    401386:	eb 13                	jmp    40139b <strings_not_equal+0x63>
    401388:	ba 00 00 00 00       	mov    $0x0,%edx
    40138d:	eb 0c                	jmp    40139b <strings_not_equal+0x63>
    40138f:	ba 01 00 00 00       	mov    $0x1,%edx
    401394:	eb 05                	jmp    40139b <strings_not_equal+0x63>
    401396:	ba 01 00 00 00       	mov    $0x1,%edx
    40139b:	89 d0                	mov    %edx,%eax
    40139d:	5b                   	pop    %rbx
    40139e:	5d                   	pop    %rbp
    40139f:	41 5c                	pop    %r12
    4013a1:	c3                   	ret
    ```

- From the code we can see that the funcion first compares the length of the two strings and then compares each character.

- If we type in `hello` as a 'test' input for this phase, we can find out that it is the first call for `string_length` that calculates the length of our input string, for `%eax` is set to `5`(as the length of `hello`) after it returns. (With the help of gdb)
    - Then this `5` is moved into `%r12` at:

        ```asm
        401347:	41 89 c4             	mov    %eax,%r12d
        ```

    - Then compared with the second length calculated at:

        ```asm
        401357:	41 39 c4             	cmp    %eax,%r12d
        40135a:	75 3f                	jne    40139b <strings_not_equal+0x63>
        ```

- So the idea is simple: just step into the second `string_length` and see what is at the memory address stored in the register corresponding to the first argument of a function call, which is `%rdi`:

    _Step into the second \<string_length\>_

    ```gdb
    (gdb) print /x $rdi
    $1 = 0x402400
    (gdb) print (char *) 0x402400
    $2 = 0x402400 "Border relations with Canada have never been better."
    ```

- And we're done for phase 1.

### Phase 2

- As usual, assembly:

    ```asm
    0000000000400efc <phase_2>:
        400efc:	55                   	push   %rbp
        400efd:	53                   	push   %rbx
        400efe:	48 83 ec 28          	sub    $0x28,%rsp
        400f02:	48 89 e6             	mov    %rsp,%rsi
        400f05:	e8 52 05 00 00       	call   40145c <read_six_numbers>
        400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)
        400f0e:	74 20                	je     400f30 <phase_2+0x34>
        400f10:	e8 25 05 00 00       	call   40143a <explode_bomb>
        400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
        400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax
        400f1a:	01 c0                	add    %eax,%eax
        400f1c:	39 03                	cmp    %eax,(%rbx)
        400f1e:	74 05                	je     400f25 <phase_2+0x29>
        400f20:	e8 15 05 00 00       	call   40143a <explode_bomb>
        400f25:	48 83 c3 04          	add    $0x4,%rbx
        400f29:	48 39 eb             	cmp    %rbp,%rbx
        400f2c:	75 e9                	jne    400f17 <phase_2+0x1b>
        400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40>
        400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx
        400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp
        400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
        400f3c:	48 83 c4 28          	add    $0x28,%rsp
        400f40:	5b                   	pop    %rbx
        400f41:	5d                   	pop    %rbp
        400f42:	c3                   	ret
    ```

- Immediately we see this `<read_six_numbers>`, so `1 2 3 4 5 6` is an ideal test string we should put in here.

- Assembly of `read_six_numbers`:

    ```asm
    000000000040145c <read_six_numbers>:
        40145c:	48 83 ec 18          	sub    $0x18,%rsp
        401460:	48 89 f2             	mov    %rsi,%rdx
        401463:	48 8d 4e 04          	lea    0x4(%rsi),%rcx
        401467:	48 8d 46 14          	lea    0x14(%rsi),%rax
        40146b:	48 89 44 24 08       	mov    %rax,0x8(%rsp)
        401470:	48 8d 46 10          	lea    0x10(%rsi),%rax
        401474:	48 89 04 24          	mov    %rax,(%rsp)
        401478:	4c 8d 4e 0c          	lea    0xc(%rsi),%r9
        40147c:	4c 8d 46 08          	lea    0x8(%rsi),%r8
        401480:	be c3 25 40 00       	mov    $0x4025c3,%esi
        401485:	b8 00 00 00 00       	mov    $0x0,%eax
        40148a:	e8 61 f7 ff ff       	call   400bf0 <__isoc99_sscanf@plt>
        40148f:	83 f8 05             	cmp    $0x5,%eax
        401492:	7f 05                	jg     401499 <read_six_numbers+0x3d>
        401494:	e8 a1 ff ff ff       	call   40143a <explode_bomb>
        401499:	48 83 c4 18          	add    $0x18,%rsp
        40149d:	c3                   	ret
    ```

    - We can see from the assembly code that this function does not explicitly use anything stored in `%rdi` while manipulating `%rsi` a lot, which is pretty strange at first glance.
    - It turns out that the value stored inside `%rdi` is implicitly passed to `sscanf` as the address to hold the user input.
    - We can trace back to `main` to see where `%rdi` is set after `read_line`:
        ```asm
        # ...Phase 1
        400e3f:	e8 80 07 00 00       	call   4015c4 <phase_defused>
        400e44:	bf a8 23 40 00       	mov    $0x4023a8,%edi
        400e49:	e8 c2 fc ff ff       	call   400b10 <puts@plt>
        400e4e:	e8 4b 06 00 00       	call   40149e <read_line>
        400e53:	48 89 c7             	mov    %rax,%rdi
        400e56:	e8 a1 00 00 00       	call   400efc <phase_2>
        400e5b:	e8 64 07 00 00       	call   4015c4 <phase_defused>
        # Phase 3...
        ```
    - It is set as the pointer pointing towards the string returned by `read_line`.

- There seems to be a lot going on before invoking `sscanf`, but it's mainly preparing space for user input on the stack.

- In short:
    - `%rdi` passed all along to `sscanf` as input string.
    - `%rsi` holds the pointer pointing to the address of the buffer which is to store the six numbers parsed by `sscanf`.
    - `%rdx`, `%rcx`, `%r8`, `%r9` holding the first four addresses of each of the target buffer block to hold the parsed numbers, while the rest 2 addresses are placed on the stack.
    - `sscanf` then set the values stored in those addresses to the six input numbers.
    - After `sscanf` returns, compare the return value with `0x5`. If the return value, which represents the number of the values parsed, is less or equal to 5, then explode the bomb.
    - Free the stack frame and return.

{% callout type="info" icon="fa-solid fa-circle-info" title="Note:" %}

```asm
    401467:	48 8d 46 14          	lea    0x14(%rsi),%rax
    40146b:	48 89 44 24 08       	mov    %rax,0x8(%rsp)
    401470:	48 8d 46 10          	lea    0x10(%rsi),%rax
    401474:	48 89 04 24          	mov    %rax,(%rsp)
```

- This is where the last two addresses of the target buffer are placed on the stack, and passed to `sscanf`.
  {% endcallout %}

- After reading six numbers from user input, first check whether the first number is `0x1`:

    ```asm
    400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)
    400f0e:	74 20                	je     400f30 <phase_2+0x34>
    ```

- Then there is a loop determining whether the input numbers are of an expected pattern:

    ```asm
    400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax
    400f1a:	01 c0                	add    %eax,%eax
    400f1c:	39 03                	cmp    %eax,(%rbx)
    400f1e:	74 05                	je     400f25 <phase_2+0x29>
    400f20:	e8 15 05 00 00       	call   40143a <explode_bomb>
    400f25:	48 83 c3 04          	add    $0x4,%rbx
    400f29:	48 39 eb             	cmp    %rbp,%rbx
    400f2c:	75 e9                	jne    400f17 <phase_2+0x1b>
    400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40>
    400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx
    400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp
    400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
    ```

- Can be written as pseudocode like:

    ```c
    int rax;
    for(int rbx = 0x7fffffffd3d4; rbx != 0x7fffffffd3e8; rbx += 4) {
        rax = *(rbx - 4);
        rax += rax;
        if (rax != *rbx) explode_bomb();
    }
    ```

- To this stage, the answer can not be more obvious.

### Phase 3

- Assembly:

    ```asm
    0000000000400f43 <phase_3>:
        400f43:	48 83 ec 18          	sub    $0x18,%rsp
        400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
        400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
        400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi
        400f56:	b8 00 00 00 00       	mov    $0x0,%eax
        400f5b:	e8 90 fc ff ff       	call   400bf0 <__isoc99_sscanf@plt>
        400f60:	83 f8 01             	cmp    $0x1,%eax
        400f63:	7f 05                	jg     400f6a <phase_3+0x27>
        400f65:	e8 d0 04 00 00       	call   40143a <explode_bomb>
        400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)
        400f6f:	77 3c                	ja     400fad <phase_3+0x6a>
        400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax
        400f75:	ff 24 c5 70 24 40 00 	jmp    *0x402470(,%rax,8)
        400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax
        400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
        400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax
        400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
        400f8a:	b8 00 01 00 00       	mov    $0x100,%eax
        400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
        400f91:	b8 85 01 00 00       	mov    $0x185,%eax
        400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
        400f98:	b8 ce 00 00 00       	mov    $0xce,%eax
        400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
        400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax
        400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
        400fa6:	b8 47 01 00 00       	mov    $0x147,%eax
        400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
        400fad:	e8 88 04 00 00       	call   40143a <explode_bomb>
        400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
        400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
        400fb9:	b8 37 01 00 00       	mov    $0x137,%eax
        400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax
        400fc2:	74 05                	je     400fc9 <phase_3+0x86>
        400fc4:	e8 71 04 00 00       	call   40143a <explode_bomb>
        400fc9:	48 83 c4 18          	add    $0x18,%rsp
        400fcd:	c3                   	ret
    ```

- From the judgment towards the return value of `sscanf`:

    ```asm
    400f5b:	e8 90 fc ff ff       	call   400bf0 <__isoc99_sscanf@plt>
    400f60:	83 f8 01             	cmp    $0x1,%eax
    400f63:	7f 05                	jg     400f6a <phase_3+0x27>
    400f65:	e8 d0 04 00 00       	call   40143a <explode_bomb>
    ```

- We can see that the number of the expected user input should be more than 1, also:

    ```asm
    400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)
    400f6f:	77 3c                	ja     400fad <phase_3+0x6a>
    # ...
    400fad:	e8 88 04 00 00       	call   40143a <explode_bomb>
    ```

- For function `sscanf`, `%rdi` holds the address of the raw input, `%esi` holds the `format` string, thus `rdx` and `rcx` holds the two addresses where the program put the two parsed numbers to.

{% callout type="info" icon="fa-solid fa-circle-info" title="Note:" %}

- The reason why I know the wanted inputs are numbers is that I've tried them in cli.
  {% endcallout %}

- So, the first input should be less or equal to `0x7`(also non-negative for `ja` reads unsigned values comparison result). As a result `1 2` should be an ideal test input.

{% callout type="info" icon="fa-solid fa-circle-info" title="Note:" %}

- Like phase 2, `%rdi` is passed from `main` function, pointing to the address of user input string.
  {% endcallout %}

- Then the 'bomb' put the second input into `%eax` and jump to somewhere in a jump table:

    ```asm
    400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax
    400f75:	ff 24 c5 70 24 40 00 	jmp    *0x402470(,%rax,8)
    ```

- The syntax here in the second line of code means:
    - Jump to address `0x402470 + 0 + (%rax) * 0x8`

- So according to our test input, the value at address stored in `%rax` should be `0x1`, Thus the target address of this `jmp` should be located at `0x402470 + 0 + 1 * 0x8 = 0x402478`, which is:

    ```gdb
    (gdb) x/wx 0x402478
    0x402478:   0x00400fb9
    ```

    - Which is:

        ```asm
        400fb9:	b8 37 01 00 00       	mov    $0x137,%eax
        400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax
        400fc2:	74 05                	je     400fc9 <phase_3+0x86>
        400fc4:	e8 71 04 00 00       	call   40143a <explode_bomb>
        400fc9:	48 83 c4 18          	add    $0x18,%rsp
        400fcd:	c3                   	ret
        ```

    - Which means the correct second input corresponding to `0x1` as the first input is `0x137 = 311`.

- We can therefore reverse the whole jump table:

    ```c
    int expected;

    switch (first_input) {
        case 0: expected = 0xcf;  break;
        case 1: expected = 0x137; break;
        case 2: expected = 0x2c3; break;
        case 3: expected = 0x100; break;
        case 4: expected = 0x185; break;
        case 5: expected = 0xce;  break;
        case 6: expected = 0x2aa; break;
        case 7: expected = 0x147; break;

        default:
            explode_bomb();
    }

    if (second_input != expected)
        explode_bomb();
    ```

### Phase 4

- Assembly:

    ```asm
    000000000040100c <phase_4>:
        40100c:	48 83 ec 18          	sub    $0x18,%rsp
        401010:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
        401015:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
        40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi
        40101f:	b8 00 00 00 00       	mov    $0x0,%eax
        401024:	e8 c7 fb ff ff       	call   400bf0 <__isoc99_sscanf@plt>
        401029:	83 f8 02             	cmp    $0x2,%eax
        40102c:	75 07                	jne    401035 <phase_4+0x29>
        40102e:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp)
        401033:	76 05                	jbe    40103a <phase_4+0x2e>
        401035:	e8 00 04 00 00       	call   40143a <explode_bomb>
        40103a:	ba 0e 00 00 00       	mov    $0xe,%edx
        40103f:	be 00 00 00 00       	mov    $0x0,%esi
        401044:	8b 7c 24 08          	mov    0x8(%rsp),%edi
        401048:	e8 81 ff ff ff       	call   400fce <func4>
        40104d:	85 c0                	test   %eax,%eax
        40104f:	75 07                	jne    401058 <phase_4+0x4c>
        401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp)
        401056:	74 05                	je     40105d <phase_4+0x51>
        401058:	e8 dd 03 00 00       	call   40143a <explode_bomb>
        40105d:	48 83 c4 18          	add    $0x18,%rsp
        401061:	c3                   	ret
    ```

- Easy part:
    - `sscanf` user input, write two numbers at address `%rsp + 0x8` and `%rsp + 0xc`.
    - The first input should be non-negative and less or equal to `0xe`.
    - Set `%rdx` to `0xe`. Set `%rsi` to `0x0`. Set `%rdi` to the first user input.
    - call `<func4>`.

- Let's take a look into `func4`:

    ```asm
    0000000000400fce <func4>:
        400fce:	48 83 ec 08          	sub    $0x8,%rsp
        400fd2:	89 d0                	mov    %edx,%eax
        400fd4:	29 f0                	sub    %esi,%eax
        400fd6:	89 c1                	mov    %eax,%ecx
        400fd8:	c1 e9 1f             	shr    $0x1f,%ecx
        400fdb:	01 c8                	add    %ecx,%eax
        400fdd:	d1 f8                	sar    $1,%eax
        400fdf:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx
        400fe2:	39 f9                	cmp    %edi,%ecx
        400fe4:	7e 0c                	jle    400ff2 <func4+0x24>
        400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx
        400fe9:	e8 e0 ff ff ff       	call   400fce <func4>
        400fee:	01 c0                	add    %eax,%eax
        400ff0:	eb 15                	jmp    401007 <func4+0x39>
        400ff2:	b8 00 00 00 00       	mov    $0x0,%eax
        400ff7:	39 f9                	cmp    %edi,%ecx
        400ff9:	7d 0c                	jge    401007 <func4+0x39>
        400ffb:	8d 71 01             	lea    0x1(%rcx),%esi
        400ffe:	e8 cb ff ff ff       	call   400fce <func4>
        401003:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
        401007:	48 83 c4 08          	add    $0x8,%rsp
        40100b:	c3                   	ret
    ```

- We can see that the calculation for `%ecx` is actually fixed, which means whatever the input numbers are, `%ecx` will be `0x7` during the two comparisons:

    ```asm
    # ...
    400fe2:	39 f9                	cmp    %edi,%ecx
    400fe4:	7e 0c                	jle    400ff2 <func4+0x24>
    400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx
    400fe9:	e8 e0 ff ff ff       	call   400fce <func4>
    # ...
    400ff7:	39 f9                	cmp    %edi,%ecx
    400ff9:	7d 0c                	jge    401007 <func4+0x39>
    400ffb:	8d 71 01             	lea    0x1(%rcx),%esi
    400ffe:	e8 cb ff ff ff       	call   400fce <func4>
    # ...
    ```

- Also we notice that each time the function returns from a recursion, it doubles the value in `%rax` and increments it by 1.

- As the assembly code we can see after `func4` returns:

    ```asm
    # ...
    40104d:	85 c0                	test   %eax,%eax
    40104f:	75 07                	jne    401058 <phase_4+0x4c>
    # ...
    401058:	e8 dd 03 00 00       	call   40143a <explode_bomb>
    # ...
    ```

- We definitely don't want `%rax` to be non-zero. As a result, we should pass a value to `func4` that both `jle` and `jge` fires, which means `%edi` should equal to `%ecx` at the moment of comparison.

    {% callout type="warning" icon="fa-solid fa-circle-info" title="Note:" %}
    - Turns out `func4` is doing something similiar to a binary search, while the return value is not the offset/index of the target value.

    - So as long as the function goes into this branch:

        ```asm
        400fe4:	7e 0c                	jle    400ff2 <func4+0x24>
        400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx
        400fe9:	e8 e0 ff ff ff       	call   400fce <func4>
        400fee:	01 c0                	add    %eax,%eax
        400ff0:	eb 15                	jmp    401007 <func4+0x39>
        ```

    - And get a return value 0 inside the recurrsion, we can keep `%rax` as 0.

    - Really weird a binary search function does not return an index, hmm.

    {% endcallout %}

- Since we've already know that `%ecx` will be a fixed `0x7` on the first call, the value of `%edi` is now revealed.

- The second input is an easy zero:

    ```asm
    # ...
    401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp)
    401056:	74 05                	je     40105d <phase_4+0x51>
    401058:	e8 dd 03 00 00       	call   40143a <explode_bomb>
    40105d:	48 83 c4 18          	add    $0x18,%rsp
    401061:	c3                   	ret
    ```

### Phase 5

- Assembly:

    ```asm
    0000000000401062 <phase_5>:
        401062:	53                   	push   %rbx
        401063:	48 83 ec 20          	sub    $0x20,%rsp
        401067:	48 89 fb             	mov    %rdi,%rbx
        40106a:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
        401071:	00 00
        401073:	48 89 44 24 18       	mov    %rax,0x18(%rsp)
        401078:	31 c0                	xor    %eax,%eax
        40107a:	e8 9c 02 00 00       	call   40131b <string_length>
        40107f:	83 f8 06             	cmp    $0x6,%eax
        401082:	74 4e                	je     4010d2 <phase_5+0x70>
        401084:	e8 b1 03 00 00       	call   40143a <explode_bomb>
        401089:	eb 47                	jmp    4010d2 <phase_5+0x70>
        40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx
        40108f:	88 0c 24             	mov    %cl,(%rsp)
        401092:	48 8b 14 24          	mov    (%rsp),%rdx
        401096:	83 e2 0f             	and    $0xf,%edx
        401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx
        4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)
        4010a4:	48 83 c0 01          	add    $0x1,%rax
        4010a8:	48 83 f8 06          	cmp    $0x6,%rax
        4010ac:	75 dd                	jne    40108b <phase_5+0x29>
        4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)
        4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi
        4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi
        4010bd:	e8 76 02 00 00       	call   401338 <strings_not_equal>
        4010c2:	85 c0                	test   %eax,%eax
        4010c4:	74 13                	je     4010d9 <phase_5+0x77>
        4010c6:	e8 6f 03 00 00       	call   40143a <explode_bomb>
        4010cb:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
        4010d0:	eb 07                	jmp    4010d9 <phase_5+0x77>
        4010d2:	b8 00 00 00 00       	mov    $0x0,%eax
        4010d7:	eb b2                	jmp    40108b <phase_5+0x29>
        4010d9:	48 8b 44 24 18       	mov    0x18(%rsp),%rax
        4010de:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax
        4010e5:	00 00
        4010e7:	74 05                	je     4010ee <phase_5+0x8c>
        4010e9:	e8 42 fa ff ff       	call   400b30 <__stack_chk_fail@plt>
        4010ee:	48 83 c4 20          	add    $0x20,%rsp
        4010f2:	5b                   	pop    %rbx
        4010f3:	c3                   	ret
    ```

- Length of the input string should be 6:

    ```asm
    40107a:	e8 9c 02 00 00       	call   40131b <string_length>
    40107f:	83 f8 06             	cmp    $0x6,%eax
    401082:	74 4e                	je     4010d2 <phase_5+0x70>
    401084:	e8 b1 03 00 00       	call   40143a <explode_bomb>
    ```

- The core logic of this function is at here:

    ```asm
    40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx
    40108f:	88 0c 24             	mov    %cl,(%rsp)
    401092:	48 8b 14 24          	mov    (%rsp),%rdx
    401096:	83 e2 0f             	and    $0xf,%edx
    401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx
    4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)
    4010a4:	48 83 c0 01          	add    $0x1,%rax
    4010a8:	48 83 f8 06          	cmp    $0x6,%rax
    4010ac:	75 dd                	jne    40108b <phase_5+0x29>
    ```

- What this while-loop is doing is basically pick the latter 4 bits of each character of the input string, and add the number represented by that 4 bits to `0x4024b0` to get another byte.

- By the way, the target string wanted is `flyers`:

    ```asm
    4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi
    4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi
    4010bd:	e8 76 02 00 00       	call   401338 <strings_not_equal>
    ```

    ```gdb
    (gdb) print (char *) 0x40245e
    $1 = 0x40245e "flyers"
    ```

- So the way to get the target string is to seek the correct letters in the string located at `0x4024b0`, and prepare the characters with the corresponding last 4 bits.

- We can print out what exactly is stored at `0x4024b0`:

    ```gdb
    (gdb) x/16c 0x4024b0
    0x4024b0 <array.3449>:	109 'm'	97 'a'	100 'd'	117 'u'	105 'i'	101 'e'	114 'r'	115 's'
    0x4024b8 <array.3449+8>:	110 'n'	102 'f'	111 'o'	116 't'	118 'v'	98 'b'	121 'y'	108 'l'
    ```

    ```
    m a d u i e r s n f o t v b y l
    0 1 2 3 4 5 6 7 8 9 a b c d e f
    ```

- String `flyers` would require such serial of index: `9 f e 5 6 7`. The input should be a string, so we might take a look at the ASCII table:

    {% folding blue::ASCII %}

    | Dec | Hex | Binary   | Char |
    | --- | --- | -------- | ---- |
    | 69  | 45  | 01000101 | E    |
    | 70  | 46  | 01000110 | F    |
    | 71  | 47  | 01000111 | G    |
    | 72  | 48  | 01001000 | H    |
    | 73  | 49  | 01001001 | I    |
    | 74  | 4A  | 01001010 | J    |
    | 75  | 4B  | 01001011 | K    |
    | 76  | 4C  | 01001100 | L    |
    | 77  | 4D  | 01001101 | M    |
    | 78  | 4E  | 01001110 | N    |
    | 79  | 4F  | 01001111 | O    |
    | 80  | 50  | 01010000 | P    |
    | 81  | 51  | 01010001 | Q    |
    | 82  | 52  | 01010010 | R    |
    | 83  | 53  | 01010011 | S    |
    | 84  | 54  | 01010100 | T    |
    | 85  | 55  | 01010101 | U    |
    | 86  | 56  | 01010110 | V    |
    | 87  | 57  | 01010111 | W    |
    | 88  | 58  | 01011000 | X    |
    | 89  | 59  | 01011001 | Y    |
    | 90  | 5A  | 01011010 | Z    |
    | 91  | 5B  | 01011011 | [    |
    | 92  | 5C  | 01011100 | \    |
    | 93  | 5D  | 01011101 | ]    |
    | 94  | 5E  | 01011110 | ^    |
    | 95  | 5F  | 01011111 | \_   |
    | 96  | 60  | 01100000 | `    |
    | 97  | 61  | 01100001 | a    |
    | 98  | 62  | 01100010 | b    |
    | 99  | 63  | 01100011 | c    |
    | 100 | 64  | 01100100 | d    |
    | 101 | 65  | 01100101 | e    |
    | 102 | 66  | 01100110 | f    |
    | 103 | 67  | 01100111 | g    |
    | 104 | 68  | 01101000 | h    |
    | 105 | 69  | 01101001 | i    |
    | 106 | 6A  | 01101010 | j    |
    | 107 | 6B  | 01101011 | k    |
    | 108 | 6C  | 01101100 | l    |
    | 109 | 6D  | 01101101 | m    |
    | 110 | 6E  | 01101110 | n    |
    | 111 | 6F  | 01101111 | o    |
    | 112 | 70  | 01110000 | p    |
    | 113 | 71  | 01110001 | q    |
    | 114 | 72  | 01110010 | r    |
    | 115 | 73  | 01110011 | s    |
    | 116 | 74  | 01110100 | t    |
    | 117 | 75  | 01110101 | u    |
    | 118 | 76  | 01110110 | v    |
    | 119 | 77  | 01110111 | w    |
    | 120 | 78  | 01111000 | x    |
    | 121 | 79  | 01111001 | y    |
    | 122 | 7A  | 01111010 | z    |
    | 123 | 7B  | 01111011 | {    |
    | 124 | 7C  | 01111100 | \|   |
    | 125 | 7D  | 01111101 | }    |
    | 126 | 7E  | 01111110 | ~    |
    | 127 | 7F  | 01111111 | DEL  |

    {% endfolding %}

- To get an input with `9 f e 5 6 7` as the four last bits of each character:
    - `ionefg`
    - `Y_^UVW`
    - ...

### Phase 6

- Assembly:

    ```asm
    00000000004010f4 <phase_6>:
        4010f4:	41 56                	push   %r14
        4010f6:	41 55                	push   %r13
        4010f8:	41 54                	push   %r12
        4010fa:	55                   	push   %rbp
        4010fb:	53                   	push   %rbx
        4010fc:	48 83 ec 50          	sub    $0x50,%rsp
        401100:	49 89 e5             	mov    %rsp,%r13
        401103:	48 89 e6             	mov    %rsp,%rsi
        401106:	e8 51 03 00 00       	call   40145c <read_six_numbers>
        40110b:	49 89 e6             	mov    %rsp,%r14
        40110e:	41 bc 00 00 00 00    	mov    $0x0,%r12d
        401114:	4c 89 ed             	mov    %r13,%rbp
        401117:	41 8b 45 00          	mov    0x0(%r13),%eax
        40111b:	83 e8 01             	sub    $0x1,%eax
        40111e:	83 f8 05             	cmp    $0x5,%eax
        401121:	76 05                	jbe    401128 <phase_6+0x34>
        401123:	e8 12 03 00 00       	call   40143a <explode_bomb>
        401128:	41 83 c4 01          	add    $0x1,%r12d
        40112c:	41 83 fc 06          	cmp    $0x6,%r12d
        401130:	74 21                	je     401153 <phase_6+0x5f>
        401132:	44 89 e3             	mov    %r12d,%ebx
        401135:	48 63 c3             	movslq %ebx,%rax
        401138:	8b 04 84             	mov    (%rsp,%rax,4),%eax
        40113b:	39 45 00             	cmp    %eax,0x0(%rbp)
        40113e:	75 05                	jne    401145 <phase_6+0x51>
        401140:	e8 f5 02 00 00       	call   40143a <explode_bomb>
        401145:	83 c3 01             	add    $0x1,%ebx
        401148:	83 fb 05             	cmp    $0x5,%ebx
        40114b:	7e e8                	jle    401135 <phase_6+0x41>
        40114d:	49 83 c5 04          	add    $0x4,%r13
        401151:	eb c1                	jmp    401114 <phase_6+0x20>
        401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi
        401158:	4c 89 f0             	mov    %r14,%rax
        40115b:	b9 07 00 00 00       	mov    $0x7,%ecx
        401160:	89 ca                	mov    %ecx,%edx
        401162:	2b 10                	sub    (%rax),%edx
        401164:	89 10                	mov    %edx,(%rax)
        401166:	48 83 c0 04          	add    $0x4,%rax
        40116a:	48 39 f0             	cmp    %rsi,%rax
        40116d:	75 f1                	jne    401160 <phase_6+0x6c>
        40116f:	be 00 00 00 00       	mov    $0x0,%esi
        401174:	eb 21                	jmp    401197 <phase_6+0xa3>
        401176:	48 8b 52 08          	mov    0x8(%rdx),%rdx
        40117a:	83 c0 01             	add    $0x1,%eax
        40117d:	39 c8                	cmp    %ecx,%eax
        40117f:	75 f5                	jne    401176 <phase_6+0x82>
        401181:	eb 05                	jmp    401188 <phase_6+0x94>
        401183:	ba d0 32 60 00       	mov    $0x6032d0,%edx
        401188:	48 89 54 74 20       	mov    %rdx,0x20(%rsp,%rsi,2)
        40118d:	48 83 c6 04          	add    $0x4,%rsi
        401191:	48 83 fe 18          	cmp    $0x18,%rsi
        401195:	74 14                	je     4011ab <phase_6+0xb7>
        401197:	8b 0c 34             	mov    (%rsp,%rsi,1),%ecx
        40119a:	83 f9 01             	cmp    $0x1,%ecx
        40119d:	7e e4                	jle    401183 <phase_6+0x8f>
        40119f:	b8 01 00 00 00       	mov    $0x1,%eax
        4011a4:	ba d0 32 60 00       	mov    $0x6032d0,%edx
        4011a9:	eb cb                	jmp    401176 <phase_6+0x82>
        4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx
        4011b0:	48 8d 44 24 28       	lea    0x28(%rsp),%rax
        4011b5:	48 8d 74 24 50       	lea    0x50(%rsp),%rsi
        4011ba:	48 89 d9             	mov    %rbx,%rcx
        4011bd:	48 8b 10             	mov    (%rax),%rdx
        4011c0:	48 89 51 08          	mov    %rdx,0x8(%rcx)
        4011c4:	48 83 c0 08          	add    $0x8,%rax
        4011c8:	48 39 f0             	cmp    %rsi,%rax
        4011cb:	74 05                	je     4011d2 <phase_6+0xde>
        4011cd:	48 89 d1             	mov    %rdx,%rcx
        4011d0:	eb eb                	jmp    4011bd <phase_6+0xc9>
        4011d2:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx)
        4011d9:	00
        4011da:	bd 05 00 00 00       	mov    $0x5,%ebp
        4011df:	48 8b 43 08          	mov    0x8(%rbx),%rax
        4011e3:	8b 00                	mov    (%rax),%eax
        4011e5:	39 03                	cmp    %eax,(%rbx)
        4011e7:	7d 05                	jge    4011ee <phase_6+0xfa>
        4011e9:	e8 4c 02 00 00       	call   40143a <explode_bomb>
        4011ee:	48 8b 5b 08          	mov    0x8(%rbx),%rbx
        4011f2:	83 ed 01             	sub    $0x1,%ebp
        4011f5:	75 e8                	jne    4011df <phase_6+0xeb>
        4011f7:	48 83 c4 50          	add    $0x50,%rsp
        4011fb:	5b                   	pop    %rbx
        4011fc:	5d                   	pop    %rbp
        4011fd:	41 5c                	pop    %r12
        4011ff:	41 5d                	pop    %r13
        401201:	41 5e                	pop    %r14
        401203:	c3                   	ret
    ```

- Holy, that is quite a large piece of code.

- Well, its core logic can be cut into several parts.

#### Checking Input

- The input is parsed by `read_six_numbers` as the one explained before.

- Then the function perform a loop in order to check whether any input is less than 6 (and also larger than 0), and to ensure that each number is unique to others:

    ```asm
    40110e:	41 bc 00 00 00 00    	mov    $0x0,%r12d
    401114:	4c 89 ed             	mov    %r13,%rbp
    401117:	41 8b 45 00          	mov    0x0(%r13),%eax
    40111b:	83 e8 01             	sub    $0x1,%eax
    40111e:	83 f8 05             	cmp    $0x5,%eax
    401121:	76 05                	jbe    401128 <phase_6+0x34>
    401123:	e8 12 03 00 00       	call   40143a <explode_bomb>
    401128:	41 83 c4 01          	add    $0x1,%r12d
    40112c:	41 83 fc 06          	cmp    $0x6,%r12d
    401130:	74 21                	je     401153 <phase_6+0x5f>
    401132:	44 89 e3             	mov    %r12d,%ebx
    401135:	48 63 c3             	movslq %ebx,%rax
    401138:	8b 04 84             	mov    (%rsp,%rax,4),%eax
    40113b:	39 45 00             	cmp    %eax,0x0(%rbp)
    40113e:	75 05                	jne    401145 <phase_6+0x51>
    401140:	e8 f5 02 00 00       	call   40143a <explode_bomb>
    401145:	83 c3 01             	add    $0x1,%ebx
    401148:	83 fb 05             	cmp    $0x5,%ebx
    40114b:	7e e8                	jle    401135 <phase_6+0x41>
    40114d:	49 83 c5 04          	add    $0x4,%r13
    401151:	eb c1                	jmp    401114 <phase_6+0x20>
    ```

    - Pseudocode:

        ```c
        // r13 points to the current input number
        int *r13 = rsp;
        int r12 = 0;

        while (1) {
            int *rbp = r13;
            int rax = *r13;
            rax -= 1;

            if (rax > 0x5) explode_bomb();

            r12 += 1;
            if (r12 == 6) break;

            for (int rbx = r12; rbx <= 0x5; rbx ++) {
                rax = rbx;
                rax = nums[rax];

                if (rax == *rbp) explode_bomb();
            }

            r13 += 4;
        }
        ```

#### Subtract Each Number From 7

- Then the function subtract each of the six input number from 7:

    ```asm
    401153:	48 8d 74 24 18       	lea    0x18(%rsp),%rsi
    401158:	4c 89 f0             	mov    %r14,%rax
    40115b:	b9 07 00 00 00       	mov    $0x7,%ecx
    401160:	89 ca                	mov    %ecx,%edx
    401162:	2b 10                	sub    (%rax),%edx
    401164:	89 10                	mov    %edx,(%rax)
    401166:	48 83 c0 04          	add    $0x4,%rax
    40116a:	48 39 f0             	cmp    %rsi,%rax
    40116d:	75 f1                	jne    401160 <phase_6+0x6c>
    ```

- Pseudocode:

    ```c
    int rcx = 7;

    for (int rax = 0; rax <= 5; rax++) {
        int rdx = rcx;
        rdx -= nums[rax];
        nums[rax] = rdx;
    }
    ```

#### Load Linked List

- After that, the function load six nodes of a linked list onto the stack:

    ```asm
    40116f:	be 00 00 00 00       	mov    $0x0,%esi
    401174:	eb 21                	jmp    401197 <phase_6+0xa3>
    401176:	48 8b 52 08          	mov    0x8(%rdx),%rdx
    40117a:	83 c0 01             	add    $0x1,%eax
    40117d:	39 c8                	cmp    %ecx,%eax
    40117f:	75 f5                	jne    401176 <phase_6+0x82>
    401181:	eb 05                	jmp    401188 <phase_6+0x94>
    401183:	ba d0 32 60 00       	mov    $0x6032d0,%edx
    401188:	48 89 54 74 20       	mov    %rdx,0x20(%rsp,%rsi,2)
    40118d:	48 83 c6 04          	add    $0x4,%rsi
    401191:	48 83 fe 18          	cmp    $0x18,%rsi
    401195:	74 14                	je     4011ab <phase_6+0xb7>
    401197:	8b 0c 34             	mov    (%rsp,%rsi,1),%ecx
    40119a:	83 f9 01             	cmp    $0x1,%ecx
    40119d:	7e e4                	jle    401183 <phase_6+0x8f>
    40119f:	b8 01 00 00 00       	mov    $0x1,%eax
    4011a4:	ba d0 32 60 00       	mov    $0x6032d0,%edx
    4011a9:	eb cb                	jmp    401176 <phase_6+0x82>
    ```

- We can see it from memory `0x6032d0`:

    ```gdb
    x/wx 0x6032d0
    0x6032d0 <node1>:	0x0000014c
    (gdb) x/wx 0x6032d0 + 0x4
    0x6032d4 <node1+4>:	0x00000001
    (gdb) x/wx 0x6032d0 + 0x8
    0x6032d8 <node1+8>:	0x006032e0
    (gdb) x/wx 0x6032e0
    0x6032e0 <node2>:	0x000000a8
    ```

- There are six nodes in the linked list, their values are as follow:

    ```plaintext
    node1: 332
    node2: 168
    node3: 924
    node4: 691
    node5: 477
    node6: 443
    ```

    _values are shown as decimal numbers_

- They are loaded on to the upper area of the stack:

    ```
          ┌──────────────┐
    +0x48 │ node_addr[5] │
          ├──────────────┤
          │ node_addr[4] │
          ├──────────────┤
          │ node_addr[3] │
          ├──────────────┤
          │ node_addr[2] │
          ├──────────────┤
          │ node_addr[1] │
          ├──────────────┤
    +0x20 │ node_addr[0] │
          ├──────────────┤
          │     ...      │
          ├──────────────┤
    +0x18 │ int arr[5]   │
          ├──────────────┤
          │ int arr[4]   │
          ├──────────────┤
          │ int arr[3]   │
          ├──────────────┤
          │ int arr[2]   │
          ├──────────────┤
          │ int arr[1]   │
          ├──────────────┤
    +0x00 │ int arr[0]   │  <---  %rsp
          └──────────────┘
    ```

- The order of the nodes is the same as user input subtracted from 7. For example if the user input `6 5 4 3 2 1`, then the nodes pointed by the address on the stack is `node1 node2 node3 node4 node5 node6`, upwards.

#### Reorder linked list

- Then the function would reorder the linked list, while the old order is a straight from 1 to 6.

    ```asm
    4011ab:	48 8b 5c 24 20       	mov    0x20(%rsp),%rbx
    4011b0:	48 8d 44 24 28       	lea    0x28(%rsp),%rax
    4011b5:	48 8d 74 24 50       	lea    0x50(%rsp),%rsi
    4011ba:	48 89 d9             	mov    %rbx,%rcx
    4011bd:	48 8b 10             	mov    (%rax),%rdx
    4011c0:	48 89 51 08          	mov    %rdx,0x8(%rcx)
    4011c4:	48 83 c0 08          	add    $0x8,%rax
    4011c8:	48 39 f0             	cmp    %rsi,%rax
    4011cb:	74 05                	je     4011d2 <phase_6+0xde>
    4011cd:	48 89 d1             	mov    %rdx,%rcx
    4011d0:	eb eb                	jmp    4011bd <phase_6+0xc9>
    4011d2:	48 c7 42 08 00 00 00 	movq   $0x0,0x8(%rdx)
    ```

- This would reorder the linked list by the order of the addresses on the stack. The nodes are now pointing upwards after the loop, from the perspective of the addresses on the stack.

#### Examine The Order

- The final loop would check whether the new order of the linked nodes are **descending** to their values.

    ```asm
    4011da:	bd 05 00 00 00       	mov    $0x5,%ebp
    4011df:	48 8b 43 08          	mov    0x8(%rbx),%rax
    4011e3:	8b 00                	mov    (%rax),%eax
    4011e5:	39 03                	cmp    %eax,(%rbx)
    4011e7:	7d 05                	jge    4011ee <phase_6+0xfa>
    4011e9:	e8 4c 02 00 00       	call   40143a <explode_bomb>
    4011ee:	48 8b 5b 08          	mov    0x8(%rbx),%rbx
    4011f2:	83 ed 01             	sub    $0x1,%ebp
    4011f5:	75 e8                	jne    4011df <phase_6+0xeb>
    ```

- A reminder of the linked list:

    ```plaintext
    node1: 332
    node2: 168
    node3: 924
    node4: 691
    node5: 477
    node6: 443
    ```

- Obviously the expected order is : `3 -> 4 -> 5 -> 6 -> 1 -> 2`.

#### Conclusion

- From these four parts we can see that the function is basically manipulating the user input, and then examine whether the order is as expected or not.

- The expected order(nodes & input):

    ```plaintext
    3 -> 4 -> 5 -> 6 -> 1 -> 2
    ```

- The order before reorder(nodes & input):

    ```plaintext
    3 -> 4 -> 5 -> 6  1 -> 2 ─┐
    │                         │
    └─────────────────────────┘
    ```

- The order before being subtracted from 7 (input):

    ```plaintext
    4 3 2 1 6 5
    ```

- Which is the answer.

---

_Phew!_

## End
