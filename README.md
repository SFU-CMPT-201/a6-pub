# Assignment 7: Memory layout

When you have a program as an executable, it is a file stored on your disk. However, when you run
the program, your OS loads the program to the main memory and starts executing the program. This
means that there exist two copies of the program when you run it---one copy on the disk and another
copy in memory. At this point, the program/executable/file on your disk doesn't really matter. What
matters is the copy of the program in memory.

When an OS loads a program to memory, it uses a particular format suitable for executing a program.
This format is called *memory layout*. Understanding this layout is crucial for writing more
reliable and correct code, and also for deepening your understanding of pointers and memory
management. In this assignment, you will learn the background and do some exercises.

## Task 0: Understanding the Linux memory layout and C pointers

Below is a simplified diagram of how the memory layout looks like when Linux loads a program.

.───────────────────────────────────────.  Address 2^n - 1, where n is the number of address bits
|                                       |  (0xFFFFFFFF for the 32-bit address space)
|         Kernel address space          |
|                                       |
+───────────────────────────────────────+
|                                       |
|                 Stack                 |
|                (grows ↓)              |
|                                       |
+───────────────────────────────────────+
|                                       |
|                                       |
+───────────────────────────────────────+
|                                       |
|             Memory Mapping            |
|                (grows ↓)              |
|                                       |
+───────────────────────────────────────+
|                                       |
|                                       |
+───────────────────────────────────────+
|                                       |
|                  Heap                 |
|                (grows ↑)              |
|                                       |
+───────────────────────────────────────+
|                                       |
+───────────────────────────────────────+
|                                       |
|         BSS (Uninitialized data)      |
|                                       |
+───────────────────────────────────────+
|                                       |
|         Data (Initialized data)       |
|                                       |
+───────────────────────────────────────+
|                                       |
|             Text (Code)               |
|                                       |
+───────────────────────────────────────+
|                                       |
'───────────────────────────────────────'  Address 0x00000000

Before examining the diagram in more detail, there are a few things to note.

* The top of the diagram is the highest memory address and the bottom of the diagram is the lowest
  memory address.
* Each memory address identifies a single byte.
* The memory address space starts from 0.
* In a C program, the value of a pointer is a memory address. For example, `int *ptr = 0` means that
  the pointer `ptr` now holds the memory address 0 as its value. Here, `ptr` is a variable that has
  the type `int *` (which makes `ptr` a pointer). Just like any other variables, a pointer has a
  size limit according to the size of its type. For example, we know that if a variable called `i`
  is of the type `int32_t`, `i`'s value can only range from the minimum value to the maximum value
  that `int32_t` can represent. This range is determined by the size of the type `int32_t`. Since it
  uses 32 bits, the minimum value is -2^31 (`INT32_MIN`) and the maximum value is 2^31-1
  (`INT32_MAX`). Similarly, if a variable called `ptr` is of the type `int *`, its value can only
  range from the minimum value to the maximum value that `int *` can represent. All pointer types
  (`int *`, `char *`, `void *`, etc.) are of the same size, and they are typically either 32 bits or
  64 bits, depending on what CPU you use. If you use a 32-bit CPU, it is 32 bits. If you use a
  64-bit CPU, it is 64 bits.
* The size of the pointer types effectively limits the size of the memory space that a program can
  use. For example, if the pointer types are represented as 32 bits, their values can only be
  between 0 and 2^32-1, which means that the range of memory addresses must be between 0 and 2^32-1.
  This effectively limits the size of the memory address space.
* The memory layout consists of *segments*, e.g., the kernel address segment, the stack segment,
  etc.

Let's examine the address layout segment-by-segment.

* Kernel address space: An OS consists of two
