# Building an ARM64 Toolchain Using Crosstool-NG

> **Goal:** Build a complete `aarch64-unknown-linux-gnu` cross-compilation toolchain from source on an x86-64 Linux host using [Crosstool-NG](https://crosstool-ng.github.io/).

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Install Crosstool-NG](#2-install-crosstool-ng)
3. [List and Select a Sample Configuration](#3-list-and-select-a-sample-configuration)
4. [Customize the Configuration](#4-customize-the-configuration)
5. [Build the Toolchain](#5-build-the-toolchain)
6. [Verify the Toolchain](#6-verify-the-toolchain)
7. [Use the Toolchain](#7-use-the-toolchain)
8. [Understanding the Output Directory](#8-understanding-the-output-directory)
9. [Troubleshooting](#9-troubleshooting)

---

## Full plan is 

Step 1 — Install dependencies on Ubuntu
      ↓
Step 2 — Build toolchain with Crosstool-NG
         (binutils + gcc + kernel headers + glibc)
      ↓
Step 3 — Compile test programs with the toolchain
         (C program, float test, struct layout test)
      ↓
Step 4 — Install QEMU
      ↓
Step 5 — Run and test binaries on QEMU
         (dynamic binary, static binary, ABI tests)
      ↓
Step 6 — Deliberately break ABI and observe/debug

## 1. Prerequisites

### 1.1 Install Host Dependencies

```bash
sudo apt update
sudo apt install -y \
    build-essential \
    git \
    autoconf \
    automake \
    bison \
    flex \
    gawk \
    gettext \
    help2man \
    libncurses5-dev \
    libtool \
    libtool-bin \
    python3-dev \
    texinfo \
    unzip \
    wget \
    xz-utils \
    libssl-dev \
    bc

# Verify key tools
gcc --version
make --version
bison --version
```

> **Why all these?** Crosstool-NG downloads and compiles binutils, GCC, glibc, GDB, etc. from source. Each component has its own set of build-time dependencies.

### 1.2 Verify Key Tools

```bash
gcc --version    # Should be GCC 10+ for building GCC 12+
make --version   # GNU Make 4.x
python3 --version
```

---

## 2. Install Crosstool-NG

### 2.1 Build from Source (Recommended)

```bash
# Clone the repository
git clone https://github.com/crosstool-ng/crosstool-ng.git
cd crosstool-ng

# (Optional) check out a stable release tag
git tag | grep '^crosstool-ng-' | sort -V | tail -5
git checkout crosstool-ng-1.26.0   # use the latest stable tag

# Bootstrap the build system (generates ./configure)
./bootstrap

# Configure for local installation (installs into the current directory)
./configure --enable-local

# Build ct-ng
make -j$(nproc)

# Verify
./ct-ng version
This is crosstool-NG version 1.28.0.31_d04b732

Copyright (C) 2008  Yann E. MORIN <yann.morin.1998@free.fr>
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.
```

> `--enable-local` installs everything inside the current directory — no `sudo` needed and easy to remove.

---

## 3. List and Select a Sample Configuration

Crosstool-NG ships with pre-defined **sample configurations** for common targets. These are excellent starting points.

### 3.1 List Available Samples

```bash
./ct-ng list-samples
```

Sample output (abbreviated):
```
[G...]   aarch64-unknown-linux-gnu
[G...]   aarch64-unknown-linux-musl
[G...]   arm-cortex_a8-linux-gnueabi
[G...]   arm-unknown-linux-gnueabihf
[G...]   mipsel-unknown-linux-gnu
[G...]   riscv64-unknown-linux-gnu
[G...]   x86_64-unknown-linux-gnu
```

The prefix `[G...]` means the sample is known to build correctly (Green).

### 3.2 Load the ARM64 glibc Sample

```bash
./ct-ng aarch64-unknown-linux-gnu
```

This writes a `.config` file in the current directory. It is similar to Linux kernel's `defconfig`.

### 3.3 (Alternative) ARM64 + musl libc

For a smaller, statically-linkable toolchain:

```bash
./ct-ng aarch64-unknown-linux-musl
```

> **Which to choose?**
> - `gnu` (glibc): Best compatibility with third-party software.
> - `musl`: Better for statically linked binaries and size-constrained targets.

---

## 4. Customize the Configuration

Run the interactive menu to review or change settings before building:

```bash
# Create a clean working directory
mkdir -p ~/toolchain-build && cd ~/toolchain-build

./ct-ng menuconfig

# Key settings to verify in menuconfig:

# Paths and misc options:
# Maximum log level to see: DEBUG     ← see every build step
# Number of parallel jobs:  8         ← match your CPU cores
# Prefix directory: ${HOME}/x-tools/${CT_TARGET}
# Local tarballs directory: ${HOME}/src

# Target options:
# Target Architecture:    arm
# Bitness:                32-bit
# Architecture level:     armv7-a
# Float point:            hardware (FPU)
# FPU:                    neon

# Operating System:
# Version of linux:   5.15.x   ← LTS kernel headers

# C-library:
# C library:          glibc
# Version of glibc:   2.31     ← compatible with most targets

# C compiler:
# Version of gcc:     13.x
# C++:                yes      ← enable for later
# Save and exit.

```


## 5. Build the Toolchain

```bash
# Create tarball cache directory
mkdir -p ~/src

# Build (takes 20-60 minutes)
ct-ng build.$(nproc)

#output will be here
[INFO ]  Performing some trivial sanity checks
[WARN ]  Number of open files 1024 may not be sufficient to build the toolchain; increasing to 2048
[INFO ]  Build started 20260427.194258
[INFO ]  Building environment variables
[EXTRA]  Preparing working directories
[EXTRA]  Installing user-supplied crosstool-NG configuration
[EXTRA]  =================================================================
[EXTRA]  Dumping internal crosstool-NG configuration
[EXTRA]    Building a toolchain for:
[EXTRA]      build  = x86_64-pc-linux-gnu
[EXTRA]      host   = x86_64-pc-linux-gnu
[EXTRA]      target = arm-unknown-linux-gnueabihf
[EXTRA]  Dumping internal crosstool-NG configuration: done in 0.04s (at 00:01)
[INFO ]  =================================================================
```

> This single command:
> 1. Downloads all source tarballs (binutils, GCC, glibc, kernel headers, GDB, …)
> 2. Builds a temporary **bootstrap compiler** (host GCC → minimal cross-gcc)
> 3. Cross-compiles glibc and kernel headers using the bootstrap compiler
> 4. Builds the final full cross-GCC linked against the cross-compiled glibc
> 5. Installs everything into the prefix directory

---

## 6. Verify the Toolchain

## 6.1 find the toolchain path

```bash
find . -name "arm-unknown-linux-gnueabihf-gcc" 
./.build/arm-unknown-linux-gnueabihf/buildtools/bin/arm-unknown-linux-gnueabihf-gcc
```

### 6.1 Add to PATH

```bash
export PATH="${HOME}/x-tools/aarch64-unknown-linux-gnu/bin:${PATH}"
```

Add this to `~/.bashrc` or `~/.profile` to make it permanent:

```bash
echo 'export PATH="${HOME}/x-tools/aarch64-unknown-linux-gnu/bin:${PATH}"' >> ~/.bashrc
source ~/.bashrc
```

### 6.2 Check the Compiler

```bash
aarch64-unknown-linux-gnu-gcc --version
```

Expected output:
```
aarch64-unknown-linux-gnu-gcc (crosstool-NG 1.26.0) 13.2.0
Copyright (C) 2023 Free Software Foundation, Inc.
```

### 6.3 Compile a Test Program

```bash
# create a directory to test the toolchain
 mkdir -p ~/cross-tests && cd ~/cross-tests

# set environment variables
export PATH="/home/aojha/Practice/toolchain/Crosstool-NG/crosstool-ng/.build/arm-unknown-linux-gnueabihf/buildtools/bin/:$PATH"
export CC=arm-unknown-linux-gnueabihf-gcc
export READELF=arm-unknown-linux-gnueabihf-readelf
export OBJDUMP=arm-unknown-linux-gnueabihf-objdump

# create a test program
cat > hello.c << 'EOF'
#include <stdio.h>
int main() {
    printf("Hello from ARM!\n");
    return 0;
}
EOF

# cross compile the test program
$CC -o hello hello.c

# check the file type
file hello

hello: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, for GNU/Linux 6.18.3, with debug_info, not stripped

# check dynamic library dependencies
arm-unknown-linux-gnueabihf-readelf -d hello |grep NEEDED
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
```
---

## 7. Understanding the Output Directory

After a successful build, the toolchain is installed at:

```
tree -L 1 .build/arm-unknown-linux-gnueabihf/
.build/arm-unknown-linux-gnueabihf/
├── build
├── buildtools
└── src

.build/arm-unknown-linux-gnueabihf/buildtools/
├── arm-unknown-linux-gnueabihf
├── bin           # binutils, GCC, GDB, etc. for the target architecture.
├── include       # GCC internal headers
├── lib           # GCC internal libraries (libgcc, libstdc++)
├── libexec       # GCC internal tools (cc1, collect2, ...)
└── share         # Man pages, locale data

```

Key paths:

| Path | Purpose |
|---|---|
| `bin/aarch64-*-gcc` | The cross-compiler you invoke |
| `aarch64-.../sysroot/` | Headers and libs the compiler links against |
| `aarch64-.../sysroot/lib/libc.so.6` | The target C library |
| `bin/aarch64-*-gdb` | Cross-GDB for remote debugging |

---


## References

- [Crosstool-NG Official Documentation](https://crosstool-ng.github.io/docs/)
- [Crosstool-NG GitHub Repository](https://github.com/crosstool-ng/crosstool-ng)
- [ARM Architecture Reference Manual (ARMv8)](https://developer.arm.com/documentation/ddi0487/latest)
- [GCC Cross-Compilation Documentation](https://gcc.gnu.org/onlinedocs/gccint/Configure-Terms.html)
- [Mastering Embedded Linux Programming — Chapter 2: Toolchains](https://www.packtpub.com/product/mastering-embedded-linux-programming-third-edition/9781789530384)
