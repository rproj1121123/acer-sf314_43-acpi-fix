# ACPI DSDT patch for Acer Swift 3 SF314-43

This laptop is designed for Windows and [modern standby](https://docs.microsoft.com/en-us/windows-hardware/design/device-experiences/modern-standby),
and the firmware does not advertise S3 sleep state by default:

```
# dmesg | grep ACPI | grep supports
kernel: ACPI: PM: (supports S0 S4 S5)
```
```
# cat /sys/power/mem_sleep                                                            ✔  
[s2idle]
```
This means that Linux never uses [suspend-to-RAM](https://www.kernel.org/doc/html/latest/admin-guide/pm/sleep-states.html#suspend-to-ram)
but only [suspend-to-idle](https://www.kernel.org/doc/html/latest/admin-guide/pm/sleep-states.html#suspend-to-idle)
which uses more battery power when sleeping.


However, it's possible to patch the firmware's ACPI DSDT table to restore S3 support and
[instruct Linux to load it at boot](https://www.kernel.org/doc/html/latest/admin-guide/acpi/initrd_table_override.html):
```
# dmesg | grep ACPI | grep supports
kernel: ACPI: PM: (supports S0 S3 S4 S5)
```
```
# cat /sys/power/mem_sleep                                                            ✔  
s2idle [deep]
```

This repository contains the [patch to the DSDT](dsdt.patch) and a Makefile to automate the process.

## Requirements

- Laptop model: Acer Swift 3 SF314-43
- BIOS version: [1.04](https://global-download.acer.com/GDFiles/BIOS/BIOS/BIOS_Acer_1.04_A_A.zip?acerid=637659969200273816) (2021-08-31, "Enable fTPM support for China")
- `acpica` package

## Building initrd with patched ACPI DSDT

To automatically patch the ACPI DSDT and generate an initrd image, just run `make`.
Patching will fail if the BIOS is not an exact match (check requirements above).

Manual process:
1. Install the `acpica` package
2. Run `acpidump -b` to dump all ACPI tables to `*.dat` files
3. Run `iasl -e *.dat -d dsdt.dat` to decompile the DSDT table to `dsdt.dsl`
4. Patch the DSDT via `patch < dsdt.patch`
5. Recompile the DSDT via `iasl -ve dsdt.dsl`
6. Generate initrd image:
   1. `mkdir -p kernel/firmware/acpi`
   2. `cp dsdt.aml kernel/firmware/acpi/`
   3. `find kernel | cpio -H newc --create > /boot/acpi-override.img`

## Configuring boot loader

Two things need to be set up in the boot loader:
1. Load `/boot/acpi-override.img` as an (additional) initrd.\
   This will make Linux use the patched ACPI DSDT with S3 support instead of the one from the firmware.
2. Add `mem_sleep_default=deep` to the
   [kernel's command line parameters](https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html).\
   This will make Linux use S3 when going to sleep (otherwise it will still default to `s2idle`, suspend-to-idle).

### GRUB

1. Change [`/etc/default/grub`](https://www.gnu.org/software/grub/manual/grub/html_node/Simple-configuration.html):
   ```
   GRUB_CMDLINE_LINUX_DEFAULT = "... mem_sleep_default=deep"
   GRUB_EARLY_INITRD_LINUX_CUSTOM = "... acpi-override.img"
   ```
2. Run `update-grub`

### systemd-boot

Change entries in [`/boot/loader/entries/*.conf`](https://www.freedesktop.org/software/systemd/man/loader.conf.html):
```
title    ...
linux    ...
initrd   /acpi-override.img
initrd   ...
options  ... mem_sleep_default=deep
```

### rEFInd

Change [`/boot/refind-linux.conf`](https://www.rodsbooks.com/refind/linux.html#refind_linux):
```
"Boot with standard options" "initrd=acpi-override.img ... mem_sleep_default=deep"
```
Change [manual boot stanzas](https://www.rodsbooks.com/refind/configfile.html#stanzas):
```
menuentry "..." {
  ostype Linux
  loader ...
  initrd ...
  options "initrd=acpi-override.img ... mem_sleep_default=deep"
}
```
Note that multiple `initrd` entries *do not work*, and any additional initrd files
must be declared in `options`.