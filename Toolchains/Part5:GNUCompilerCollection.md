# 1. Compiling C files with gcc

Compilation is the translation of source code (the code we write) into object code (sequence of statements in machine language) by a compiler.
The compilation process has four different steps:
1. Preprocessing
2. Compiling
3. Assembling
4. Linking

![GCC Compilation Pipeline](assets/gcc-compilation-steps.svg)

The compiler we will be using as an example is `gcc` which stands for **GNU Compiler Collection**.

**GCC** supports various programming languages, including C, is completely free and is the go-to compiler for most Unix-like operating systems. In order to use it, we should make sur we install it on our computer, if it’s not already there

We will take a very know c code example to explain the above four steps of compilation.

```c
#include <stdio.h>

int main() {
    printf("Hello, World!\n");
    return 0;
}
```

## 1. Preprocessing
* Get rid of comments
* Expand macros
* Handle include files
* Conditional Compilation

The output of this will be stored in a file with `.i` extension.

```bash
gcc -E main.c -o main.i
```

## 2. Compiling
* Convert `.i` file to assembly language
* The assembly code is architecture specific

The preprocessed C code is translated into assembly language.
Compiler checks:
* Syntax
* Data types
* Semantic errors
* Optimizations

```bash
gcc -S main.i -o main.s
```

```bash
cat main.s

# Directives & Setup
.arch armv8-a          ; Target architecture: ARMv8-A (64-bit ARM)
.file "main.c"         ; Debug info: source file name
.text

# Read-Only Data Section
.section .rodata       ; Switch to read-only data section
.align 3               ; Align to 2³ = 8 bytes
.LC0:
.string "Hello, World!" ; The string literal, null-terminated

# Function Header
.text                  ; Back to code section
.align 2               ; Align to 2² = 4 bytes (one instruction)
.global main           ; Make 'main' visible to the linker
.type main, %function  ; Tell the linker this symbol is a function

# Function Body
Prologue — saving the stack frame:
asmstp x29, x30, [sp, -16]!  ; Push frame pointer (x29) and link register (x30)
                            ; onto the stack. The ! means sp = sp - 16 first.
mov x29, sp                ; Set frame pointer to current stack pointer

```

* `x30` is the link register (return address). `x29` is the frame pointer. Both must be saved before calling another function.

```bash
# Loading the string address
asmadrp x0, .LC0         ; Load the PAGE address of .LC0 into x0
add  x0, x0, :lo12:.LC0 ; Add the page OFFSET (lower 12 bits) to get exact address

# Calling puts:
asmbl puts                ; Branch with Link — calls puts(x0)
                       ; x0 is the first argument by AArch64 calling convention
# The compiler optimized printf("Hello, World!\n") into puts("Hello, World!") since there are no format arguments.

# Epilogue — returning:
asmmov w0, 0              ; Set return value to 0 (int, so w0 not x0)
ldp x29, x30, [sp], 16 ; Restore x29 and x30 from stack; sp = sp + 16
ret                    ; Jump to address in x30 (return to caller)
```

```bash
# CFI Directives
# All the .cfi_* lines are Call Frame Information — metadata for the unwinder/debugger (used by stack traces and exception handling). They don't generate instructions, just describe how to unwind the stack.

# Metadata
asm.size main, .-main          ; Record function size for the symbol table
.ident "GCC: ..."           ; GCC version stamp in the binary
.section .note.GNU-stack,"",@progbits  ; Marks stack as non-executable (security)
```

* `.file "main.c"`: Source file information: Used for debugging purposes
* `.text`: Code Section: Contains executable instructions
* `.rodata`: Read Only Data Section: Contains read only data like string literals
* `.LC0`: Label for the string "Hello,World"

Linux uses ELF format, a file format for executables, object code, shared libraries, and core dumps.

## 3. Assembler (as)

The assembler:

* Converts assembly instructions into machine code
* Creates ELF object file structure
* Builds symbol table
* Creates relocation entries

Organizes section like:

* `.text`: Code Section: Contains executable instructions
* `.data`: Data Section: Contains initialized data
* `.bss`: BSS Section: Contains uninitialized data
* `.rodata`: Read Only Data Section: Contains read only data like string literals


```bash
gcc -c main.s -o main.o
```

## 4. Linker (ld)
The linker:

* Combines multiple object files into a single executable
* Resolves symbols (finds where functions and variables are defined)
* Performs relocation (updates addresses to match final layout)
* Stitches together standard libraries (libc, etc.)
* Creates the final ELF executable

linker uses `ld scripts` to map sections into memory. we can aslo write our own ld scripts to customize the memory layout

