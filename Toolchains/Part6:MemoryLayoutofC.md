# C Memory Layout

The memory layout of a program shows how its data is stored in memory during
execution.
It helps developers understand and manage memory efficiently.

* Memory is divided into sections such as `text`, `data`, `heap`, and `stack`.
* Knowing the memory layout is useful for optimizing performance, debugging and
  prevent errors like segmentation fault and memory leak.

## 1. `Text` section (code)
* The `text` section (or code section) stores the executable code of the program
  like program’s functions and instructions.
* The `text` section is usually read-only to prevent accidental modification
  during execution.
* It is typically stored in the lower part of memory.
* The size of the `text` section depends on the number of instructions and the
  program’s complexity.

Example:

```c
#include <stdio.h>
void greet(const char *name) { ... }  /* → text section */
int  main()                  { ... }  /* → text section */
```

## 2. `Data` section
* The `data` section stores **global variables** and **static variables** that
  are initialized by the programmer.

* It is divided into two parts:
    * Read-only data section (`.rodata`)
    * Writable data section (`.data`)

* Initialized global/static variables → `.data` section
* Uninitialized global/static variables → `.bss` section
* Static local variables → `.data` section (if initialized) or 
  `.bss` (if uninitialized)

#### Initialized data section (`.data`)
* Initialized global variables → `.data` section
* Static local variables → `.data` section (if initialized)
* Static global variables → `.data` section

Example:
```c
int global_init = 42; // .data section
static int s_count = 0; // .data section

int main() {
    static int s_count = 10; // .data section
....
}
```

#### Read only data section (`.rodata`)
* String literals → `.rodata` section
* Constant variables → `.rodata` section

Example:
```c
const char *msg = "hello"; // "hello" in .rodata, msg in .data
const int MAX = 256; // MAX in .rodata
const char VER[] = "1.0.0"; // VER in .rodata
```

### Uninitialized data section (`.bss`)
* Uninitialized global variables → `.bss` section
* Static local variables → `.bss` section (if uninitialized)
* Static global variables → `.bss` section

**BSS** variable are initialized to 0 by the os at load time and this section
does not take any space in the executable file. because it only contains the
size.

```
char big_buffer[1000000]; // 1MB array
```
If ELF stored `one million` zeros physically, executable size would become huge.
Instead:
ELF says:
"Reserve 1,000,000 bytes here and initialize to zero"

```bash
readelf -S a.out
```

**Output:**
```
[Nr] Name      Type     Addr      Off    Size
[24] .bss      NOBITS   00004040  0040   0x0f4240
```
**Type: NOBITS**

**Meaning:**
* occupies memory at runtime
* occupies NO space in file


Example:
```c
int global_uninit; // .bss section
static int s_count; // .bss section
static char str2[]; // str2 in .bss
```

[[NOTE]]
Initialized and Unintialized are with respect to programmer. If 
programmer does not initialize a variable, it is considered as uninitialized.
 even if static variable is initialized to 0 or NULL, it is still considered 
as uninitialized.

## 4. Heap
* The heap is a region of memory that is used for dynamic memory allocation.
* It is used to store variables that are created during the execution of the
  program.
* The heap is managed by the programmer and is used to store variables that are
  created during the execution of the program.

Example:
```c
int *p = malloc(64); // p on STACK, block on HEAP
```

## 5. Stack
* The stack is a region of memory that is used for local variables and function
  calls.
* It is used to store variables that are created during the execution of the
  program.
* The stack is managed by the compiler and is used to store variables that are
  created during the execution of the program.

Example:
```c
int x = 7; // STACK: local int
```

# Explain with one example

## The Example C Program

Save this as `mem.c` — all verification steps in this guide use it.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* Initialized global → DATA segment */
int global_init   = 42; //.data section

/* Uninitialized global → BSS segment (zeroed by OS) */
int global_uninit; //.bss section

