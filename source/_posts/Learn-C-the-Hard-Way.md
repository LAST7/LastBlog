---
title: Learn C the Hard Way
date: 2024-10-30 17:14:52
categories: 笔记
tags:
    - c
    - make
excerpt: Interesting & useful functions or header files from _Learn C the Hard Way_
---

## General Project Structure

-   A general C project structure with a well-functioning Makefile, automated tests(unit test):

    ```plaintext
    .
    ├── bin
    ├── build
    │   ├── libYOUR_LIBRARY.a
    │   └── libYOUR_LIBRARY.so
    ├── LICENSE
    ├── Makefile
    ├── README.md
    ├── src
    │   ├── libex29.c
    │   └── libex29.o
    └── tests
        ├── libex29_tests
        ├── libex29_tests.c
        ├── minunit.h
        ├── runtests.sh
        └── tests.log

    5 directories, 13 files
    ```

-   For details inside of each file, see down below.

## Makefile

```makefile
.PHONY: all dev clean test

CFLAGS = -g -O2 -Wall -Wextra -Isrc -rdynamic -DNDEBUG $(OPTFLAGS)
LIBS = -ldl $(OPTLIBS)
PREFIX ?= /usr/local

SOURCES = $(wildcard src/**/*.c src/*.c)
OBJECTS = $(patsubst %.c,%.o,$(SOURCES))

TEST_SRC = $(wildcard tests/*_tests.c)
TESTS = $(patsubst %.c,%,$(TEST_SRC))

TARGET = build/libYOUR_LIBRARY.a
SO_TARGET = $(patsubst %.a,%.so,$(TARGET))

# the target build
all: $(TARGET) $(SO_TARGET) test

dev: CFLAGS = -g -Wall -Isrc -Wall -Wextra $(OPTFLAGS)
dev: all

# `-fPIC` means Position Independent Code, which is essential for creating
# shared libraries
$(TARGET): CFLAGS += -fPIC
$(TARGET): build $(OBJECTS)
	ar rcs $@ $(OBJECTS)
	ranlib $@

$(SO_TARGET): $(TARGET) $(OBJECTS)
	$(CC) -shared -o $@ $(OBJECTS)

build:
	@mkdir -p build
	@mkdir -p bin

# unit test
test: CFLAGS += $(TARGET)
test: $(TESTS)
	sh ./tests/runtests.sh

valgrind:
	VALGRIND="valgrind --log-file=/tmp/valgrind-%p.log" $(MAKE)

# cleaner
clean:
	rm -rf build $(OBJECTS) $(TESTS)
	rm -f tests/tests.log
	find . -name "*.gc" -exec rm {} \;
	rm -rf `find . -name "*.dSYM" -print`

# install
install: all
	install -d $(DESTDIR)/$(PREFIX)/lib/
	install $(TARGET) $(DESTDIR)/$(PREFIX)/lib/

# checker
BADFUNCS = [^_.>a-zA-Z0-9](str(n?cpy|n?cat|xfrm|n?dup|str|pbrk|tok|_)|stpn?cpy|a?sn?printf|byte_)
check:
	@echo Files with potentially dangerous functions.
	@grep -E "$(BADFUNCS)" $(SOURCES) || true
```

-   This is quite a large Makefile, containing mechanisms which allow automatic build and test process.

-   In this blog, I will only focus on the important parts as I am too lazy to explain it thoroughly. It's just doesn't worth it.

### Main Logic

-   This Makefile automatically collects all C source files from the `src/` directory and its subdirectories, compiles them into object files, and bundles them into both static (.a) and shared (.so) library formats while also setting up a testing framework - all orchestrated through a hierarchy of targets where `all` builds everything.
-   `dev` provides a debug-friendly version, and `clean` removes all generated files.

### Tests

-   This Makefile contains logic for automatic tests, right after the build process.
-   The test process is actually written in `tests/runtests.sh`:

    ```bash
    # color definition(see folder down below)
    # ...

    echo -e "${BG_BLUE}${FG_ORANGE} Running unit tests:${NONE}"

    for i in tests/*_tests
    do
        if test -f $i; then
            if $VALGRIND ./$i 2 >> tests/tests.log; then
                echo -e "${FG_GREEN} $i PASS${NONE}"
            else
                echo -e "${FG_RED}ERROR in test $i:${NONE}"
                echo -e "${FG_RED}-----${NONE}"
                tail tests/tests.log
                exit 1
            fi
        fi
    done

    echo ""
    ```

