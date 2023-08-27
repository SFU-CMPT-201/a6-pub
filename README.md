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

## Linux memory layout and C pointers

Every single thing in your program, all the statements, all the variables, and all the function
calls, occupies memory space when you run the program. Your OS organizes the memory space in a
certain way using its pre-defined memory layout and your program's code interacts with the memory
space to actually run the program.

Below is a simplified diagram of how Linux organizes the memory space when it runs a program.

```bash
.──────────────────────.
|                      | Address 2^n - 1
| Kernel address space | (n is the number of address bits)
|                      |
+──────────────────────+
|                      |
| Stack                |
| (grows ↓)            |
|                      |
+──────────────────────+
|                      |
|                      |
+──────────────────────+
|                      |
| Memory mapping       |
| (grows ↓)            |
|                      |
+──────────────────────+
|                      |
|                      |
+──────────────────────+
|                      |
| Heap                 |
| (grows ↑)            |
|                      |
+──────────────────────+
|                      |
+──────────────────────+
|                      |
| BSS                  |
| (Uninitialized       |
|  global              |
|  or static data)     |
|                      |
+──────────────────────+
|                      |
| Data                 |
| (Initialized         |
|  global              |
|  or static data)     |
|                      |
+──────────────────────+
|                      |
| Text (Code)          |
|                      |
+──────────────────────+
|                      |
|                      |
|                      | Address 0
'──────────────────────'
```

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

## Task 0: Understanding the text segment

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

(You can stop recording and come back later, or continue with the next task.)

## Task 1: Understanding the data and BSS segments

The data and BSS segments store the values for static variables in a program. The data segment
stores *initialized* global or static variables, while the BSS segment stores *uninitialized* global
or static variables. For example, if you have `static char *example_string = "This is
initialized.\n";` somewhere in your program, it gets stored in the data segment. On the other hand,
if you have `static char *example_string;` somewhere in your program, it gets stored in the BSS
segment. Let's do an activity to understand this better. Make sure you `record`.

Create a file named `data_and_bss.c` and write the following code.

```c
#include <stdio.h>

char bss_char0;
char bss_char1;
char data_char0 = '0';
char data_char1 = '1';

int main() {
  printf("data_char0 address: %p\n", &data_char0);
  printf("data_char1 address: %p\n", &data_char1);
  printf("bss_char0 address: %p\n", &bss_char0);
  printf("bss_char1 address: %p\n", &bss_char1);

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

```bash
data_char0 address: 0xaaaacdfe1038
data_char1 address: 0xaaaacdfe1039
bss_char0 address: 0xaaaacdfe103b
bss_char1 address: 0xaaaacdfe103c
```

Based on this output, you can draw a diagram that visualizes how these are stored such as the
following:

```bash
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
```

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

(You can stop recording and come back later, or continue with the next task.)

## Task 2: Understanding the stack segment

The stack segment, sometimes referred to as *the call stack* or just *the stack*, is the memory
space where local variables, function parameters, and function return values are stored. A typical
program has multiple functions, and the stack's size changes as different functions get called.
Let's do a couple of activities to learn about this more. Make sure you `record` if you are not
doing it already.

### Stack basics

Create a file named `stack_basics.c` and write the code below. Also, add a new target `stack_basics`
to the same Makefile from previous sections that produces an executable of the same name with `make
stack_basics`.

```c
#include <stdint.h>
#include <stdio.h>

void test0(void) {
  int8_t local0 = 1;
  int8_t local_array[3] = {2, 3, 4};
  int8_t local1 = 5;

  printf("test0:\n");
  printf("  local0 address: %p\n", &local0);
  printf("  local_array[0] address: %p\n", local_array);
  printf("  local_array[1] address: %p\n", (local_array + 1));
  printf("  local_array[2] address: %p\n", (local_array + 2));
  printf("  local1 address: %p\n", &local1);
}

