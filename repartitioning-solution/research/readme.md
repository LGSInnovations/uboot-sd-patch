Steps to Create Repartitioning Patch
====================================

Important Files:

`edison-src/meta-intel-edison/meta-intel-edison-bsp/recipes-bsp/u-boot/files/edison.env`
`edison-src/meta-intel-edison/utils/flash/ota_update.cmd`
`edison-src/meta-intel-edison/meta-intel-edison-distro/recipes-core/post-install/files/post-install.service`

```bash
# Partition definition
partitions=uuid_disk=${uuid_disk};name=u-boot0,start=1MiB,size=2MiB,uuid=${uuid_uboot0};name=u-boot-env0,size=1MiB,uuid=${uuid_uboot_env0};name=u-boot1,size=2MiB,uuid=${uuid_uboot1};name=u-boot-env1,size=1MiB,uuid=${uuid_uboot_env1};name=factory,size=1MiB,uuid=${uuid_factory};name=panic,size=24MiB,uuid=${uuid_panic};name=boot,size=32MiB,uuid=${uuid_boot};name=rootfs,size=1536MiB,uuid=${uuid_rootfs};name=update,size=768MiB,uuid=${uuid_update};name=home,size=-,uuid=${uuid_home};
```

You have to change the `ota_update.cmd` script from `fat` to `ext4`, also make it check for `10` and give back `a`.

Also, change `edison.env` to point do_load_ota_src to recovery/ota_update.scr.

Also, resize the update partition to be 1MiB (you could remove it, but you'd have to change other files in the system)