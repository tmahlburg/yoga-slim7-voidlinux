# Linux on Lenovo Yoga Slim 7 AMD

## About

This are various tweaks and fix to run Linux on Lenovo Yoga Slim 7. This note is validated on the following configuration
- Void Linux on 2020-10-03
- Lenovo Yoga Slim 7 AMD (Ryzen 7)

## Summary

Legend:
- :heavy_check_mark: Works out of the box (Kernel 5.4)
- :hammer_and_wrench: Require tweaking
- :negative_squared_cross_mark: Not working
- :question: Unknown

| Feature                      | Status              | Description                                                                      |
| ---------------------------- | ------------------- | -------------------------------------------------------------------------------- |
| Power (battery and charging) | :heavy_check_mark:  |                                                                                  |
| Storage                      | :heavy_check_mark:  | Disable bitlocker on windows to access windows partition from Linux              |
| Graphic                      | :heavy_check_mark:  |                                                                                  |
| USB                          | :heavy_check_mark:  |                                                                                  |
| Keyboard                     | :heavy_check_mark:  |                                                                                  |
| Speakers                     | :heavy_check_mark:  |                                                                                  |
| Microphone                   | :hammer_and_wrench: | Have not found a perfect solution yet, but you can record audio without tweaking |
| Audio jack                   | :heavy_check_mark:  |                                                                                  |
| Wifi and Bluetooth           | :heavy_check_mark:  |                                                                                  |
| Webcam                       | :heavy_check_mark:  |                                                                                  |
| External display (HDMI)      | :heavy_check_mark:  |                                                                                  |
| Suspend                      | :hammer_and_wrench: | See detail [below](#Suspend)                                                     |

## Fixes

**DISCLAIMER** I am not responsible for any damage and negative consequences to your system

### Bootloader

Trying to use `efibootmgr` to boot my kernel directly, I ran into some problems. This configuration works:

Assuming your root partition is `/dev/nvme0n1p2` and your EFI system partition (esp) is the vfat-formatted partition `/dev/nvme0n1p1`, mount the esp to /boot. This way the UEFI can read the kernel and initrd without needing extra filesystem drivers. Next, configure `efibootmgr` using Voids `/etc/default/efibootmgr-kernel-hook`:

```
# Options for the kernel hook script installed by the efibootmgr package.
#MODIFY_EFI_ENTRIES=0
# To allow efibootmgr to modify boot entries, set
MODIFY_EFI_ENTRIES=1
# Kernel command-line options.  Example:
OPTIONS="root=/dev/nvme0n1p2 quiet"
# Disk where EFI Partition is.  Default is /dev/sda
DISK="/dev/nvme0n1"
# Partition number of EFI Partition.  Default is 1
PART=1
```

At the end, run 

```
root# xbps-reconfigure -f linuxX.Y
````

replacing *X.Y* with your current kernel version. Now comes the unexpected part: On my system the UEFI wouldn't show me the (according to `efibootmgr`) successfully created boot entry, so I couldn't boot from it. The workaround I found is to set the newly created boot entry as *next boot* using `efibootmgr` once, rebooting and setting it as the default boot entry, like so:

```sh
root# efibootmgr
----
# example output:
BootCurrent: 0001
Timeout: 0 seconds
BootOrder: 2001,2002,2003,0001
Boot0001* Void Linux with kernel 5.8
Boot2001* EFI USB Device
Boot2002* EFI DVD/CDROM
Boot2003* EFI Network
----
root# efibootmgr -n 0001 # set Void Linux entry as next boot
root# reboot
----
# after reboot:
root# efibootmgr -o 0001,2001,2002,2003 # set Void Linux entry as default boot entry
```
Now your system should boot as expected. At least mine does.

### Suspend

#### Background

In recent times Microsoft has introduced something called "[Modern Standby](https://docs.microsoft.com/en-us/windows-hardware/design/device-experiences/modern-standby)" which is essentially a new way to suspend with the advantage of allowing the system to do some task while suspending (e.g. fetching emails). In order to support this mode, the BIOS must not advertise support for the traditional suspend (S3) <sup>[Citation needed]</sup> 

```bash
$ dmesg | grep ACPI:\ \(
[    0.383096] ACPI: (supports S0 S4 S5)

```

By not advertising support for S3, the kernel will only support s2idle sleep mode which is also supported by Linux

```bash
$ cat /sys/power/mem_sleep 
[s2idle]
```

However there seems to be a problem with the amdgpu driver on resuming from suspend in this mode.

```
amdgpu 0000:03:00.0: [drm:amdgpu_ring_test_helper [amdgpu]] *ERROR* ring gfx test failed (-110)
[drm:amdgpu_device_ip_resume_phase2 [amdgpu]] *ERROR* resume of IP block <gfx_v9_0> failed -110
[drm:amdgpu_device_resume [amdgpu]] *ERROR* amdgpu_device_ip_resume failed (-110).
```

#### Possible Solutions

There are three solutions:
- Wait until the problem in amdgpu driver is fixed in newer kernel version
- Wait until Lenovo adds an option in the UEFI to advertise S3 support (similar to the options available in Thinkpad)
- Modify the system to advertise S3 support (see below)

#### Advertise S3

**0. Important notes**

All of the following commands assume **root shell** (sudo -i)

**1. Get the required tools**

Download and compile from https://www.acpica.org/downloads
```sh
# Install some required dependencies
root# xbps-install m4 build-devel bison flex

# Download and extract
$ wget https://acpica.org/sites/acpica/files/acpica-unix-20200717.tar.gz
$ tar -xvf acpica-unix-20200717.tar.gz
$ cd acpica-unix-20200717

# Make
$ make clean
$ make -j$(nproc)

# Add to PATH for easy access
$ PATH=$PATH:$(realpath ./generate/unix/bin/)
```

**2. Dump the ACPI files and decompile the DSDT table**

```sh
# Create new working directory
$ mkdir acpi
$ cd acpi

# Dump acpi table to binary files
root# acpidump -b

# Decompile dsdl.bat with all of the .dat as external symbol
$ iasl -e *.dat -d dsdt.dat
```

**3. Apply patch**

Copy dsdt.patch from this repo and lid_wakeup.patch [this gist](https://gist.github.com/polikutinevgeny/7d673fe2453d88461ab06edfd7556d14) (Backup in this repo) and patch dsdt.dsl
```sh
$ patch <dsdt.patch
$ patch <lid_wakeup.patch
```

**4. Recompile the modified table**

```sh
# Recompile dsdt to new hex asml table (ignore warning)
$ iasl -ve -tc dsdt.dsl
```

**5. Make override archive**
```bash
# copy to /boot
root# cp dsdt.aml /boot/
```

**6. Set the default sleep type to S3 (deep)**

Add `mem_sleep_default=deep` to your kernel command line. For grub open `/etc/default/grub` and follow this example:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet mem_sleep_default=deep"
```

For efibootmgr open `/etc/default/efibootmgr-kernel-hook` and follow this example:

```
OPTIONS="quiet mem_sleep_default=deep"
```

**7. Add the new ACPI table to the initrd**

Create `/etc/dracut.d/acpi-override` and add the following:

```
acpi_override="yes"
acpi_table_dir="/boot"
```

**8. Reconfigure the kernel package to apply the changes**

Run the following command, but replace the version number *X.Y* with the one that's installed.

```
root# xbps-reconfigure -f linuxX.Y
```

## Thanks
- @SteveImmanuel for the information regarding microphone on kernel 5.7
- https://www.reddit.com/r/linuxhardware/comments/i28nm5/ideapad_14are05_s3_sleep_fix/ 
- https://wiki.archlinux.org/index.php/Lenovo_ThinkPad_X1_Yoga_(Gen_3)#Enabling_S3_(before_BIOS_version_1.33)