int main(void) {
  test0();
}
```

Compile and run it. You should get an output similar to the following:

```bash
test0:
  local0 address: 0xffffdf138fbf
  local_array[0] address: 0xffffdf138fbc
  local_array[1] address: 0xffffdf138fbd
  local_array[2] address: 0xffffdf138fbe
  local1 address: 0xffffdf138fbb
```

You can visualize this as follows:

```bash
+────────────────+
|     local0     | 0xffffdf138fbf
+────────────────+
| local_array[2] | 0xffffdf138fbe
+────────────────+
| local_array[1] | 0xffffdf138fbd
+────────────────+
| local_array[0] | 0xffffdf138fbc
+────────────────+
|     local1     | 0xffffdf138fbb
+────────────────+
```

These local variables are placed in the stack in the order of definition. However, the stack grows
*downward*, meaning that a variable occupies a higher address first. In the above example, `local0`
occupies the highest address above `local_array`. In turn, `local_array` occupies the next highest
address above `local1`. Notice that for `local_array`, a lower index still occupies a lower address.

Since all the variables are placed consecutively in the stack, we need to be careful when using
pointers or arrays, as we can inadvertently access other variables. Add the following function to
`stack.c` and call the function from `main()`.

```c
void test1(void) {
  int8_t local0 = 1;
  int8_t local_array[3] = {2, 3, 4};
  int8_t local1 = 5;

  printf("test1:\n");
  printf("  local0 value: %d\n", local0);

  local_array[3] = 24;

  printf("  local0 value: %d\n", local0);
}
```

If you add this code, our linter will give a warning regarding the array usage. As the warning
message says, `local_array` only has three elements. However, `local_array[3]` is accessing the 4th
element, which `local_array` does not have by definition. Regardless, we can still use the
expression `local_array[3]` because an array in C is nothing more than a convenient pointer.
`local_array[3]` is simply `*(local_array + 3)`.

Now, compile the code. The compiler will also give an array warning but we can ignore it for the
sake of our activity. Run it and check the output.

Excluding the output from `test0()`, your output should show the following:

```bash
test1:
  local0 value: 1
  local0 value: 24
