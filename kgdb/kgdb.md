# KGDB - Debug Kernel

---

## About KGDB
Kgdb is intended to be used as a source level debugger for the Linux kernel. It is used along with gdb to debug a Linux kernel. The expectation is that gdb can be used to “break in” to the kernel to inspect memory, variables and look through call stack information similar to the way an application developer would use gdb to debug an application. It is possible to place breakpoints in kernel code and perform some limited execution stepping.

---

## Block Diagram
TBD

---

## Kernel Configuration
* `CONFIG_KGDB` to enable remote debugging.
* `CONFIG_KGDB_SERIAL_CONSOLE` to allow gdb to access the serial console.
* `CONFIG_KALLSYMS_ALL` add all symbols into the kernel image.
* `CONFIG_MAGIC_SYSRQ` to be able to make sysrq.
* `CONFIG_FRAME_POINTER` this is to use the frame pointer to save the stack information and backtrace will use this.

---

## Required Tools
* `Agent-proxy` is to split the serial port into two, one is for console and another is for gdb.

```bash
git clone http://git.kernel.org/pub/scm/utils/kernel/kgdb/agent-proxy.git
cd agent-proxy
make
```

---

## Run Agent Proxy

```bash
./agent-proxy/agent-proxy '127.0.0.1:5550^127.0.0.1:5551' 0 /dev/ttyUSB0,115200
```

To connect to the console of the device, a simple telnet or telnet-like tool is enough:

```bash
screen //telnet localhost 5550
```

## Disable KASLR

Kernel Address Space Layout Randomization (KASLR) randomizes the kernel’s memory
layout at boot time, making it difficult to resolve symbol addresses for
debugging or development. To simplify this process, it is strongly recommended
to disable KASLR by booting the kernel with the nokaslr parameter.

### Through Kernel Configuration

```
CONFIG_RANDOMIZE_BASE=y  
```

### By Modifying Boot Arguments

```
setenv bootargs "${bootargs} nokaslr"
saveenv
```

---


## Using lx-symbols in GDB
lx-symbols loads kernel and module symbols automatically. However, if KASLR is
enabled, symbol addresses will not match the actual runtime locations, leading
to incorrect symbol resolution.

After disabling KASLR, you can correctly load symbols as follows:

```
(gdb) add-symbol-file vmlinux
(gdb) lx-symbols
```

---

## DUT/QEMU setup

* `kgdb` debugging can be enabled at run time or at boot time.

### Enable kgdb at run time
* `kgdb` can be enabled at runtime using sysfs

```bash
echo ttymxc0 > /sys/module/kgdboc/parameters/kgdboc
```

### Enable kgdb at boot time
* Configure bootargs

```
kgdboc=ttymxc0,115200 kgdbwait
```
Here, kgdbdoc means kgdb over console. `ttymxc0` is the serial console `kgdb` will use for debugging and `kgdbwait` stop the
kernel execution and enable debugger

---

## Debugging setup
* To connect gdb, kernel executions should be stopped and wait for debugger connection.
* `kgdbwait` bootargs stops the kernel at boot time and wait.

```
Starting kernel ...
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]
....
[    0.270256] 30860000.serial: ttymxc0 at MMIO 0x30860000 (irq = 15, base_baud = 1562500) is a IMX
[    0.270307] printk: console [ttymxc0] enabled
[    1.375461] 30880000.serial: ttymxc2 at MMIO 0x30880000 (irq = 16, base_baud = 5000000) is a IMX
[    1.385505] KGDB: Registered I/O driver kgdboc
[    1.390020] KGDB: Waiting for connection from remote gdb...

Entering kdb (current=0xffff0000c0270000, pid 1) on processor 2 due to Keyboard Entry
[2]kdb>
```

* At runtime, sysrq-g is used to stop the kernel execution and wait for gdb connection.

```
echo g > /proc/sysrq-trigger
Entering kdb (current=0xffff0000c0270000, pid 1) on processor 2 due to Keyboard Entry
[2]kdb>
```

---

## Connect the gdb 
* we took this example from yocto sdk.
* GDB is build with toolchain. Use the gdb to connect to tagret board.

```bash
aarch64-linux-gnu-gdb  ./vmlinux
```
NOTE: vmlinux can be found in the build kernel directory. It is adviced to build the kernel with debugging symbols enabled.

*  Here is the sample output.

```
aarch64-poky-linux-musl-gdb ./vmlinux
GNU gdb (GDB) 13.2
Copyright (C) 2023 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=x86_64-pokysdk-linux --target=aarch64-poky-linux".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./vmlinux...

warning: could not convert 'main' from the host encoding (UTF-8) to UTF-32.
This normally should not happen, please file a bug report.
(gdb)
```

* Connect the gdb to target board using below command.

```
(gdb) target remote localhost:4441 
```

* If you are not using agent proxy then use this command

```
(gdb) set remotebaud 115200
(gdb) target remote /dev/ttyS0
```

* gdb should be connected to target board.

