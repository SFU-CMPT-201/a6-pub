# Assignment 7: Memory layout

When you have a program as an executable, it is a file stored on your disk. However, when you run
the program, your OS loads the program to the main memory and starts executing the program. This
means that there exist two copies of the program when you run it---one copy on the disk and another
copy in memory. At this point, the program/executable/file on your disk doesn't really matter. What
matters is the copy of the program in memory.

When an OS loads a program to memory, it uses a particular format suitable for executing a program.
This format is called *memory layout*. Understanding this layout is crucial for writing more
reliable and correct code, and also for deepening your understanding of pointers and memory
management. In this assignment, you will learn the background and do some activities.

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
|             Memory mapping            |
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
* We assume that you know this already, but in a C program, the value stored in a pointer is a
  memory address. For example, `int *ptr = 0` means that the pointer `ptr` now holds the memory
  address 0 as its value. Here, `ptr` is a variable that has the type `int *` (which makes `ptr` a
  pointer), similar to how `int i` means `i` is a variable that has the type `int`. Just like any
  other variables, a pointer has a size limit according to the size of its type. For example, we
  know that if a variable called `i` is of the type `int32_t`, `i`'s value can only range from the
  minimum value to the maximum value that `int32_t` can represent. This range is determined by the
  size of the type `int32_t`. Since it uses 32 bits, the minimum value is -2^31 (`INT32_MIN`) and
  the maximum value is 2^31-1 (`INT32_MAX`). Similarly, if a variable called `ptr` is of the type
  `int *`, its value can only range from the minimum value to the maximum value that `int *` can
  represent. All pointer types (`int *`, `char *`, `void *`, etc.) are of the same size, and they
  are typically either 32 bits or 64 bits, depending on what CPU you use. If you use a 32-bit CPU,
  it is 32 bits. If you use a 64-bit CPU, it is 64 bits. For example, if the pointer types are
  represented as 32 bits, their values can only be between 0 and 2^32-1
* What this means is that the size of the pointer types effectively limits the size of the memory
  space that a program can use. For a 32-bit pointer, since its value can only range between 0 and
  2^32-1, the range of memory addresses must be between 0 and 2^32-1. This effectively limits the
  size of the memory address space.
* The memory layout consists of *segments*, e.g., the kernel address segment, the stack segment,
  etc.

Let's examine the address layout segment-by-segment from the bottom. There are some activities you
need to do, so don't forget to start recording with `record`.

### Text segment

The OS loads the program itself to this segment, i.e., the text segment contains the code of the
program. This means that your code resides somewhere in memory when you run your program. Indeed, we
can examine the memory and print out your (compiled) code at run time.

Let's create a file named `main_dump.c` and write the following program. (Don't forget that you need
to `record`.)

```c
#include <stdint.h>
#include <stdio.h>

int main(void) {
  int (*main_ptr)(void); // Define a function pointer
  main_ptr = main; // Assign the address of main()
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
feature](https://en.wikipedia.org/wiki/Function_pointer) to get a pointer to the address of
`main()`'s code, stored in the text segment in memory. The first line demonstrates how to declare a
variable as a function pointer, since the syntax is not like other variables. It has a variable name
(`main_ptr`) and a function pointer type that returns an integer (`int` at the beginning) and takes
no parameters (`(void)` at the end). The function pointer type is defined exactly the same as `int
main(void)` (except for `*` that indicates a function pointer), since we want `main_ptr` to point to
the address of `main()` (as done in the second line). The third line casts the function pointer's
type to a regular pointer type, so we can easily access the values byte by byte. You don't actually
need three lines and replace them with a single line, `uint8_t *start_address = (uint8_t *)main;`.
The rest of the code is just printing and formatting, but one thing to note is that `%p` for
`printf()` is the format string to use to print out a pointer value, i.e., a memory address.

Now, compile and run the code to check the output. Make sure you compile it with basic options,
e.g., just `-o` since we don't want the compiler to do extra things. What's printed out is the first
64 bytes of `main()`'s compile code loaded in the text segment in memory. How do we know if this is
correct code? There's a convenient utility called `objdump` that prints out what is in an
executable. Enter `objdump -s <the name of the compiled executable>` or for easy navigation, pipe
the output to `less` (`objdump -s <the name of the compiled executable> | less`). It will show you
different sections contained in the executable. Look for the text segment and look for the matching
bytes that you get from running your code. You will find that all 64 bytes are present in exactly
the same order.

### Data and BSS segments

The data and BSS segments store the values for static variables in a program. The data segment
stores *initialized* static variables, while the BSS segment stores *uninitialized* static
variables. For example, if you have `static char *example_string = "This is initialized.\n";`
somewhere in your program, it gets stored in the data segment. On the other hand, if you have
`static char *example_string;` somewhere in your program, it gets stored in the BSS segment. Let's
do a quick activity to understand better.

Create a file named `data_and_bss.c` and write the following code.

```c
#include <stdio.h>

int main() {
  static char bss_char;
  static char data_char0 = '0';
  static char data_char1 = '1';

  printf("bss_char: %p\n", &bss_char);
  printf("data_char0: %p\n", &data_char0);
  printf("data_char1: %p\n", &data_char1);

  return 0;
}

```

### Stack

This is the space where local variables, function parameters, and function return values are stored.

### Memory mapping

### Heap

### Kernel address space

This space is only used by the Linux kernel and user programs cannot access it. If a user program
accesses this address space, the OS will terminate the program with a segmentation fault.