```

In the code, `local0` is initialized to 1, and there is no other place where we directly change the
value of `local0`. However, the second `printf()` prints out 24, not 1. What happened? The reason is
that `local_array[3]` points to the memory address of `local0`. So when we assign 24 to
`local_array[3]`, we are writing to the memory space occupied by `local0`. Let's visualize this.

```bash
+───────────────────────────────+
| local0 (also, local_array[3]) | 0xffffdf138fbf
+───────────────────────────────+
| local_array[2]                | 0xffffdf138fbe
+───────────────────────────────+
| local_array[1]                | 0xffffdf138fbd
+───────────────────────────────+
| local_array[0]                | 0xffffdf138fbc
+───────────────────────────────+
| local1                        | 0xffffdf138fbb
+───────────────────────────────+
```

In fact, this is a very common error called *array out of bounds*. As the name suggests, it is an
error where an array variable is used to access memory locations outside the array's bounds.

Beside arrays, we can use any local variable to manipulate other local variables. Add the following
function and call it from `main()`.

```c
void test2(void) {
  int8_t local_array[3] = {2, 3, 4};
  int8_t local0 = 5;

  printf("test2:\n");
  printf("  local_array[1] value: %d\n", local_array[1]);

  int8_t *ptr = &local0 + 2; // This points to `local_array[1]`
  *ptr = 36;

  printf("  local_array[1] value: %d\n", local_array[1]);
}
```

Before running this code, think about what the output will be. The first `printf()` should print out
the value of `local_array[1]`. How about the second `printf()`? In `int8_t *ptr = &local0 + 2;` we
are adding 2 to the address of `local0`, which is the location for `local_array[1]`. Since `ptr`
points to `local_array[1]`, `*ptr = 36;` should assign 36 to `local_array[1]`. We can visualize this
again.

```bash
+──────────────────────────────────+
| local_array[2]                   | 0xffffdf138fbe
+──────────────────────────────────+
| local_array[1] (or *(&local0+2)) | 0xffffdf138fbd
+──────────────────────────────────+
| local_array[0]                   | 0xffffdf138fbc
+──────────────────────────────────+
| local0                           | 0xffffdf138fbb
+──────────────────────────────────+
```

The important point here is that it is possible to use a local variable's address to access other
local variables. This often causes problems, generally called *stack corruption*---you access stack
locations unintentionally and corrupts the stack. Thus, we need to be extremely careful when using
arrays and pointers.

(You can stop recording and come back later, or continue on.)

### Stack with function calls

The stack not only stores local variables but also stores arguments passed for function calls. Since
there are potentially many function calls, the stack organizes a function's local variables and
arguments in a place called a *stack frame*. Whenever there is a new function call, the stack
creates a new stack frame and stores local variables, arguments, and other things necessary.

The stack with stack frames looks like the following.

```bash
+─────────────────────+ Higher address
| main() stack frame  |
+─────────────────────+
| foo() stack frame   |
+─────────────────────+
| (grows down)        |
+─────────────────────+ Lower address
```

Here, we are showing an example where, inside `main()`, there is a function call to `foo()`. For
each function call, a new stack frame gets created and pushed to the stack, and it grows downward.
Thus, if `foo()` makes a function call to `bar()`, the stack will look like the following.

```bash
+─────────────────────+ Higher address
| main() stack frame  |
+─────────────────────+
| foo() stack frame   |
+─────────────────────+
| bar() stack frame   |
+─────────────────────+
| (grows down)        |
+─────────────────────+ Lower address
```

If `bar()` is done and the execution returns to `foo()`, the stack frame for `bar()` will be
popped, and the stack will look like the following.

```bash
+─────────────────────+ Higher address
| main() stack frame  |
+─────────────────────+
| foo() stack frame   |
+─────────────────────+
| (grows down)        |
+─────────────────────+ Lower address
```

Now, a single stack frame *generally* looks like the diagram below, but it depends on many factors
such as your CPU, compiler, OS, configuration, etc. The diagram uses the basic Linux configuration
of the 32-bit x86 CPUs as an example.

```bash
+────────────────────────+ Higher address
| (more arguments)       |
+────────────────────────+
| argument0              |
+────────────────────────+
| Return address         |
+────────────────────────+
| Previous frame pointer |
+────────────────────────+
| local_variable0        |
+────────────────────────+
| (more local variables) |
+────────────────────────+ Lower address
```

Below are a few points about the diagram. We use *caller* and *callee* to distinguish the function
that makes a function call (caller) and the function that gets invoked (callee).

* Function arguments occupy the highest-addressed region of a stack frame. (This is traditionally
  called the *bottom* of the stack.)
* A stack frame saves the *return address*, which points to the caller's code that needs to be
  executed right after the function call returns. This is necessary to remember where to go back
  when the callee's code execution is done.
* A stack frame also saves the *previous frame pointer*, which points to the stack frame of the
  caller. This is necessary when popping the callee's stack frame after the callee's code execution
  is done.

The important point here is again the same point made earlier. Since these are all stored in the
same memory region, it is possible to use arrays/pointers of local variables to access other stack
frames, leading to potential stack corruption. Let's do an activity to understand this more. Make
sure you `record` if you are not doing it already.

Create a file named `stack_frames.c` and write the code below. Also, add a new target `stack_frames`
to the same Makefile from previous sections that produces an executable of the same name with `make
stack_frames`. We will first print out the addresses of the variables in the program to understand
where they are located in the stack.

```c
#include <stdint.h>
#include <stdio.h>

void foo(uint8_t argument) {
  uint8_t foo_local = 16;
  printf("Local variable address in foo: %p\n", &foo_local);
}