```
(gdb) target remote localhost:4441
Remote debugging using localhost:4441
warning: multi-threaded target stopped without sending a thread-id, using first non-exited thread
[Switching to Thread 4294967294]
arch_kgdb_breakpoint () at /usr/src/kernel/arch/arm64/include/asm/kgdb.h:21
21      /usr/src/kernel/arch/arm64/include/asm/kgdb.h: No such file or directory.
(gdb)
```

* Start your debugging.

---

## Example(imx8mq: nxp processor)
* Here we are trying to dump value of a register.

```
aarch64-poky-linux-musl-gdb ./vmlinux
GNU gdb (GDB) 13.2
Copyright (C) 2023 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=x86_64-pokysdk-linux --target=aarch64-poky-linux".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./vmlinux...

warning: could not convert 'main' from the host encoding (UTF-8) to UTF-32.
This normally should not happen, please file a bug report.
(gdb) 
(gdb) target remote localhost:4441
Remote debugging using localhost:4441
warning: multi-threaded target stopped without sending a thread-id, using first non-exited thread
[Switching to Thread 4294967294]
arch_kgdb_breakpoint () at /usr/src/kernel/arch/arm64/include/asm/kgdb.h:21
21      /usr/src/kernel/arch/arm64/include/asm/kgdb.h: No such file or directory.
(gdb) b dcss-dtg.c:134
Breakpoint 1 at 0xffff80008095ecac: file /usr/src/kernel/drivers/gpu/drm/imx/dcss/dcss-dtg.c, line 134
.
(gdb) c
Continuing.
[New Thread 61]
[New Thread 62]
[New Thread 63]
[New Thread 64]
[New Thread 65]
[New Thread 66]
[New Thread 67]
[New Thread 68]
[New Thread 69]
[New Thread 70]
[New Thread 71]
[New Thread 72]
[New Thread 73]
[New Thread 74]
[Switching to Thread 47]

Thread 51 hit Breakpoint 1, dcss_dtg_irq_config (pdev=<optimized out>, dtg=0xffff0000c10675c0)
    at /usr/src/kernel/drivers/gpu/drm/imx/dcss/dcss-dtg.c:134
134     /usr/src/kernel/drivers/gpu/drm/imx/dcss/dcss-dtg.c: No such file or directory.
(gdb) p dtg
$1 = (struct dcss_dtg *) 0xffff0000c10675c0
(gdb) p *dtg
$2 = {dev = 0xffff0000c02c5810, ctxld = 0xffff0000c093da00, base_reg = 0xffff800082363000, base_ofs = 
853671936, ctx_id = 0, 
  in_use = false, hdmi_output = false, dis_ulc_x = 0, dis_ulc_y = 0, control_status = 4278190216, alph
a = 255, alpha_cfg = 0, 
  ctxld_kick_irq = 203, ctxld_kick_irq_en = false}
(gdb) x/1x 0xffff800082363000
0xffff800082363000:     0x00000400
(gdb) x/1x 0xffff800082363068
0xffff800082363068:     0x00000000
(gdb) info frame
Stack level 0, frame at 0xffff80008275baf0:
 pc = 0xffff80008095ecac in dcss_dtg_irq_config (/usr/src/kernel/drivers/gpu/drm/imx/dcss/dcss-dtg.c:1
34); 
    saved pc = 0xffff80008095dcc0
 inlined into frame 1
 source language c.
 Arglist at unknown address.
 Locals at unknown address, Previous frame's sp in sp
(gdb) p *(0xffff800082363068)
(gdb) ptype *dtg
type = struct dcss_dtg {
struct device *dev;
struct dcss_ctxld *ctxld;
void *base_reg;
u32 base_ofs;
u32 ctx_id;
bool in_use;
bool hdmi_output;
u32 dis_ulc_x;
u32 dis_ulc_y;
u32 control_status;
u32 alpha;
u32 alpha_cfg;
int ctxld_kick_irq;
bool ctxld_kick_irq_en;
}
```

```
(gdb) lx-clk-summary
clock                      enable  prepare  protect  rate
----------------------------------------------------------
sys2_pll_out                         2        3        0  1000000000 
   sys_pll2_out_monitor              0        0        0  1000000000
   sys2_pll_1000m                    0        0        0  1000000000
....
....
----------------------------------------------------------
Total clocks: 128

```

* You could also try following

```
lx-clk-summary      Prints the kernel clock tree summary (rate, enables, etc.)
lx-symbols          Loads kernel and module symbols
lx-dmesg            Dumps the kernel log buffer
lx-list             Lists elements of a kernel linked list
lx-device-list      Lists registered devices
```

---

## QEMU Setup
* Run command to start the qemu with `kgdb` options enabled
```bash
qemu-system-arm -M vexpress-a9 -m 512M -dtb vexpress-a9.dtb -kernel zImage -initrd rootfs.img.gz -append "kgdbwait kgdboc=ttyAMC0,115200 root=/dev/ram rdinit=/linuxrc -serial tcp::1234,server,nowait" 
```

---

## References
* https://www.marcusfolkesson.se/blog/debug-kernel-with-kgdb/
* https://sergioprado.blog/debugging-the-linux-kernel-with-gdb/
* https://www.kernel.org/doc/html/v4.14/dev-tools/kgdb.html
