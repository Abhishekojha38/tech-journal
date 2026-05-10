# C Memory Layout

The memory layout of a program shows how its data is stored in memory during execution. It helps developers understand and manage memory efficiently.

* Memory is divided into sections such as code, data, heap, and stack.
* Knowing the memory layout is useful for optimizing performance, debugging and prevent errors like segmentation fault and memory leak.

## 1. `Text` segment (code)
* The text segment (or code segment) stores the executable code of the program like program’s functions and instructions.
* The segment is usually read-only to prevent accidental modification during execution.
* It is typically stored in the lower part of memory.
* The size of the text segment depends on the number of instructions and the program’s complexity.

## 2. Data segment
* The data segment stores initialized data and static variables.
* It is divided into two parts:
    * Read-only data segment
    * Writable data segment
* Initialized global variables → `.data` segment
* Uninitialized global variables → `.bss` segment
* Static local variables → `.data` segment (if initialized) or `.bss` (if uninitialized)

### Read only data segment (`.rodata`)
* String literals → `.rodata` segment
* Constant variables → `.rodata` segment

Example:
```c
const char *msg = "hello"; // "hello" in .rodata, msg in .data
const int MAX = 256; // MAX in .rodata
const char VER[] = "1.0.0"; // VER in .rodata
```

### Writable data segment (`.data`)
* Initialized global variables → `.data` segment
* Static local variables → `.data` segment (if initialized)
* Static global variables → `.data` segment

Example:
```c
int global_init = 42; // .data segment
static int s_count = 0; // .data segment
const char *msg = "hello"; // "hello" in .rodata, msg in .data
const char * const msg2 = "hello2"; // msg2 in .rodata, "hello2" in .rodata
```

## 3. `BSS` segment (Uninitialized data segment )
* Uninitialized global variables → `.bss` segment
* Static local variables → `.bss` segment (if uninitialized)
* Static global variables → `.bss` segment

Example:
```c
int global_uninit; // .bss segment
static int s_count; // .bss segment
static char str2[]; // str2 in .bss
```

NOTE:Initialized and Unintialized are with respect to programmer. If programmer does not initialize a variable, it is considered as uninitialized. even if static variable is initialized to 0 or NULL, it is still considered as uninitialized.

## 4. Heap
* The heap is a region of memory that is used for dynamic memory allocation.
* It is used to store variables that are created during the execution of the program.
* The heap is managed by the programmer and is used to store variables that are created during the execution of the program.

Example:
```c
int *p = malloc(64); // p on STACK, block on HEAP
```

## 5. Stack
* The stack is a region of memory that is used for local variables and function calls.
* It is used to store variables that are created during the execution of the program.
* The stack is managed by the compiler and is used to store variables that are created during the execution of the program.

Example:
```c
int x = 7; // STACK: local int
```

## 6. The Example C Program

Save this as `mem.c` — all verification steps in this guide use it.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* Initialized global → DATA segment */
int global_init   = 42;

/* Uninitialized global → BSS segment (zeroed by OS) */
int global_uninit;

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

A C program's virtual address space is divided into fixed segments loaded by the OS, plus two runtime-managed regions.

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

| Segment | Writable | On Disk | Lifetime        | Who manages it |
|---------|----------|---------|-----------------|----------------|
| Text    | No       | Yes     | Program         | Linker         |
| RODATA  | No       | Yes     | Program         | Linker         |
| Data    | Yes      | Yes     | Program         | Linker         |
| BSS     | Yes      | No      | Program         | OS (zeroes)    |
| Heap    | Yes      | No      | Until `free()`  | Programmer     |
| Stack   | Yes      | No      | Until return    | CPU / compiler |

## Segment-by-Segment Breakdown

### 1. Text Segment

Contains the compiled machine instructions for every function. Marked read-only and executable by the OS — writing to it causes a segfault.

```c
void greet(const char *name) { ... }  /* → text segment */
int  main()                  { ... }  /* → text segment */
```

