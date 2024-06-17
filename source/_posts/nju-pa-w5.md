---
title: PA W5 - NEMU framework
date: 2024-06-17 16:07:49
categories: 技术
tags:
  - c
  - make
  - gdb
excerpt: Notes taken for class W5 of NJU PA
---

## Building Project: Using `Make`

-   When a project grows to a very large scale, even just using `gcc` from the command line a few times will become painful and cumbersome as there might be tens of commands to execute. Not to mention that the whole thing needs to be done again and again during development.
-   To resolve this dilemma, we can use a build system called `make`.

-   `make` uses `Makefile` to build the project. `Makefile` is a special file which contains declarative code to describe the dependent relationship between the build targets and ways to update them.
-   It can automatically detect the files which need to be updated, avoiding repeatitive build.

-   Tricks When Studying Behaviors of `Make` :

    -   `-n` : Dry-run, simulate the process of build without actually running it, and prints out all the commands.
    -   `-B` : Force `gcc` to update all of the make target.

-   Combined(transmit the output of `grep` to `nvim` by using pipeline `|`):
    ```bash
    make -nB \
        | grep -ve '^\(\#\|echo\|mkdir\)' \
        | nvim -
    ```

## Reading `Makefile` in Browser

-   Use this command to convert a `Makefile` into a `html` file:

    ```bash
    ### *Get a more readable version of this Makefile* by `make html` (requires python-markdown)
    html:
        cat Makefile | sed 's/^\([^#]\)/    \1/g' | markdown_py > Makefile.html
    ```

-   NB: command `markdown_py` is provided by package `python-markdwon`.

## A Simple `main` Function:

-   `main` function in nemu:

    ```c
    #include <common.h>

    void init_monitor(int, char *[]);
    void am_init_monitor();
    void engine_start();
    int is_exit_status_bad();

    int main(int argc, char *argv[]) {
        /* Initialize the monitor. */
    #ifdef CONFIG_TARGET_AM
        am_init_monitor();
    #else
        init_monitor(argc, argv);
    #endif

        /* Start engine. */
        engine_start();

        return is_exit_status_bad();
    }
    ```

-   We can see that this main function only serves as a director--it only calls other functions without executing any actual logic of the project.

## Some Code Explanation

### `getopt_long`

```c
static int parse_args(int argc, char *argv[]) {
    const struct option table[] = {
        // clang-format off
        {"batch"    , no_argument      , NULL, 'b'},
        {"log"      , required_argument, NULL, 'l'},
        {"diff"     , required_argument, NULL, 'd'},
        {"port"     , required_argument, NULL, 'p'},
        {"help"     , no_argument      , NULL, 'h'},
        {0          , 0                , NULL,  0 },
        // clang-format on
    };
    int o;
    while ((o = getopt_long(argc, argv, "-bhl:d:p:", table, NULL)) != -1) {
        switch (o) {
        case 'b':
            sdb_set_batch_mode();
            break;
        case 'p':
            sscanf(optarg, "%d", &difftest_port);
            break;
        case 'l':
            log_file = optarg;
            break;
        case 'd':
            diff_so_file = optarg;
            break;
        case 1:
            img_file = optarg;
            return 0;
        default:
            printf("Usage: %s [OPTION...] IMAGE [args]\n\n", argv[0]);
            printf("\t-b,--batch              run with batch mode\n");
            printf("\t-l,--log=FILE           output log to FILE\n");
            printf("\t-d,--diff=REF_SO        run DiffTest with reference "
                   "REF_SO\n");
            printf("\t-p,--port=PORT          run DiffTest with port PORT\n");
            printf("\n");
            exit(0);
        }
    }
    return 0;
}
```

-   The `getopt_long` function is used to parse command-line options, especially for parsing long arguments.
-   Check the detailed definition and usage with `man 3 getopt_long`.

### `static`

-   Take a look at this function:

    ```c
    static void restart() {
        /* Set the initial program counter. */
        cpu.pc = RESET_VECTOR;

        /* The zero register is always 0. */
        cpu.gpr[0] = 0;
    }
    ```

