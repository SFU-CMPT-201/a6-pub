# Assignment 7: Memory layout

When you have a program as an executable, it is a file stored on your disk. However, when you run
the program, your OS loads the program to the main memory and starts executing the program. This
means that there exist two copies of the program when you run it---one copy on the disk and another
copy in memory. At this point, the program/executable/file on your disk doesn't really matter. What
matters is the copy of the program in memory.

When an OS loads a program to memory, it uses a particular format suitable for executing a program.
This format is called *memory layout*. Understanding this layout is crucial for writing more
reliable and correct code, and also for deepening your understanding of pointers and memory
management. In this assignment, you will learn about Linux's memory layout. Along the way, you will
also write a few C programs that examine the memory layout, which also give us an opportunity to
take a deeper look at pointers.

## Task 0: Understanding the Linux memory layout and C pointers

Every single thing in your program, all the statements, all the variables, all the function calls,
occupies memory space when you run the program. Your OS organizes the memory space in a certain way
using its pre-defined memory layout and your program's code interacts with the organized memory
space to actually run the program.

Below is a simplified diagram of how Linux organizes the memory space when it runs a program.

.───────────────────────────────────────.  Address 2^n - 1, where n is the number of address bits
|                                       |  (0xFFFFFFFF for the 32-bit address space)
|         Kernel address space          |
|                                       |
+───────────────────────────────────────+
|                                       |
|                 Stack                 |
|               (grows ↓)               |
|                                       |
+───────────────────────────────────────+
|                                       |
|                                       |
+───────────────────────────────────────+
|                                       |
|             Memory mapping            |
|               (grows ↓)               |
|                                       |
+───────────────────────────────────────+
|                                       |
|                                       |
+───────────────────────────────────────+
|                                       |
|                 Heap                  |
|               (grows ↑)               |
|                                       |
+───────────────────────────────────────+
|                                       |
+───────────────────────────────────────+
|                                       |
|                 BSS                   |
| (Uninitialized global or static data) |
|                                       |
+───────────────────────────────────────+
|                                       |
|                 Data                  |
|  (Initialized global or static data)  |
|                                       |
+───────────────────────────────────────+
|                                       |
|              Text (Code)              |
|                                       |
+───────────────────────────────────────+
|                                       |
'───────────────────────────────────────'  Address 0x00000000

Before examining the diagram in more detail, there are a few things to note.

* The memory layout determines the memory space that a program can use. Notice that the size of the
  diagram is *finite*. This means that the memory space your program can use is also finite.
* A program accesses memory by address. The top of the diagram is the highest memory address and the
  bottom of the diagram is the lowest memory address.
* Each memory address identifies a single byte.
* The memory address space starts from 0.
* The memory layout consists of *segments*, e.g., the kernel address segment, the stack segment,
  etc.
* We assume that you know this already, but in a C program, the value stored in a pointer is a
  memory address. For example, `int *ptr = 0` means that the pointer `ptr` now holds the memory
  address 0 as its value. There are a few things to note here.
    * If you define a pointer, e.g., `int *ptr;`, what you're saying is that `ptr` is a variable of
      the type `int *` (which is an integer pointer type). This is similar to how `int i` means that
      `i` is a variable of the type `int`.
    * Just like any other variables, a pointer has a size limit according to the size of its type.
      For example, we know that if a variable called `i` is of the type `int32_t`, `i`'s value can
      only range from the minimum value to the maximum value that `int32_t` can represent. This
      range is determined by the size of the type `int32_t`. Since it uses 32 bits, the minimum
      value is -2^31 (`INT32_MIN`) and the maximum value is 2^31-1 (`INT32_MAX`). Similarly, if a
      variable called `ptr` is of the type `int *`, its value can only range from the minimum value
      to the maximum value that `int *` can represent. All pointer types (`int *`, `char *`, `void
      *`, etc.) are of the same size, and they are typically either 32 bits or 64 bits, depending on
      what CPU you use. If you use a 32-bit CPU, it is 32 bits. If you use a 64-bit CPU, it is 64
      bits. For example, if the pointer types are represented as 32 bits, their values can only be
      between 0 and 2^32-1.
    * What this means is that the size of a pointer type effectively limits the size of the memory
      space that a program can use. For a 32-bit pointer, since its value can only range between 0
      and 2^32-1, the range of memory addresses must be between 0 and 2^32-1. This effectively
      limits the size of the memory address space.

Let's examine the address layout segment-by-segment from the bottom. There are some activities you
need to do, so don't forget to start recording with `record`.

### Text segment

The OS loads the program itself to this segment, i.e., the text segment contains the code of the
program. This means that your code resides somewhere in memory when you're running your program. In
fact, we can examine the memory and print out your (compiled) code at run time.

Let's create a file named `main_dump.c` and write the following program. (Don't forget that you need
to `record`.)

```c
#include <stdint.h>
#include <stdio.h>

int main(void) {
  int (*main_ptr)(void); // Define a function pointer
  main_ptr = &main; // Assign the address of main() to the pointer. If you'd like, you can omit `&`.
  uint8_t *start_address = (uint8_t *)main_ptr; // Cast it to an unsigned byte pointer for reading

  printf("Dumping memory from address %p:\n", start_address);

  for (size_t i = 0; i < 64; i++) {
    printf("%02X ", start_address[i]); // Hex formatting

    if ((i + 1) % 16 == 0) // Just print 16 bytes per line
      printf("\n");
  }

  return 0;
}
```

