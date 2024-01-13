![Apache NuttX RTOS in the Web Browser: TinyEMU with VirtIO](https://lupyuen.github.io/images/tinyemu-title.png)

# Apache NuttX RTOS in the Web Browser: TinyEMU with VirtIO

[(Live Demo of NuttX on TinyEMU)](https://lupyuen.github.io/nuttx-tinyemu)

[(Watch on YouTube)](https://youtu.be/KYrdwzIsgeQ)

Apache NuttX RTOS is a tiny operating system for 64-bit RISC-V Machines and many other platforms. (Arm, x64, ESP32, ...)

[TinyEMU](https://github.com/fernandotcl/TinyEMU) is a barebones RISC-V Emulator that runs in a [Web Browser](https://www.barebox.org/jsbarebox/?graphic=1). (Thanks to WebAssembly)

Can we boot NuttX in a Web Browser, with a little help from TinyEMU? Let's find out!

_Why are we doing this?_

We might run NuttX in a Web Browser and emulate the Ox64 BL808 RISC-V SBC. Which is great for testing NuttX Apps like [Nim Blinky LED](https://lupyuen.github.io/articles/nim)! Or even LVGL Apps with VirtIO Framebuffer?

Also imagine: A NuttX Dashboard that lights up in Real-Time, as the various NuttX Modules are activated! This is all possible when NuttX runs in a Web Browser!

(Sorry QEMU Emulator is a bit too complex to customise)

# Install TinyEMU

_How to run TinyEMU in the Command Line?_

We begin with TinyEMU in the Command Line, then move to WebAssembly. To install TinyEMU on macOS:

```bash
brew tap fernandotcl/homebrew-fernandotcl
brew install --HEAD fernandotcl/fernandotcl/tinyemu
temu https://bellard.org/jslinux/buildroot-riscv64.cfg
```

Or build TinyEMU on Ubuntu and macOS [with these steps](https://github.com/lupyuen/TinyEMU/blob/master/.github/workflows/ci.yml).

[(Generate the Emscripten JavaScript)](https://github.com/lupyuen/nuttx-tinyemu#build-tinyemu-for-webassembly-with-emscripten)

# RISC-V Addresses for TinyEMU

_Where in RAM will NuttX boot?_

TinyEMU is hardcoded to run at these RISC-V Addresses (yep it's really barebones): [riscv_machine.c](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L66-L82)

```c
#define LOW_RAM_SIZE   0x00010000 /* 64KB */
#define RAM_BASE_ADDR  0x80000000
#define CLINT_BASE_ADDR 0x02000000
#define CLINT_SIZE      0x000c0000
#define DEFAULT_HTIF_BASE_ADDR 0x40008000
#define VIRTIO_BASE_ADDR 0x40010000
#define VIRTIO_SIZE      0x1000
#define VIRTIO_IRQ       1
#define PLIC_BASE_ADDR 0x40100000
#define PLIC_SIZE      0x00400000
#define FRAMEBUFFER_BASE_ADDR 0x41000000

#define RTC_FREQ 10000000
#define RTC_FREQ_DIV 16 /* arbitrary, relative to CPU freq to have a
                           10 MHz frequency */
```

Thus we shall compile NuttX Kernel to boot at 0x8000_0000.

We begin with the NuttX Port for QEMU 64-bit RISC-V...

TODO: Can we change the above addresses to emulate a RISC-V SoC, like Ox64 BL808?

TODO: Wrap TinyEMU with Zig for safety and WebAssembly

# TinyEMU Config

_What's inside a TinyEMU Config?_

RISC-V Virtual Machines for TinyEMU are configured like this: [buildroot-riscv64.cfg](https://bellard.org/jslinux/buildroot-riscv64.cfg)

```text
/* VM configuration file */
{
  version: 1,
  machine: "riscv64",
  memory_size: 256,
  bios: "bbl64.bin",
  kernel: "kernel-riscv64.bin",
  cmdline: "loglevel=3 swiotlb=1 console=hvc0 root=root rootfstype=9p rootflags=trans=virtio ro TZ=${TZ}",
  fs0: { file: "https://vfsync.org/u/os/buildroot-riscv64" },
  eth0: { driver: "user" },
}
```

`bbl64.bin` is the [Barebox Bootloader](https://www.barebox.org). (Similar to U-Boot)

_Will NuttX go into `bios` or `kernel`?_

According to [copy_bios](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L753-L812), the BIOS is mandatory, the Kernel is optional.

Thus we put NuttX Kernel into `bios` and leave `kernel` empty.

[copy_bios](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L753-L812) will load NuttX Kernel at RAM_BASE_ADDR (0x8000_0000).

# Build NuttX for TinyEMU

_Will NuttX boot on TinyEMU?_

NuttX for QEMU RISC-V is already configured to boot at 0x8000_0000: [ld.script](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/boards/risc-v/qemu-rv/rv-virt/scripts/ld.script#L21-L27)

```text
SECTIONS
{
  . = 0x80000000;
  .text :
    {
      _stext = . ;
```

So we build NuttX for QEMU RISC-V (64-bit, Flat Mode)...

```bash
## Download WIP NuttX
git clone --branch tinyemu https://github.com/lupyuen2/wip-pinephone-nuttx nuttx
git clone --branch tinyemu https://github.com/lupyuen2/wip-pinephone-nuttx-apps apps

## Configure NuttX for QEMU RISC-V (64-bit, Flat Mode)
cd nuttx
tools/configure.sh rv-virt:nsh64
make menuconfig
## Device Drivers
##   Enable "Simple AddrEnv"
##   Enable "Virtio Device Support"

## Device Drivers > Virtio Device Support
##   Enable "Virtio MMIO Device Support"

## Build Setup > Debug Options >
##   Enable Debug Features
##   Enable "Debug Assertions > Show Expression, Filename"
##   Enable "Binary Loader Debug Features > Errors, Warnings, Info"
##   Enable "File System Debug Features > Errors, Warnings, Info"
##   Enable "C Library Debug Features > Errors, Warnings, Info"
##   Enable "Memory Manager Debug Features > Errors, Warnings, Info"
##   Enable "Scheduler Debug Features > Errors, Warnings, Info"
##   Enable "Timer Debug Features > Errors, Warnings, Info"
##   Enable "IPC Debug Features > Errors, Warnings, Info"
##   Enable "Virtio Debug Features > Error, Warnings, Info"

## Application Configuration > Testing >
##   Enable "OS Test Example"

## RTOS Features > Tasks and Scheduling >
##   Set "Application Entry Point" to "ostest_main"
##   Set "Application Entry Name" to "ostest_main"
## Save and exit menuconfig

## Build NuttX
make

## Export the Binary Image to nuttx.bin
riscv64-unknown-elf-objcopy \
  -O binary \
  nuttx \
  nuttx.bin

## Dump the disassembly to nuttx.S
riscv64-unknown-elf-objdump \
  --syms --source --reloc --demangle --line-numbers --wide \
  --debugging \
  nuttx \
  >nuttx.S \
  2>&1
```

# Run NuttX on TinyEMU

_How to boot NuttX on TinyEMU?_

We create a TinyEMU Config for NuttX and run it: [root-riscv64.cfg](https://github.com/lupyuen/nuttx-tinyemu/blob/main/docs/root-riscv64.cfg)

```bash
$ cat nuttx.cfg
/* VM configuration file */
{
  version: 1,
  machine: "riscv64",
  memory_size: 256,
  bios: "nuttx.bin",
}

$ temu nuttx.cfg
```

TinyEMU hangs, nothing happens. Let's print something to TinyEMU HTIF Console...

# Print to HTIF Console

_What's HTIF?_

From [RISC-V Spike Emulator](https://github.com/riscv-software-src/riscv-isa-sim/issues/364#issuecomment-607657754)...

> HTIF is a tether between a simulation host and target, not something that's supposed to resemble a real hardware device. It's not a RISC-V standard; it's a UC Berkeley standard. 

> Bits 63:56 indicate the "device".

> Bits 55:48 indicate the "command".

> Device 1 is the blocking character device.

> Command 0 reads a character

> Command 1 writes a character from the 8 LSBs of tohost

![TinyEMU with HTIF Console](https://lupyuen.github.io/images/tinyemu-htif.jpg) 

TinyEMU handles HTIF Commands like this: [riscv_machine.c](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L129-L153)

```c
static void htif_handle_cmd(RISCVMachine *s)
{
    uint32_t device, cmd;

    device = s->htif_tohost >> 56;
    cmd = (s->htif_tohost >> 48) & 0xff;
    if (s->htif_tohost == 1) {
        /* shuthost */
        printf("\nPower off.\n");
        exit(0);
    } else if (device == 1 && cmd == 1) {
        uint8_t buf[1];
        buf[0] = s->htif_tohost & 0xff;
        s->common.console->write_data(s->common.console->opaque, buf, 1);
        s->htif_tohost = 0;
        s->htif_fromhost = ((uint64_t)device << 56) | ((uint64_t)cmd << 48);
    } else if (device == 1 && cmd == 0) {
        /* request keyboard interrupt */
        s->htif_tohost = 0;
    } else {
        printf("HTIF: unsupported tohost=0x%016" PRIx64 "\n", s->htif_tohost);
    }
}
```

So to print `1` (ASCII 0x31) to the HTIF Console...

- device <br> = (htif_tohost >> 56) <br> = 1

- cmd <br> = (htif_tohost >> 48) <br> = 1

- buf <br> = (htif_tohost & 0xff) <br> = 0x31

Which means we write this value to htif_tohost...

- (1 << 56) | (1 << 48) | 0x31 <br> = 0x0101_0000_0000_0031

_Where is htif_tohost?_

According to [riscv_machine_init](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L913-L927) and [htif_write](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L154-L178), htif_tohost is at [DEFAULT_HTIF_BASE_ADDR](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L66-L82): 0x4000_8000

Thus we print to HTIF Console like this...

```c
// Print `1` to HTIF Console
*(volatile uint64_t *) 0x40008000 = 0x0101000000000031ul;
```

Let's print something in our NuttX Boot Code...

# Print in NuttX Boot Code

_How to print to HTIF Console in the NuttX Boot Code? (RISC-V Assembly)_

[Based on Star64 Debug Code](https://lupyuen.github.io/articles/nuttx2#print-to-qemu-console), we code this in RISC-V Assembly...

```text
/* Load HTIF Base Address to Register t0 */
li  t0, 0x40008000

/* Load to Register t1 the HTIF Command to print `1` */
li  t1, 0x0101000000000031
/* Store 64-bit double-word from Register t1 to HTIF Base Address, Offset 0 */
sd  t1, 0(t0)

/* Load to Register t1 the HTIF Command to print `2` */
li  t1, 0x0101000000000032
/* Store 64-bit double-word from Register t1 to HTIF Base Address, Offset 0 */
sd  t1, 0(t0)

/* Load to Register t1 the HTIF Command to print `3` */
li  t1, 0x0101000000000033
/* Store 64-bit double-word from Register t1 to HTIF Base Address, Offset 0 */
sd  t1, 0(t0)
```

We insert the above code into the NuttX Boot Code: [qemu_rv_head.S](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/arch/risc-v/src/qemu-rv/qemu_rv_head.S#L43-L61)

_Does it work?_

NuttX prints to the HTIF Console yay! Now we know that NuttX Boot Code is actually running on TinyEMU...

```text
$ temu nuttx.cfg
123
```

Let's fix the NuttX UART Driver...

# Fix the NuttX UART Driver for TinyEMU

_NuttX on TinyEMU has been awfully quiet. How to fix the UART Driver so that NuttX can print things?_

NuttX is still running on the QEMU UART Driver (16550). Let's make a quick patch so that we will see something in the TinyEMU HTIF Console: [uart_16550.c](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/drivers/serial/uart_16550.c#L1701-L1720)

```c
// Write one character to the UART (polled)
static void u16550_putc(FAR struct u16550_s *priv, int ch) {

  // Hardcode the HTIF Base Address
  *(volatile uint64_t *) 0x40008000 = 0x0101000000000000ul | ch;

  // Previously:
  // while ((u16550_serialin(priv, UART_LSR_OFFSET) & UART_LSR_THRE) == 0);
  // u16550_serialout(priv, UART_THR_OFFSET, (uart_datawidth_t)ch);
}
```

We skip the reading and writing of other UART Registers, because we'll patch them later: [uart_16550.c](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/drivers/serial/uart_16550.c#L604-L635)

```c
// Read UART Register
static inline uart_datawidth_t u16550_serialin(FAR struct u16550_s *priv, int offset) {
  return 0; ////
  // Commented out the rest
}

// Write UART Register
static inline void u16550_serialout(FAR struct u16550_s *priv, int offset, uart_datawidth_t value) {
  // Commented out the rest
}
```

And we won't wait for UART Ready, since we're not accessing the Line Control Register: [uart_16550.c](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/drivers/serial/uart_16550.c#L635-L673)

```c
// Wait until UART is not busy. This is needed before writing to Line Control Register.
// Otherwise we will get spurious interrupts on Synopsys DesignWare 8250.
static int u16550_wait(FAR struct u16550_s *priv) {
  // Nopez! No waiting for now
  return OK; ////
}
```

Now we see NuttX booting OK on TinyEMU yay!

```text
+ temu nuttx.cfg
123ABCnx_start: Entry
mm_initialize: Heap: name=Umem, start=0x80035700 size=33335552
mm_addregion: [Umem] Region 1: base=0x800359a8 size=33334864
mm_malloc: Allocated 0x800359d0, size 48
mm_malloc: Allocated 0x80035a00, size 288
mm_malloc: Allocated 0x80035b20, size 32
mm_malloc: Allocated 0x80035b40, size 720
mm_malloc: Allocated 0x80035e10, size 80
mm_malloc: Allocated 0x80035e60, size 64
mm_malloc: Allocated 0x80035ea0, size 240
mm_malloc: Allocated 0x80035f90, size 464
mm_malloc: Allocated 0x80036160, size 176
mm_malloc: Allocated 0x80036210, size 336
mm_malloc: Allocated 0x80036360, size 464
mm_malloc: Allocated 0x80036530, size 464
mm_malloc: Allocated 0x80036700, size 528
builtin_initialize: Registering Builtin Loader
elf_initialize: Registering ELF
uart_register: Registering /dev/console
mm_malloc: Allocated 0x80036910, size 80
mm_malloc: Allocated 0x80036960, size 80
uart_register: Registering /dev/ttyS0
mm_malloc: Allocated 0x800369b0, size 80
mm_malloc: Allocated 0x80036a00, size 80
mm_malloc: Allocated 0x80036a50, size 80
mm_malloc: Allocated 0x80036aa0, size 32
mm_malloc: Allocated 0x80036ac0, size 160
mm_malloc: Allocated 0x80036b60, size 32
mm_malloc: Allocated 0x80036b80, size 32
mm_malloc: Allocated 0x80036ba0, size 32
nx_start_application: Starting init thread
task_spawn: name=ostest_main entry=0x80006fde file_actions=0 attr=0x80035670 argv=0x80035668
mm_malloc: Allocated 0x80036bc0, size 272
mm_malloc: Allocated 0x80036cd0, size 288
mm_malloc: Allocated 0x80036df0, size 32
mm_malloc: Allocated 0x80036e10, size 720
mm_malloc: Allocated 0x800370e0, size 32
mm_malloc: Allocated 0x80037100, size 32
mm_malloc: Allocated 0x80037120, size 32
mm_malloc: Allocated 0x80037140, size 32
mm_malloc: Allocated 0x80037160, size 160
mm_malloc: Allocated 0x80037200, size 3088
mm_free: Freeing 0x80036b60
mm_free: Freeing 0x80036ba0
mm_free: Freeing 0x80036b80
mm_malloc: Allocated 0x80036b60, size 32
mm_malloc: Allocated 0x80036b80, size 32
mm_malloc: Allocated 0x80037e10, size 48
mm_free: Freeing 0x800370e0
mm_free: Freeing 0x80036b60
mm_free: Freeing 0x80036b80
mm_malloc: Allocated 0x800370e0, size 32
nx_start: CPU0: Beginning Idle Loop
```

Let's boot NuttX in the Web Browser...

# Boot NuttX in the Web Browser

_Will NuttX boot in the Web Browser?_

Yep! WebAssembly Demo is here: [Demo of NuttX on TinyEMU](https://lupyuen.github.io/nuttx-tinyemu/)

WebAssembly Files are located here: [nuttx-tinyemu/docs](https://github.com/lupyuen/nuttx-tinyemu/tree/main/docs)

![Apache NuttX RTOS in the Web Browser: TinyEMU with VirtIO](https://lupyuen.github.io/images/tinyemu-title.png)

We copied the TinyEMU Config and NuttX Kernel to the Web Server...

```bash
## Copy to Web Server: NuttX Config, Kernel, Disassembly (for troubleshooting)
cp nuttx.cfg ../nuttx-tinyemu/docs/root-riscv64.cfg
cp nuttx.bin ../nuttx-tinyemu/docs/
cp nuttx.S ../nuttx-tinyemu/docs/
```

The other files were provided by [TinyEMU](https://bellard.org/tinyemu/)...

- [jslinux-2019-12-21.tar.gz](https://bellard.org/tinyemu/jslinux-2019-12-21.tar.gz): Precompiled JSLinux demo

  [(Fixed for __Mobile Keyboards__)](https://github.com/lupyuen/nuttx-tinyemu/commit/33f0857a4a5ac8da899b159331be4ea258d490ca)

TODO: Where is the updated source code for the WebAssembly? What is the implementation of `console_resize_event`? Hmmm...

_How to test this locally?_

To test on our computer, we need to install a Local Web Server (because our Web Browser won't load WebAssembly Files from the File System)...

```bash
## Based on https://github.com/TheWaWaR/simple-http-server
$ cargo install simple-http-server
$ git clone https://github.com/lupyuen/nuttx-tinyemu
$ simple-http-server nuttx-tinyemu/docs
```

Then browse to...

```text
http://0.0.0.0:8000/index.html
```

_But there's no Console Input?_

To do Console Input, we need to implement VirtIO Console in our NuttX UART Driver...

# VirtIO Console in TinyEMU

_How will we implement Console Input / Output in NuttX TinyEMU?_

TinyEMU supports VirtIO for proper Console Input and Output...

- [TinyEMU support for VirtIO](https://bellard.org/tinyemu/readme.txt)

- [Virtio - OSDev Wiki](https://wiki.osdev.org/Virtio)

- [Virtual I/O Device (VIRTIO) Spec, Version 1.2](https://docs.oasis-open.org/virtio/virtio/v1.2/csd01/virtio-v1.2-csd01.html)

- [About VirtIO Console](https://projectacrn.github.io/latest/developer-guides/hld/virtio-console.html)

And NuttX supports VirtIO, based on OpenAMP...

- [Running NuttX with VirtIO on QEMU](https://www.youtube.com/watch?v=_8CpLNEWxfo)

- [NuttX VirtIO Framework and Future Works](https://www.youtube.com/watch?v=CYMkAv-WjQg)

- [Intro to OpenAMP](https://www.openampproject.org/docs/whitepapers/Introduction_to_OpenAMPlib_v1.1a.pdf)

- [knetnsh64: NuttX for QEMU RISC-V with VirtIO](https://github.com/apache/nuttx/blob/master/boards/risc-v/qemu-rv/rv-virt/configs/knetnsh64/defconfig#L52)

But let's create a simple VirtIO Console Driver for NuttX with OpenAMP...

- Create Queue: Call OpenAMP [virtqueue_create](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtqueue.c#L49)

  (See [virtio_mmio_create_virtqueue](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-mmio.c#L349-L414) or [virtio_create_virtqueues](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtio.c#L96-L142))

- Send Data: Call OpenAMP [virtqueue_add_buffer](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtqueue.c#L83C1-L138)

  (See [virtio_serial_dmasend](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-serial.c#L310-L345))

- Start Processing: Call OpenAMP [virtqueue_kick](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtqueue.c#L321-L336)

  (See [virtio_serial_dmasend](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-serial.c#L310-L345))

This will help us understand the inner workings of VirtIO and OpenAMP! But first we enable VirtIO and OpenAMP in NuttX...

![TinyEMU with VirtIO Console](https://lupyuen.github.io/images/tinyemu-virtio.jpg) 

# Enable VirtIO and OpenAMP in NuttX

_How do we call VirtIO and OpenAMP?_

To enable VirtIO and OpenAMP in NuttX:

```text
make menuconfig
## Device Drivers
##   Enable "Simple AddrEnv"
##   Enable "Virtio Device Support"

## Device Drivers > Virtio Device Support
##   Enable "Virtio MMIO Device Support"

## Build Setup > Debug Options >
##   Enable "Virtio Debug Features > Error, Warnings, Info"
```

_Why "Simple AddrEnv"?_

`up_addrenv_va_to_pa` is defined in [drivers/misc/addrenv.c](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/drivers/misc/addrenv.c#L89-L112). So we need `CONFIG_DEV_SIMPLE_ADDRENV` (Simple AddrEnv)

Otherwise we see this...

```text
riscv64-unknown-elf-ld: nuttx/staging/libopenamp.a(io.o): in function `metal_io_phys_to_offset_':
nuttx/openamp/libmetal/lib/system/nuttx/io.c:105: undefined reference to `up_addrenv_pa_to_va'
riscv64-unknown-elf-ld: nuttx/staging/libopenamp.a(io.o): in function `metal_io_offset_to_phys_':
nuttx/openamp/libmetal/lib/system/nuttx/io.c:99: undefined reference to `up_addrenv_va_to_pa'
```

Now we configure NuttX VirtIO...

# Configure NuttX VirtIO for TinyEMU

_How to make NuttX VirtIO talk to TinyEMU?_

Previously we saw the TinyEMU config: [riscv_machine.c](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L66-L82)

```c
#define VIRTIO_BASE_ADDR 0x40010000
#define VIRTIO_SIZE      0x1000
#define VIRTIO_IRQ       1
```

Now we set the VirtIO Parameters for TinyEMU in NuttX: [qemu_rv_appinit.c](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/boards/risc-v/qemu-rv/rv-virt/src/qemu_rv_appinit.c#L41-L49)

```c
#define QEMU_VIRTIO_MMIO_BASE    0x40010000 // VIRTIO_BASE_ADDR. Previously: 0x10001000
#define QEMU_VIRTIO_MMIO_REGSIZE 0x1000     // VIRTIO_SIZE
#ifdef CONFIG_ARCH_USE_S_MODE
#  define QEMU_VIRTIO_MMIO_IRQ   26 // TODO: Should this be 1? (VIRTIO_IRQ)
#else
#  define QEMU_VIRTIO_MMIO_IRQ   28 // TODO: Should this be 1? (VIRTIO_IRQ)
#endif
#define QEMU_VIRTIO_MMIO_NUM     1  // Number of VirtIO Devices. Previously: 8
```

With these settings, VirtIO and OpenAMP will start OK on NuttX yay!

```text
virtio_mmio_init_device: VIRTIO version: 2 device: 3 vendor: ffff
mm_malloc: Allocated 0x80046a90, size 48
test_virtio: 
mm_malloc: Allocated 0x80046ac0, size 848
nx_start: CPU0: Beginning Idle Loop
```

Which means NuttX VirtIO + OpenAMP has successfully validated the Magic Number from TinyEMU. (Otherwise NuttX will halt)

_How does it work?_

At NuttX Startup: [board_app_initialize](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/boards/risc-v/qemu-rv/rv-virt/src/qemu_rv_appinit.c#L77-L123) calls...

- [qemu_virtio_register_mmio_devices](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/boards/risc-v/qemu-rv/rv-virt/src/qemu_rv_appinit.c#L54-L73) (to register all VirtIO MMIO Devices) which calls...

- [virtio_register_mmio_device](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/drivers/virtio/virtio-mmio.c#L809-L932) 
(to register a VirtIO MMIO Device, explained below)

Let's create a VirtIO Queue for the VirtIO Console and send some data...

# Test TinyEMU VirtIO Console with NuttX

_NuttX has started VirtIO and OpenAMP and they talk nicely to TinyEMU. What next?_

We dig around NuttX and we see NuttX creating a VirtIO Queue for VirtIO Console: [virtio_serial_init](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-serial.c#L445-L511) calls...

- OpenAMP [virtio_create_virtqueues](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtio.c#L96-L142) (create data queues, explained below)

Also we see NuttX sending data to VirtIO Console: [virtio_serial_dmasend](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-serial.c#L310-L345) calls...

- OpenAMP [virtqueue_add_buffer](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtqueue.c#L83C1-L138) (send data to queue) and...

  OpenAMP [virtqueue_kick](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtqueue.c#L321-L336) (start queue processing, explained below)

Let's do all these in our NuttX Test Code: [virtio-mmio.c](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/drivers/virtio/virtio-mmio.c#L870-L925)

```c
  // Testing: Init VirtIO Device
  // Based on virtio_serial_init
  // https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-serial.c#L445-L511

  struct virtio_device *vdev = &vmdev->vdev;
  DEBUGASSERT(vdev != NULL);

  virtio_set_status(vdev, VIRTIO_CONFIG_STATUS_DRIVER);
  virtio_set_features(vdev, 0);
  virtio_set_status(vdev, VIRTIO_CONFIG_FEATURES_OK);

  #define VIRTIO_SERIAL_RX           0
  #define VIRTIO_SERIAL_TX           1
  #define VIRTIO_SERIAL_NUM          2
  const char *vqnames[VIRTIO_SERIAL_NUM];
  vqnames[VIRTIO_SERIAL_RX]   = "virtio_serial_rx";
  vqnames[VIRTIO_SERIAL_TX]   = "virtio_serial_tx";

  vq_callback callbacks[VIRTIO_SERIAL_NUM];
  callbacks[VIRTIO_SERIAL_RX] = NULL; // TODO: virtio_serial_rxready;
  callbacks[VIRTIO_SERIAL_TX] = NULL; // TODO: virtio_serial_txdone;
  ret = virtio_create_virtqueues(vdev, 0, VIRTIO_SERIAL_NUM, vqnames,
                                 callbacks);
  DEBUGASSERT(ret >= 0);
  virtio_set_status(vdev, VIRTIO_CONFIG_STATUS_DRIVER_OK);

  // Testing: Send data to VirtIO Device
  // Based on virtio_serial_dmasend
  // https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-serial.c#L310-L345

  DEBUGASSERT(vdev->vrings_info != NULL);
  struct virtqueue *vq = vdev->vrings_info[VIRTIO_SERIAL_TX].vq;
  DEBUGASSERT(vq != NULL);

  /* Set the virtqueue buffer */
  static char *HELLO_MSG = "Hello VirtIO from NuttX!\r\n";
  struct virtqueue_buf vb[2];
  vb[0].buf = HELLO_MSG;
  vb[0].len = strlen(HELLO_MSG);
  int num = 1;

  /* Get the total send length */
  uintptr_t len = strlen(HELLO_MSG);

  // TODO: What's this?
  // if (xfer->nlength != 0)
  //   {
  //     vb[1].buf = xfer->nbuffer;
  //     vb[1].len = xfer->nlength;
  //     num = 2;
  //   }

  /* Add buffer to TX virtiqueue and notify the VirtIO Host */
  virtqueue_add_buffer(vq, vb, num, 0, (FAR void *)len);
  virtqueue_kick(vq);  
  // End of Testing
```

_Does it work?_

Yep NuttX prints correctly to TinyEMU's VirtIO Console yay!

[__Demo of NuttX on TinyEMU: lupyuen.github.io/nuttx-tinyemu__](https://lupyuen.github.io/nuttx-tinyemu/)

```text
+ temu nuttx.cfg
123ABCnx_start: Entry
builtin_initialize: Registering Builtin Loader
elf_initialize: Registering ELF
uart_register: Registering /dev/console
uart_register: Registering /dev/ttyS0
nx_start_application: Starting init thread
task_spawn: name=nsh_main entry=0x8000756e file_actions=0 attr=0x80043e80 argv=0x80043e78
virtio_mmio_init_device: VIRTIO version: 2 device: 3 vendor: ffff
Hello VirtIO from NuttX!
nx_start: CPU0: Beginning Idle Loop
```

[(See the Complete Log)](https://gist.github.com/lupyuen/8805f8f21dfae237bc06dfbda210628b)

![Apache NuttX RTOS in the Web Browser: TinyEMU with VirtIO](https://lupyuen.github.io/images/tinyemu-title.png)

# Enable the VirtIO Serial Driver

Now we implement Console Input / Output with the NuttX Serial Driver for VirtIO:

- [See the Modified Files](https://github.com/lupyuen2/wip-pinephone-nuttx/pull/50/files)

- [See the Full Demo](https://lupyuen.github.io/nuttx-tinyemu)

```text
Device Drivers > Virtio Device Support
  Enable "Virtio Serial Support"

Device Drivers > Serial Driver Support
  Disable "16550 UART Chip support"
```

Which will start a tiny bit of NuttX Shell...

```text
+ temu nuttx.cfg
123Ariscv_earlyserialinit: 
BCnx_start: Entry
riscv_serialinit: 
virtio_mmio_init_device: VIRTIO version: 2 device: 3 vendor: ffff
uart_register: Registering /dev/console
virtio_register_serial_driver: ret1=0
virtio_register_serial_driver: ret2=0
nx_start_application: Starting init thread
task_spawn: name=nsh_main entry=0x80008874 file_actions=0 attr=0x80044b30 argv=0x80044b28

NuttShell (NSH) NuttX-12.3.0-RC1
nx_start: CPU0: Beginning Idle Loop
```

NuttX Console crashes because we didn't initialise VirtIO early enough. So we moved the VirtIO Init from [qemu_rv_appinit.c](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/boards/risc-v/qemu-rv/rv-virt/src/qemu_rv_appinit.c)...

```c
int board_app_initialize(uintptr_t arg) {
  ...
#ifdef CONFIG_DRIVERS_VIRTIO_MMIO
  //// Moved to nuttx/arch/risc-v/src/qemu-rv/qemu_rv_start.c
  //// Previously: qemu_virtio_register_mmio_devices();
#endif
```

To [qemu_rv_start.c](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/arch/risc-v/src/qemu-rv/qemu_rv_start.c#L233-L240)...

```c
void riscv_serialinit(void) {
  // Init the VirtIO Devices
  void qemu_virtio_register_mmio_devices(void);
  qemu_virtio_register_mmio_devices();
}
```

We created our own HTIF Driver, so 16550 UART Driver is no longer needed for Kernel Logging: [qemu_rv_start.c](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/arch/risc-v/src/qemu-rv/qemu_rv_start.c#L240-L271)

```c
// Print to HTIF Console
static void htif_putc(int ch) {
  // Hardcode the HTIF Base Address and print: device=1, cmd=1, buf=ch
  *(volatile uint64_t *) 0x40008000 = 0x0101000000000000ul | ch;
}

int up_putc(int ch) {
  irqstate_t flags;

  /* All interrupts must be disabled to prevent re-entrancy and to prevent
   * interrupts from firing in the serial driver code.
   */

  flags = enter_critical_section();

  /* Check for LF */

  if (ch == '\n')
    {
      /* Add CR */

      htif_putc('\r');
    }

  htif_putc(ch);
  leave_critical_section(flags);

  return ch;
}
```

NuttX Apps will use the VirtIO Serial Driver to access the NuttX Console...

[(Live Demo of NuttX on TinyEMU)](https://lupyuen.github.io/nuttx-tinyemu)

[(Watch on YouTube)](https://youtu.be/KYrdwzIsgeQ)

[(See the Modified Files)](https://github.com/lupyuen2/wip-pinephone-nuttx/pull/50/files)

![Live Demo of NuttX on TinyEMU](https://lupyuen.github.io/images/tinyemu-nsh.png) 

# Enable NuttX Console for VirtIO

_Nothing appears when we type in NuttX Shell. Why?_

That's because we haven't enabled the Echoing of Keypresses! Here's the fix: [virtio-serial.c](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/drivers/virtio/virtio-serial.c#L451-L490)

```c
static int virtio_serial_init(FAR struct virtio_serial_priv_s *priv, FAR struct virtio_device *vdev) {
  ...
  //// TinyEMU needs NuttX to echo the keypress and change CR to NL
  udev->isconsole = true; ////
```

This will...

- Echo all keys pressed

- If the key pressed is Carriage Return `\r`, convert to Line Feed `\n`

- TOOO: What else?

When we enable the NuttX Console for VirtIO, NuttX Shell works correctly yay!

[(Live Demo of NuttX on TinyEMU)](https://lupyuen.github.io/nuttx-tinyemu)

[(Watch on YouTube)](https://youtu.be/KYrdwzIsgeQ)

[(See the Modified Files)](https://github.com/lupyuen2/wip-pinephone-nuttx/pull/50/files)

![Live Demo of NuttX on TinyEMU](https://lupyuen.github.io/images/tinyemu-nsh2.png) 

# TinyEMU can't enable Machine-Mode Software Interrupts

[Based on our snooping](https://github.com/lupyuen/nuttx-tinyemu#virtio-console-input-in-tinyemu), we see that TinyEMU's VirtIO Console will [Trigger an Interrupt](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_cpu_template.h#L220-L258) like so...

```c
/* check pending interrupts */
if (unlikely((s->mip & s->mie) != 0)) {
  if (raise_interrupt(s)) {
    s->n_cycles--; 
    goto done_interp;
  }
}
```

This means that MIP (Machine-Mode Interrupt Pending Register) must have the same bits set as MIE (Machine-Mode Interrupt Enable Register).

But we have a problem: TinyEMU won't let us set the MEIE Bit (Machine-Mode External Interrupt Enable) in MIE Register!

From [qemu_rv_irq.c](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/arch/risc-v/src/qemu-rv/qemu_rv_irq.c#L204C1-L222):

```c
  /* Enable external interrupts (mie/sie) */
  // TODO: TinyEMU won't let us set the MEIE Bit (Machine-Mode External Interrupt Enable) in MIP!
  { uint64_t mie = READ_CSR(mie); _info("Before mie: %p\n", mie); }
  // CSR_IE is MIE
  // IE_EIE is MEIE
  SET_CSR(CSR_IE, IE_EIE);
  { uint64_t mie = READ_CSR(mie); _info("After mie: %p\n", mie); }
```

Which shows that MEIE Bit in MIE Register is NOT SET correctly: [NuttX Log](https://gist.github.com/lupyuen/8b342300f03cd4b0758995f0e0c5c646):

```text
up_irq_enable: Before mie: 0
up_irq_enable: After mie: 0
```

Our workaround is to use the SEIE Bit (Supervisor-Mode Externel Interrupt Enable) in MIE Register...

From [qemu_rv_irq.c](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/arch/risc-v/src/qemu-rv/qemu_rv_irq.c#L204C1-L222):

```c
  // TODO: TinyEMU supports SEIE but not MEIE!
  uint64_t mie = READ_CSR(mie); _info("mie: %p\n", mie);

  // TODO: This doesn't work
  // Enable MEIE: Machine-Mode External Interrupt  
  // WRITE_CSR(mie, mie | (1 << 11));

  // TODO: This works, but we need MEIE, not SEIE. We patch this in riscv_dispatch_irq()
  // Enable SEIE: Supervisor-Mode External Interrupt
  WRITE_CSR(mie, mie | (1 << 9));
  mie = READ_CSR(mie); _info("mie: %p\n", mie);
```

Which shows that SEIE Bit in MIE Register is set correctly: [NuttX Log](https://gist.github.com/lupyuen/8b342300f03cd4b0758995f0e0c5c646):

```text
up_irq_enable: mie: 0
up_irq_enable: mie: 0x200
```

Then we patch the NuttX Exception Handler to map Supervisor-Mode Interrupts into Machine-Mode Interrupts: [riscv_dispatch_irq](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/arch/risc-v/src/qemu-rv/qemu_rv_irq_dispatch.c#L52-L92)

```c
void *riscv_dispatch_irq(uintptr_t vector, uintptr_t *regs)
{
  int irq = (vector >> RV_IRQ_MASK) | (vector & 0xf);

  // TODO: TinyEMU works only with SEIE, not MEIE
  if (irq == RISCV_IRQ_SEXT) { irq = RISCV_IRQ_MEXT; }
```

TODO: Find out why TinyEMU can't set the MEIE Bit (Machine-Mode External Interrupt Enable) in MIE

# TinyEMU supports VirtIO Block, Network, Input and Filesystem Devices

_We've done VirtIO Console with TinyEMU. What other VirtIO Devices are supported in Web Browser TinyEMU?_

TinyEMU supports these VirtIO Devices:

- [Console Device](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L1259-L1361)

- [Block Device](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L979-L1133)

- [Network Device](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L1133-L1259)

- [Input Device](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L1361-L1645)

- [9P Filesystem Device](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L1645-L2649)

More details in the [TinyEMU Doc](https://bellard.org/tinyemu/readme.txt). (Are all these devices supported in the Web Browser?)

The Network Device is explained in the [JSLinux FAQ](https://bellard.org/jslinux/faq.html)...

> Can I access to the network from the virtual machine ?

> Yes it is possible. It uses the websocket VPN offered by Benjamin Burns [(see his blog)](http://www.benjamincburns.com/2013/11/10/jor1k-ethmac-support.html). The bandwidth is capped to 40 kB/s and at most two connections are allowed per public IP address. Please don't abuse the service.

# NuttX in Kernel Mode

_Right now we're running NuttX in Flat Mode..._

_Can NuttX run in Kernel Mode on TinyEMU?_

NuttX Kernel Mode requires [RISC-V Semihosting](https://lupyuen.github.io/articles/semihost#semihosting-on-nuttx-qemu) to access the NuttX Apps Filesystem. Which is supported by QEMU but not TinyEMU.

But we can [Append the Initial RAM Disk](https://lupyuen.github.io/articles/app#initial-ram-disk) to the NuttX Kernel. So yes it's possible to run NuttX in Kernel Mode with TinyEMU, with some additional [Mounting Code](https://lupyuen.github.io/articles/app#mount-the-initial-ram-disk).

# Inside the VirtIO Driver for NuttX

_How does VirtIO Guest work in NuttX?_

NuttX VirtIO Driver is based on OpenAMP with MMIO...

- [Running NuttX with VirtIO on QEMU](https://www.youtube.com/watch?v=_8CpLNEWxfo)

- [NuttX VirtIO Framework and Future Works](https://www.youtube.com/watch?v=CYMkAv-WjQg)

At NuttX Startup: [board_app_initialize](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/boards/risc-v/qemu-rv/rv-virt/src/qemu_rv_appinit.c#L77-L123) calls...

- [qemu_virtio_register_mmio_devices](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/boards/risc-v/qemu-rv/rv-virt/src/qemu_rv_appinit.c#L54-L73) (to register all VirtIO MMIO Devices) which calls...

- [virtio_register_mmio_device](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu/drivers/virtio/virtio-mmio.c#L809-L932) 
(to register a VirtIO MMIO Device) which calls...

- [virtio_mmio_init_device](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-mmio.c#L740-L805) which passes...

- [g_virtio_mmio_dispatch](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-mmio.c#L234-L254) which contains...

- [virtio_mmio_create_virtqueues](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-mmio.c#L419) which calls...

- [virtio_mmio_create_virtqueue](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-mmio.c#L349-L414) which calls...

- [virtqueue_create](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtqueue.c#L49) (OpenAMP)

To create a VirtIO Queue for VirtIO Console: [virtio_serial_probe](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-serial.c#L530) calls...

- [virtio_serial_init](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-serial.c#L445-L511) which calls...

- [virtio_create_virtqueues](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtio.c#L96-L142) (OpenAMP)

To send data to VirtIO Console: [virtio_serial_send](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-serial.c#L245) calls...

- [virtio_serial_dmatxavail](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-serial.c#L345-L357) which calls...

- [uart_xmitchars_dma](https://github.com/apache/nuttx/blob/master/drivers/serial/serial_dma.c#L86-L125) which calls...

- [virtio_serial_dmasend](https://github.com/apache/nuttx/blob/master/drivers/virtio/virtio-serial.c#L310-L345) which calls...

- [virtqueue_add_buffer](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtqueue.c#L83C1-L138) (OpenAMP) and...

  [virtqueue_kick](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtqueue.c#L321-L336) (OpenAMP)

# Inside the VirtIO Host for TinyEMU

_How does VirtIO Host work in TinyEMU?_

Let's look inside the implementation of VirtIO in TinyEMU...

## TinyEMU VirtIO

TinyEMU supports these VirtIO Devices:

- [Console Device](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L1259-L1361)

- [Block Device](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L979-L1133)

- [Network Device](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L1133-L1259)

- [Input Device](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L1361-L1645)

- [9P Filesystem Device](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L1645-L2649)

The Device IDs are: [virtio_init](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L219-L297)

```c
switch(device_id) {
case 1: /* net */ ...
case 2: /* block */ ...
case 3: /* console */ ...
case 9: /* Network Device */ ...
case 18: /* Input Device */ ...
```

TinyEMU supports VirtIO over MMIO and PCI:

- [MMIO addresses](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L37)

- [PCI registers](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L66)

TinyEMU Guests (like NuttX) are required to check the [VIRTIO_MMIO_MAGIC_VALUE](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L617) that's returned by the TinyEMU Host.

## TinyEMU VirtIO Console

From above: VirtIO Console is Device ID 3. Here's how it works...

At TinyEMU Startup: [riscv_machine_init](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L952) calls...

- [virtio_console_init](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L1347-L1361) which calls...

- [virtio_init](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L219-L297) with Device ID 3

To print to VirtIO Console: [virt_machine_run (js)](https://github.com/fernandotcl/TinyEMU/blob/master/jsemu.c#L304-L348) and [virt_machine_run (temu)](https://github.com/fernandotcl/TinyEMU/blob/master/temu.c#L545-L610) call...

- [virtio_console_write_data](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L1317-L1337) which calls...

- [memcpy_to_queue](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L451-L459) which calls...

- [memcpy_to_from_queue](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L380)

Which will access...

- [QueueState](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L97-L107): For desc_addr, avail_addr, used_addr

- [VIRTIODesc](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L111-L118): For [VirtualQueue::Buffers[QueueSize]](https://wiki.osdev.org/Virtio#Virtual_Queue_Descriptor)

TinyEMU Console Device:

- [console device decl](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.h#L108)

- [console device impl](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L1261)

## TinyEMU VirtIO MMIO Queue

TinyEMU Guest (like NuttX) is required to set the VirtIO Queue Desc / Avail / Used.

This is how TinyEMU accesses the VirtIO MMIO Queue: [virtio.c](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L645)

```c
case VIRTIO_MMIO_QUEUE_SEL:
    val = s->queue_sel;
    break;
case VIRTIO_MMIO_QUEUE_NUM_MAX:
    val = MAX_QUEUE_NUM;
    break;
case VIRTIO_MMIO_QUEUE_NUM:
    val = s->queue[s->queue_sel].num;
    break;
case VIRTIO_MMIO_QUEUE_DESC_LOW:
    val = s->queue[s->queue_sel].desc_addr;
    break;
case VIRTIO_MMIO_QUEUE_AVAIL_LOW:
    val = s->queue[s->queue_sel].avail_addr;
    break;
case VIRTIO_MMIO_QUEUE_USED_LOW:
    val = s->queue[s->queue_sel].used_addr;
    break;
#if VIRTIO_ADDR_BITS == 64
case VIRTIO_MMIO_QUEUE_DESC_HIGH:
    val = s->queue[s->queue_sel].desc_addr >> 32;
    break;
case VIRTIO_MMIO_QUEUE_AVAIL_HIGH:
    val = s->queue[s->queue_sel].avail_addr >> 32;
    break;
case VIRTIO_MMIO_QUEUE_USED_HIGH:
    val = s->queue[s->queue_sel].used_addr >> 32;
    break;
#endif
```

To Select and Notify the Queue:

- [VIRTIO_MMIO_QUEUE_SEL](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L741)

- [VIRTIO_MMIO_QUEUE_NOTIFY](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L781)

# VirtIO Console Input in TinyEMU

_How does TinyEMU handle VirtIO Console Input?_

Suppose we press a key in TinyEMU. From the [Detailed Console Input Log](https://gist.github.com/lupyuen/1f0bbf1a749e58f1c467b50a031886fd)...

```text
virtio_console_get_write_len
virtio_console_write_data: ready=1
virtio_console_write_data: last_avail_idx=0, avail_idx=1
```

TinyEMU [virt_machine_run](https://github.com/fernandotcl/TinyEMU/blob/master/temu.c#L545-L603) calls...

- [virtio_console_write_data](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L1317-L1337) to write the key pressed into the VirtIO Console's RX Queue

TinyEMU triggers an Interrupt...

```text
plic_set_irq: irq_num=1, state=1
plic_update_mip: set_mip, pending=0x1, served=0x0
virtio_console_write_data: buf[0]=l, buf_len=1
raise_exception: cause=-2147483639
raise_exception2: cause=-2147483639, tval=0x0
```

[virtio_console_write_data](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L1317-L1337) calls...

- [virtio_consume_desc](https://github.com/fernandotcl/TinyEMU/blob/master/virtio.c#L459-L479) (to notify the queue) which calls...

- [plic_set_irq](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L303-L316) (to set the PLIC Interrupt) which calls...

- [plic_update_mip](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L241C1-L253) (to set the Machine-Mode Interrupt Pending Register) which...

- [Triggers an Interrupt](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_cpu_template.h#L220-L258) which calls...

  ```c
  /* check pending interrupts */
  if (unlikely((s->mip & s->mie) != 0)) {
    if (raise_interrupt(s)) {
      s->n_cycles--; 
      goto done_interp;
    }
  }
  ```

- [raise_interrupt](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_cpu.c#L1185C1-L1198) which calls...

- [raise_exception](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_cpu.c#L1121C1-L1126) which calls...

- [raise_exception2](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_cpu.c#L1041C1-L1121)

This invokes the NuttX Exception Handler: [riscv_dispatch_irq](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/arch/risc-v/src/qemu-rv/qemu_rv_irq_dispatch.c#L52-L92)

[NuttX Exception Handler](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/arch/risc-v/src/qemu-rv/qemu_rv_irq_dispatch.c#L52-L92) handles the Interrupt...

```text
plic_read: offset=0x200004
plic_update_mip: reset_mip, pending=0x1, served=0x1
plic_set_irq: irq_num=1, state=0
plic_update_mip: reset_mip, pending=0x0, served=0x1
```

NuttX [riscv_dispatch_irq](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/arch/risc-v/src/qemu-rv/qemu_rv_irq_dispatch.c#L52-L92) reads the PLIC Interrupt Claim Register at PLIC Offset 0x200004, which calls...

- TinyEMU [plic_read](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L253-L284) (to read the PLIC Interrupt Claim Register at PLIC Offset 0x200004) which calls...

- [plic_set_irq](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L303-L316) (to clear the PLIC Interrupt) which calls...

- [plic_update_mip](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L241C1-L253) (to clear the Machine-Mode Interrupt Pending Register)

[NuttX Exception Handler](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/arch/risc-v/src/qemu-rv/qemu_rv_irq_dispatch.c#L52-L92) calls the VirtIO Serial Driver...

```text
virtio_serial_rxready: buf[0]=l, len=1
uart_recvchars_done: 
uart_datareceived: 
virtio_serial_dmarxfree: length=0
virtio_serial_dmareceive: buf[0]=, len=254
virtio_serial_dmareceive: num=1, length=254, nlength=0
uart_read: ch=l
virtio_serial_dmarxfree: length=254
uart_read: buf[0]=l, recvd=1
readline_common: ch=0x6c
virtio_serial_dmarxfree: length=254
virtio_serial_dmarxfree: length=254
```

We explain these in the next section.

To finish up, [NuttX Exception Handler](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/arch/risc-v/src/qemu-rv/qemu_rv_irq_dispatch.c#L52-L92) Completes the Interrupt by writing to PLIC Interrupt Claim Register at PLIC Offset 0x200004...

```text
plic_write: offset=0x200004, val=0x1
plic_update_mip: reset_mip, pending=0x0, served=0x0
raise_exception2: cause=11, tval=0x0
```

TinyEMU [plic_write](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L284-L303)  writes to PLIC Interrupt Claim Register at PLIC Offset 0x200004, and calls...

- [plic_update_mip](https://github.com/fernandotcl/TinyEMU/blob/master/riscv_machine.c#L241C1-L253) (to clear the Machine-Mode Interrupt Pending Register)

_How did we figure out all this?_

We added [Debug Logs to TinyEMU](https://github.com/lupyuen/TinyEMU/commits/master/).

# VirtIO Console Input in NuttX

_Inside NuttX: What happens when we press a key in TinyEMU?_

From the [Detailed Console Input Log](https://gist.github.com/lupyuen/1f0bbf1a749e58f1c467b50a031886fd)...

```text
virtio_serial_rxready: buf[0]=l, len=1
uart_recvchars_done: 
uart_datareceived: 
```

Which says that [NuttX Exception Handler](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/arch/risc-v/src/qemu-rv/qemu_rv_irq_dispatch.c#L52-L92) calls...

- [virtio_serial_rxready](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/drivers/virtio/virtio-serial.c#L397-L424) (to handle the keypress) which calls...

- OpenAMP [virtqueue_get_buffer](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtqueue.c#L138-L177) (to read the buffer from Receive Queue) and...

  [uart_recvchars_done](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/drivers/serial/serial_dma.c#L292-L361) (to receive the keypress, see below)

[uart_recvchars_done](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/drivers/serial/serial_dma.c#L292-L361) calls

- [uart_datareceived](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/drivers/serial/serial.c#L1849-L1880) (to notify the waiting threads)

```text
virtio_serial_dmarxfree: length=0
virtio_serial_dmareceive: buf[0]=, len=254
virtio_serial_dmareceive: num=1, length=254, nlength=0
```

Then [virtio_serial_rxready](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/drivers/virtio/virtio-serial.c#L397-L424) calls...

- [virtio_serial_dmarxfree](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/drivers/virtio/virtio-serial.c#L385-L397) (to free the DMA Receive Buffer) which calls...

- [uart_recvchars_dma](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/drivers/serial/serial_dma.c#L170-L292) (to receive the DMA data) which calls...

- [virtio_serial_dmareceive](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/drivers/virtio/virtio-serial.c#L357-L385) which calls...

- OpenAMP [virtqueue_add_buffer](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtqueue.c#L83C1-L138) (to release the VirtIO Queue Buffer) and...

  OpenAMP [virtqueue_kick](https://github.com/OpenAMP/open-amp/blob/main/lib/virtio/virtqueue.c#L321-L336) (to notify TinyEMU)

Finally the Waiting Thread reads the keypress...

```text
uart_read: ch=l
virtio_serial_dmarxfree: length=254
uart_read: buf[0]=l, recvd=1
readline_common: ch=0x6c
virtio_serial_dmarxfree: length=254
virtio_serial_dmarxfree: length=254
```

The Waiting Thread calls...

- [uart_read](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/drivers/serial/serial.c#L731-L1156) (to read the keypress) which calls...

- [virtio_serial_dmarxfree](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/drivers/virtio/virtio-serial.c#L385-L397) (to free the DMA Receive Buffer)

_What about NSH Shell?_

From the [Detailed Console Input Log](https://gist.github.com/lupyuen/1f0bbf1a749e58f1c467b50a031886fd)...

```text
nsh_main: 
nsh_consolemain: 
nsh_session: 
NuttShell (NSH) NuttX-12.3.0-RC1
nsh>
```

At NuttX Startup: [nsh_main](https://github.com/apache/nuttx-apps/blob/master/system/nsh/nsh_main.c#L40-L85) calls...

- [nsh_consolemain](https://github.com/apache/nuttx-apps/blob/master/nshlib/nsh_consolemain.c#L38-L82) which calls...

- [nsh_session](https://github.com/apache/nuttx-apps/blob/master/nshlib/nsh_session.c#L45-L249) (to run the NSH Session)

```text
nsh_session: Before readline_fd
readline_fd: 
readline_common: 
...
uart_read: buf[0]=l, recvd=1
readline_common: ch=0x6c
```

After NuttX Startup: [nsh_session](https://github.com/apache/nuttx-apps/blob/master/nshlib/nsh_session.c#L45-L249) calls...

- [readline_fd](https://github.com/apache/nuttx-apps/blob/master/system/readline/readline_fd.c#L173-L223) (to read one line) which calls...

- [readline_common](https://github.com/apache/nuttx-apps/blob/master/system/readline/readline_common.c#L444-L738) (to read one line) which calls...

- [uart_read](https://github.com/lupyuen2/wip-pinephone-nuttx/blob/tinyemu2/drivers/serial/serial.c#L731-L1156) (to read one keypress)

- Which is explained above

# Build TinyEMU for WebAssembly with Emscripten

Based on: https://github.com/lupyuen/TinyEMU/blob/master/.github/workflows/ci.yml

```bash
sudo apt install emscripten
make -f Makefile.js 
```

Fails to build...

```bash
emcc -O3 --memory-init-file 0 --closure 0 -s NO_EXIT_RUNTIME=1 -s NO_FILESYSTEM=1 -s "EXPORTED_FUNCTIONS=['_console_queue_char','_vm_start','_fs_import_file','_display_key_event','_display_mouse_event','_display_wheel_event','_net_write_packet','_net_set_carrier']" -s 'EXTRA_EXPORTED_RUNTIME_METHODS=["ccall", "cwrap"]' -s BINARYEN_TRAP_MODE=clamp --js-library js/lib.js -s WASM=0 -o js/riscvemu32.js jsemu.js.o softfp.js.o virtio.js.o fs.js.o fs_net.js.o fs_wget.js.o fs_utils.js.o simplefb.js.o pci.js.o json.js.o block_net.js.o iomem.js.o cutils.js.o aes.js.o sha256.js.o riscv_cpu32.js.o riscv_machine.js.o machine.js.o elf.js.o
emcc: error: Invalid command line option -s BINARYEN_TRAP_MODE=clamp: The wasm backend does not support a trap mode (it always clamps, in effect)
make: *** [Makefile.js:47: js/riscvemu32.js] Error 1
```

So we remove `-s BINARYEN_TRAP_MODE=clamp` from Makefile.js...

```bash
EMLDFLAGS=-O3 --memory-init-file 0 --closure 0 -s NO_EXIT_RUNTIME=1 -s NO_FILESYSTEM=1 -s "EXPORTED_FUNCTIONS=['_console_queue_char','_vm_start','_fs_import_file','_display_key_event','_display_mouse_event','_display_wheel_event','_net_write_packet','_net_set_carrier']" -s 'EXTRA_EXPORTED_RUNTIME_METHODS=["ccall", "cwrap"]' -s BINARYEN_TRAP_MODE=clamp --js-library js/lib.js
```

[(See the Modified File)](https://github.com/lupyuen/TinyEMU/commit/471f6e684054eec1dc2ed98207652c32b4e996e7#diff-3fc6364bd19a0e4ee8d1e0fe312541201418d80f9d1b08015db4d11e7dbde39e)

Now it builds OK...

```bash
/workspaces/bookworm/TinyEMU (master) $ ls -l js
total 1160
-rw-r--r-- 1 vscode vscode   8982 Jan 13 04:17 lib.js
-rw-r--r-- 1 vscode vscode 352884 Jan 13 04:18 riscvemu32.js
-rw-r--r-- 1 vscode vscode  45925 Jan 13 04:18 riscvemu32-wasm.js
-rwxr-xr-x 1 vscode vscode 147816 Jan 13 04:18 riscvemu32-wasm.wasm
-rw-r--r-- 1 vscode vscode 401186 Jan 13 04:18 riscvemu64.js
-rw-r--r-- 1 vscode vscode  45925 Jan 13 04:19 riscvemu64-wasm.js
-rwxr-xr-x 1 vscode vscode 164038 Jan 13 04:19 riscvemu64-wasm.wasm
```

[(See the Build Log)](https://github.com/lupyuen/TinyEMU/actions)