-   We can see that this function is a `static` one. But what does `static` mean?

    > static (C99 6.2.2 #3): If the declaration of a file scope identifier for an object or a function contains the storage- class specifier static, the identifier has internal linkage.

-   This explanation is very abstract and hard to understand for me as I don't have much knowledge about it. Let's take a look at an example of it:

    -   AWAK, in **C** if we have several functions with the same name declared in different files, these files can be compiled separately. But when it comes to linkage, it will cause errors.

        ```c
        /* a.c */ int f() { return 0; }
        /* b.c */ int f() { return 1; }
        ```

    -   Will cause:

        ```plaintext
        b.c:(.text+0x0): multiple definition of f; a.c:(.text+0xb): first defined here
        ```

-   This is also the reason why we shouldn't write definitions of functions into head files, since a head file could be possibly included multiple times in different source files.
-   Somebody may argue that 'no one does this, it's just stupid'. Well, there are actually multiple functions were declared also defined in head files in PA. Therefore, they need the `static` specifier, making the corresponding identifier to have _internal linkage_ and avoid the errors.

### `inline`

-   Further more, if we take a look to a function written in `reg.h`, we would find that it also has a specifier called `inline` :

    ```c
    static inline int check_reg_idx(int idx) {
        ...
    }
    ```

-   It turns out to be another reminder for the compiler, telling the compiler that, for every call for this function, the function call will be replaced with the actual code of the function, rather than just pushing the original loaded function into stack.
-   The purpose of doing so is to **improve performance** by eliminating the overhead of a function call, at a price of an increased code size.
-   The `inline` function **_shouldn't_** be too long(large) as it would increase size of the code.

### `Assert`

-   In nemu project, `Assert` is actually a macro defined in the framework, see below:

    ```c
    #define Assert(cond, format, ...)                                              \
        do {                                                                       \
            if (!(cond)) {                                                         \
                MUXDEF(CONFIG_TARGET_AM,                                           \
                       printf(ANSI_FMT(format, ANSI_FG_RED) "\n", ##__VA_ARGS__),  \
                       (fflush(stdout),                                            \
                        fprintf(stderr, ANSI_FMT(format, ANSI_FG_RED) "\n",        \
                                ##__VA_ARGS__)));                                  \
                IFNDEF(CONFIG_TARGET_AM, extern FILE *log_fp; fflush(log_fp));     \
                extern void assert_fail_msg();                                     \
                assert_fail_msg();                                                 \
                assert(cond);                                                      \
            }                                                                      \
        } while (0)
    ```

### `Log` && `ASNI_FMT`

-   In `monitor.c`, function `welcome` :

    ```c
    Log("Trace: %s", MUXDEF(CONFIG_TRACE, ANSI_FMT("ON", ANSI_FG_GREEN),
                            ANSI_FMT("OFF", ANSI_FG_RED)));
    ```

-   `Log` :

    -   This macro can be used to print out log info and write them into specified log files automatically.
    -   `debug.h` :

        ```c
        #define Log(format, ...)                                                       \
            _Log(ANSI_FMT("[%s:%d %s] " format, ANSI_FG_BLUE) "\n", __FILE__,          \
                 __LINE__, __func__, ##__VA_ARGS__)
        ```

    -   `utils.h` :

        ```c
        #define log_write(...)                                                         \
            IFDEF(                                                                     \
                CONFIG_TARGET_NATIVE_ELF, do {                                         \
                    extern FILE *log_fp;                                               \
                    extern bool log_enable();                                          \
                    if (log_enable()) {                                                \
                        fprintf(log_fp, __VA_ARGS__);                                  \
                        fflush(log_fp);                                                \
                    }                                                                  \
                } while (0))

        #define _Log(...)                                                              \
            do {                                                                       \
                printf(__VA_ARGS__);                                                   \
                log_write(__VA_ARGS__);                                                \
            } while (0)
        ```

-   `ASNI_FMT` :

    -   This macro is used to define the foreground & background of the information printed out to the command line. Namely, customize the theme of the output.
    -   `utils.h` :

        ```c
        #define ANSI_FG_BLACK "\33[1;30m"
        #define ANSI_FG_RED "\33[1;31m"
        #define ANSI_FG_GREEN "\33[1;32m"
        #define ANSI_FG_YELLOW "\33[1;33m"
        #define ANSI_FG_BLUE "\33[1;34m"
        #define ANSI_FG_MAGENTA "\33[1;35m"
        #define ANSI_FG_CYAN "\33[1;36m"
        #define ANSI_FG_WHITE "\33[1;37m"
        #define ANSI_BG_BLACK "\33[1;40m"
        #define ANSI_BG_RED "\33[1;41m"
        #define ANSI_BG_GREEN "\33[1;42m"
        #define ANSI_BG_YELLOW "\33[1;43m"
        #define ANSI_BG_BLUE "\33[1;44m"
        #define ANSI_BG_MAGENTA "\33[1;35m"
        #define ANSI_BG_CYAN "\33[1;46m"
        #define ANSI_BG_WHITE "\33[1;47m"
        #define ANSI_NONE "\33[0m"

        #define ANSI_FMT(str, fmt) fmt str ANSI_NONE
        ```

    -   Effect:

        ![Shot-2024-06-17-215810.png](https://s2.loli.net/2024/06/17/4d1RrQPqzJ7mgWv.png)

## Debugging `core dump: Segmentation Fault`

-   Sometimes(like `core dump`) it is hard to debug nemu, since you might not know whether the error occurs in nemu itself or the program which it is running.
-   In this situation, we would need to use the `backtrace` feature of `gdb` to go back to the frame which causes the problem.

## Expanding Macros in a Better Way

-   There are many nested macros in the source code of PA. Expanding them manually doesn't seem to be a good idea.
-   We can add these lines to the Makefile(`build.mk`):

    ```Makefile
    # Compilation patterns
    $(OBJ_DIR)/%.o: %.c
        @echo + CC $<
        @mkdir -p $(dir $@)
        @$(CC) $(CFLAGS) -c -o $@ $<

        @# -----Expand macros-----
        @$(CC) $(CFLAGS) -E -MF /dev/null $@ $< | \
            grep -ve '^#' | \
            clang-format - > \
            $(basename $@).i
        @# -----Expand macros-----

        $(call call_fixdep, $(@:.o=.d), $@)
    ```

-   _Though, I think the output file of this command is even harder to read..._

## End.
