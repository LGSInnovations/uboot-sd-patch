Edison U-Boot OTA w/ External SD Card
========================================

**Performing an OTA Update**: Place the contents of the `toFlash` directory onto the update partition of the Intel Edison. Then, at the U-Boot prompt run `do_ota`. Or, if the system is running in multi-user mode, run as root: `reboot ota`.

**Motivation**: Currently ([Yocto 2.1](https://software.intel.com/en-us/iot/hardware/edison/downloads), Intel Edison) has an update partition of ~800MB. The Yocto `toFlash` directory is ~500MB, so placing it on to the update partition is no problem, allowing an OTA update through U-Boot. However, to do an OTA update using [Emutex Labs' Ubilinxu](http://www.emutexlabs.com/ubilinux) is currently impossible as the Ubilinux `toFlash` directory is ~1.6GB.

**Goal**: The Intel Edison allows for an external SD card; the goal is to point the U-Boot `do_load_ota_scr` env variable to the external SD card instead of the eMMC update partition (`mmc 0:9`).