```bash
ENTRY(_start)

SECTIONS
{
    /* Set the initial execution address */
    . = 0x40000000; 

    /* Group all execution code */
    .text : { 
        *(.text) 
    }

    /* Align to an 8-byte boundary for AArch64 */
    . = ALIGN(8);   
    
    /* Group all read-only constants */
    .rodata : { 
        *(.rodata) 
    }

    /* Group initialized global variables */
    .data : { 
        *(.data) 
    }

    /* Group uninitialized variables */
    .bss : { 
        __bss_start = .;
        *(.bss) 
        _end = .;
    }
}

aarch64-linux-gnu-ld -T custom-script.ld main.o -o output
```

```bash
nm output
000000004000001c r $d
0000000040000000 t $x
0000000040000030 R __bss_start
0000000040000030 R _end
0000000040000000 T main
                 U _start
```

### 4.1 Static Linking vs Dynamic Linking

#### Static Linking

```bash
gcc main.o -o program_static
```

* All library code is copied into the executable
* Larger executable size
* No runtime dependency on libraries
* Faster startup (no dynamic loading)
* Safer (all code is present at compile time)

#### Dynamic Linking

```bash
gcc main.o -o program_dynamic
```

* Uses shared libraries (.so files)
* Smaller executable size
* Multiple programs share library code in memory
* Easier updates (update library, all programs automatically use it)
* Slower startup (dynamic loading required)
* Dynamic linker (ld.so) resolves symbols at runtime

ELF format:

```bash
+----------------------+
| ELF Header           |
+----------------------+
| Program Headers      |
+----------------------+
| .text                |
+----------------------+
| .rodata              |
+----------------------+
| .data                |
+----------------------+
| .bss                 |
+----------------------+
| PLT                  |
+----------------------+
| GOT                  |
+----------------------+
| Dynamic Symbol Table |
+----------------------+
```

### 4.2 How Linking Works (Simplified)

Imagine:

```c
// file1.c
void print_hello();
int main() {
    print_hello();
    return 0;
}

// file2.c
void print_hello() {
    puts("Hello, World!");
}
```

**File A (main.o)**:

```assembly
main:
    call print_hello@PLT   # PLT = Procedure Linkage Table
    ret
```

**File B (print.o)**:

```assembly
.globl print_hello
print_hello:
    mov $msg, %rdi
    call puts@PLT
    ret
```

```bash
# File A: main.o
0000000000000000 T main
                 U print_hello
                 U puts

# File B: print.o
0000000000000000 T print_hello
                 U puts
```

**Relocation entries in each object file tell the linker:**

* "I call print_hello — find where it is"
* "I use puts — I need its address from libc"

The linker:

1. Finds print_hello in print.o
2. Finds puts in libc.so
3. Updates main.o to point to the actual addresses in print.o and libc.so
4. Combines everything into a single executable

### 4.4 Default linker script

The linker script is a file that tells the linker where to place the different sections of the executable in memory. It is a text file that is written in a special syntax that is specific to the linker.

### Dynamic Linker/Loader

The dynamic loader, also known as the dynamic linker, is a small program that is loaded into memory by the kernel before your application starts. It is responsible for finding and loading all the shared libraries your application needs, resolving symbols, and connecting everything together.

When you run a dynamically linked executable:

1. Kernel loads the executable
2. Dynamic linker (ld.so) starts

```bash
$ ldd ./program_dynamic
    linux-vdso.so.1 (0x00007ffeb694c000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f2836426000)
    /lib64/ld-linux-x86-64.so.2 => /lib64/ld-linux-x86-64.so.2 (0x00007f283667a000)
```

* `linux-vdso.so.1`: Kernel-provided virtual dynamic shared object (for performance)
* `libc.so.6`: The standard C library (contains puts())
* `ld-linux-x86-64.so.2`: The dynamic linker itself

3. ld.so reads the executable's NEEDED entries (which libraries it requires)
4. ld.so searches system paths (/lib, /usr/lib, LD_LIBRARY_PATH)
5. ld.so loads required libraries into memory
6. ld.so resolves all remaining symbols (the .PLT entries)
7. Control jumps to main()

This all happens in milliseconds — fast enough to be transparent.

```bash
$ ldd ./program_dynamic
    linux-vdso.so.1 (0x00007ffeb694c000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f2836426000)
    /lib64/ld-linux-x86-64.so.2 => /lib64/ld-linux-x86-64.so.2 (0x00007f283667a000)
```

```bash
$ cat /etc/ld.so.cache | grep libc
# or
$ grep libc /etc/ld.so.conf /etc/ld.so.conf.d/*
```
