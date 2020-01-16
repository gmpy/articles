# The test I have done to pstore/blk

## introduction

Pstore/blk(including blkoops and mtdpstore) is a new module to dump crash log to device.

Before upstream submission, pstore/blk is tested on arch ARM and x84_64, block device and mtd device, built as modules and in kernel.

Here are the details.

## ubuntu with U disk

### environment

```
arch: x86_64
system: ubuntu 16.04 (on VirtualBox)
linux: 5.5.0-rc2+ (tree from pstore subsystem, branch for-next/pstore)
built as: modules
storage: mechanical hard drive and U disk
```

### do before test

Firstly, download the lastest linux and commit patches of pstore/blk.

```
git://git.kernel.org/pub/scm/linux/kernel/git/kees/linux.git for-next/pstore
```

Secondly, build and install kernel. Here we build pstore/blk and blkoops as modules.

```
CONFIG_PSTORE_BLK=m
CONFIG_PSTORE_BLKOOPS=m
CONFIG_PSTORE_BLKOOPS_DMESG_SIZE=64
CONFIG_PSTORE_BLKOOPS_PMSG_SIZE=64
CONFIG_PSTORE_BLKOOPS_CONSOLE_SIZE=64
CONFIG_PSTORE_BLKOOPS_FTRACE_SIZE=64
CONFIG_PSTORE_BLKOOPS_BLKDEV=""
CONFIG_PSTORE_BLKOOPS_DUMP_OOPS=y
```

Commands like this.

```Bash
$ cp /boot/config-`uname -r` .config
$ make menuconfig
$ make -j4
$ make modules_install
$ make install
```

After reboot, kernel will be updated.

```
$ uname -a
Linux gmpy-VirtualBox 5.5.0-rc2+ #54 SMP Tue Jan 14 09:27:48 CST 2020 x86_64 x86_64 GNU/Linux
```

Thirdly, build a sample module to pretend block driver.

```C
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/blkoops.h>
#include <linux/major.h>

static int disk_pstore_init(void)
{
	/*
	 * As no real panic write for block device driver, we pass NULL here.
	 * In this case, all recorders except panic are enabled.
	 */
	return blkoops_register_blkdev(SCSI_DISK0_MAJOR,
			BLKOOPS_DEV_SUPPORT_ALL, NULL);
}
module_init(disk_pstore_init);

static void disk_pstore_exit(void)
{
	blkoops_unregister_blkdev(SCSI_DISK0_MAJOR);
}
module_exit(disk_pstore_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("WeiXiong Liao <liaoweixiong@allwinnertech.com>");
MODULE_DESCRIPTION("Sample of blkoops for Block device");
```

The last step, create a little partition on U disk or hard drive. For instance, I have created a 1M partition (sdb2) for pstore/blk.

```
$ cat /proc/parition
major minor  #blocks  name

[...]
   8       16    3872256 sdb
   8       17    3773976 sdb1
   8       18      10240 sdb2
```

### testing

Firstly, insert your U disk and get device number of partition you created. In my case, it is ```8:18```.

Secondly, install all modules built before:

```
# insmod pstore_blk.ko
# insmod blkoops.ko blkdev=8:18 # the device number you want
# insmod disk_pstore.ko # a test module, codes as above
```

The following log from kernel:

```
[  469.245029] pstore-blk: Registered blkoops as blkzone backend for Oops Pmsg Console Ftrace
[  469.245078] pstore: Using crash dump compression: deflate
[  470.593736] printk: console [pstore-1] enabled
[  470.596540] pstore: Registered pstore-blk as persistent store backend
```

Note that, ```pstore``` is built-in kernel and ```pstore filesystem``` is mounted automatically.

Then, try each funtions of pstore:

```
# echo "psmg log" > /dev/pmsg0                                    # pmsg
# echo '<1>console err log' > /dev/kmsg                           # console
# echo 1 > /sys/kernel/debug/pstore/record_ftarce                 # ftrace
# echo c > /proc/sysrq-trigger                                    # dmesg
```

After rebooting and re-installing pstore/blk modules, we should find that pstore/blk works.