In the first three lines of `main()`, we use C's [function pointer
feature](https://en.wikipedia.org/wiki/Function_pointer) to get the address of the first byte of
`main()`'s code, stored in the text segment in memory.
* The first line demonstrates how to define a variable as a function pointer, since the syntax is
  not like any other variable definitions. `int (*main_ptr)(void);` says two things---(i) `main_ptr`
  is a variable that is a function pointer, and (ii) it points to a function that returns an integer
  (`int` at the beginning) and takes no parameters (`(void)` at the end).
* We define this function pointer exactly the same as `int main(void)`, except for the variable name
  `main_ptr` and `*` to indicate that we're defining a function pointer. This is because we want
  `main_ptr` to point to the function `main()`. This is in fact done in the second line, where we
  assign the address of `main()` to `main_ptr`. This effectively means that `main_ptr` now has the
  address of the first byte of `main()`'s code.
* The third line casts the function pointer's type to a regular unsigned byte pointer type, so we
  can easily access the values that it points to, byte by byte.
* You can replace the first three lines with a single line, `uint8_t *start_address = (uint8_t
  *)main;`. We're using those three lines for demonstration purposes.
* The `for` loop performs 64 iterations, where each iteration reads one byte via `start_address[i]`.
  Since `start_address[i]` reads the *i*th byte from the address stored in `start_address`, and
  `start_address` has the address of the function `main()`, the `for` loop effectively reads the
  first 64 bytes of the code in the function `main()`.
* The rest of the code is just printing and formatting, but one thing to note is that `%p` for
  `printf()` is the format string to use to print out a pointer value, i.e., a memory address.

Now, write a Makefile that produces an executable named `main_dump` with `make main_dump`. Make sure
you compile it with basic options, e.g., just `-o`, since we don't want the compiler to do extra
things that could make it difficult to access the text segment. Once that's done, compile and run it
to check the output. What's printed out is the first 64 bytes of `main()`'s compile code loaded in
the text segment in memory.

How do we know if this is correct code? There's a utility called `objdump` that prints out what is
in an executable. Similar to the memory layout, your executable (the file itself) is organized into
sections with similar names as the memory layout, e.g., the text section, the data section, etc.
`objdump` allows us to examine these sections. Enter `objdump -s <the name of the compiled
executable>` or, for easy navigation, pipe the output to `less` (`objdump -s <the name of the
compiled executable> | less`). It will show you what is in the executable, organized as sections.
Look for the text section and look for the matching bytes that you get from running your code. You
will find that all 64 bytes are present in the output of `objdump` in exactly the same order as our
program prints out.

### Data and BSS segments

The data and BSS segments store the values for static variables in a program. The data segment
stores *initialized* global or static variables, while the BSS segment stores *uninitialized* global
or static variables. For example, if you have `static char *example_string = "This is
initialized.\n";` somewhere in your program, it gets stored in the data segment. On the other hand,
if you have `static char *example_string;` somewhere in your program, it gets stored in the BSS
segment. Let's do a quick activity to understand this better. Make sure you `record`.

Create a file named `data_and_bss.c` and write the following code.

```c
#include <stdio.h>

char bss_char0;
char bss_char1;
char data_char0 = '0';
char data_char1 = '1';

int main() {
  printf("data_char0: %p\n", &data_char0);
  printf("data_char1: %p\n", &data_char1);
  printf("bss_char0: %p\n", &bss_char0);
  printf("bss_char1: %p\n", &bss_char1);

  return 0;
}
```

Before compiling and running this code, look at the code and look at the diagram above to think
about where these variables would be placed. Consider the fact that the data segment is for
initialized global or static variables and the BSS segment is for uninitialized global or static
variables.

Now, let's compile and run it to see which addresses we get for these variables. In the same
Makefile from the above [Text segment](#text-segment) section, add a new target named `data_and_bss`
that produces an executable of the same name (`data_and_bss`). Once that's done, compile it and run
it. You should get an output similar to the following:

```c
data_char0: 0xaaaacdfe1038
data_char1: 0xaaaacdfe1039
bss_char0: 0xaaaacdfe103b
bss_char1: 0xaaaacdfe103c
```

Based on this output, you can draw a diagram that visualizes how these are stored such as the
following:

+────────────+
| bss_char1  | 0xaaaacdfe103c
+────────────+
| bss_char0  | 0xaaaacdfe103b
+────────────+
|            | 0xaaaacdfe103a
+────────────+
| data_char1 | 0xaaaacdfe1039
+────────────+
| data_char0 | 0xaaaacdfe1038
+────────────+

There are a few important points to observe:

* As we can see, each variable takes up exactly one byte because of their type (`char`) that has the
  size of one byte.
* `data_char0` gets placed at a *lower* address than `data_char1` because we define `data_char0`
  *before* `data_char1` in the program.
* Similarly, `bss_char0` gets placed at a lower address than `bss_char1` due to the order of
  definition in the program.
* `data_char0` and `data_char1` are placed in the data segment (since they are initialized), and
  `bss_char0` and `bss_char1` are placed in the BSS segment (since they are not initialized). We can
  see this from their addresses---`bss_char0` and `bss_char` are placed on top of `data_char0` and
  `data_char1`. (There is a one-byte gap in the output. The reason is that the linker automatically
  adds some start-up code for a C program, which uses the BSS segment.)

Why do we need a separate BSS segment? Why can't it be just the data segment? The answer is that the
BSS segment is automatically filled with zeros so we can still initialize uninitialized global and
static variables. This helps ensure that these variables start with a known and predictable value.

### Stack

The stack is the memory space where local variables, function parameters, and function return values
are stored. The size of the stack typically changes as your program runs

### Memory mapping

### Heap

### Kernel address space

This space is only used by the Linux kernel and user programs cannot access it. If a user program
accesses this address space, the OS will terminate the program with a segmentation fault.
