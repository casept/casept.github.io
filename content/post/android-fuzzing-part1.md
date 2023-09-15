+++
title = "Fuzzing Android binaries under Qemu - part 1"
date = "2023-09-15T01:56:50+02:00"
categories = ["linux", "android", "fuzzing", "smartwatch"]
tags = ["linux", "android", "fuzzing", "smartwatch"]
description ="Fuzzing a weird Android binary extracted from a smartwatch recovery image"
author = "casept"
+++

# Introduction

For over a year now, I've been working on-and-mostly-off on researching Samsung smartwatches. I'll document my efforts here at a later date
(including a new tool for flashing Samsung devices that's hopefully less broken than Heimdall).

As part of that effort, I wanted to find out whether a binary Samsung ships in the recovery partition for flashing firmware can be exploited to achieve RCE.
This could be useful as a way to bypass bootloader locks (judging by pictures extracted from the bootloader binary, this only affects US Verizon models).
However, my proximate use case is extracting stock firmware of a friend's watch without the need for a root exploit.

In this post, I'll cover how to extract the binary from a recovery image found online for a related watch and how to get it to start under AFL++.
Building an actually useful fuzz harness for it was a much bigger undertaking, and will be documented in future installments.

# Extracting the binary

I obtained the firmware for a related watch from https://samfw.com. It came packed in a ZIP archive containing `.tar.md5` archives typically used in
Samsung's proprietary Odin flash tool. These are just regular tar archives with a proprietary trailer appended, and can be extracted with standard `tar`.

The recovery image is stored in the `AP` file, as `recovery.img.lz4`. This can be extracted using the `lz4` utility:

```shell
$ lz4 recovery.img.lz4
```

The unpacked `recovery.img` is an Android bootimage, which is basically a custom format for storing a kernel and ramdisk together.
Extracting this with `binwalk` created a directory structure that initially looked promising:

```shell
$ binwalk -e recovery.img
$ ls _recovery.img.extracted
11AB56F.xz    16A2290     185D800  cpio-root  dev     E86242.7z   root
145F510.cpio  16A2290.7z  console  DCBCA8     E86242  FDCED0.lzo
```

Unfortunately, the rootfs appears empty. It seems like we'll need more specific tooling:

```shell
$  tree _recovery.img.extracted
.
├── 11AB56F.xz
├── 145F510.cpio
├── 16A2290
├── 16A2290.7z
├── 185D800
├── console
├── cpio-root
│   ├── dev
│   └── root
├── DCBCA8
├── dev
├── E86242
├── E86242.7z
├── FDCED0.lzo
└── root
```

After some googling, I found [this tool](https://github.com/anestisb/android-unpackbootimg), which is a version of a tool used during Android compilation
that has been modified to let it run outside the Android build system.

This looks much more promising:

```shell
$ android-unpackbootimg/unpackbootimg -i recovery.img -o recovery
BOARD_KERNEL_CMDLINE
BOARD_KERNEL_BASE 10000000
BOARD_NAME SRPUD05C001
BOARD_PAGE_SIZE 2048
BOARD_HASH_TYPE sha1
BOARD_KERNEL_OFFSET 00008000
BOARD_RAMDISK_OFFSET 00000000
BOARD_SECOND_OFFSET f0000000
BOARD_TAGS_OFFSET 00000000
BOARD_OS_VERSION 11.0.0
BOARD_OS_PATCH_LEVEL 2023-01
BOARD_DT_SIZE 2
$ ls recovery/
recovery.img-base     recovery.img-dtb        recovery.img-oslevel    recovery.img-ramdisk.gz  recovery.img-tagsoff
recovery.img-board    recovery.img-hash       recovery.img-osversion  recovery.img-ramdiskoff  recovery.img-zImage
recovery.img-cmdline  recovery.img-kerneloff  recovery.img-pagesize   recovery.img-secondoff
```

`recovery.img-ramdisk.gz` can be easily extracted, revealing first a CPIO archive and then the contents we're after:

```shell
$ unar recovery.img-ramdisk.gz
recovery.img-ramdisk.gz: Gzip
  recovery.img-ramdisk... OK.
Successfully extracted to "./recovery.img-ramdisk".
$ file recovery.img-ramdisk
recovery.img-ramdisk: ASCII cpio archive (SVR4 with no CRC)
$ unar recovery.img-ramdisk
unar recovery.img-ramdisk
recovery.img-ramdisk: Cpio
  acct/  (dir)... OK.
  apex/  (dir)... OK.
  audit_filter_table  (38553 B)... OK.
  bin  (link)... OK.
  bugreports  (link)... OK.
  cache/  (dir)... OK.
  config/  (dir)... OK.
  ...
  <snip>
  ...
  system/bin/which  (link)... OK.
  system/bin/whoami  (link)... OK.
  system/bin/wirelessd  (89536 B)... OK.
  system/bin/wl  (2128392 B)... OK.
  system/bin/xargs  (link)... OK.
  ...
  <snip>
  ...
  Successfully extracted to "recovery".
```

After this, I cleaned up the Matroska-like directory structure and proceeded with fuzzer setup.

# Building the fuzzer

 I decided to use [AFL++](https://github.com/AFLplusplus/AFLplusplus) as my fuzzer, as it seemed reasonably documented and supported fuzzing
binaries for other architectures relatively easily via Qemu. To build it:

```shell
$ apt build-dep -y qemu afl++
$ apt install -y python3-dev python3-pip cmake
$ git clone https://github.com/AFLplusplus/AFLplusplus
$ cd AFLplusplus
$ NO_NYX=1 NO_UNICORN_ARM64=1 CPU_TARGET=arm STATIC=1 make binary-only
```

On Debian, you'll get errors when building Unicorn mode, as it tries to install Python modules from
pip with the system Python, which is prohibited on new Debian versions. You can safely ignore this, as that mode is not needed.

You'll also want to build some helper libraries specifically for your target, as these are going to be `AFL_PRELOAD`-ed into it.
Install [the NDK](https://developer.android.com/ndk/downloads/) and put it on your `PATH`, then run:

```shell
$ cd utils/libdislocator/
$ make clean
# Replace as appropriate for your arch and Android API level
$ CC=armv7a-linux-androideabi30-clang make
```

# Fuzzing

## Take 1

At this point, we have the tools needed to make a first fuzzing attempt.
Because AFL++ is configured mainly by env vars and there are a lot we need to set, we'll create a script to run it called `fuzz.sh`:

```shell
#!/usr/bin/env bash
set -eo pipefail

# Definitions to make our life easier below
export PATH="$(pwd)/AFLplusplus:$PATH" # So AFL++ tools are available
export TARGET_ROOT="$(pwd)/recovery-rootfs"
export TARGET="$TARGET_ROOT/system/bin/wirelessd"
export TARGET_LIB_ROOT="$TARGET_ROOT/system/lib" # Where Qemu should look for target libs
export QEMU_LD_PREFIX="$TARGET_LIB_ROOT/"
# Custom allocator to hopefully find smaller memory misuses as well.
# Add any other custom libs you want to LD_PRELOAD into the target here (absolute path, space-separated).
export AFL_PRELOAD="$TARGET_LIB_ROOT/libdislocator.so"

# Actual fuzzer invocation, see AFL++ docs for what the flags mean. Everything after "--" is passed to the target.
afl-fuzz -Q -a "binary" -i "fuzzcases" -o "out" -- "$TARGET" -i 127.0.0.1 -p 14344 &
```

Finally, we need to create a directory called `fuzzcases` and fill it with some valid program inputs.
Then, we can try fuzzing:

```shell
$ ./fuzz.sh
[+] Enabled environment variable AFL_PRELOAD with value /root/wirelessdl-exploit/recovery-rootfs/system/lib/libdislocator.so
afl-fuzz++4.09a based on afl by Michal Zalewski and a large online community
[+] AFL++ is maintained by Marc "van Hauser" Heuse, Dominik Maier, Andrea Fioraldi and Heiko "hexcoder" Eißfeldt
[+] AFL++ is open source, get it at https://github.com/AFLplusplus/AFLplusplus
[+] NOTE: AFL++ >= v3 has changed defaults and behaviours - see README.md
[+] No -M/-S set, autoconfiguring for "-S default"
[*] Getting to work...
[+] Using exponential power schedule (FAST)
[+] Enabled testcache with 50 MB
[+] Generating fuzz data with a length of min=1 max=1048576
[*] Checking core_pattern...
[!] WARNING: Could not check CPU scaling governor
[+] You have 3 CPU cores and 3 runnable tasks (utilization: 100%).
[*] Setting up output directories...
[*] Checking CPU core loadout...
[+] Found a free CPU core, try binding to #0.
[*] Scanning 'fuzzcases'...
[+] Loaded a total of 5 seeds.
[*] Creating hard links for all input files...
[*] Validating target binary...
[*] No auto-generated dictionary tokens to reuse.
[*] Attempting dry run with 'id:000000,time:0,execs:0,orig:begin-end-session'...
[*] Spinning up the fork server...

[-] Hmm, looks like the target binary terminated before we could complete a
handshake with the injected code. You can try the following:

    - The target binary crashes because necessary runtime conditions it needs
      are not met. Try to:
      1. Run again with AFL_DEBUG=1 set and check the output of the target
         binary for clues.
      2. Run again with AFL_DEBUG=1 and 'ulimit -c unlimited' and analyze the
         generated core dump.

    - Possibly the target requires a huge coverage map and has CTORS.
      Retry with setting AFL_MAP_SIZE=10000000.

Otherwise there is a horrible bug in the fuzzer.
Poke the Awesome Fuzzing Discord for troubleshooting tips.

[-] PROGRAM ABORT : Fork server handshake failed
         Location : afl_fsrv_start(), src/afl-forkserver.c:1423
```

That doesn't look good!

## Take 2

Let's add `export AFL_DEBUG=1` to our script so we can see program output and deduce what's going on:

```shell
$ ./fuzz.sh
<snip>
AFL forkserver entrypoint: 0x40006560
afl-qemu-trace: Could not open '/system/bin/linker': No such file or directory

[-] Hmm, looks like the target binary terminated before we could complete a
handshake with the injected code. You can try the following:

    - The target binary crashes because necessary runtime conditions it needs
      are not met. Try to:
      1. Run again with AFL_DEBUG=1 set and check the output of the target
         binary for clues.
      2. Run again with AFL_DEBUG=1 and 'ulimit -c unlimited' and analyze the
         generated core dump.

    - Possibly the target requires a huge coverage map and has CTORS.
      Retry with setting AFL_MAP_SIZE=10000000.

Otherwise there is a horrible bug in the fuzzer.
Poke the Awesome Fuzzing Discord for troubleshooting tips.

[-] PROGRAM ABORT : Fork server handshake failed
         Location : afl_fsrv_start(), src/afl-forkserver.c:1423
```

Aha, so the binary can't be linked because Android's dynamic linker is not at the expected location.
This is because AFL does't chroot into the recovery rootfs, so the linker is actually located under `$(pwd)/recovery-rootfs/system/bin/linker`.
Let's tell the binary that using [patchelf](https://github.com/NixOS/patchelf):

```shell
$ apt install -y patchelf
<snip>
$ patchelf --set-interpreter "$TARGET_ROOT/system/bin/linker" "$TARGET"
```

I recommend adding this into your `fuzz.sh`, as every time you replace the target binary (e.g. because you patched and exported it from an external tool)
you'll have to remember to re-run this.

Let's try fuzzing again:

```shell
$ ./fuzz.sh
<snip>
CANNOT LINK EXECUTABLE "/root/wirelessdl-exploit/recovery-rootfs/system/bin/wirelessd": library "libbacktrace.so" not found: needed by main executable
linker: CANNOT LINK EXECUTABLE "/root/wirelessdl-exploit/recovery-rootfs/system/bin/wirelessd": library "libbacktrace.so" not found: needed by main executable


[-] Hmm, looks like the target binary terminated before we could complete a
handshake with the injected code. You can try the following:

    - The target binary crashes because necessary runtime conditions it needs
      are not met. Try to:
      1. Run again with AFL_DEBUG=1 set and check the output of the target
         binary for clues.
      2. Run again with AFL_DEBUG=1 and 'ulimit -c unlimited' and analyze the
         generated core dump.

    - Possibly the target requires a huge coverage map and has CTORS.
      Retry with setting AFL_MAP_SIZE=10000000.

Otherwise there is a horrible bug in the fuzzer.
Poke the Awesome Fuzzing Discord for troubleshooting tips.

[-] PROGRAM ABORT : Fork server handshake failed
         Location : afl_fsrv_start(), src/afl-forkserver.c:1423
```

Hmm, something is clearly still broken.

# Take 3

This is the first of many points where I was stumped. Why are libraries not being found? We set `QEMU_LD_PREFIX`, after all!

After asking in the fuzzing Discord, I learned that `CANNOT LINK EXECUTABLE` is not a message from Qemu at all, but from Android's `linker`!
The easiest way to rectify this is to set the proper library search path in `fuzz.sh`, as `/system/lib` doesn't exist on our host:

```shell
# For the Android linker, so it doesn't expect /system/lib to exist on the host
export AFL_TARGET_ENV="LD_LIBRARY_PATH=$TARGET_LIB_ROOT"
```

And let's test:

```shell
$ ./fuzz.sh
<snip>
[*] Spinning up the fork server...
AFL forkserver entrypoint: 0x40006560
AFL forkserver entrypoint: 0x40006560
Limited by the size of pthread_mutex_t, 32 bit bionic libc only accepts pid <= 65535, but current pid is 301599
libc: Limited by the size of pthread_mutex_t, 32 bit bionic libc only accepts pid <= 65535, but current pid is 301599
libc: Fatal signal 6 (SIGABRT), code -1 (SI_QUEUE) in tid 301599 (afl-qemu-trace), pid 301599 (afl-qemu-trace)
libc: failed to spawn debuggerd dispatch thread: Invalid argument

[-] Hmm, looks like the target binary terminated before we could complete a
handshake with the injected code. You can try the following:

    - The target binary crashes because necessary runtime conditions it needs
      are not met. Try to:
      1. Run again with AFL_DEBUG=1 set and check the output of the target
         binary for clues.
      2. Run again with AFL_DEBUG=1 and 'ulimit -c unlimited' and analyze the
         generated core dump.

    - Possibly the target requires a huge coverage map and has CTORS.
      Retry with setting AFL_MAP_SIZE=10000000.

Otherwise there is a horrible bug in the fuzzer.
Poke the Awesome Fuzzing Discord for troubleshooting tips.

[-] PROGRAM ABORT : Fork server handshake failed
         Location : afl_fsrv_start(), src/afl-forkserver.c:1422
```

Ah, a bionic libc issue!

# Take 4

Guess we need to limit the maximum PID somehow. After a bit of browsing stackoverflow, I found the correct incantation and added it to `fuzz.sh`:

```shell
# Bionic doesn't like high PIDs on a 64-bit host
sudo sh -c "echo 65535 > /proc/sys/kernel/pid_max"
```

Let's test again:

```shell
$ ./fuzz.sh
<snip>
2023-09-15 20:39:03 [I]  wd_handler()
2023-09-15 20:39:03 [I]  *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
2023-09-15 20:39:03 [I]  ABI: 'arm'
2023-09-15 20:39:03 [I]  pid: 61410, tid: 4, name: afl-qemu-trace
2023-09-15 20:39:03 [I]  signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 3fffee9000000000
2023-09-15 20:39:03 [I]       r0 0x00000004  r1 0x00000000  r2 0x40127bc8  r3 0x00000000
2023-09-15 20:39:03 [I]       r4 0xffffffff  r5 0x00000000  r6 0x00000005  r7 0x3ffff204
2023-09-15 20:39:03 [I]       r8 0x00000000  r9 0x3f2fc2b4  sl 0x400165dc  fp 0x00000002
2023-09-15 20:39:03 [I]       ip 0x3f2f8574  sp 0x3ffff188  lr 0x40008457  pc 0x4000f6f0  cpsr 0x800f0030
2023-09-15 20:39:03 [I]
backtrace:
<this goes on in an infinite loop>
```

`wd_handler()` is a function I saw when reversing the target in Ghidra earlier, so that means the output must come from it and we're running it!

The bad news is that:

1) the target is crashing really early in initialization, so it'll probably need to be patched extensively to run at all.
2) the target is trying to handle segfaults itself, which hides them from qemu and AFL++. We'll need to work around that.

However, if yor target is more amenable you might be done at this point! As for me, my target required quite some more convincing to run.
More on that in part 2.