/* Static local: file scope, init value → DATA segment */
static int s_count = 0;

/* Pointer variable in DATA; points into RODATA */
const char *msg = "hello";

/* pointer variable in rodata, points to rodata*/
const char * const msg2 = "hello2";

void greet(const char *name) {
    char buf[32];                     /* STACK: local array */
    snprintf(buf, 32, "Hi %s", name);
    puts(buf);
}

int main() {
    int x = 7;                        /* STACK: local int */
    char *p = malloc(64);             /* p on STACK, block on HEAP */
    strcpy(p, "dynamic");
    greet("world");
    free(p);
    return 0;
}
```

## Memory Segments Overview

A C program's virtual address space is divided into fixed segments loaded by
the OS, plus two runtime-managed regions.

```
High address  ┌─────────────────────────┐
              │   Kernel space          │  (inaccessible to user code)
              ├─────────────────────────┤
              │   Stack                 │  ← grows downward
              │   (local vars, frames)  │
              │         ↓               │
              │                         │
              │         ↑               │
              │   Heap                  │  ← grows upward
              │   (malloc / free)       │
              ├─────────────────────────┤
              │   BSS                   │  uninit globals, zeroed
              ├─────────────────────────┤
              │   Data                  │  init globals & statics
              ├─────────────────────────┤
              │   RODATA                │  string literals, const
              ├─────────────────────────┤
              │   Text                  │  machine code (read+exec)
Low address   └─────────────────────────┘
```


## Section-by-Section Breakdown

### 1. Text Section

Contains the compiled machine instructions for every function. Marked
read-only and executable by the OS — writing to it causes a segfault.

```c
void greet(const char *name) { ... }  /* → text segment */
int  main()                  { ... }  /* → text segment */
```

- Functions `greet` and `main` have fixed addresses assigned by the linker.
- The CPU fetches instructions from this region.
- Shared between processes running the same binary (copy-on-write mapping).

### 2. RODATA (Read-Only Data)

String literals and `const`-qualified globals are placed here by the compiler. 
The OS marks this region non-writable — writing to it causes a segfault.

```c
const char *msg    = "hello";   /* "hello\0" in .rodata; msg pointer in .data */
const int   MAX    = 256;       /* MAX in .rodata */
const char  VER[]  = "1.0.0";  /* array in .rodata */

/* Danger: */
char *bad = "test"; /* bad is in .data and "test" is in .rodata */
bad[0] = 'X';  /* SIGSEGV — .rodata is read-only */
```

**Key distinction:** `msg` (the pointer variable) lives in `.data`. The string
bytes `"hello\0"` live in `.rodata`. Two different locations.

Identical string literals may be merged by the compiler into a single address
(controlled by `-fmerge-constants`).

### 3. Data Segment (Initialized data segment, Globals)

Global and `static` variables that have an explicit initializer. Their initial
values are stored inside the ELF binary on disk and loaded at startup.

```c
int  global_init = 42;    /* .data — value 42 stored in binary */
static int s_count = 0;   /* .data — static local, one copy ever */
```

- Address is fixed for the entire program lifetime.
- Calling a function that modifies `s_count` increments the same memory slot
  every time — no new allocation per call.
- Costs disk space proportional to the data size.

**If the initialized variable value is large like array[1000000] then that much
space will be occupied in the executable file which will increase the size of
the executable file. And hence will increase the size of the binary. But in case
of `.bss` we can declare the array of any size and it will not increase the size
of the binary. Because the OS will allocate the memory for the `.bss` segment at
the time of loading and will zero it out. It will not store the initial values
of the `.bss` segment in the executable file**

If we add array[1000000] to the above c file then the output of `size mem` will
be:

* size before arr[1000000]

```bash
int arr[10000000] = {0}; // Stored in .bss (Safe) optimized by compiler
int arr[10000000] = {1}; // Stored in .data

