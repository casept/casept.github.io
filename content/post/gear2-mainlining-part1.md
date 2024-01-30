+++
title = "Getting mainline running on the Samsung Gear 2 - part 1"
date = "2024-01-15T01:56:50+02:00"
categories = ["linux", "tizen", "smartwatch"]
tags = ["linux", "tizen", "smartwatch"]
description ="Booting mainline Linux on the Samsung Gear 2"
author = "casept"
+++

## Introduction

After putting my work on the Galaxy Gear S2 on hold due to how tricky it is to work on without a reliable USB connection, I've decided to hack on easier targets first.

The main results of this work are a patched kernel tree hosted [here](https://github.com/casept/linux-samsung-smartwatch/tree/rinato) and a much-expanded 
postmarketOS wiki article [here](https://wiki.postmarketos.org/wiki/Samsung_Gear_2_(samsung-rinato)).

An [AsteroidOS port](https://github.com/casept/meta-smartwatch) is also work-in-progress, but blocked
by having to split all of the libhybris-related changes out, as this is the first mainline watch supported.
Getting hardware accelerated rendering working seems like an especially large challenge, as the launcher / compositor codebase is quite tightly coupled
to Android's HWC compositing framework.

## Choosing a target

Unlike most newer smartwatches, the Gear 2 actually has exposed USB pins, making it much easier to upload kernels and debug without risking a brick.
Even better, unlike some other watches like e.g. the Gear Fit 2, this one has five pins exposed rather than only four, which means that the USB ID
pin is usable for [serial debugging](https://wiki.postmarketos.org/wiki/Serial_debugging:Cable_schematics#microUSB.2FCarkit_debug_cable).

The watches are no longer supported on recent Android versions and therefore very cheap (~25 EUR on Kleinanzeigen),
meaning I could easily buy several.
They also sold better than the original Samsung Gear, meaning that it's actually possible to find them.

More importantly, the SoC (Exynos 3250) seemed well-supported in mainline and Samsung even contributed an (as we'll soon see, both incomplete and somewhat incorrect)
[device tree](https://github.com/torvalds/linux/blob/master/arch/arm/boot/dts/samsung/exynos3250-rinato.dts).

## Initial recon

Whenever attempting mainlining the vendor source tree is a must. Samsung really dropped the ball here, as their download server is both super slow
and drops archives for older devices quite quickly. I had to utilize a form on the site to manually request source code and wait until it
was reinstated, then wait several hours until the slow download finished.
To save you the trouble, I've mirrored the source code [on Github](https://github.com/casept/samsung-rinato-sources).

From taking apart freely-available stock ROMs for the Watch, I learned that the boot image format is not an Android bootimg (as used for e.g. WearOS watches),
but a plain kernel `zImage`. By using Heimdall's `--print-pit` command I determined that the partition holding the kernel image is `BOOT`. As there's no device
tree partition, it's probably not passed by the bootloader and the device tree should simply be appended to the kernel image.

I also created a rooted variant of the stock Tizen, so I could poke at debug functionality and reverse engineer userspace applications if needed.

## Does it boot?

After attempting the initial kernel boot I got no output at all, just the bootloader continuing to display the logo. At this point, I didn't even know whether the kernel ran.
I suspected an issue like an incorrect kernel base address, but after wasting too much time digging I figured out that the kernel is position-independent.

The first attempt to actually debug this was via UART, as I knew from Samsung phones and my experiments with the S2 that the bootloader prints logs via Carkit serial.
Unfortunately, the output was totally silent. By measuring out all the charging cradle's pins with a multimeter, I came to the conclusion that the USB ID pin
was not connected to the microUSB port for some reason. I later resolved this by modding the cradle, but that required tools I didn't have handy at the time.

Next, I remembered that the stock kernel's cmdline documented on the pmOS wiki has addresses for what looks like debug functionality in it, including for a framebuffer.
Writing to the area from rooted Tizen resulted in flipped pixels right after reboot, but defining a `simplefb` in the device tree didn't result in any output.
I didn't know what caused this and gave up on this approach.

However, investigating the other addresses proved fruitful, as they turned out to contain bootloader log buffers which confirmed that mainline was indeed
being loaded! So the kernel was crashing at some later stage. I confirmed this theory by adding `panic.reboot=5` to the kernel cmdline and observing
that the watch did indeed reboot.

## Debugging the crash

As I didn't know whether any peripherals work at all, I needed to find a way to debug the kernel via RAM only.
The traditional way to do this is via the kernel's `pstore` mechanism, but that'd require a sufficiently new, working kernel. Linux 3.4 the watch shipped
with is simply too old, and backporting pstore seemed like a pain.

Fortunately, Samsung has a solution for this - upload mode. This special bootloader debug features enables dumping RAM regions,
and there's [even a tool for that](https://github.com/bkerler/sboot_dump). After enabling it via the magic dialer combination `*#9900#`, it can be entered
via the bootloader menu.

Using it to dump the `pstore` region results in somewhat garbled, but readable kernel logs!

With the downstream kernel, upload mode also has some additional features - it can display a cause for the crash and the kernel panic string.
I've implemented a simplistic driver for this in mainline. Initially, only enough to get into upload mode after a panic and show some random memory contents:

![watch displaying random gibberish in upload mode](/images/upload.jpeg)

And then, with the ability to print kernel panic messages:

![watch displaying kernel panic](/images/upload-panic.jpeg)

{{< details "Click for kernel logs" >}}
~~~
[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 6.7.0-rc8-next-20240104-ga8f3062caf0c-dirty (casept@l13) (arm-none-eabi-gcc (Arm GNU Toolchain 13.2.rel1 (Build arm-13.7)) 13.2.1 20231009, GNU ld (Arm GNU Toolchain 13.2.rel1 (Build arm-13.7)) 2.41.0.20231009) #59 SMP PREEMPT Fri Jan  5 15:43:00 CET 2024
[    0.000000] CPU: ARMv7 Processor [410fc073] revision 3 (ARMv7), cr=10c5387d
[    0.000000] CPU: div instructions available: patching division code
[    0.000000] CPU: PIPT / VIPT nonaliasing data cache, VIPT aliasing instruction cache
[    0.000000] OF: fdt: Machine model: Samsung Rinato board
[    0.000000] Memory policy: Data cache writealloc
[    0.000000] OF: reserved mem: 0x51000000..0x510fffff (1024 KiB) map non-reusable ramoops@51000000
[    0.000000] cma: Reserved 96 MiB at 0x59c00000 on node -1
[    0.000000] Samsung CPU ID: 0xe3472000
[    0.000000] Zone ranges:
[    0.000000]   Normal   [mem 0x0000000040004000-0x000000005fefffff]
[    0.000000]   HighMem  empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000040004000-0x00000000401fffff]
[    0.000000]   node   0: [mem 0x0000000040200000-0x000000005fefffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000040004000-0x000000005fefffff]
[    0.000000] On node 0, zone Normal: 4 pages in unavailable ranges
[    0.000000] Running under secure firmware.
[    0.000000] percpu: Embedded 17 pages/cpu s39732 r8192 d21708 u69632
[    0.000000] Kernel command line: console=ttyGS0,115200 console=ttySAC1,115200 fbcon=font:VGA8x8 printk.always_kmsg_dump=1 crash_kexec_post_notifiers panic=5 nosgx
[    0.000000] Unknown kernel command line parameters "nosgx", will be passed to user space.
[    0.000000] Dentry cache hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.000000] Inode-cache hash table entries: 32768 (order: 5, 131072 bytes, linear)
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 129790
[    0.000000] mem auto-init: stack:all(zero), heap alloc:off, heap free:off
[    0.000000] Memory: 394448K/523248K available (10240K kernel code, 1401K rwdata, 3604K rodata, 1024K init, 6604K bss, 30496K reserved, 98304K cma-reserved, 0K highmem)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
[    0.000000] trace event string verifier disabled
[    0.000000] Running RCU self tests
[    0.000000] Running RCU synchronous self tests
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu: 	RCU event tracing is enabled.
[    0.000000] rcu: 	RCU lockdep checking is enabled.
[    0.000000] 	Trampoline variant of Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 10 jiffies.
[    0.000000] Running RCU synchronous self tests
[    0.000000] NR_IRQS: 16, nr_irqs: 16, preallocated irqs: 16
[    0.000000] rcu: srcu_init: Setting srcu_struct sizes based on contention.
[    0.000000] Switching to timer-based delay loop, resolution 41ns
[    0.000000] clocksource: mct-frc: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 79635851949 ns
[    0.000003] sched_clock: 32 bits at 24MHz, resolution 41ns, wraps every 89478484971ns
[    0.001421] Console: colour dummy device 80x30
[    0.001552] Lock dependency validator: Copyright (c) 2006 Red Hat, Inc., Ingo Molnar
[    0.001569] ... MAX_LOCKDEP_SUBCLASSES:  8
[    0.001585] ... MAX_LOCK_DEPTH:          48
[    0.001600] ... MAX_LOCKDEP_KEYS:        8192
[    0.001615] ... CLASSHASH_SIZE:          4096
[    0.001630] ... MAX_LOCKDEP_ENTRIES:     32768
[    0.001645] ... MAX_LOCKDEP_CHAINS:      65536
[    0.001659] ... CHAINHASH_SIZE:          32768
[    0.001675]  memory used by lock dependency info: 4125 kB
[    0.001690]  memory used for stack traces: 2112 kB
[    0.001705]  per task-struct memory footprint: 1536 bytes
[    0.001817] Calibrating delay loop (skipped), value calculated using timer frequency.. 48.00 BogoMIPS (lpj=240000)
[    0.001859] CPU: Testing write buffer coherency: ok
[    0.002084] pid_max: default: 32768 minimum: 301
[    0.002915] Mount-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.002953] Mountpoint-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.007787] Running RCU synchronous self tests
[    0.007826] Running RCU synchronous self tests
[    0.009336] CPU0: update cpu_capacity 1024
[    0.009363] CPU0: thread -1, cpu 0, socket 0, mpidr 80000000
[    0.020201] RCU Tasks: Setting shift to 1 and lim to 1 rcu_task_cb_adjust=1.
[    0.020796] Running RCU Tasks wait API self tests
[    0.130457] Setting up static identity map for 0x40100000 - 0x40100060
[    0.131330] rcu: Hierarchical SRCU implementation.
[    0.131348] rcu: 	Max phase no-delay instances is 1000.
[    0.135919] smp: Bringing up secondary CPUs ...
[    0.141181] CPU1: update cpu_capacity 1024
[    0.141202] CPU1: thread -1, cpu 1, socket 0, mpidr 80000001
[    0.142765] smp: Brought up 1 node, 2 CPUs
[    0.142792] SMP: Total of 2 processors activated (96.00 BogoMIPS).
[    0.142814] CPU: All CPU(s) started in SVC mode.
[    0.146651] devtmpfs: initialized
[    0.249316] VFP support v0.3: implementor 41 architecture 2 part 30 variant 7 rev 3
[    0.251391] Running RCU synchronous self tests
[    0.251516] Running RCU synchronous self tests
[    0.252082] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
[    0.252165] futex hash table entries: 512 (order: 3, 32768 bytes, linear)
[    0.256910] pinctrl core: initialized pinctrl subsystem
[    0.264594] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.270834] DMA: preallocated 256 KiB pool for atomic coherent allocations
[    0.280921] thermal_sys: Registered thermal governor 'step_wise'
[    0.281401] cpuidle: using governor menu
[    0.282303] hw-breakpoint: Failed to enable monitor mode on CPU 0.
[    0.287679] printk: legacy console [ramoops-1] enabled
[    0.292108] pstore: Registered ramoops as persistent store backend
[    0.292270] ramoops: using 0x100000@0x51000000, ecc: 0
[    0.351557] Callback from call_rcu_tasks() invoked.
[    0.352327] platform 11000000.pinctrl: Fixed dependency cycle(s) with /soc/pinctrl@11000000/sleep-state
[    0.352567] platform 11000000.pinctrl: Fixed dependency cycle(s) with /soc/pinctrl@11000000/initial-state
[    0.405245] platform 11400000.pinctrl: Fixed dependency cycle(s) with /soc/pinctrl@11400000/sleep-state
[    0.405380] platform 11400000.pinctrl: Fixed dependency cycle(s) with /soc/pinctrl@11400000/initial-state
[    0.488727] iommu: Default domain type: Translated
[    0.488812] iommu: DMA domain TLB invalidation policy: strict mode
[    0.503393] SCSI subsystem initialized
[    0.508360] i2c-gpio i2c-gpio-0: using lines 659 (SDA) and 660 (SCL)
[    0.527093] nfc: nfc_init: NFC Core ver 0.1
[    0.527507] NET: Registered PF_NFC protocol family
[    0.529837] clocksource: Switched to clocksource mct-frc
[    0.595168] NET: Registered PF_INET protocol family
[    0.596195] IP idents hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.600766] tcp_listen_portaddr_hash hash table entries: 256 (order: 1, 10240 bytes, linear)
[    0.600886] Table-perturb hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.600956] TCP established hash table entries: 4096 (order: 2, 16384 bytes, linear)
[    0.601228] TCP bind hash table entries: 4096 (order: 6, 327680 bytes, linear)
[    0.602722] TCP: Hash tables configured (established 4096 bind 4096)
[    0.603239] UDP hash table entries: 256 (order: 2, 24576 bytes, linear)
[    0.603434] UDP-Lite hash table entries: 256 (order: 2, 24576 bytes, linear)
[    0.604087] NET: Registered PF_UNIX/PF_LOCAL protocol family
[    0.607043] RPC: Registered named UNIX socket transport module.
[    0.607195] RPC: Registered udp transport module.
[    0.607240] RPC: Registered tcp transport module.
[    0.607282] RPC: Registered tcp-with-tls transport module.
[    0.607324] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.610217] armv7-pmu pmu: hw perfevents: no interrupt-affinity property, guessing.
[    0.611875] hw perfevents: enabled with armv7_cortex_a7 PMU driver, 5 counters available
[    0.620957] Initialise system trusted keyrings
[    0.622138] workingset: timestamp_bits=30 max_order=17 bucket_order=0
[    0.624459] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    0.626067] NFS: Registering the id_resolver key type
[    0.626300] Key type id_resolver registered
[    0.626439] Key type id_legacy registered
[    0.626583] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    0.626864] nfs4flexfilelayout_init: NFSv4 Flexfile Layout Driver Registering...
[    0.627093] romfs: ROMFS MTD (C) 2007 Red Hat, Inc.
[    0.628320] Key type asymmetric registered
[    0.628465] Asymmetric key parser 'x509' registered
[    0.628848] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 247)
[    0.628980] io scheduler mq-deadline registered
[    0.629031] io scheduler kyber registered
[    0.629128] io scheduler bfq registered
[    0.630416] samsung-pinctrl 11000000.pinctrl: Failed to create device link (0x180) with soc
[    0.665476] dma-pl330 12680000.dma-controller: Loaded driver for PL330 DMAC-141330
[    0.665546] dma-pl330 12680000.dma-controller: 	DBUFF-32x4bytes Num_Chans-8 Num_Peri-32 Num_Events-32
[    0.686031] dma-pl330 12690000.dma-controller: Loaded driver for PL330 DMAC-141330
[    0.686092] dma-pl330 12690000.dma-controller: 	DBUFF-32x4bytes Num_Chans-8 Num_Peri-32 Num_Events-32
[    0.688148] exynos-chipid 10000000.chipid: Exynos: CPU[EXYNOS3250] PRO_ID[0xe3472000] REV[0x0] Detected
[    1.053498] Serial: 8250/16550 driver, 4 ports, IRQ sharing disabled
[    1.074991] 13800000.serial: ttySAC0 at MMIO 0x13800000 (irq = 56, base_baud = 0) is a S3C6400/10
[    1.078760] serial serial0: tty port ttySAC0 registered
[    1.084241] 13810000.serial: ttySAC1 at MMIO 0x13810000 (irq = 57, base_baud = 0) is a S3C6400/10
[    1.085162] printk: legacy console [ttySAC1] enabled
[    1.981273] exynos4-fb 11c00000.fimd: Adding to iommu group 0
[    1.984545] OF: graph: no port node found in /soc/fimd@11c00000
[    2.062482] brd: module loaded
[    2.106425] loop: module loaded
[    2.106590] sboot_upload: Init
[    2.107086] sboot_upload: Init OK
[    2.114382] max14577 7-0025: Device type: 2 (ID: 0xe, vendor: 0x5)
[    2.129926] max14577-regulator max77836-regulator: DMA mask not set
[    2.130812] max77836-battery: Failed to locate of_node [id: -1]
[    2.144781] MOT_2.7V: Bringing 3300000uV into 2700000-2700000uV
[    2.184428] i2c_dev: i2c /dev entries driver
[    2.191162] max14577-charger max77836-charger: DMA mask not set
[    2.243977] device-mapper: ioctl: 4.48.0-ioctl (2023-03-01) initialised: dm-devel@redhat.com
[    2.253283] sdhci: Secure Digital Host Controller Interface driver
[    2.253874] sdhci: Copyright(c) Pierre Ossman
[    2.258806] Synopsys Designware Multimedia Card Interface Driver
[    2.267983] dwmmc_exynos 12520000.mmc: IDMAC supports 32-bit address mode.
[    2.273671] dwmmc_exynos 12520000.mmc: Using internal DMA controller.
[    2.277501] dwmmc_exynos 12520000.mmc: Version ID is 260a
[    2.283757] dwmmc_exynos 12520000.mmc: DW MMC controller at irq 60,32 bit host data width,128 deep fifo
[    2.294119] dwmmc_exynos 12520000.mmc: allocated mmc-pwrseq
[    2.297812] mmc_host mmc1: card is non-removable.
[    2.316937] mmc_host mmc1: Bus speed (slot 0) = 50000000Hz (slot req 400000Hz, actual 396825HZ div = 63)
[    2.350327] exynos-ppmu 106a0000.ppmu: cannot get PPMU clock
[    2.351483] exynos-ppmu: new PPMU device registered 106a0000.ppmu (ppmu-event3-dmc0)
[    2.358956] exynos-ppmu 106b0000.ppmu: cannot get PPMU clock
[    2.364471] exynos-ppmu: new PPMU device registered 106b0000.ppmu (ppmu-event3-dmc1)
[    2.372982] exynos-ppmu: new PPMU device registered 112a0000.ppmu (ppmu-event3-rightbus)
[    2.381162] exynos-ppmu: new PPMU device registered 116a0000.ppmu (ppmu-event3-leftbus)
[    2.388555] max14577-muic max77836-muic: DMA mask not set
[    2.466147] mmc_host mmc1: Bus speed (slot 0) = 50000000Hz (slot req 50000000Hz, actual 50000000HZ div = 0)
[    2.475627] max14577-muic max77836-muic: device ID : 0x75
[    2.478138] mmc1: new high speed SDIO card at address 0001
[    2.483267] NET: Registered PF_INET6 protocol family
[    2.491118] Segment Routing with IPv6
[    2.491407] In-situ OAM (IOAM) with IPv6
[    2.493950] sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
[    2.504169] NET: Registered PF_PACKET protocol family
[    2.504758] NET: Registered PF_KEY protocol family
[    2.510214] Key type dns_resolver registered
[    2.514679] Registering SWP/SWPB emulation handler
[    2.583910] Loading compiled-in X.509 certificates
[    3.025524] input: gpio-keys as /devices/platform/gpio-keys/input/input0
[    3.034515] clk: Disabling unused clocks
[    3.133889] /dev/root: Can't open blockdev
[    3.134340] VFS: Cannot open root device "" or unknown-block(0,0): error -6
[    3.139467] Please append a correct "root=" boot option; here are the available partitions:
[    3.148111] 0100           65536 ram0
[    3.148143]  (driver?)
[    3.153893] 0101           65536 ram1
[    3.153919]  (driver?)
[    3.160051] 0102           65536 ram2
[    3.160077]  (driver?)
[    3.166041] 0103           65536 ram3
[    3.166065]  (driver?)
[    3.172118] 0104           65536 ram4
[    3.172143]  (driver?)
[    3.178193] 0105           65536 ram5
[    3.178217]  (driver?)
[    3.184270] 0106           65536 ram6
[    3.184296]  (driver?)
[    3.190427] 0107           65536 ram7
[    3.190453]  (driver?)
[    3.196421] 0108           65536 ram8
[    3.196445]  (driver?)
[    3.202498] 0109           65536 ram9
[    3.202523]  (driver?)
[    3.208573] 010a           65536 ram10
[    3.208598]  (driver?)
[    3.214737] 010b           65536 ram11
[    3.214762]  (driver?)
[    3.220956] 010c           65536 ram12
[    3.220981]  (driver?)
[    3.227061] 010d           65536 ram13
[    3.227085]  (driver?)
[    3.233226] 010e           65536 ram14
[    3.233251]  (driver?)
[    3.239387] 010f           65536 ram15
[    3.239411]  (driver?)
[    3.245572] List of all bdev filesystems:
[    3.249527]  ext3
[    3.249546]  ext4
[    3.251518]  ext2
[    3.253346]  cramfs
[    3.255256]  squashfs
[    3.257339]  vfat
[    3.259596]  msdos
[    3.261563]  romfs
[    3.263502]
[    3.266976] Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
[    3.275222] CPU: 0 PID: 1 Comm: swapper/0 Not tainted 6.7.0-rc8-next-20240104-ga8f3062caf0c-dirty #59
[    3.284418] Hardware name: Samsung Exynos (Flattened Device Tree)
[    3.290500]  unwind_backtrace from show_stack+0x10/0x14
[    3.295703]  show_stack from dump_stack_lvl+0x58/0x70
[    3.300737]  dump_stack_lvl from panic+0x12c/0x374
[    3.305513]  panic from mount_root_generic+0x1e8/0x298
[    3.310633]  mount_root_generic from prepare_namespace+0x1e4/0x240
[    3.316795]  prepare_namespace from kernel_init+0x18/0x12c
[    3.322263]  kernel_init from ret_from_fork+0x14/0x28
[    3.327297] Exception stack(0xe0045fb0 to 0xe0045ff8)
[    3.332337] 5fa0:                                     00000000 00000000 00000000 00000000
[    3.340497] 5fc0: 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
[    3.348655] 5fe0: 00000000 00000000 00000000 00000000 00000013 00000000
[    3.355268] CPU 1 will stop doing anything useful since another CPU has crashed
[    3.356272] sboot_upload: Handling kernel panic, will enter upload mode after reboot
[    3.370369] sboot_upload: Panic string: VFS: Unable to mount root fs on unknown-block(0,0)
[    3.381590] Rebooting in 5 seconds..
~~~
{{< /details >}}

From reading the logs, we can see that the eMMC is not detected. I'll leave fixing that for part 2. 