- Functions `greet` and `main` have fixed addresses assigned by the linker.
- The CPU fetches instructions from this region.
- Shared between processes running the same binary (copy-on-write mapping).

### 2. RODATA (Read-Only Data)

String literals and `const`-qualified globals are placed here by the compiler. The OS marks this region non-writable — writing to it causes a segfault.

```c
const char *msg    = "hello";   /* "hello\0" in .rodata; msg pointer in .data */
const int   MAX    = 256;       /* MAX in .rodata */
const char  VER[]  = "1.0.0";  /* array in .rodata */

/* Danger: */
char *bad = "test"; /* bad is in .data and "test" is in .rodata */
bad[0] = 'X';  /* SIGSEGV — .rodata is read-only */
```

**Key distinction:** `msg` (the pointer variable) lives in `.data`. The string bytes `"hello\0"` live in `.rodata`. Two different locations.

Identical string literals may be merged by the compiler into a single address (controlled by `-fmerge-constants`).

### 3. Data Segment (Initialized data segment, Globals)

Global and `static` variables that have an explicit initializer. Their initial values are stored inside the ELF binary on disk and loaded at startup.

```c
int  global_init = 42;    /* .data — value 42 stored in binary */
static int s_count = 0;   /* .data — static local, one copy ever */
```

- Address is fixed for the entire program lifetime.
- Calling a function that modifies `s_count` increments the same memory slot every time — no new allocation per call.
- Costs disk space proportional to the data size.

**If the initialized variable value is large like array[1000000] then that much space will be occupied in the executable file which will increase the size of the executable file. And hence will increase the size of the binary. But in case of `.bss` we can declare the array of any size and it will not increase the size of the binary. Because the OS will allocate the memory for the `.bss` segment at the time of loading and will zero it out. It will not store the initial values of the `.bss` segment in the executable file**

If we add array[1000000] to the above c file then the output of `size mem` will be:

* size before arr[1000000]

```bash
int arr[10000000] = {0}; // Stored in .bss (Safe) optimized by compiler
int arr[10000000] = {1}; // Stored in .data

size mem
   text	   data	    bss	    dec	    hex	filename
   1992	40000688	16	40002696	2626488	mem
```


### 4. BSS Segment (Uninitialized data segment )

Global and `static` variables with no initializer, or initialized to zero. The C standard guarantees they are zero-initialized before `main()` runs. The OS performs this zeroing — the BSS section occupies **zero bytes** in the ELF file on disk.

```c
int  global_uninit;        /* .bss — guaranteed 0, no disk cost */
static int call_count;     /* .bss — same guarantee */
```

**Practical test:** Add `int arr[10000000];` at file scope and run `size mem`. BSS grows by 40,000 bytes; the binary on disk stays the same size.

* size before arr[10000000]

```bash
$ size mem
   text	   data	    bss	    dec	    hex	filename
   1992	    648	     16	   2656	    a60	mem
```


Add `int arr[10000000];` at file scope and compare the output of `size mem` again. You will see that the `bss` segment has increased by 40,000 bytes, but the executable file size on disk has not changed.

* size after arr[10000000]

```bash
$ size mem
   text	   data	    bss	    dec	    hex	filename
   1992	    648	40000072	40002712	2626498	mem
```

### 5. Heap

