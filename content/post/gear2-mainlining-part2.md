+++
title = "Getting mainline running on the Samsung Gear 2 - part 2 (eMMC)"
date = "2024-04-19T01:56:50+02:00"
categories = ["linux", "tizen", "smartwatch"]
tags = ["linux", "tizen", "smartwatch"]
description ="Getting eMMC working on the Samsung Gear 2"
author = "casept"
+++

## Introduction

In part 1, I got the kernel booting and providing debug output,
to the point where I could figure out that the kernel was failing to find the rootfs.

The kernel logs suspiciously did not show proper enumeration of the eMMC flash, so that's
what we need to fix.


## Initial hypothesis

We know that the eMMC controller is at least somewhat working, as the Wi-Fi/bluetooth chip
is connected to it via SDIO and the kernel logs show that it's being enumerated as `mmc1`.
This means that something is wrong with the way the flash chip itself is set up.

Unfortunately, the rest of the kernel output is not (directly) helpul. I investigated some
of the warnings (like the `Failed to create device link with soc` one),
but all of them seemed unrelated.

## Building a debug userspace

I figured it'd be helpful if I could poke at the kernel interactively, investigate the contents
of `/dev` and `/sys` etc. To do that, at least a basic userspace with a shell and some tools is needed -
but where can we put it if no storage devices are recognized?

The simplest option is to make the bootloader load the rootfs into memory for us,
alongside the rest of the kernel image. Linux supports this via a so-called `initrd`,
which is a rootfs packed in a particular format. This can be enabled in `menuconfig`
under `General setup -> Initial RAM filesystem and RAM disk (initramfs/initrd) support`.
The path specified must be a `.cpio` image relative to the kernel source root.
As the kernel partition is only 8 MiB, I also enabled XZ compression support here.

The rootfs is easiest to build using `buildroot`. It can directly generate a `.cpio` archive,
which can be enabled by setting `Filesystem images -> cpio the root filesystem (for use as an initial RAM filesystem)`.
Compression must be disabled here, as the kernel also compresses the image by itself and a double-compressed
image will not work.

Under `System configuration`, I set a root password and enabled `getty` in order to get a shell prompt immediately on boot.
This was enough to get a shell via UART through my modified charging cradle!

By the way, just in case you don't want to take your charging cradle apart or don't have a dentist
friend with sufficiently steady hands to solder a thin piece of wire to the microUSB port's ID pin - 
aftermarket cradles are available on AliExpress for cheap, and many of them actually connect all 5 pins.

## Taking another look at kernel logs

After playing around in the shell for a bit, I took another look at `dmesg`:

