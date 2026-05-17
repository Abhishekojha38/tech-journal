# 1. Architecture Tuning

Architecture tuning usually refers to compiling your code so it’s optimized for
a specific CPU architecture and micro-architecture, instead of using generic
defaults.

* GCC provides several config time options to tune for specific architecture.

| GCC Config Options | Runtime Flags | Description |
|-------------------|---------------|-------------|
| `--with-arch` | `-march` | **Architecture Level**: Which CPU features are available |
| `--with-tune` | `-mtune` | **Micro-architecture**: How to schedule instructions for best performance |
| `--with-cpu` | `-mcpu` | **Specific CPU Model**: Subset of -march, implies tuning |
| `--with-fpu` | `-mfpu` | **Floating-Point Unit**: Which FPU to use (vfp, neon, etc.) |
| `--with-abi` | `-mabi` | **Application Binary Interface**: Calling convention, data layout |
| `--with-float-abi` | `-mfloat-abi` | **Floating-Point ABI**: Soft-float vs hardware-float |


## 6.1 -march (Instruction Set Architecture)

Defines the instruction set architecture (ISA). 

`-march` is a compiler option (commonly used with GCC/Clang) that specifies the
target CPU architecture instruction set to generate code for.


```
-march=armv7-a  // for 32-bit ARM CPUs
-march=armv8-a  // for 64-bit ARM CPUs
```

* `armv7-a` → Cortex-A7/A8/A9 class CPUs
* `armv8-a` → 64-bit ARMv8 CPUs like Cortex-A53

```
aarch64-linux-gnu-gcc test.c -march=armv8-a -o test
```
- This generates code compatible with most 64-bit ARM CPUs.

```
aarch64-linux-gnu-gcc test.c \
    -march=armv8-a+crc+crypto \
    -mtune=cortex-a53 \
    -o test
```

`-march=armv8-a+crc+crypto` tells GCC: "Generate code for `ARMv8-A`, and enable
`CRC` and `Crypto` instructions." If you don't include the `+crc+crypto` part,
GCC won't use those instructions, even if the CPU supports them.

## 6.2 -mtune (CPU Core tuning)

`-mtune` tells the compiler which CPU to optimize performance for, without
changing instruction-set compatibility.

It optimizes:
- instruction scheduling
- pipeline usage
- cache behavior
- branch prediction patterns

But it does NOT enable new instructions.
So binaries remain compatible with the architecture specified by -march.

```
-mtune=cortex-a53  // optimize for Cortex-A53
-mtune=cortex-a72  // optimize for Cortex-A72
-mtune=cortex-a710 // optimize for Cortex-A710
```

For example, if you compile with:
```
aarch64-linux-gnu-gcc test.c -march=armv8-a -mtune=cortex-a53 -o test
```

The binary will run on any ARMv8-A CPU (because of `-march=armv8-a`), but it
will run *best* on Cortex-A53 (because of `-mtune=cortex-a53`).

**Example for better understanding:**

Compatible with all ARMv8-A CPUs

```
-march=armv8-a
Generates generic ARMv8 code.
```

Optimize for Cortex-A53

```
-march=armv8-a -mtune=cortex-a53
```

Now compiler rearranges instructions for better performance on Cortex-A53 CPUs
like the ones in i.MX8MQ.

## 6.3 -mcpu (−march+−mtune)

`-mcpu` combines both:

* architecture selection (`-march`)
* CPU tuning (`-mtune`)

into a single option.

`-mcpu` is similar to `-mtune`, but it also implies a specific CPU micro-architecture, which may include **additional optimizations** or **instruction set extensions** beyond what `-mtune` alone provides.

```
-mcpu=native          // compile for the CPU you're currently building on
-mcpu=cortex-a53      // compile for Cortex-A53 (implies tuning + specific features)
-mcpu=cortex-a72      // compile for Cortex-A72 (implies tuning + specific features)
```

**Example for better understanding:**

Compile for generic ARMv8 (compatible with everything)

```
-march=armv8-a
Generates generic ARMv8 code.
```

Compile for Cortex-A53 (includes tuning + maybe extra features)

```
-mcpu=cortex-a53
Now compiler uses Cortex-A53-specific instructions and optimizations.
```

**Note:**

* `-mcpu` is often a **shorthand** for `-march` + `-mtune` + potentially other
flags.
* It's useful when you want the **absolute best performance** on a specific CPU
model and don't need backward compatibility.

## 6.4 -mfloat-abi and -mfpu (Floating Point Options)

### -mfloat-abi

Controls how floating-point numbers are handled:

* `-mfloat-abi=soft` — Software-only floating-point (no hardware FPU used)
* `-mfloat-abi=softfp` — Mixed mode: code calls FPU instructions, but arguments
passed in integer registers
* `-mfloat-abi=hard` — Hard-float: uses FPU instructions AND passes
floating-point arguments in FPU registers

### -mfpu

Selects the floating-point unit (FPU) to target:

* `-mfpu=vfp` — VFPv2 (older, widely supported)
* `-mfpu=neon` — Neon (SIMD unit, includes VFPv4)
* `-mfpu=neon-fp-armv8` — Neon + ARMv8 FP
* `-mfpu=auto` — Let GCC pick the best one

## 6.5 -mabi (Application Binary Interface)

What it controls:
* How function arguments are passed
* How return values are handled
* Size of int, long, pointers (in some cases)
* Stack alignment rules

Common values for ARM64 (aarch64):
* `lp64` — Long/pointer are 64-bit, int is 32-bit (Linux standard)
* `ilp32` — All integer types are 32-bit (rare, Windows-style ARM)
* `-mabi=aapcs-linux` — ARM 32-bit calling convention for Linux.

# Mental Models

```
Application level
   ↓
-mabi        → how functions talk
   ↓
-mfloat-abi  → how floats are passed
   ↓
-mfpu        → what FP hardware instructions exist
   ↓
CPU (Cortex-A53 etc.)
```