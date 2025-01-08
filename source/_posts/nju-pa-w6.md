---
title: PA W6 - Machine Level Representation of Data
date: 2024-08-16 18:26
categories: 笔记
tags:
    - pa
    - c
    - make
    - gdb
excerpt: Notes taken for class W6 of NJU PA
mathjax: true
---

{% notel orange fa-triangle-exclamation Warning %}
If someone is reading this blog, please be aware that the writer **did not** consider the experience of the other readers.
After all, the most important part is about writing things down for better memorization.
{% endnotel %}

## Bit Operation

-   In the world of Math, the most basic operations on numbers are plus, subtract, multiply and division.
-   However, in the world of computer, the most basic operations on bits, are bit operations like:

    -   $, |, ~
    -   ^
    -   <<, >>

-   But what can we do with bit operations in C? See this example:

    ```plaintext
        a = 0 1 0 1 = 5
        b = 0 1 1 0 = 10
            | | | |
    a & b = 0 1 0 0 = 8
    ```

-   From the example calculation above we can see that a trait of bit operations is that: **the calculations are done separately for each bit**.

### Digit Slice

-   Now, let's consider a problem: How to take second _byte_ from, let's say, a 32-bit integer?

    ```plaintext
    142857 = 0000 0000 0000 0010 0010 1110 0000 1001
    ```

    _(MSB: left, LSB: right)_

    -   Let's first take a glance at a easy version of this problem: _how to take the third digit from a four bit number?_

        ```plaintext
        x = 0 1 0 1 = 5
        ```

    -   The answer is simple:

        ```plaintext
        (x >> 1) & 1 = 0
        ```

    -   First we cut the first digit on the right, then use a bit operation(and) to get the target digit.

    -   As for the original problem, we can do this:

        ```plaintext
        (142857 >> 16) & 0000 0000 1111 1111
        ```

### Representation of a Set

-   Another implementation is that a binary numbers can be used to represent the status of elements a set.
-   By using a binary number with the same length as the number of elements in the set, we can determine whether a particular element is present or absent by examining the corresponding bit in the binary number.

-   e.g. Say we have a set of four elements, and it is now set to `{0, 2}`, the corresponding binary number should be `0 1 0 1`.
-   Now if we want to check whether the **_third_** element is present in the set(which is obviously not), we can do some bit operations like this:

    ```plaintext
    (0 1 0 1 >> 3) & 1 = 0
    ```

-   The same thing happens when we want to add/remove elements from the set.

### Length of a Set

-   How to count how many elements are there in a set with its corresponding binary number? By checking how many `1` are there in the bin number.

    ```c
    int bitset_size(uint32_t S) {
        int n = 0;
        for (int i = 0; i < 32; i++) {
            n += bitset_contains(S, i);
        }

        return n;
    }
    ```

-   But consider this function below:

    ```c
    int bitset_size1(uint32_t S) {
        S = (S & 0x55555555) + ((S >> 1) & 0x55555555);
        S = (S & 0x33333333) + ((S >> 2) & 0x33333333);
        S = (S & 0x0F0F0F0F) + ((S >> 4) & 0x0F0F0F0F);
        S = (S & 0x00FF00FF) + ((S >> 8) & 0x00FF00FF);
        S = (S & 0x0000FFFF) + ((S >> 16) & 0x0000FFFF);

        return S;
    }
    ```

    -   This is a quite ingenious and effective method to calculate the number of `1` in a binary number. The logic of it is stated below:

        1. The function first combines the bits in odd positions with those in even positions using the mask `0x55555555`. This operation reduces the problem by half, where each pair of bits now represents the number of `1`s in the original pair.
        2. Next, it groups the results into 4-bit segments, summing the number of `1`s in the first two bits with those in the last two bits using the mask `0x33333333`.
        3. 8-bit...
        4. 16-bit...

### Finding low-bit:

-   Suppose we have a rather large binary number, how to find the lowest `1` bit?

    -   For example: `0b+++++100`.
    -   We can do some common bit operations and look at the results:

        ```plaintext
        x      = 0b+++++100
        x - 1  = 0b+++++011
        ~x     = 0b-----011
        ~x + 1 = 0b-----100
        ```

    -   We can see that by performing a bitwise AND operation between `x` and `~x + 1`, we could get the lowest `1` bit.

        ```plaintext
        x & (~x + 1) = 0b00000100
        ```

-   In this process, we have actually encounter something called two's complement, which is `~x + 1`.
-   It is also represented by `-x`, so instead of using `x & (~x + 1)`, we could just use `x & -x`.

### Calculating $log_2(x)$