{{< details "Click for kernel logs" >}}
~~~
[    0.000000] Linux version 6.7.0-rc8-next-20240104-ga8f3062caf0c-dirty (casept@l13) (arm-none-eabi-gcc (Arm GNU Toolchain 13.2.rel1 (Build arm-13.7)) 13.2.1 20231009, GNU ld (Arm GNU Toolchain 13.2.rel1 (Build arm-13.7)) 2.41.0.20231009) #58 SMP PREEMPT Fri Jan  5 13:10:53 CET 2024
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
[    0.000000] Memory: 390348K/523248K available (11264K kernel code, 1401K rwdata, 3620K rodata, 4096K init, 6604K bss, 34596K reserved, 98304K cma-reserved, 0K highmem)
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
[    0.001441] Console: colour dummy device 80x30
[    0.001572] Lock dependency validator: Copyright (c) 2006 Red Hat, Inc., Ingo Molnar
[    0.001588] ... MAX_LOCKDEP_SUBCLASSES:  8
[    0.001605] ... MAX_LOCK_DEPTH:          48
[    0.001620] ... MAX_LOCKDEP_KEYS:        8192
[    0.001635] ... CLASSHASH_SIZE:          4096
[    0.001650] ... MAX_LOCKDEP_ENTRIES:     32768
[    0.001666] ... MAX_LOCKDEP_CHAINS:      65536
[    0.001680] ... CHAINHASH_SIZE:          32768
[    0.001696]  memory used by lock dependency info: 4125 kB
[    0.001711]  memory used for stack traces: 2112 kB
[    0.001727]  per task-struct memory footprint: 1536 bytes
[    0.001840] Calibrating delay loop (skipped), value calculated using timer frequency.. 48.00 BogoMIPS (lpj=240000)
[    0.001884] CPU: Testing write buffer coherency: ok
[    0.002110] pid_max: default: 32768 minimum: 301
[    0.002967] Mount-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.003003] Mountpoint-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.007841] Running RCU synchronous self tests
[    0.007882] Running RCU synchronous self tests
[    0.009412] CPU0: update cpu_capacity 1024
[    0.009438] CPU0: thread -1, cpu 0, socket 0, mpidr 80000000
[    0.015718] RCU Tasks: Setting shift to 1 and lim to 1 rcu_task_cb_adjust=1.
[    0.016315] Running RCU Tasks wait API self tests
[    0.120460] Setting up static identity map for 0x40100000 - 0x40100060
[    0.121355] rcu: Hierarchical SRCU implementation.
[    0.121373] rcu: 	Max phase no-delay instances is 1000.
[    0.126041] smp: Bringing up secondary CPUs ...
[    0.131367] CPU1: update cpu_capacity 1024
[    0.131390] CPU1: thread -1, cpu 1, socket 0, mpidr 80000001
[    0.133042] smp: Brought up 1 node, 2 CPUs
[    0.133069] SMP: Total of 2 processors activated (96.00 BogoMIPS).
[    0.133091] CPU: All CPU(s) started in SVC mode.
[    0.136960] devtmpfs: initialized
[    0.238681] VFP support v0.3: implementor 41 architecture 2 part 30 variant 7 rev 3
[    0.240691] Running RCU synchronous self tests
[    0.240822] Running RCU synchronous self tests
[    0.241496] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
[    0.241581] futex hash table entries: 512 (order: 3, 32768 bytes, linear)
[    0.246325] pinctrl core: initialized pinctrl subsystem
[    0.254099] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.260289] DMA: preallocated 256 KiB pool for atomic coherent allocations
[    0.270347] thermal_sys: Registered thermal governor 'step_wise'
[    0.270784] cpuidle: using governor menu
[    0.271753] hw-breakpoint: Failed to enable monitor mode on CPU 0.
[    0.277181] printk: legacy console [ramoops-1] enabled
[    0.281642] pstore: Registered ramoops as persistent store backend
[    0.281805] ramoops: using 0x100000@0x51000000, ecc: 0
[    0.341788] Callback from call_rcu_tasks() invoked.
[    0.342615] platform 11000000.pinctrl: Fixed dependency cycle(s) with /soc/pinctrl@11000000/sleep-state
[    0.342853] platform 11000000.pinctrl: Fixed dependency cycle(s) with /soc/pinctrl@11000000/initial-state
[    0.396434] platform 11400000.pinctrl: Fixed dependency cycle(s) with /soc/pinctrl@11400000/sleep-state
[    0.396570] platform 11400000.pinctrl: Fixed dependency cycle(s) with /soc/pinctrl@11400000/initial-state
[    0.480616] iommu: Default domain type: Translated
[    0.480701] iommu: DMA domain TLB invalidation policy: strict mode
[    0.493389] SCSI subsystem initialized
[    0.498388] i2c-gpio i2c-gpio-0: using lines 659 (SDA) and 660 (SCL)
[    0.517307] nfc: nfc_init: NFC Core ver 0.1
[    0.517715] NET: Registered PF_NFC protocol family
[    0.520068] clocksource: Switched to clocksource mct-frc
[    0.585581] NET: Registered PF_INET protocol family
[    0.586605] IP idents hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.591034] tcp_listen_portaddr_hash hash table entries: 256 (order: 1, 10240 bytes, linear)
[    0.591330] Table-perturb hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.591400] TCP established hash table entries: 4096 (order: 2, 16384 bytes, linear)
[    0.591682] TCP bind hash table entries: 4096 (order: 6, 327680 bytes, linear)
[    0.593176] TCP: Hash tables configured (established 4096 bind 4096)
[    0.593696] UDP hash table entries: 256 (order: 2, 24576 bytes, linear)
[    0.593894] UDP-Lite hash table entries: 256 (order: 2, 24576 bytes, linear)
[    0.594554] NET: Registered PF_UNIX/PF_LOCAL protocol family
[    0.597527] RPC: Registered named UNIX socket transport module.
[    0.597678] RPC: Registered udp transport module.
[    0.597723] RPC: Registered tcp transport module.
[    0.597765] RPC: Registered tcp-with-tls transport module.
[    0.597807] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.599652] armv7-pmu pmu: hw perfevents: no interrupt-affinity property, guessing.
[    0.601868] hw perfevents: enabled with armv7_cortex_a7 PMU driver, 5 counters available
[    0.621202] Initialise system trusted keyrings
[    0.630474] workingset: timestamp_bits=30 max_order=17 bucket_order=0
[    0.632925] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    0.650835] NFS: Registering the id_resolver key type
[    0.651110] Key type id_resolver registered
[    0.651253] Key type id_legacy registered
[    0.651407] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    0.651708] nfs4flexfilelayout_init: NFSv4 Flexfile Layout Driver Registering...
[    0.651976] romfs: ROMFS MTD (C) 2007 Red Hat, Inc.
[    0.653506] Key type asymmetric registered
[    0.653697] Asymmetric key parser 'x509' registered
[    0.654149] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 247)
[    0.654309] io scheduler mq-deadline registered
[    0.654363] io scheduler kyber registered
[    0.654473] io scheduler bfq registered
[    0.656003] samsung-pinctrl 11000000.pinctrl: Failed to create device link (0x180) with soc
[    0.723417] dma-pl330 12680000.dma-controller: Loaded driver for PL330 DMAC-141330
[    0.723505] dma-pl330 12680000.dma-controller: 	DBUFF-32x4bytes Num_Chans-8 Num_Peri-32 Num_Events-32
[    0.758788] dma-pl330 12690000.dma-controller: Loaded driver for PL330 DMAC-141330
[    0.758874] dma-pl330 12690000.dma-controller: 	DBUFF-32x4bytes Num_Chans-8 Num_Peri-32 Num_Events-32
[    0.771392] exynos-chipid 10000000.chipid: Exynos: CPU[EXYNOS3250] PRO_ID[0xe3472000] REV[0x0] Detected
[    1.331638] Serial: 8250/16550 driver, 4 ports, IRQ sharing disabled
[    1.376291] 13800000.serial: ttySAC0 at MMIO 0x13800000 (irq = 56, base_baud = 0) is a S3C6400/10
[    1.390386] serial serial0: tty port ttySAC0 registered
[    1.394791] 13810000.serial: ttySAC1 at MMIO 0x13810000 (irq = 57, base_baud = 0) is a S3C6400/10
[    1.395387] printk: legacy console [ttySAC1] enabled
[    2.303441] exynos4-fb 11c00000.fimd: Adding to iommu group 0
[    2.306893] OF: graph: no port node found in /soc/fimd@11c00000
[    2.472953] brd: module loaded
[    2.559471] loop: module loaded
[    2.559641] sboot_upload: Init
[    2.570085] sboot_upload: Init OK
[    2.574288] max14577 7-0025: Device type: 2 (ID: 0xe, vendor: 0x5)
[    2.614336] max77836-battery: Failed to locate of_node [id: -1]
[    2.630444] max14577-regulator max77836-regulator: DMA mask not set
[    2.655348] MOT_2.7V: Bringing 3300000uV into 2700000-2700000uV
[    2.665707] i2c_dev: i2c /dev entries driver
[    2.685000] max14577-charger max77836-charger: DMA mask not set
[    2.755932] device-mapper: ioctl: 4.48.0-ioctl (2023-03-01) initialised: dm-devel@redhat.com
[    2.765460] sdhci: Secure Digital Host Controller Interface driver
[    2.766047] sdhci: Copyright(c) Pierre Ossman
[    2.771296] Synopsys Designware Multimedia Card Interface Driver
[    2.783285] dwmmc_exynos 12520000.mmc: IDMAC supports 32-bit address mode.
[    2.800398] dwmmc_exynos 12520000.mmc: Using internal DMA controller.
[    2.801289] dwmmc_exynos 12520000.mmc: Version ID is 260a
[    2.807134] dwmmc_exynos 12520000.mmc: DW MMC controller at irq 60,32 bit host data width,128 deep fifo
[    2.831937] dwmmc_exynos 12520000.mmc: allocated mmc-pwrseq
[    2.832128] mmc_host mmc1: card is non-removable.
[    2.860179] mmc_host mmc1: Bus speed (slot 0) = 50000000Hz (slot req 400000Hz, actual 396825HZ div = 63)
[    2.915541] exynos-ppmu 106a0000.ppmu: cannot get PPMU clock
[    2.917359] exynos-ppmu: new PPMU device registered 106a0000.ppmu (ppmu-event3-dmc0)
[    2.925422] exynos-ppmu 106b0000.ppmu: cannot get PPMU clock
[    2.931280] exynos-ppmu: new PPMU device registered 106b0000.ppmu (ppmu-event3-dmc1)
[    2.939342] exynos-ppmu: new PPMU device registered 112a0000.ppmu (ppmu-event3-rightbus)
[    2.947519] exynos-ppmu: new PPMU device registered 116a0000.ppmu (ppmu-event3-leftbus)
[    2.954794] max14577-muic max77836-muic: DMA mask not set
[    3.031606] mmc_host mmc1: Bus speed (slot 0) = 50000000Hz (slot req 50000000Hz, actual 50000000HZ div = 0)
[    3.045248] mmc1: new high speed SDIO card at address 0001
[    3.058020] max14577-muic max77836-muic: device ID : 0x75
[    3.065496] NET: Registered PF_INET6 protocol family
[    3.070926] Segment Routing with IPv6
[    3.071210] In-situ OAM (IOAM) with IPv6
[    3.073111] sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
[    3.083747] NET: Registered PF_PACKET protocol family
[    3.083964] NET: Registered PF_KEY protocol family
[    3.089218] Key type dns_resolver registered
[    3.093972] Registering SWP/SWPB emulation handler
[    3.162605] Loading compiled-in X.509 certificates
[    3.612997] input: gpio-keys as /devices/platform/gpio-keys/input/input0
[    3.621405] clk: Disabling unused clocks
[    3.638638] Freeing unused kernel image (initmem) memory: 4096K
[    3.639557] Run /init as init process
[    3.950067] ------------[ cut here ]------------
[    3.950372] WARNING: CPU: 1 PID: 0 at include/trace/events/lock.h:24 lock_acquire+0x2c4/0x39c
[    3.957713] Modules linked in:
[    3.960761] CPU: 1 PID: 0 Comm: swapper/1 Not tainted 6.7.0-rc8-next-20240104-ga8f3062caf0c-dirty #58
[    3.969949] Hardware name: Samsung Exynos (Flattened Device Tree)
[    3.976040]  unwind_backtrace from show_stack+0x10/0x14
[    3.981232]  show_stack from dump_stack_lvl+0x58/0x70
[    3.986268]  dump_stack_lvl from __warn+0x78/0x1c4
[    3.991044]  __warn from warn_slowpath_fmt+0xc4/0x1c4
[    3.996076]  warn_slowpath_fmt from lock_acquire+0x2c4/0x39c
[    4.001718]  lock_acquire from _raw_spin_lock_irqsave+0x4c/0x68
[    4.007619]  _raw_spin_lock_irqsave from cpu_pm_enter+0xc/0x50
[    4.013440]  cpu_pm_enter from exynos_cpu1_powerdown+0x8/0x84
[    4.019164]  exynos_cpu1_powerdown from exynos_enter_coupled_lowpower+0x44/0x74
[    4.026462]  exynos_enter_coupled_lowpower from cpuidle_enter_state+0x144/0x4dc
[    4.033749]  cpuidle_enter_state from cpuidle_enter_state_coupled+0x36c/0x3bc
[    4.040864]  cpuidle_enter_state_coupled from cpuidle_enter+0x3c/0x54
[    4.047288]  cpuidle_enter from do_idle+0x224/0x2cc
[    4.052152]  do_idle from cpu_startup_entry+0x28/0x2c
[    4.057184]  cpu_startup_entry from secondary_start_kernel+0x1a0/0x230
[    4.063695]  secondary_start_kernel from 0x401018c0
[    4.068740] irq event stamp: 2057
[    4.071852] hardirqs last  enabled at (2057): [<c08204bc>] cpuidle_enter_state_coupled+0x36c/0x3bc
[    4.080793] hardirqs last disabled at (2056): [<c017ce5c>] do_idle+0xb4/0x2cc
[    4.087908] softirqs last  enabled at (2054): [<c0101634>] __do_softirq+0x32c/0x50c
[    4.095546] softirqs last disabled at (2049): [<c012f16c>] __irq_exit_rcu+0x13c/0x190
[    4.103359] ---[ end trace 0000000000000000 ]---
[    4.107972] ------------[ cut here ]------------
[    4.112587] WARNING: CPU: 1 PID: 0 at include/trace/events/notifier.h:59 notifier_call_chain+0x100/0x198
[    4.122023] Modules linked in:
[    4.125064] CPU: 1 PID: 0 Comm: swapper/1 Tainted: G        W          6.7.0-rc8-next-20240104-ga8f3062caf0c-dirty #58
[    4.135737] Hardware name: Samsung Exynos (Flattened Device Tree)
[    4.141815]  unwind_backtrace from show_stack+0x10/0x14
[    4.147022]  show_stack from dump_stack_lvl+0x58/0x70
[    4.152056]  dump_stack_lvl from __warn+0x78/0x1c4
[    4.156829]  __warn from warn_slowpath_fmt+0xc4/0x1c4
[    4.161865]  warn_slowpath_fmt from notifier_call_chain+0x100/0x198
[    4.168115]  notifier_call_chain from raw_notifier_call_chain_robust+0x40/0x94
[    4.175317]  raw_notifier_call_chain_robust from cpu_pm_enter+0x28/0x50
[    4.181915]  cpu_pm_enter from exynos_cpu1_powerdown+0x8/0x84
[    4.187645]  exynos_cpu1_powerdown from exynos_enter_coupled_lowpower+0x44/0x74
[    4.194935]  exynos_enter_coupled_lowpower from cpuidle_enter_state+0x144/0x4dc
[    4.202226]  cpuidle_enter_state from cpuidle_enter_state_coupled+0x36c/0x3bc
[    4.209344]  cpuidle_enter_state_coupled from cpuidle_enter+0x3c/0x54
[    4.215767]  cpuidle_enter from do_idle+0x224/0x2cc
[    4.220628]  do_idle from cpu_startup_entry+0x28/0x2c
[    4.225662]  cpu_startup_entry from secondary_start_kernel+0x1a0/0x230
[    4.232173]  secondary_start_kernel from 0x401018c0
[    4.237067] irq event stamp: 2057
[    4.240329] hardirqs last  enabled at (2057): [<c08204bc>] cpuidle_enter_state_coupled+0x36c/0x3bc
[    4.249271] hardirqs last disabled at (2056): [<c017ce5c>] do_idle+0xb4/0x2cc
[    4.256388] softirqs last  enabled at (2054): [<c0101634>] __do_softirq+0x32c/0x50c
[    4.264026] softirqs last disabled at (2049): [<c012f16c>] __irq_exit_rcu+0x13c/0x190
[    4.271838] ---[ end trace 0000000000000000 ]---
[    4.276452] ------------[ cut here ]------------
[    4.281039] WARNING: CPU: 1 PID: 0 at include/trace/events/lock.h:69 lock_release+0x250/0x388
[    4.289547] Modules linked in:
[    4.292585] CPU: 1 PID: 0 Comm: swapper/1 Tainted: G        W          6.7.0-rc8-next-20240104-ga8f3062caf0c-dirty #58
[    4.303260] Hardware name: Samsung Exynos (Flattened Device Tree)
[    4.309338]  unwind_backtrace from show_stack+0x10/0x14
[    4.314545]  show_stack from dump_stack_lvl+0x58/0x70
[    4.319579]  dump_stack_lvl from __warn+0x78/0x1c4
[    4.324354]  __warn from warn_slowpath_fmt+0xc4/0x1c4
[    4.329388]  warn_slowpath_fmt from lock_release+0x250/0x388
[    4.335030]  lock_release from _raw_spin_unlock_irqrestore+0x18/0x60
[    4.341367]  _raw_spin_unlock_irqrestore from cpu_pm_enter+0x38/0x50
[    4.347703]  cpu_pm_enter from exynos_cpu1_powerdown+0x8/0x84
[    4.353432]  exynos_cpu1_powerdown from exynos_enter_coupled_lowpower+0x44/0x74
[    4.360723]  exynos_enter_coupled_lowpower from cpuidle_enter_state+0x144/0x4dc
[    4.368014]  cpuidle_enter_state from cpuidle_enter_state_coupled+0x36c/0x3bc
[    4.375132]  cpuidle_enter_state_coupled from cpuidle_enter+0x3c/0x54
[    4.381555]  cpuidle_enter from do_idle+0x224/0x2cc
[    4.386417]  do_idle from cpu_startup_entry+0x28/0x2c
[    4.391449]  cpu_startup_entry from secondary_start_kernel+0x1a0/0x230
[    4.397961]  secondary_start_kernel from 0x401018c0
[    4.402823] irq event stamp: 2057
[    4.406118] hardirqs last  enabled at (2057): [<c08204bc>] cpuidle_enter_state_coupled+0x36c/0x3bc
[    4.415059] hardirqs last disabled at (2056): [<c017ce5c>] do_idle+0xb4/0x2cc
[    4.422176] softirqs last  enabled at (2054): [<c0101634>] __do_softirq+0x32c/0x50c
[    4.429815] softirqs last disabled at (2049): [<c012f16c>] __irq_exit_rcu+0x13c/0x190
[    4.437626] ---[ end trace 0000000000000000 ]---
[    4.442531]
[    4.443773] =============================
[    4.447698] WARNING: suspicious RCU usage
[    4.451695] 6.7.0-rc8-next-20240104-ga8f3062caf0c-dirty #58 Tainted: G        W
[    4.459760] -----------------------------
[    4.463754] include/linux/rcupdate.h:751 rcu_read_lock() used illegally while idle!
[    4.471393]
[    4.471393] other info that might help us debug this:
[    4.471393]
[    4.479381]
[    4.479381] rcu_scheduler_active = 2, debug_locks = 1
[    4.485890] 1 lock held by swapper/1/0:
[    4.489707]  #0: c147a6c0 (rcu_read_lock){....}-{1:2}, at: cpu_pm_notify+0x0/0x13c
[    4.497260]
[    4.497260] stack backtrace:
[    4.501606] CPU: 1 PID: 0 Comm: swapper/1 Tainted: G        W          6.7.0-rc8-next-20240104-ga8f3062caf0c-dirty #58
[    4.512275] Hardware name: Samsung Exynos (Flattened Device Tree)
[    4.518389]  unwind_backtrace from show_stack+0x10/0x14
[    4.523563]  show_stack from dump_stack_lvl+0x58/0x70
[    4.528601]  dump_stack_lvl from lockdep_rcu_suspicious+0x150/0x1c4
[    4.534849]  lockdep_rcu_suspicious from cpu_pm_notify+0xe0/0x13c
[    4.540925]  cpu_pm_notify from exynos_cpu1_powerdown+0x60/0x84
[    4.546825]  exynos_cpu1_powerdown from exynos_enter_coupled_lowpower+0x44/0x74
[    4.554119]  exynos_enter_coupled_lowpower from cpuidle_enter_state+0x144/0x4dc
[    4.561407]  cpuidle_enter_state from cpuidle_enter_state_coupled+0x36c/0x3bc
[    4.568522]  cpuidle_enter_state_coupled from cpuidle_enter+0x3c/0x54
[    4.574946]  cpuidle_enter from do_idle+0x224/0x2cc
[    4.579811]  do_idle from cpu_startup_entry+0x28/0x2c
[    4.584842]  cpu_startup_entry from secondary_start_kernel+0x1a0/0x230
[    4.591352]  secondary_start_kernel from 0x401018c0
[    4.596247]
[    4.597684] =============================
[    4.601677] WARNING: suspicious RCU usage
[    4.605672] 6.7.0-rc8-next-20240104-ga8f3062caf0c-dirty #58 Tainted: G        W
[    4.613744] -----------------------------
[    4.617735] include/linux/rcupdate.h:779 rcu_read_unlock() used illegally while idle!
[    4.625548]
[    4.625548] other info that might help us debug this:
[    4.625548]
[    4.633535]
[    4.633535] rcu_scheduler_active = 2, debug_locks = 1
[    4.640044] 1 lock held by swapper/1/0:
[    4.643863]  #0: c147a6c0 (rcu_read_lock){....}-{1:2}, at: cpu_pm_notify+0x0/0x13c
[    4.651415]
[    4.651415] stack backtrace:
[    4.655755] CPU: 1 PID: 0 Comm: swapper/1 Tainted: G        W          6.7.0-rc8-next-20240104-ga8f3062caf0c-dirty #58
[    4.666432] Hardware name: Samsung Exynos (Flattened Device Tree)
[    4.672510]  unwind_backtrace from show_stack+0x10/0x14
[    4.677716]  show_stack from dump_stack_lvl+0x58/0x70
[    4.682751]  dump_stack_lvl from lockdep_rcu_suspicious+0x150/0x1c4
[    4.689001]  lockdep_rcu_suspicious from cpu_pm_notify+0x130/0x13c
[    4.695164]  cpu_pm_notify from exynos_cpu1_powerdown+0x60/0x84
[    4.701066]  exynos_cpu1_powerdown from exynos_enter_coupled_lowpower+0x44/0x74
[    4.708358]  exynos_enter_coupled_lowpower from cpuidle_enter_state+0x144/0x4dc
[    4.715648]  cpuidle_enter_state from cpuidle_enter_state_coupled+0x36c/0x3bc
[    4.722766]  cpuidle_enter_state_coupled from cpuidle_enter+0x3c/0x54
[    4.729189]  cpuidle_enter from do_idle+0x224/0x2cc
[    4.734050]  do_idle from cpu_startup_entry+0x28/0x2c
[    4.739083]  cpu_startup_entry from secondary_start_kernel+0x1a0/0x230
[    4.745595]  secondary_start_kernel from 0x401018c0
[   14.017483] platform 11c80000.dsi: deferred probe pending: platform: wait for supplier /soc/i2c@13860000/pmic@66/regulators/LDO6
[   14.024465] platform 13000000.gpu: deferred probe pending: platform: wait for supplier /soc/i2c@13860000/pmic@66/regulators/BUCK3
[   14.038675] platform 12480000.usb: deferred probe pending: platform: wait for supplier /soc/i2c@13860000/pmic@66/regulators/LDO12
[   14.047571] platform 10070000.rtc: deferred probe pending: platform: wait for supplier /soc/i2c@13860000/pmic@66/clocks
[   14.057723] platform 100c0000.tmu: deferred probe pending: platform: wait for supplier /soc/i2c@13860000/pmic@66/regulators/LDO7
[   14.069262] platform cpufreq-dt: deferred probe pending: (reason unknown)
[   14.076034] platform bus-dmc: deferred probe pending: platform: wait for supplier /soc/i2c@13860000/pmic@66/regulators/BUCK1
[   14.087228] platform 12510000.mmc: deferred probe pending: platform: wait for supplier /soc/i2c@13860000/pmic@66/regulators/LDO12
[   14.098859] platform bus-fsys: deferred probe pending: (reason unknown)
[   14.105481] platform bus-isp: deferred probe pending: (reason unknown)
[   14.111967] platform bus-lcd0: deferred probe pending: (reason unknown)
[   14.118484] platform bus-leftbus: deferred probe pending: platform: wait for supplier /soc/i2c@13860000/pmic@66/regulators/BUCK3
[   14.130110] platform bus-mcuisp: deferred probe pending: (reason unknown)
[   14.136798] platform bus-mfc: deferred probe pending: (reason unknown)
[   14.143388] platform bus-peril: deferred probe pending: (reason unknown)
[   14.150070] platform bus-rightbus: deferred probe pending: (reason unknown)
[   14.156935] platform 126c0000.adc: deferred probe pending: platform: wait for supplier /soc/i2c@13860000/pmic@66/regulators/LDO3
[   33.762779] SAFEOUT: disabling
[   33.767953] CHARGER: disabling
~~~
{{< /details >}}


