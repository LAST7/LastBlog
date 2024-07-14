---
title: PA PA1 - Monitor & Infrastructure
date: 2024-07-03 17:00:55
categories: 技术
tags:
    - c
excerpt: First Lab done in PA
---

## _strtok_

-   I used it to extract argument(substing) from a string.

    -   `sdb.c`: _line 137_

        ```c
        char *len_c = strtok(args, " ");
        char *addr_c = strtok(NULL, " ");
        ```

-   From the example above we can see that on the first time we call this method, the first parameter it acccepts is the string to be parsed, and the second parameter is the token as a 'separator'.
-   While, when the second time we use it for more arguments from the same string, we could use `NULL` as the first paramter. By this way, the method would look for the first string we passed to it.

## Using _memset_ to empty buffer

-   I used it to reset the `tokens` buffer, since it's not just a character array but a struct array.

    -   `expr.c`: _line 297_

        ```c
        // empty the buffer
        memset(tokens, 0, sizeof(tokens));
        nr_token = 0;
        ```

## Buffer overload

-   Sometimes `gen-expr.c` produces extremely long expressions. I had to set `MAX_TOKEN` to 256 to avoid assertion failure as much as possible, even then the expression could still exceed the limit.
-   I do not have a better way to solve this problem, but since the test could be done(just try a few more times with the `gen-expr` function or delete some unnecessary parentheses manually), I see no reason spending more time.

## Returning address of stack memory

-   When I was implementing the function `WP *new_wp(char *e)`, I initialized and returned the new watch point variable as the following way:

    ```c
    WP new = *free_;
    ...
    return &new;
    ```

-   The clangd lsp immediately warns me that:
    > **Address of stack memory associated with local variable 'new' returned**
-   Reason of this warning:
    -   **When declaring a variable this way inside a function, the variable is actually put on _stack_, which is a temporary memory area that is allocated when a function is called and _deallocated_ when the function returns.**
    -   **If the address of this _local variable_ `new` is returned, the function is basically returning a pointer to memory that will become invalid ASA the function returns.**
-
-   This could lead to a problem referred to as a **_dangling pointer_**, which is a very dangerous problem since it grant the program access to an unauthorized memory area, which could lead to undefined behavior, often resulting in crashes or data corruption.

-   So, the correct way of returning the wanted `WP` variable is to initialize it as a pointer and returns it:
    ```c
    WP *new = free_;
    ...
    return new;
    ```

## Including headfiles from different directory

-   The makefiles in this project seems too complicated for me, and I have no intention to read it for now.
-   So, I used an ugly method: relative path.

    -   `cpu-exec.c`: _line 21_

        ```c
        #include "../monitor/sdb/watchpoint.h"
        ```

## End of PA1

-   This task is actually quite difficult for me, though I've finished it.(Without the testing of OJ)
-   The most impressive feeling during this process is that 'it seems that the C programs I've written before are nothing but some toys to play with'. TBH, most of the codes in the framework provided by the course seemed unfamiliar and difficult for me, especially when I first glanced at the dense definitions of the macros used almost everywhere throughout the project.
-   Moreover, when reading the project manual, I could clearly feel that there were a lot of concepts that the author had persumed the reader to be aware of, which I had totally no idea what these concepts are.
-   Though these challenges made my journey uneasy, they also provided excitement since there're so many new things to learn and practice. Comparing to pure and boring learning of concepts and theories, it feels great to put the knowledge into practice right after learning it.

-   Hope I don't get toasted in PA2, let's go!