-   Given a 32-bit integer `x`, we can calculate $log_2(x)$ by observing its binary representation. The idea is straightforward: $log_2(x)$ corresponds to the position of the highest `1` bit in the binary representation of `x`.
-   This works because each bit position in a binary number represents a power of 2. The highest `1` bit indicates the most significant power of 2 that contributes to the value of `x`. Therefore, the position of this bit (counting from 0) is the value of $log_2(x)$

-   When it comes to coding, we can consider how to find out how many consistent `0`s are there on the left side of the number, and then subtract it from `31` to get the answer.

    ```c
    int clz(uint32_t x) {
        int n = 0;
        if (x <= 0x0000ffff) n += 16, x <<= 16;
        if (x <= 0x00ffffff) n += 8, x <<= 8;
        if (x <= 0x0fffffff) n += 4, x <<= 4;
        if (x <= 0x3fffffff) n += 2, x <<= 2;
        if (x <= 0x7fffffff) n += 1;

        return n;
    }

    int ans = 31 - clz(x);
    ```

-   This code first compares `x` with `0x0000ffff`, in order to see whether the first 16 bits of `x` are all zero. If they are indeed all `0`, add `16` to `n`(answer holder), and then move the 16 bits on the right to the left for further calculation. If not, precede immediately.
-   It then narrow down the range for comparison to 8 bits, and so on.

-   Finally we get the position of the highest `1` in the original number, and then subtract it from 31 to get the answer of $log_2(x)$.

---

-   Here's another way of calculating $log_2(x)$:

    ```c
    #define LOG2(X) "-01J2GK-3@HNL;-=47A-IFO?M:<6-E>95D8CB"[(X) % 37] - '0'
    ```

-   This is a very peculiar and intriguing method to solve the log2 problem. It has a table represented by a string, which could be used for finding the answer of certain integar.
-   Because this method requires minimum calculation, it has an outstanding efficiency.
-   Also, as it is written in a _macro_, it gains even more performance as it will get expanded into a _constant_ after the compilation.

---

-   Finally, the fastest and the easiest way to calculate $log_2(x)$ is to use a builtin method of gcc: `__builtin_ctz` (ctz = count trailing zeros).

    ```c
    printf("log_2(1024) = %d\n", __builtin_ctz(1024));
    // output: log_2(1024) = 10
    ```

~~Though using this method requires zero bit operation knowledge~~

-   The `__builtin_ctz` function is a compiler intrinsic that is often **directly translated into a single CPU instruction**. This makes it much faster than manually implementing the logarithm function or using more complex mathematical operations.

## Undefined Behavior & Integer Overflow

> Undefined behavior(UB) is the result of executing computer code whose behavior is not prescribed by the language specification to which the code adheres, for the current state of the program. This happens when the translator of the source code makes certain assumptions, but these assumptions are not satisfied during execution.
> _--Wikipedia_

-   The C programming language does not have any limitation for UB.
-   Common UB:

    -   illegal memory access(null pointer, index out of range...)
    -   divided by zero
    -   signed integer overflow
    -   ...

-   When UB occurs inside the code, what it will end up to is completely depends on the compiler.
-   Some compilers would exit when they encounter a UB, while some does not. Some compilers would just delete the UB related code, while others might understand the UB in a strange way.
-   For example:

    -   Take this function:

        ```c
        int f() { return 1 << -1; }
        ```

    -   **clang** recognise this `1 << -1` as an undefined behavior, then it will delete this calculation.
    -   As a result, this function would just normally return and ignores this `1 << -1` thing.

### Common Integer Related UB

| Expression                 | Value         |
| -------------------------- | ------------- |
| UINT_MAX + 1               | 0             |
| INT_MAX + 1; LONG_MAX + 1; | **undefined** |
| char c = CHAR_MAX; c++;    | varies(???)   |
| 1 << -1                    | **undefined** |
| 1 << 0                     | 1             |
| 1 << 31                    | **undefined** |
| 1 << 32                    | **undefined** |
| 1 / 0                      | **undefined** |
| INT_MAX % -1               | **undefined** |

## Floating Point Number: IEEE 754

-   It is quite difficult to map decimals into a 32-bit/64-bit binary, since real number is infinite, while the number represented by a 32-bit/64-bit binary is limited.