int main(void) {
  uint8_t main_local = 32;
  foo(main_local);
  printf("Local variable address in main: %p\n", &main_local);
}
```

Compile and run the program. You will get an output similar to the following:

```bash
Local variable address in foo: 0xffffd064d30e
Local variable address in main: 0xffffd064d32f
```

The address difference is 0x21 or 33 bytes. Understanding this requires an in-depth discussion about
how things work, which we won't get into here. But briefly, our VM uses a 64-bit CPU and a stack
allocation always takes up a multiple of 16 bytes for faster execution. Thus, in order to store
`uint8_t main_local` in `main()`, the stack allocates 16 bytes instead of one byte. After that, the
return address and the previous frame pointer shown in the diagram above take up 8 bytes each.
Lastly, `uint8_t argument` takes up another one byte. Thus, those four variables take up 33 bytes in
the stack. The local variable `uint8_t foo_local` comes after those variables in the stack, so the
address difference between `uint8_t foo_local` and `uint8_t main_local` is 33 bytes.

Needless to say, misusing arrays or pointers with `uint8_t foo_local` in `foo()` can affect `uint8_t
argument`, the return address, the previous frame pointer, and `uint8_t main_local`. In fact, this
can cause a very serious problem known as *stack buffer overflow attack*, where the return address
is replaced with an address that points to a malicious piece of code by overflowing a buffer. The
simplest example, though not an actual attack, is something like the following:

```c
#include <stdio.h>

int main(void) {
  char buffer[5] = {0};
  scanf("%s", buffer);
}
```

Create a file named `buffer_overflow.c` and write the above code. You will immediately see that our
linter complains about `scanf()` being insecure, which is exactly what this example is about. Add a
new target for `make buffer_overflow` to produce an executable named `buffer_overflow`. Compile and
run it. `scanf()` requires an input, so provide `this-is-a-long-string` as an input. It is going to
cause a segmentation fault.

Since this is a common, yet serious problem, there is a sanitizer that detects this problem. In the
Makefile, add a new target named `buffer_overflow_sanitizer`, that produces an executable of the
same name. There, enable the `AddressSanitizer` by adding a compiler option `-fsanitize=address`.
Compile and run it. Provide the same input `this-is-a-long-string`. It should immediately show a
long error message that explains the stack buffer overflow error.

The reason is that the size of `buffer` is 5 bytes and `this-is-a-long-string` is 22 bytes, which is
clearly more than 5 bytes. When `scanf()` stores it in `buffer`, it *overflows* and overruns the
stack as follows.

```bash
+─────+
| ... |
+─────+
| '-' | buffer[7] (beyond the bound; overwriting the stack)
+─────+
| 's' | buffer[6] (beyond the bound; overwriting the stack)
+─────+
| 'i' | buffer[5] (beyond the bound; overwriting the stack)
+─────+
| '-' | buffer[4]
+─────+
| 's' | buffer[3]
+─────+
| 'i' | buffer[2]
+─────+
| 'h' | buffer[1]
+─────+
| 't' | buffer[0]
+─────+
```

Other standard library functions that read user inputs, e.g., `gets()`, have similar problems and
their use is explicitly discouraged. For example, if you look at the manpage of `gets()` (`man
gets`), you will see that the description says `Never use this function`. You can use a safer
alternative, such as `fgets()`, where you need to specify the size that you can read. (Of course,
you need to provide the right size to be safe.)

In the next assignment, we will continue the discussion on Linux's memory layout. Make sure you
submit all the files you created for this assignment, including `.record/` and `.vim/`.

# Next steps

You need to accept the invite for the next assignment (A8).

* Go to this URL: [https://classroom.github.com/a/PGPiPLGw](https://classroom.github.com/a/PGPiPLGw)
* Accept the invite for Assignment 8 (A8).
* If you are not in `units/03-memory` directory, go to that directory.
* Clone the assignment repo.