Memory allocated dynamically at runtime via `malloc` / `calloc` / `realloc`. Grows upward from a low base address. The allocator (`glibc`'s ptmalloc) manages an internal arena; `free()` returns memory to that arena.

```c
char *p   = malloc(64);       /* p: 8-byte pointer on stack */
                               /* block: 64 bytes on heap   */
strcpy(p, "dynamic");
free(p);                       /* must call or memory leaks */
```

**Critical rule:** Only the _pointer variable_ is on the stack (8 bytes on x86-64). The allocated block is on the heap until `free()` is called explicitly. There is no automatic reclamation.

```c
/* Memory leak: p goes out of scope, block is never freed */
void leak() {
    char *p = malloc(64);
    /* forgot free(p) */
}
```

### 6. Stack

Local variables, function parameters, return addresses, and saved registers. Grows downward from a high base address. Each function call pushes a _stack frame_; returning pops it — the memory is instantly reclaimed.

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

When `main()` calls `compute()` which calls `add()`, three frames are live simultaneously:

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

When `add` returns: SP moves back up, frame gone. `result`'s memory is no longer valid.

---

## Heap vs Stack — Pointer Indirection

```c
typedef struct {
    int  id;
    char name[32];
} User;

int main() {
    User *u   = malloc(sizeof(User));  /* sizeof(User) = 36 bytes */
    u->id     = 1;
    strcpy(u->name, "Alice");

    int *arr  = malloc(10 * sizeof(int));
    arr[0]    = 99;

    free(u);
    free(arr);
    return 0;
}
```

Memory layout:

```
Stack (main frame)
  rbp-8   u    = 0x02601000   ← 8-byte pointer
  rbp-16  arr  = 0x02601030   ← 8-byte pointer

          │                   │
          ▼                   ▼
Heap
  0x02601000  User struct (36 bytes)
    +0   .id   = 1
    +4   .name = "Alice\0…"

  0x02601030  int arr[10] (40 bytes)
    +0   arr[0] = 99
    +4…  arr[1..9] = 0
```

---

## String Literals — Three Ways

```c
int main() {
    const char *lit  = "hello";  /* pointer to RODATA — read-only    */
    char        buf[] = "hello"; /* array copy on STACK — mutable    */
    char       *heap  = malloc(strlen(lit) + 1);
    strcpy(heap, lit);           /* heap copy — mutable, must free() */

    buf[0]  = 'H';  /* OK */
    heap[0] = 'H';  /* OK */
    /* lit[0] = 'H'; */  /* SIGSEGV */

    free(heap);
    return 0;
}
```

| Variable | Location | Mutable | Free required |
|----------|----------|---------|---------------|
| `lit`    | pointer in stack → bytes in RODATA | No  | No  |
| `buf[]`  | array on stack (copy)              | Yes | No (auto) |
| `heap`   | pointer in stack → bytes on heap   | Yes | Yes |

---

## GCC Verification Steps

### Step 1: Compile with Debug Info

Always compile with `-g` (debug symbols) and `-O0` (no optimisation) for verification. Optimisation can eliminate variables or move them into registers, making them invisible to inspection tools.

```bash
# Basic compile
gcc -g -O0 -o mem mem.c

# With linker map file
gcc -g -O0 -Wl,-Map=mem.map -o mem mem.c

# Verify the output is an ELF binary
file mem
# mem: ELF 64-bit LSB pie executable, x86-64, ...
```

> **Note:** All verification commands in subsequent steps assume the binary is named `mem` and was compiled with `-g -O0`.

---

### Step 2: `size` — Section Byte Counts

The fastest way to confirm segment sizes. One command, one line of output.

```bash
size mem
```

**Example output:**

```
   text    data     bss     dec     hex filename
   2156     616      12    2784     ae0 mem
```

| Column | What it measures |
|--------|-----------------|
| `text` | Machine code + RODATA (on some toolchains) |
| `data` | Initialized globals stored in the binary |
| `bss`  | Uninitialized globals (zeroed at runtime, no disk cost) |
| `dec`  | Total of all three in decimal |

**Experiment:** Add `int arr[100];` at file scope, recompile, and run `size` again. BSS grows by 400 bytes; the binary on disk stays the same size.

---

### Step 3: `nm` — Symbol Table

`nm` lists every symbol with its virtual address, size, and a type letter indicating which segment it lives in.

```bash
# All symbols, sorted by size
nm --print-size --size-sort mem | grep -v ' [Uw] '

# Only data and BSS symbols
nm mem | grep ' [bBdD] '

# Only RODATA symbols
nm mem | grep ' [rR] '

# Demangle C++ names (useful for C++ projects)
nm --demangle mem
```

**Example output:**

```
0000000000004010 0000000000000004 b global_uninit
0000000000004014 0000000000000004 b s_count.0
0000000000004008 0000000000000004 d global_init
0000000000004004 0000000000000008 d msg
0000000000001169 000000000000005e T greet
0000000000001140 0000000000000029 T main
```

#### nm Type Letters Reference

| Letter | Segment | Meaning |
|--------|---------|---------|
| `T` / `t` | Text    | Function code. Uppercase = exported, lowercase = file-static |
| `D` / `d` | Data    | Initialized global/static. Uppercase = global, lowercase = file-static |
| `B` / `b` | BSS     | Uninitialized global/static |
| `R` / `r` | RODATA  | Read-only data (string literals, const globals) |
| `U`       | —       | Undefined (resolved from a shared library at runtime) |
| `W`       | —       | Weak symbol |

---

### Step 4: `readelf` — Section Headers

`readelf -S` shows every ELF section with flags, virtual address, file offset, and size in bytes.

```bash
# List all sections
readelf -S --wide mem

# Dump .rodata as hex + ASCII
readelf -x .rodata mem

# Dump .data section
readelf -x .data mem

# Show program headers (load segments)
readelf -l mem
```

**Example output (`readelf -S --wide`):**

```
  [Nr] Name           Type       Address          Off    Size   Flg
  [14] .text          PROGBITS   0000000000001060 001060 000185  AX
  [16] .rodata        PROGBITS   0000000000002000 002000 000038   A
  [24] .data          PROGBITS   0000000000004000 003000 000268  WA
  [25] .bss           NOBITS     0000000000004268 003268 000010  WA
```

#### Section Flags

| Flag | Meaning |
|------|---------|
| `A`  | Allocate in memory at load time |
| `X`  | Executable |
| `W`  | Writable |

| Type       | Meaning |
|------------|---------|
| `PROGBITS` | Has real bytes stored in the ELF file |
| `NOBITS`   | No disk storage; OS allocates and zeroes at load (BSS) |

**Key insight:** The `Address` column is the runtime virtual address. The `Off` column is the byte offset inside the ELF file on disk. These are different — the linker maps one to the other.

---

### Step 5: `objdump` — Disassembly

`objdump -d` shows machine instructions; `-S` interleaves original C source lines (requires `-g` at compile time).

```bash
# Disassemble all functions (Intel syntax)
objdump -d -M intel mem

# Interleave C source with assembly
objdump -d -S -M intel mem | less

# Disassemble only the greet() function
objdump -d mem | awk '/\<greet\>/,/^$/'

# Show full contents of .rodata as hex+ASCII
objdump -s -j .rodata mem
```

**Example output (main's prologue):**

```asm
0000000000001140 <main>:
    1140: 55                   push   rbp
    1141: 48 89 e5             mov    rbp, rsp
    1144: 48 83 ec 20          sub    rsp, 0x20    ; 32 bytes for locals
    ; int x = 7
    1148: c7 45 fc 07 00 00 00 mov    DWORD PTR [rbp-0x4], 0x7
    ; char *p = malloc(64)
    114e: bf 40 00 00 00       mov    edi, 0x40
    1153: e8 d8 fe ff ff       call   1030 <malloc@plt>
    1158: 48 89 45 f0          mov    QWORD PTR [rbp-0x10], rax
```

#### Reading the Disassembly

| Instruction | What it confirms |
|-------------|-----------------|
| `sub rsp, 0x20` | 32 bytes reserved for local variables on the stack |
| `mov [rbp-0x4], 7` | `int x = 7` placed at offset -4 from the frame base |
| `call malloc@plt` | PLT stub for `malloc`; return value (heap ptr) in `rax` |
| `mov [rbp-0x10], rax` | Heap pointer stored as `p` at rbp-16 on the stack |

---

### Step 6: GDB — Live Stack Inspection

Run the binary under GDB to observe actual runtime addresses and confirm the memory regions.

```bash
gdb ./mem
```

**GDB commands:**

```gdb
# Set a breakpoint in main and run
(gdb) break main
(gdb) run

# Step past variable initializations
(gdb) next   # past: int x = 7
(gdb) next   # past: char *p = malloc(64)

# Print addresses of stack variables
(gdb) print &x
# $1 = (int *) 0x7fffffffdc8c    ← stack (high address)

(gdb) print &p
# $2 = (char **) 0x7fffffffdc80  ← stack

# Print value of p (the heap address it holds)
(gdb) print p
# $3 = 0x5555555592a0            ← heap (low address)

# Print address of globals
(gdb) print &global_init
# $4 = (int *) 0x555555558008    ← .data (fixed)

(gdb) print &global_uninit
# $5 = (int *) 0x55555555800c    ← .bss (fixed)

# Show the entire current stack frame
(gdb) info frame
(gdb) info locals

# Step into greet() and inspect its frame
(gdb) break greet
(gdb) continue
(gdb) print &buf                 # stack address in greet's frame
(gdb) backtrace                  # full call chain
```

**Address range summary from GDB output:**

| Variable       | Address range    | Segment |
|----------------|-----------------|---------|
| `x`, `p`, `buf` | `0x7fff…`      | Stack   |
| `malloc()` result | `0x5555…5…`  | Heap    |
| `global_init`  | `0x5555…8…`    | .data   |
| `global_uninit`| `0x5555…8…+4`  | .bss    |

> **Tip:** Stack at `0x7fff…`, heap at `0x5555…5…`, globals at `0x5555…8…` — these ranges confirm the virtual address layout diagram. PIE (position-independent executable) offsets will vary per run with ASLR enabled; use `set disable-randomization on` in GDB for reproducible addresses.

---

### Step 7: Valgrind — Heap Verification

Valgrind's `memcheck` tool instruments every memory operation and reports leaks, use-after-free, and invalid accesses.

```bash
# Check for memory leaks
valgrind --leak-check=full --track-origins=yes ./mem
```

**Clean output (all frees correct):**

```
==1234== HEAP SUMMARY:
==1234==     in use at exit: 0 bytes in 0 blocks
==1234==   total heap usage: 2 allocs, 2 frees, 1,088 bytes allocated
==1234==
==1234== All heap blocks were freed -- no leaks are possible
```

**Output when `free(p)` is omitted:**

```
==1235== LEAK SUMMARY:
==1235==    definitely lost: 64 bytes in 1 blocks
==1235==  at 0x...: malloc (vg_replace_malloc.c:...)
==1235==  by 0x...: main (mem.c:17)
```

```bash
# Profile heap usage over time
valgrind --tool=massif ./mem
ms_print massif.out.*

# Check for use-after-free
valgrind --leak-check=full --error-exitcode=1 ./mem
```

> **Note:** The "2 allocs, 1088 bytes" reflects `malloc(64)` plus ~1024 bytes of internal `stdio` buffer allocation. Both are freed, so the summary is clean.

---

### Step 8: Verifying RODATA String Literals

Three complementary commands confirm that string literals are baked into `.rodata` at compile time.

```bash
# Extract all printable strings from the binary
strings mem | grep -E 'hello|world|Hi %s|dynamic'
```

Output:
```
hello
Hi %s
world
dynamic
```

```bash
# Dump .rodata as hex + ASCII
objdump -s -j .rodata mem
```

Output:
```
Contents of section .rodata:
 2000 01000200 00000000 68656c6c 6f00     ........ hello.
 200e 48692025 7300     776f726c 6400     Hi %s. world.
```

Hex breakdown: `68 65 6c 6c 6f 00` = `'h','e','l','l','o','\0'`

```bash
# Confirm write-protection — compile this and run:
cat > test_rodata.c << 'EOF'
int main() {
    char *p = "hello";
    p[0] = 'H';   /* SIGSEGV expected */
    return 0;
}
EOF
gcc -o test_rodata test_rodata.c && ./test_rodata
# Segmentation fault (core dumped)
```

```bash
# Cross-check: find the msg pointer address, then trace it into .rodata
nm mem | grep msg            # address of the pointer variable in .data
readelf -x .data mem         # find the pointer's value — it points into .rodata
readelf -x .rodata mem       # confirm the bytes at that address are "hello\0"
```

---

## Quick Reference: nm Type Letters

| Letter | Segment | Rule |
|--------|---------|------|
| `T` / `t` | `.text`   | Uppercase = globally visible function |
| `D` / `d` | `.data`   | Initialized global/static, has value in binary |
| `B` / `b` | `.bss`    | Uninitialized global/static, zeroed at load |
| `R` / `r` | `.rodata` | Read-only: string literals, `const` globals |
| `U`       | external  | Undefined — resolved from shared lib |
| `W` / `w` | —         | Weak symbol — linker picks one definition |
| `A`       | absolute  | Value is fixed; not relocated |

---

## Address Range Cheat Sheet

On a typical x86-64 Linux process (without ASLR, or in GDB with randomization disabled):

| Region     | Typical address range | Notes |
|------------|----------------------|-------|
| Text       | `0x0000555555554000` | Executable, read-only |
| RODATA     | `0x0000555555556000` | Read-only, immediately after text |
| Data       | `0x0000555555558000` | Writable, fixed address |
| BSS        | just after Data      | Writable, zero-filled |
| Heap       | `0x0000555555559000` | Grows upward |
| Stack      | `0x00007ffffffde000` | Grows downward from top of user space |

With ASLR enabled (default on Linux), base addresses are randomized per run. Use `echo 0 > /proc/sys/kernel/randomize_va_space` or GDB's `set disable-randomization on` for reproducible results.

---

## Common Pitfalls

### 1. Returning a pointer to a local variable

```c
int *bad() {
    int local = 99;
    return &local;   /* undefined behavior — frame is gone */
}
```

**Fix:** allocate on the heap and document that the caller must `free()`.

### 2. Writing to a string literal

```c
char *s = "hello";
s[0] = 'H';   /* SIGSEGV — .rodata is read-only */
```

**Fix:** use `char s[] = "hello";` to get a mutable stack copy.

### 3. Missing `free()`

```c
void process() {
    char *buf = malloc(1024);
    if (error) return;    /* leak: buf never freed */
    free(buf);
}
```

**Fix:** use a `goto cleanup` pattern or always set `buf = NULL` after `free`.

### 4. Double `free()`

```c
free(p);
free(p);   /* undefined behavior — heap corruption */
```

**Fix:** set `p = NULL` immediately after `free(p)`. Calling `free(NULL)` is a no-op.

### 5. Buffer overflow on stack

```c
char buf[8];
strcpy(buf, "this string is way too long");   /* smashes stack */
```

**Fix:** use `strncpy(buf, src, sizeof(buf) - 1)` or `snprintf`.

### 6. Uninitialized local variables

```c
int x;
printf("%d\n", x);   /* undefined — stack contains whatever was there */
```

This is different from a global: globals in BSS are guaranteed zero; locals are not.

---

## Tool Summary

| Tool       | Command                          | What you learn |
|------------|----------------------------------|----------------|
| `gcc`      | `gcc -g -O0 -o mem mem.c`       | Compile with debug info |
| `file`     | `file mem`                       | Confirm ELF format |
| `size`     | `size mem`                       | Byte counts per segment |
| `nm`       | `nm --print-size --size-sort mem`| Symbol → segment mapping |
| `readelf`  | `readelf -S --wide mem`          | Full section headers and flags |
| `readelf`  | `readelf -x .rodata mem`         | Raw hex dump of a section |
| `objdump`  | `objdump -d -S -M intel mem`     | Disassembly interleaved with source |
| `strings`  | `strings mem`                    | Printable strings in binary |
| `gdb`      | `gdb ./mem`                      | Live address inspection at runtime |
| `valgrind` | `valgrind --leak-check=full ./mem` | Heap leak and error detection |

---

*Generated from the C Memory Layout & GCC Verification interactive guide.*
