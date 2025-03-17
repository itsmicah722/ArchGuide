# Pre-Installation Guide
The pre-installation includes the setup prior to the dual boot and later intallation of arch linux along side your existing x64-bit Windows 10/11 OS. We will perform the following in this guide:

:white_check_mark: Ensure your hardware is **compatible** with Arch Linux. See the [Arch Wiki Hardware Compatibility](https://wiki.archlinux.org/title/Category%3AHardware). Arch Linux runs on any x86_64 compatible machine with ~1GiB of RAM, so most modern systems are supported.

:white_check_mark: Verify that your system is in **UEFI Mode**, not legacy BIOS.

:white_check_mark: Ensure you have a **USB drive** with atleast ~2GiB of space. If you don't have one, Sandisk is a good option for a reasonable price. 

:white_check_mark: Download the latest Arch Linux **ISO** from the official [Arch Linux website](https://archlinux.org/download/).

:white_check_mark: Use a tool like [Ventoy](https://ventoy.net/en/download.html) to create a bootable USB.

:white_check_mark: **Deallocate space for Arch Linux in Windows** via the `Disk Management` app.

:white_check_mark: **Disable Fast Boot** in Windows: 
   - Go to *Control Panel* > *Power Options* > *Choose What the Power Buttons Do* > *Change settings that are unavailable* 
   - Disable *Turn on fast startup*.

:white_check_mark: **Disable Secure Boot** in your system's UEFI/BIOS settings.