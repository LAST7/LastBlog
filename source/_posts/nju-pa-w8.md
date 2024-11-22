---
title: PA W8 - IO Devices
date: 2024-11-08 15:21:10
categories: PA
tags:
    - c
excerpt: Notes taken for class W8 of NJU PA
---

{% notel orange fa-triangle-exclamation Warning %}
If someone is reading this blog, please be aware that the writer **DID NOT** consider the experience of the other readers.
After all, the most important thing is about writing things down for better memorization.
{% endnotel %}

{% notel red fa-triangle-exclamation Warning %}
Starting from this blog, the lesson is updated to ICS2024.
Video link: [ICS2024-南京大学软件学院计算机系统基础实验](https://www.bilibili.com/video/BV1ztSiYPEr8/)
{% endnotel %}

## IO

-   The computer itself doesn't pretty much do anything apart from 'computing'. It is the IO devices that connect the computer with the outer world, allowing more application and wider promotion.

-   We included two instruction sets(on x86) `in` and `out` to handle IO operations. The first one is to read data from the devices into the cpu, and the latter one does the opposite.

### Memory Mapping IO

-   One very basic way of reading data from outer devices is that, we simply connect the cpu with the registers in the outer devices with circuits. Though being simple, it is obviously not a good way as the number of physical interface on cpu is limited, which means we can't just design an interface for each of the possible device.

---

-   That introduces the second way of connecting cpu to the outer world, **memory mapped IO**.
-   Memory mapped IO is basically saying that, we could set up a part of the memory for a specific outer device. When writing/reading data from that part of memory, we are essentially reading data from the designated device.

-   The advantage of this way is that we avoid the problem that we couldn't connect infinite devices to the cpu.
-   But the drawback is that a part of the memory is consumed for this mapping.

---

-   Something worth mentioning is that, the compiler doesn't actually know the writing memory section is mapped to a device or not. Sometimes it could do unwanted optimizations. Check the example down below:

    ```c
    void foo() {
        for (int i = 0; i < 1024; i++) {
            // out(ADDR, 0)
            (*(char *)ADDR) = 0;
        }
    }
    ```

    -   Here we intended to write a `0` to `ADDR` for 1024 times. But the compiler does not know that this is a spectial section of memory which is mapped to a outer device, so it would consider no need to write the same value in repeatedly, you will end up only writing `0` one time to `ADDR`.
    -   Specifically, the loop logic will be deserted with the compilation flag `-O2`(or higher optimization levels).

-   The solution is to add `volatile` keyword, which prevent the compiler from optimizing the code.

    ```c
    (*(volatile char *)ADDR) = 0;
    ```

## Two Special IO devices

### Bus

From [Wikipedia](<https://en.wikipedia.org/wiki/Bus_(computing)>):

> In computer architecture, a bus is a communication system that transfers data between components inside a computer, or between computers. This expression covers all related hardware components (wire, optical fiber, etc.) and software, including communication protocols.

-   The bus connect the IO devices in the computer with the cpu.

### Programmable Interrupt Controller(PIC)

From [Wikipedia](https://en.wikipedia.org/wiki/Programmable_interrupt_controller)

> In computing, a programmable interrupt controller (PIC) is an integrated circuit that helps a microprocessor (or CPU) handle interrupt requests (IRQs) coming from multiple different sources (like external I/O devices) which may occur simultaneously. It helps prioritize IRQs so that the CPU switches execution to the most appropriate interrupt handler (ISR) after the PIC assesses the IRQs' relative priorities.

-   Generally speaking, PIC is a device which sends a interruption signal to the cpu at a fixed frequency. This allows interaction with the executing process, and furthur more, concurrency.

## Interruption: Compensating for IO Device Speed Deficiencies

-   Most interaction with I/O devices are slow, as there might be interaction with the real world which are of course slower than the electronic world inside the cpu.
-   Because of that, we don't want the cpu to wait until the IO jobs are done before executing other instructions.

-   This requires two things:

    1. After sending data/instructions to I/O devices, the cpu needs to turn to other tasks/processes.
    2. Once the I/O jobs are done, the devices should inform the cpu to do the remaining jobs or retrieve the data from the devices.

-   This way, the relatively slower I/O operation could be executed without halting the cpu.