```
# ll
drwxr-x--- 2 root root     0 1月  15 11:52 ./
drwxr-xr-x 8 root root     0 1月  15 11:52 ../
-r--r--r-- 1 root root    31 1月  15 11:53 console-pstore-blk-0
-r--r--r-- 1 root root  3666 1月  15 11:53 demsg-pstore-blk-0
-r--r--r-- 1 root root 65524 1月  15 11:53 ftrace-pstore-blk-0
-r--r--r-- 1 root root     9 1月  15 11:53 pmsg-pstore-blk-0

# cat pmsg-pstore-blk-0
pmsg log

# head -n 4 demsg-pstore-blk-0
Oops#1 Part1
<4>[  211.645867] atkbd serio0: Spurious NAK on isa0060/serio0. Some program might be trying to access hardware directly.
<4>[  268.357587] atkbd serio0: Spurious NAK on isa0060/serio0. Some program might be trying to access hardware directly.
<1>[  311.451585] BUG: kernel NULL pointer dereference, address: 0000000000000000

# cat console-pstore-blk-0
console err log

# head -n 3 ftrace-pstore-blk-0
CPU:2 ts:51984994049982464 b8a083a0ffffffff  2facf000ffffffff  0xb8a083a0ffffffff <- 0x2facf000ffffffff
CPU:2 ts:51984907882201088 b8b05157ffffffff  2facf100ffffffff  0xb8b05157ffffffff <- 0x2facf100ffffffff
CPU:2 ts:51984888017977344 b8b013d3ffffffff  2facf200ffffffff  0xb8b013d3ffffffff <- 0x2facf200ffffffff
```

As we have no panic_write registered, pstore/blk can not store panic log.

## Allwinner Android with mmc

### environment

```
arch: arm
system: Android (on Allwinner R16)
linux: 3.4 (adapted from newest pstore and pstore/blk)
built as: built-in
storage: mmc and TF card
```

### do before test

Firstly, create a 8M partition:

```
$cat /proc/partition
[...]
 259        7      8192 mmcblk0p15
[...]
```

Secondly, build in kernel with dmesg and psmg recorder enabled as below:

```
CONFIG_PSTORE=y
CONFIG_PSTORE_PMSG=y
CONFIG_PSTORE_BLK=y
CONFIG_PSTORE_BLKOOPS=y
CONFIG_PSTORE_BLKOOPS_DMESG_SIZE=64
CONFIG_PSTORE_BLKOOPS_PMSG_SIZE=64
CONFIG_PSTORE_BLKOOPS_BLKDEV=""
CONFIG_PSTORE_BLKOOPS_DUMP_OOPS=y
```

Thirdly, add module parameter to cmdline:

```
# cat /proc/cmdline
[...] blkoops.blkdev=259:7
```

What has mmc driver done to adapt to pstore/blk? Allwinner mmc driver provides panic_write and registers to pstore/blk on function
```mmc_rescan_try_freq```, which is really *NOT* a standard and perfect way, but it works well.

```
static int mmc_rescan_try_freq(struct mmc_host *host, unsigned freq)
{
[...]
	if (!mmc_attach_mmc(host)) {
		pr_info("*******************mmc init ok *******************\n");
		ret = blkoops_register_blkdev(MMC_BLOCK_MAJOR, sunxi_mci_panic_write);
		if (ret)
			ret = blkoops_register_blkdev(BLOCK_EXT_MAJOR, sunxi_mci_panic_write))
	}
[...]
}
```

### testing

Firstly, trigger a crash.

```
echo c > /proc/sysrq-trigger
```

Note that, ```pstore``` is built-in kernel and ```pstore filesystem``` is mounted automatically.

After rebooting, we will get log files on ```/sys/fs/pstore```

```
# ls -al /sys/fs/pstore
-r--r--r-- root     root        65520 1970-01-01 23:16 dmesg-pstore-blk-2
-r--r--r-- root     root        65521 1970-01-01 23:16 dmesg-pstore-blk-3
-r--r--r-- root     root        65524 1970-01-01 23:16 pmsg-pstore-blk-0

# head -n 4 dmesg-pstore-blk-2
Oops: Total 2 times
Oops#1 Part1
c1 set ios: clk 150000Hz bm PP pm ON vdd 3.3V width 4 timing SD-HS(SDR25) dt B
<4>[   67.407799] [mmc]: sdc1 set ios: clk 50000000Hz bm PP pm ON vdd 3.3V width 4 timing SD-HS(SDR25) dt B

# head -n 4 dmesg-pstore-blk-3
Panic: Total 2 times
Panic#2 Part1
 67.415107] [wifi_pm]: wifi power off
<4>[   67.415119] wl_android_wifi_on: Failed

# hexdump -C -n 128 pmsg-pstore-blk-0
00000000  6c 52 00 e8 03 86 00 65  64 20 77 69 74 68 20 63  |lR.....ed with c|
00000010  6f 6e 74 65 78 74 20 61  6e 64 72 6f 69 64 2e 61  |ontext android.a|
00000020  70 70 2e 52 65 63 65 69  76 65 72 52 65 73 74 72  |pp.ReceiverRestr|
00000030  69 63 74 65 64 43 6f 6e  74 65 78 74 40 38 37 66  |ictedContext@87f|
00000040  36 34 66 39 20 61 6e 64  20 69 6e 74 65 6e 74 20  |64f9 and intent |
00000050  49 6e 74 65 6e 74 20 7b  20 61 63 74 3d 61 6e 64  |Intent { act=and|
00000060  72 6f 69 64 2e 69 6e 74  65 6e 74 2e 61 63 74 69  |roid.intent.acti|
00000070  6f 6e 2e 4d 45 44 49 41  5f 53 43 41 4e 4e 45 52  |on.MEDIA_SCANNER|
00000080
```

