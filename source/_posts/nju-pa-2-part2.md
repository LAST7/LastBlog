---
title: PA2 Part 2 - Emulated Hardware Device
date: 2024-11-22 18:10:25
categories: 笔记
tags:
    - 笔记
    - c
excerpt: Implementing emulated device such as timer, keyboard, VGA and audio
---

{% notel orange fa-triangle-exclamation Warning %}
If someone is reading this blog, please be aware that the writer **DID NOT** consider the experience of the other readers.
After all, the most important thing is about writing things down for better memorization.
{% endnotel %}

## IO Interaction Between AM and NEMU

-   Generally speaking, NEMU would register some special memory address as MMIO abstract registers, such as `CONFIG_AUDIO_CTL_MMIO`_(0xa0000200)_ when initializing corresponding devices.
-   When the guest program(application) is trying to read/write data from these special memory address, NEMU would intercept those read/write operations and run the correlated callback functions which are set up with the special register, to simulate read/write op to hardware registers.

## Timer

-   The memory address of RTC is _0xa0000048_, where NEMU registers 8 bytes of memory space.
-   Accessing this memory area would invoke function `rtc_io_handler` which effectively returns the uptime in us as a 64-bit int.
-   As AM could conduct a read operation of at most 32 bits(`inl`), we have to 'read' twice for the complete 64-bit data.
-   The first 4 bytes starting from _0xa0000048_ is the lower part of the 8-byte uptime.

## Keyboard

-   There seems to be quite a lot of things going on in `nemu/src/device/keyboard.c`, but it's a quite simple task as long as one understand the interaction logic between AM and NEMU, as there're only 4 bytes(AKA 1 32-bit int) representing a keycode being registered at `CONFIG_I8042_DATA_MMIO`_(0xa0000060)_.

-   There's an interesting design `#define KEYDOWN_MASK 0x8000` in AM. It is basically an indicator of whether the key is pressed or released. Here's how it works:

    -   First of all, a convention in NEMU states that, if the keycode's 15th bit is non-zero, that means the key is pressed down, and vice versa.
    -   Then, we can use this mask to judge whether the keycode represents a pressed key or a released one by conducting bitwise operation AND(&) between the keycode and the `KEYDOWN_MASK`, if the result is `1`, that means a pressed key, and vice versa.

---

{% notel blue fa-circle-info Fun_Fact %}
This 15th bit convention is primarily a Windows-specific implementation detail.
Other systems have other conventions, like using `XKeyEvent` structure on X11(Linux/Unix).
{% endnotel %}

## VGA

-   This is a quite troublesome part. Here's the deal:

    -   The guest program(application) could write frame data to frame buffer register(`CONFIG_FB_ADDR`_0xa1000000_) with the api provided by AM(`io_write`).
    -   AM would then write the pixels sent by the client to `FB_ADDR`(same as `CONFIG_FB_ADDR`). There're no callback function for frame buffer in NEMU, so it is direct write to `vmem`.
    -   NEMU constantly updates vga, whenever the value stored in the abstract sync register turns into `1`, NEMU will use the data in frame buffer to update the screen.

---

-   The code in `am-kernels/kernels/typing-game` is really fun, especially this part:

    ```c
    for (int ch = 0; ch < 26; ch++) {
        char *c = &font[CHAR_H * ch];
        for (int i = 0, y = 0; y < CHAR_H; y++)
            for (int x = 0; x < CHAR_W; x++, i++) {
                int t = (c[y] >> (CHAR_W - x - 1)) & 1;
                texture[WHITE][ch][i] = t ? COL_WHITE : COL_PURPLE;
                texture[GREEN][ch][i] = t ? COL_GREEN : COL_PURPLE;
                texture[RED][ch][i] = t ? COL_RED : COL_PURPLE;
            }
    }
    ```

-   This is where each english letter is 'pre-painted'. By checking each bit(pixel) of the letter with bitwise operation, the program effectively find out the alpha value of each pixel. Then, it writes color data into the corresponding place in `texture`, `COL_PURPLE` for zero-alpha pixel.

## Audio

-   This is the most difficult part of the emulated device done in PA2. Mostly because it requires a ring buffer queue which is written and read by differenct programs(AM & NEMU). Also, Learning SDL is also a troublesome part, cuz I really don't consider its official wiki very friendly.

-   Most of my time was spent on tuning the ring buffer, which is kinda like a producer-consumer model I learnt in OS lessons before. It was the first time that I implement some theoretical OS stuff into actual program, it really took me a while to fix all these bugs created by a fool filled with ignorance, which is me...

---

-   Anyway, the core idea is like this:

    -   The guest program(application) could write audio data to sound buffer register(`CONFIG_SB_ADDR`).
    -   AM would then write the audio data to the queue(`sbuf`, the ring buffer) when there're enough space.
    -   NEMU would read audio data from the queue(`sbuf`) and copy them to SDL2's stream. BTW, this process is actually done in SDL_Audio's callback function.

---

-   Key difficulties when maintaining `sbuf`:

    -   When AM writes audio data:

        1. Check whether there're enough space(`count + len <= sbuf_size`). If not, halt until there are.
        2. Check whether the write-pointer has reached the boundary of the queue. If so, reset the pointer to the start, and write the rest of the data.

    -   When NEMU reads audio data:

        1. Check whether there're valid data in `sbuf`(`count >= 0`). If not, return.
        2. Check whether the read-pointer has reached the boundary of the queue. If so, reset the pointer to the start, and read the rest of the required data.

-   In real practice, the speed of AM writing audio data is significantly faster than NEMU reading, as the frequency of calling SDL2's callback function is pretty low.

## End of PA2

-   It's been quite a journey since I started learning PA. I consider myself utterly fortunate to have found a lesson that aligns so perfectly with what psychologists called the _Zone of Proximal Development_. PA has been exactly that for me -- a space where I can challenge myself while steadily improving.
-   When I first began learning PA, I couldn't even write a basic linked list in C. Now, I found myself maintaining a ring buffer queue that is accessed by differenct programs. How exciting to see how far I've come!

---

-   I am not a student at NJU, not even a CS major student to be precise. I am also not a 985/211 university student. However, my passion for programming, curiosity about the knowledge behind the computing machine, and the desire for meaningful challenges have guided me to this point. This lesson, PA, has shown me the way of learning. I know there's still a long way to go, but the courage I've gained will surely motivate me to continue the rest of my journey.
-   Looking back, now I feel my college life has not been wasted.

---

I want to express my gratitude to:

-   Dr. **HuiYan Wang** (http://www.why.ink:8080) for her excellent [open classes](https://www.bilibili.com/video/BV11BpFe4EmM) for PA on Bilibili.
-   Dr. **ZiHao Yu** (https://sashimi-yzh.github.io) for his brilliant emulator [NEMU](https://github.com/NJU-ProjectN/nemu) and the detailed [manual](https://nju-projectn.github.io/ics-pa-gitbook/ics2024/index.html) for PA.
-   The members of the ysyx QQ group, who provided invaluable support.
-   And, of course, the vast resources available on the Internet -- wiki, documentation, forums -- and the assistance from LLMs like ChatGPT and Claude.

---

I will be resting for a while, and then head for the pre-learning of ysyx, which is basicaly knowledge and expirement of Digital Circuit.

Cheers!