size mem
   text	   data	    bss	    dec	    hex	filename
   1992	40000688	16	40002696	2626488	mem
```


### 4. BSS Segment (Uninitialized data segment )

Global and `static` variables with no initializer, or initialized to zero. The C
standard guarantees they are zero-initialized before `main()` runs. The OS
performs this zeroing — the BSS section occupies **zero bytes** in the ELF file
on disk.

```c
int  global_uninit;        /* .bss — guaranteed 0, no disk cost */
static int call_count;     /* .bss — same guarantee */
```

**Practical test:** Add `int arr[10000000];` at file scope and run `size mem`.
BSS grows by 40,000 bytes; the binary on disk stays the same size.

* size before arr[10000000]

```bash
$ size mem
   text	   data	    bss	    dec	    hex	filename
   1992	    648	     16	   2656	    a60	mem
```


Add `int arr[10000000];` at file scope and compare the output of `size mem`
again. You will see that the `bss` segment has increased by 40,000 bytes, but
the executable file size on disk has not changed.

* size after arr[10000000]

```bash
$ size mem
   text	   data	    bss	    dec	    hex	filename
   1992	    648	40000072	40002712	2626498	mem
```

### 5. Heap

Memory allocated dynamically at runtime via `malloc` / `calloc` / `realloc`.
Grows upward from a low base address. The allocator (`glibc`'s ptmalloc)
manages an internal arena; `free()` returns memory to that arena.

```c
char *p   = malloc(64);       /* p: 8-byte pointer on stack */
                               /* block: 64 bytes on heap   */
strcpy(p, "dynamic");
free(p);                       /* must call or memory leaks */
```

**Critical rule:** Only the _pointer variable_ is on the stack(8 bytes on x86-64).
The allocated block is on the heap until `free()` is called explicitly. There
is no automatic reclamation.

```c
/* Memory leak: p goes out of scope, block is never freed */
void leak() {
    char *p = malloc(64);
    /* forgot free(p) */
}
```

### 6. Stack

Local variables, function parameters, return addresses, and saved
registers.Grows downward from a high base address. Each function call pushes a
_stack frame_; returning pops it — the memory is instantly reclaimed.

```c
int main() {
    int  x = 7;          /* rbp-4  on stack */
    char *p = malloc(64);/* rbp-8  on stack (pointer only) */
    greet("world");      /* pushes new frame for greet */
    return 0;            /* pops main's frame */
}
```

**Never return a pointer to a local variable:**

```c
int *dangerous() {
    int local = 99;
    return &local;  /* UNDEFINED BEHAVIOR: frame gone after return */
}
```

---

## Stack Frames In Depth

When `main()` calls `compute()` which calls `add()`, three frames are live
simultaneously:

```c
int add(int a, int b) {
    int result = a + b;   /* stack */
    return result;
}

int compute(int n) {
    int tmp = n * 2;      /* stack */
    return add(tmp, 5);
}

int main() {
    int x  = 10;          /* stack */
    int y  = compute(x);
    printf("%d\n", y);
    return 0;
}
```

Stack layout at the deepest point (inside `add`), addresses decreasing downward:

```
High addr  ┌──────────────────────────────────┐
           │ main's frame                     │
           │   return addr → (caller of main) │
           │   rbp-4   x = 10                 │
           │   rbp-8   y = ? (pending)        │
           ├──────────────────────────────────┤
           │ compute's frame                  │
           │   return addr → main             │
           │   rbp-4   n = 10  (copy)         │
           │   rbp-8   tmp = 20               │
           ├──────────────────────────────────┤
           │ add's frame   ← SP here          │
           │   return addr → compute          │
           │   rbp-4   a = 20  (copy)         │
           │   rbp-8   b = 5   (copy)         │
           │   rbp-12  result = 25            │
Low addr   └──────────────────────────────────┘
```

When `add` returns: SP moves back up, frame gone. `result`'s memory is no longer
valid.