{% folding blue::Color_Definition %}

```bash
# Foreground colors
FG_BLACK="\033[1;30m"
FG_RED="\033[1;31m"
FG_GREEN="\033[1;32m"
FG_YELLOW="\033[1;33m"
FG_ORANGE="\033[44;93m"
FG_BLUE="\033[1;34m"
FG_MAGENTA="\033[1;35m"
FG_CYAN="\033[1;36m"
FG_WHITE="\033[1;37m"

# Background colors
BG_BLACK="\033[1;40m"
BG_RED="\033[1;41m"
BG_GREEN="\033[1;42m"
BG_YELLOW="\033[1;43m"
BG_ORANGE="\033[48;93m"
BG_BLUE="\033[1;44m"
BG_MAGENTA="\033[1;45m"
BG_CYAN="\033[1;46m"
BG_WHITE="\033[1;47m"

# Reset
NONE="\033[0m"
```

_Stolen from `debug.h` from PA :)_

{% endfolding %}

{% notel blue fa-circle-info Note: %}

Variable `$VALGRIND` in the script should be defined in the Makefile when running `make valgrind`.

When running `make` or `make test`, `$VALGRIND` will be empty/undefined, so the command simply runs `./$i`.

---

Also, the `2` in line `if $VALGRIND ./$i 2 >> tests/tests.log; then` refers to file descriptor 2, which is **stderr(standard error)**.

> In Unix/Linux systems, there are three standard file descriptors:
>
> 0: stdin (standard input)
> 1: stdout (standard output)
> 2: stderr (standard error)

So `2 >>` means "append stderr output to the specified file". This ensures that error messages and debugging information (which typically go to stderr) are captured in tests/tests.log.

{% endnotel %}

## Debug Helper

{% folding blue::dbg.h/log.h %}

`dbg.h`:

```c
#ifndef __DBG_H__
#define __DBG_H__

#include "log.h"
#include <errno.h>
#include <stdio.h>
#include <string.h>

#ifdef NDEBUG
#define debug(M, ...)
#else
#define debug(M, ...)                                                          \
    fprintf(stderr, ANSI_FMT("DEBUG %s:%d: " M "\n", ANSI_BG_BLACK), __FILE__, \
            __LINE__, ##__VA_ARGS__)
#endif // DEBUG

#define clean_errno() (errno == 0 ? "None" : strerror(errno))

#define log_err(M, ...)                                                        \
    fprintf(stderr,                                                            \
            ANSI_FMT("[ERROR] (%s:%d: errno: %s) " M "\n", ANSI_FG_RED),       \
            __FILE__, __LINE__, clean_errno(), ##__VA_ARGS__)

#define log_warn(M, ...)                                                       \
    fprintf(stderr,                                                            \
            ANSI_FMT("[WARN] (%s:%d: errno: %s) " M "\n", ANSI_FG_YELLOW),     \
            __FILE__, __LINE__, clean_errno(), ##__VA_ARGS__)

#define log_info(M, ...)                                                       \
    fprintf(stderr, ANSI_FMT("[INFO] (%s:%d:) " M "\n", ANSI_FG_BLUE),         \
            __FILE__, __LINE__, ##__VA_ARGS__)

#define check(A, M, ...)                                                       \
    if (!(A)) {                                                                \
        log_err(M, ##__VA_ARGS__);                                             \
        errno = 0;                                                             \
        goto error;                                                            \
    }

#define sentinel(M, ...)                                                       \
    {                                                                          \
        log_err(M, ##__VA_ARGS__);                                             \
        errno = 0;                                                             \
        goto error;                                                            \
    }

#define check_mem(A) check((A), "Out of memory.")

#define check_debug(A, M, ...)                                                 \
    if (!(A)) {                                                                \
        debug(M, ##__VA_ARGS__);                                               \
        errno = 0;                                                             \
        goto error;                                                            \
    }

#endif // !__DBG_H__
```

`log.h`:

```c
#ifndef __LOG_H__
#define __LOG_H__

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

#endif // !__LOG_H__
```

{% endfolding %}

## To Be Continued...