Interestingly, it seems like the kernel generates some more output after running for a while,
and it mentions a `12510000.mmc` device!
Seems like it's waiting on the regulator supplying power to it to become available for whatever reason.

At this point, I guessed (incorrectly!) that this might be due to the devices
being hooked up to the wrong sub-regulator or getting an incorrect voltage delivered,
causing me to spend the next hour fruitlessly and laboriously comparing the device tree
to downstream kernel sources, only to find no significant differences.

Demotivated, I just wanted to get it working somehow, even if not in a "clean" way.
I figured that the regulator must've been set up to provide power by the bootloader, as it could clearly load the kernel.
Therefore, it should be possible to just comment out the link between eMMC and regulator to prevent waiting on it.
By applying this workaround to other device tree nodes, I also managed to get the screen and USB working.

## Debugging regulator issues

This is not a great fix, as peripherals will remain powered on when the device
goes to sleep. That's obviously a problem on a device with a battery merely holding a few hundred
mAh of charge, so I needed to dig deeper.

The I2C controller to which the regulator is attached was strangely missing under `/sys/class`, and
running `i2cdetect` didn't make it show up either. This made me suspect that the actual issue was not
with the regulator's configuration itself, but actually with it's controller.

Other devices, such as the main PMIC/MFD were actually controlled via bitbanged I2C
over GPIOs instead of the built-in I2C peripheral, and switching to bitbanging immediately made the regulator work.
I'm still not sure why the built-in peripheral doesn't work, but aside from slightly higher CPU and power usage
this workaround worked well enough that I really didn't feel like sinking even more time into root-causing the issue.