It seems that pmsg is used for android system logs.

## Allwinner TinaLinux with spinand (mtd)

### environment

```
arch: arm
system: TinaLinux (on Allwinner R328)
linux: 4.9 (adapted from newest pstore and pstore/blk)
built as: built-in
storage: spinand (mtd device)
```

### do before test

Firstly, create a small size mtd partition named pstore as the same as testing *allwinner android with mmc*.

```
# cat /sys/class/mtd/mtd3/name
pstore
```

Secondly, build blkoops/blk and mtdpstore in kernel. We need to enable full functions of pstore and set device by Kconfig for test:

```
CONFIG_MTD_PSTORE=y
CONFIG_PSTORE=y
CONFIG_PSTORE_CONSOLE=y
CONFIG_PSTORE_PMSG=y
CONFIG_PSTORE_BLK=y
CONFIG_PSTORE_BLKOOPS=y
CONFIG_PSTORE_BLKOOPS_DMESG_SIZE=64
CONFIG_PSTORE_BLKOOPS_PMSG_SIZE=64
CONFIG_PSTORE_BLKOOPS_CONSOLE_SIZE=64
CONFIG_PSTORE_BLKOOPS_FTRACE_SIZE=64
CONFIG_PSTORE_BLKOOPS_BLKDEV="pstore"
CONFIG_PSTORE_BLKOOPS_DUMP_OOPS=y
```

### testing

During booting, the log shows as below:

```
Creating 5 MTD partitions on "sunxi_mtd_nand":
[...]
0x000000500000-0x000000580000 : "pstore"
pstore-blk: Registered blkoops as blkzone backend for Oops Panic
pstore: Registered pstore-blk as persistent store backend
mtdoops-pstore: Attached to MTD device 3
[...]
```

Pstore/blk does not enable console, pmsg and ftarce even if Kconfig is set, because mtd device driver only support dmesg at the moment.

Then, we try to trigger a crash and get OOPS log.

```
# ll /sys/fs/pstore
drwxr-x---    2 root     root             0 Jan  1  1970 .
drwxr-xr-x    4 root     root             0 Jan  1  1970 ..
-r--r--r--    1 root     root         15579 Jan 16  2020 dmesg-pstore-blk-0

# head -n 4 dmesg-pstore-blk-0
Oops: Total 1 times
Oops#1 Part1
<6>[    1.793240] sunxi-mmc sdc1: Can't get vdmmc regulator string
<6>[    1.799556] sunxi-mmc sdc1: Can't get vdmmc33sw regulator string
```

As my spinand driver does not support panic_write, pstore/blk can only log OOPS.

We trigger crash for several times.

```
# ll /sys/fs/pstore
drwxr-x---    2 root     root             0 Jan  1  1970 .
drwxr-xr-x    4 root     root             0 Jan  1  1970 ..
-r--r--r--    1 root     root         15579 Jan 16  2020 dmesg-pstore-blk-0
-r--r--r--    1 root     root         15712 Jan 16 09:15 dmesg-pstore-blk-1
-r--r--r--    1 root     root         15712 Jan 16 09:06 dmesg-pstore-blk-2
-r--r--r--    1 root     root         15712 Jan 16 09:06 dmesg-pstore-blk-3
-r--r--r--    1 root     root         15713 Jan 16 09:06 dmesg-pstore-blk-4
-r--r--r--    1 root     root         15713 Jan 16 09:06 dmesg-pstore-blk-5
```

It turns out they all work well.
