Intel Edison OTA w/ SD Card
===========================

This Gist attempts to document the research and (hopefully) steps to successfully allowing U-Boot to perform an [OTA](https://en.wikipedia.org/wiki/Over-the-air_programming) (over-the-air) update.

***Please comment with corrections and suggestions!***

**Performing an OTA Update**: Place the contents of the `toFlash` directory onto the update partition of the Intel Edison. Then, at the U-Boot prompt run `do_ota`. Or, if the system is running in multi-user mode, run as root: `reboot ota`.

**Motivation**: Currently ([Yocto 2.1](https://software.intel.com/en-us/iot/hardware/edison/downloads), Intel Edison) has an update partition of ~800MB. The Yocto `toFlash` directory is ~500MB, so placing it on to the update partition is no problem, allowing an OTA update through U-Boot. However, to do an OTA update using [Emutex Labs' Ubilinxu](http://www.emutexlabs.com/ubilinux) is currently impossible as the Ubilinux `toFlash` directory is ~1.6GB.

**Goal**: The Intel Edison allows for an external SD card; the goal is to point the U-Boot `do_load_ota_scr` env variable to the external SD card instead of the eMMC update partition (`mmc 0:9`). This is a superior solution to resizing partitions and allowing a larger update partition.

### Intel Edison System Overview ###

##### CPU #####

The Intel Edison uses a SoC that is a 22nm Intel Atom "Tangier" (Z34XX) that includes two Atom Silvermont cores (see [Wikipedia](https://en.wikipedia.org/wiki/Intel_Edison) and Intel's [Z34XX Brief](http://www.intel.com/content/www/us/en/processors/atom/atom-z34xx-smartphones-tablets-brief.html)). It's based on the "Merrifield" and "Moorefield" families.

##### GPIO #####

The Edison uses a Langwell GPIO interface:

```bash
# cat /proc/iomem
  ff008000-ff008fff : 0000:00:0c.0
    ff008000-ff008fff : langwell_gpio
```

##### eMMC #####

*See section 3.3 Managed NAND (eMMC) Flash of the [Intel Edison Compute Module Hardware Guide (Rev 4)](http://www.intel.com/support/edison/sb/CS-035274.htm)*

The Intel Edison has a 4GB [eMMC](https://en.wikipedia.org/wiki/MultiMediaCard#eMMC) device for its filesystem and the U-Boot environment partitions. This can be seen in the output of the `parted` command below.


```bash
# parted /dev/mmcblk0 print
Model: MMC H4G1d (sd/mmc)
Disk /dev/mmcblk0: 3909MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name         Flags
 1      1049kB  3146kB  2097kB               u-boot0
 2      3146kB  4194kB  1049kB               u-boot-env0
 3      4194kB  6291kB  2097kB               u-boot1
 4      6291kB  7340kB  1049kB               u-boot-env1
 5      7340kB  8389kB  1049kB  ext2         factory
 6      8389kB  33.6MB  25.2MB               panic
 7      33.6MB  67.1MB  33.6MB  fat16        boot
 8      67.1MB  1678MB  1611MB  ext4         rootfs
 9      1678MB  2483MB  805MB   fat32        update
10      2483MB  3909MB  1426MB  ext4         home
```

```bash
# cat /sys/kernel/debuc/mmc0/ios
clock:          200000000 Hz
actual clock:   200000000 Hz
vdd:            7 (1.65 - 1.95 V)
bus mode:       2 (push-pull)
chip select:    0 (don't care)
power mode:     2 (on)
bus width:      3 (8 bits)
timing spec:    8 (mmc high-speed SDR200)
signal voltage: 0 (1.80 V)
```

##### External SD Card Interface #####

*See section 4.3 SD Card Interface of the [Intel Edison Compute Module Hardware Guide (Rev 4)](http://www.intel.com/support/edison/sb/CS-035274.htm)*

There is one external SD card interface, according to the Edison's Hardware Guide. Although, `cat /proc/iomem` and `lspci -vvv` reveal three SD Host Controllers:

```bash
# cat /proc/iomem
  ff3fa000-ff3fa0ff : 0000:00:01.2
    ff3fa000-ff3fa0ff : mmc1
  ff3fb000-ff3fb0ff : 0000:00:01.3
    ff3fb000-ff3fb0ff : mmc2
  ff3fc000-ff3fc0ff : 0000:00:01.0
    ff3fc000-ff3fc0ff : mmc0
```

```bash
# lspci -vvv
00:01.0 SD Host controller: Intel Corporation Device 1190 (rev 01) (prog-if 01)
        Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 64
        Interrupt: pin A routed to IRQ 0
        Region 0: Memory at ff3fc000 (32-bit, non-prefetchable) [size=256]
        Capabilities: [b0] Power Management version 3
                Flags: PMEClk- DSI+ D1- D2- AuxCurrent=0mA PME(D0+,D1-,D2-,D3hot+,D3cold-)
                Status: D3 NoSoftRst- PME-Enable- DSel=0 DScale=0 PME-
        Capabilities: [b8] Vendor Specific Information: Len=08 <?>
        Capabilities: [c0] PCI-X non-bridge device
                Command: DPERE- ERO+ RBC=512 OST=1
                Status: Dev=ff:1f.7 64bit- 133MHz+ SCD- USC- DC=simple DMMRBC=512 DMOST=1 DMCRS=8 RSCEM- 266MHz+ 533MHz-
        Capabilities: [100 v1] Vendor Specific Information: ID=0000 Rev=0 Len=024 <?>
        Kernel driver in use: sdhci-pci

00:01.2 SD Host controller: Intel Corporation Device 1190 (rev 01) (prog-if 01)
        Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 64
        Interrupt: pin B routed to IRQ 37
        Region 0: Memory at ff3fa000 (32-bit, non-prefetchable) [size=256]
        Capabilities: [b0] Power Management version 3
                Flags: PMEClk- DSI+ D1- D2- AuxCurrent=0mA PME(D0+,D1-,D2-,D3hot+,D3cold-)
                Status: D3 NoSoftRst- PME-Enable- DSel=0 DScale=0 PME-
        Capabilities: [b8] Vendor Specific Information: Len=08 <?>
        Capabilities: [c0] PCI-X non-bridge device
                Command: DPERE- ERO+ RBC=512 OST=1
                Status: Dev=ff:1f.7 64bit- 133MHz+ SCD- USC- DC=simple DMMRBC=512 DMOST=1 DMCRS=8 RSCEM- 266MHz+ 533MHz-
        Capabilities: [100 v1] Vendor Specific Information: ID=0000 Rev=0 Len=024 <?>
        Kernel driver in use: sdhci-pci

00:01.3 SD Host controller: Intel Corporation Device 1190 (rev 01) (prog-if 01)
        Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 64
        Interrupt: pin C routed to IRQ 38
        Region 0: Memory at ff3fb000 (32-bit, non-prefetchable) [size=256]
        Capabilities: [b0] Power Management version 3
                Flags: PMEClk- DSI+ D1- D2- AuxCurrent=0mA PME(D0+,D1-,D2-,D3hot+,D3cold-)
                Status: D3 NoSoftRst- PME-Enable- DSel=0 DScale=0 PME-
        Capabilities: [b8] Vendor Specific Information: Len=08 <?>
        Capabilities: [c0] PCI-X non-bridge device
                Command: DPERE- ERO+ RBC=512 OST=1
                Status: Dev=ff:1f.7 64bit- 133MHz+ SCD- USC- DC=simple DMMRBC=512 DMOST=1 DMCRS=8 RSCEM- 266MHz+ 533MHz-
        Capabilities: [100 v1] Vendor Specific Information: ID=0000 Rev=0 Len=024 <?>
        Kernel driver in use: sdhci-pci
```

The eMMC has been verified to be the `mmc0` device at PCI address `0000:00:01.0` and memory address `0xff3fc00`. I believe that `mmc1` (`0xff3fa00`) is the external SD card and that `mmc2` is a hardware disabled controller (although, I'm still confused about this, as seen in the IRQ section below). The following output from `dmesg` cross-referenced with `lspci -vvv` (above) and the device tree in `/sys/bus/mmc/drivers/mmcblk` seems to verify:

```bash
# dmesg | grep mmc
[    1.595165] mmc1: no vqmmc regulator found
[    1.595544] mmc1: SDHCI controller on PCI [0000:00:01.2] using ADMA
[    1.606109] mmc2: no vqmmc regulator found
[    1.606576] mmc2: SDHCI controller on PCI [0000:00:01.3] using ADMA
[    1.739385] mmc1: new high speed SDHC card at address aaaa
[    1.740215] mmcblk1: mmc1:aaaa SL32G 28.7 GiB 
[    1.751294]  mmcblk1: p1
```

```bash
root@edison:/sys/bus/mmc/drivers/mmcblk# ll
--w-------    1 root     root        4.0K Jun  8 15:40 bind
lrwxrwxrwx    1 root     root           0 Jun  8 15:40 mmc0:0001 -> ../../../../devices/pci0000:00/0000:00:01.0/mmc_host/mmc0/mmc0:0001
lrwxrwxrwx    1 root     root           0 Jun  8 15:40 mmc1:aaaa -> ../../../../devices/pci0000:00/0000:00:01.2/mmc_host/mmc1/mmc1:aaaa
--w-------    1 root     root        4.0K Jan  1  2000 uevent
--w-------    1 root     root        4.0K Jun  8 15:40 unbind
```

It can be shown that device `mmc1` is at address `aaaa` which points to `pci0000:00/0000:00:01.2` which is at address `ff3fa000`. This address is important when telling U-Boot where to look for the device.

```bash
# cat /sys/kernel/debug/mmc1/ios
clock:          50000000 Hz
actual clock:   50000000 Hz
vdd:            17 (2.9 ~ 3.0 V)
bus mode:       2 (push-pull)
chip select:    0 (don't care)
power mode:     2 (on)
bus width:      2 (4 bits)
timing spec:    2 (sd high-speed)
signal voltage: 0 (3.30 V)
```

##### Interrupts and IRQs ######

*[Intro to Linux Interrupts](http://www.thegeekstuff.com/2014/01/linux-interrupts/)*

In the output below, we have, by colum:

1. IRQ (Interrupt Request) number
2. Interrupts fired on CPU0
3. Interrupts fired on CPU1
4. Type of interrupt
5. Name of the interrupt

Interrupts are registered in Kernel code using the `request_irq` function:
```c
int request_irq (	unsigned int  	irq,
 	                irq_handler_t  	handler,
 	                unsigned long  	irqflags,
 	                const char *  	devname,
 	                void *  		dev_id);
```

```bash
# cat /proc/interrupts | grep 'mmc\|sd_cd\|CPU0'
           CPU0       CPU1       
  0:       5166          0   IO-APIC-fasteoi   mmc0
 37:       2341          0   IO-APIC-fasteoi   mmc1
 38:       5612          0   IO-APIC-fasteoi   mmc2
320:        175          0  LNW-GPIO-demux     bcmsdh_sdmmc
333:          2          0  LNW-GPIO-demux     sd_cd
```
The `sd_cd` interrupt is coming from the Langwell GPIO block and is for SD Card Detection. Every insert/removal of the SD card causes an interrupt on this handler. The confusing thing is that all three mmc devices have registered interrupts, and in this specific case, `mmc2` (which is though to be disabled) has had more interrupts than `mmc1`. However, after inserting/removing an SD card, `mmc1` not `mmc2` increments. I am unsure what `bcmsdh_sdmmc` is, although the `bcm` portion of the name scares me that it's a proprietary Broadcomm thing.

*I have a hunch that `mmc2` might be a SDIO device -- possibly the wlan?*

```bash
# dmesg | grep -i gpio
[    1.595518] sdhci-pci 0000:00:01.2: GPIO: 77
[    1.595541] sdhci-pci 0000:00:01.2: request_irq(333, handler, 4d00000003, sd_cd_parker, slot);
```

### U-Boot Source Code ###

Digging through the U-Boot source code after the `edison-src/meta-intel-edison/meta-intel-edison-bsp/recipes-bsp/u-boot/files/upstream_to_edison.path` has been applied shows promising results. It seems like support for the external `mmc1` can be enabled by adding some extra code.

Starting from `boards.cfg` we will see how the eMMC (device `mmc0`) is being enabled:

```bash
# edison-src/out/current/build/tmp/work/edison-poky-linux/u-boot/2014.04-1-r0/git/boards.cfg
#Status, Arch, CPU:SPLCPU, SoC, Vendor, Board name, Target, Options, Maintainers
Active  x86         sivermont      tangier     intel           edison              edison                               edison:SYS_USB_OTG_BASE=0xf9100000,SYS_EMMC_PORT_BASE=0xff3fc000,SYS_TEXT_BASE=0x1101000                                          -
```

This config file shows that `CONFIG_SYS_EMMC_PORT_BASE=0xff3fc000`, which is used later in `tangier.c:board_mmc_init`.

```c
// edison-src/out/current/build/tmp/work/edison-poky-linux/u-boot/2014.04-1-r0/git/include/configs/edison.h
/*
* MMC
 */
#define CONFIG_MD5
#define CONFIG_GENERIC_MMC
#define CONFIG_MMC
#define CONFIG_SDHCI
#define CONFIG_TANGIER_SDHCI
#define CONFIG_CMD_MMC
#define CONFIG_MMC_SDMA
/*#define CONFIG_MMC_TRACE*/
```

This header config file seems to include support for `MMC` and the `TANGIER_SDHCI`. Uncommenting `CONFIG_MMC_TRACE` will print out the SD/MMC commands and show the contents of the registers from the MMC/SD device.

```c
// edison-src/out/current/build/tmp/work/edison-poky-linux/u-boot/2014.04-1-r0/git/arch/x86/cpu/tangier/tangier.c
int board_mmc_init(bd_t * bis)
{
	int index = 0;
	unsigned int base = CONFIG_SYS_EMMC_PORT_BASE + (0x40000 * index);

	return tangier_sdhci_init(base, index, 4);
}
```

This code initializes the eMMC. It is odd that `0x40000` is being multiplied by the index (which is `0` in this case). One would think duplicating this code with `index=1` would add the next device, but it doesn't seem to work.

### Attempting to Add Support for External SD Card ###

I changed the `tangier.c:board_mmc_init` function as follows:

```c
int board_mmc_init(bd_t * bis)
{
	int index = 0;
	unsigned int base = CONFIG_SYS_EMMC_PORT_BASE + (0x40000 * index);

	// eMMC at 0cff3fc000
	tangier_sdhci_init(base, index, 4)

	// mmc1 external SD card, index=1, width=4
	parker_sdhci_init(0xff3fa000, 1, 4);
	
	return 0;
}
```

Then, I copied and pasted the `tangier_sdhci_init` function and renamed to `parker_sdhci_init` in `drivers/mmc/tangier_sdhci.c`:

```c
int parker_sdhci_init(u32 regbase, int index, int bus_width)
{
	struct sdhci_host *host = NULL;

	host = (struct sdhci_host *)malloc(sizeof(struct sdhci_host));
	if (!host) {
		printf("sdhci__host malloc fail!\n");
		return 1;
	}

	memset(host, 0x00, sizeof(struct sdhci_host));

	host->name = "parker_sdhci";
	host->ioaddr = (void *)regbase;

	host->quirks = SDHCI_QUIRK_NO_HISPD_BIT | SDHCI_QUIRK_BROKEN_VOLTAGE |
	    SDHCI_QUIRK_BROKEN_R1B | SDHCI_QUIRK_32BIT_DMA_ADDR |
	    SDHCI_QUIRK_WAIT_SEND_CMD;
	host->voltages = MMC_VDD_29_30; // | MMC_VDD_32_33 | MMC_VDD_33_34 | MMC_VDD_165_195;


	printf("About to run sdhci_readw.\n");
	
	// All this function does is dereferrences the value at 0xff3fa0fe (which is the SDHC version reg)
	host->version = sdhci_readw(host, SDHCI_HOST_VERSION);
	
	// I am getting back 0xffff for version!
	// I should be expecting the lower 8 bits to be 0x02 (SD v3.0)
	printf("Version returned: %d\n", host->version);

	host->index = index;

	host->host_caps = MMC_MODE_HC;

	// Added to see if bus_width needed to be 4 bits (as per docs)
	if (bus_width == 4)
		host->host_caps |= MMC_MODE_4BIT;

	printf("About to run add_sdhci...\n");

	// Changed MAX clock to 50MHz
	add_sdhci(host, 50000000, 400000);

	printf("Ran add_sdhci.\n");

	return 0;
}
```
When I upload and run, U-Boot will hang and then restart.