## Building a nicer rootfs

At this point, I wanted to test hardware which required larger userspace components,
such as the GPU with Mesa and inputs with `evtest`. The tiny kernel partition was simply too
small and having to re-flash the device just to install a package via buildroot was getting old fast.

It'd be nice to have a conventional Linux distro with a rich package repository available, and
creating a Debian rootfs for ARMv7 is easy enough:
```sh
sudo debootstrap --arch=armhf --foreign bookworm root
```
I also installed some basic packages by running the system under Qemu user-mode emulation on my workstation. Finally, I created an ext4 disk image of the correct size and copied Debian into it,
before flashing the image over the watch's userdata partition via `heimdall`.

The janky serial console was also getting annoying and took up the only USB port on the device,
so I instead enabled USB serial gadget support in the kernel.
This permits accessing the serial console via `/dev/ttyACM0` on the host,
while also allowing simultenous use of e.g. USB networking for installing packages from the internet
or getting access to an even more comfortable shell via SSH once the system has fully booted.

Of course, once you have a proper Linux running there's always one thing that needs to be done first:
![neofetch](/images/neofetch.png)

And, if the device has a display:

![sway](/images/sway.jpeg)

## Outlook

Though many peripherals are still broken,
the device is now at a point where solid groundwork has been laid for bringing up the rest
of the hardware.

In part 3 we'll get the GPU, touchscreen and RTC working.