-   **IEEE 754** is a standard of how a floating point number should be stored in a 32-bit binary:

    -   1 bit S, 23/52 bits Fraction, 8/11 bits Exponent:

        $$
        x = (-1)^S \times (1.F) \times 2^{E - B}
        $$

    -   ![32-bit example](https://upload.wikimedia.org/wikipedia/commons/thumb/d/d2/Float_example.svg/590px-Float_example.svg.png)

    _[How to calculate this?](https://www.youtube.com/watch?v=yh2m7BSzRRo)_

### Distribution && Precision

-   Despite the floating point number could represent a large amount of real number, **most of them are concentrated around zero**.

-   The program down below shows the precision loss of floating point number when representing a large number, and their distribution around zero.

    ```c
    #include <float.h>
    #include <math.h>
    #include <stdint.h>
    #include <stdio.h>

    int main(int argc, char *argv[]) {
        float x = FLT_MAX;
        printf("x = %e (10^%.1f)\n", x, log10(x));

        printf("=============================\n");
        printf("float type variable can represent a fairly large number, but the "
               "loss will be tremendous\n");

        float y = 1e38;
        printf("y         = %.0f\n", y);
        printf("y + 1e30f = %.0f\n", y + 1e30f);
        printf("y + 1e31f = %.0f\n", y + 1e31f);

        printf("=============================\n");

        unsigned long n1 = 0, n2 = 0, n3 = 0;
        union {
            float f;
            int i;
        } z;

        for (uint32_t i = 0;; i++) {
            z.i = i;
            if (-1.0f < z.f && z.f < 1.0f)
                n1++;
            if (-0.5f < z.f && z.f < 0.5f)
                n2++;
            if (-0.001f < z.f && z.f < 0.001f)
                n3++;
            if (i == UINT32_MAX)
                break;
        }

        double n = (double)UINT32_MAX + 1;
        printf("%.2lf%% of floats are in (-1, 1)\n", (double)n1 / n * 100);
        printf("%.2lf%% of floats are in (-0.5, 0.5)\n", (double)n2 / n * 100);
        printf("%.2lf%% of floats are in (-0.001, 0.001)\n", (double)n3 / n * 100);
        printf("Most of the decimals represented by float type variable are very "
               "close to 0\n");

        return 0;
    }
    ```

    :

    ```plaintext
    x = 3.402823e+38 (10^38.5)
    =============================
    float type variable can represent a fairly large number, but the loss will be tremendous
    y         = 99999996802856924650656260769173209088
    y + 1e30f = 99999996802856924650656260769173209088
    y + 1e31f = 100000006944061726476491472742798852096
    =============================
    49.61% of floats are in (-1, 1)
    49.22% of floats are in (-0.5, 0.5)
    45.71% of floats are in (-0.001, 0.001)
    Most of the decimals represented by float type variable are very close to 0
    ```

-   From the output we can see that: large portions of possible float values fall within the intervals (-1, 1), (-0.5, 0.5), and (-0.001, 0.001). This illustrates that the float type has a dense distribution of representable values near zero, while the precision decreases as values move farther from zero.

-   This is the reason why most of the math calculation(especially in machine learning and deep learning) requires **normalization** -- to ensure precision.

---

-   This feature could cause trouble when doing calculation among extreme numbers. Take an example of the fomula below:

    $$
    \frac{-b-\sqrt{b^2 - 4ac}}{2a}
    $$

-   According to a paper:
    > P.Panchekha, et al. Automatically improving accuracy for floating point expressions. ln Proc. of PLDI, 2015.
-   Better fomula in certain condition:

    -   $b<0$ :

        $$
        \frac{4ac}{-b+\sqrt{b^2-4ac}} \cdot \frac{1}{2a}
        $$

    -   $0 \le b \le 10^{127}$ :

        $$
        (-b-\sqrt{b^2-4ac}) \cdot \frac{1}{2a}
        $$

    -   $b \gt 10^{127}$ :

        $$
        -\frac{b}{a} + \frac{c}{b}
        $$

---

-   A more common and practical example in game engine is:

    $$
    \frac{1}{\sqrt{x}}
    $$

-   There is a famous and magic solution for this problem:

    > Matthew Robertson. A Brief History of InvSqrt, Bachelor Thesis, The University of New Brunswick, 2012.

    ```c
    float Q_rsqrt(float number) {
        union { float f; uint32_t i; } conv;
        float x2 = number * 0.5F;
        conv.f = number;
        conv.i = 0x5f3759df - (conv.i >> 1); // ???
        conv.f = conv.f * (1.5F - (x2 * conv.f * conv.f));

        return conv.f;
    }
    ```

-   This solution uses a 'magic trick' to conduct an _O(1)_ algorithm.

---

-   And yet another example on calculating $(a \times b) \mod m$ :

    ```c
    int64_t multimod_fast(int64_t a, int64_t b, int64_t m) {
        int64_t x = (int64_t)((double)a * b / m) * m;
        int64_t t = (a * b - x) % m;

        return t < 0 ? t + m : t;
    }
    ```

## End
