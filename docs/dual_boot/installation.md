# Arch Linux Dual-Boot Install Guide

This guide provides a step-by-step walkthrough for installing Arch Linux on a **Btrfs filesystem** while dual-booting with **Windows 10/11**. It is designed to be beginner-friendly while also including best practices and optimizations for experienced users. This serves as a personal reference for future installations and is available for everyone.

---

## 1. Prerequisites

✅ Ensure your hardware is compatible with Arch Linux. See the [Arch Wiki Hardware Compatibility](https://wiki.archlinux.org/title/Category%3AHardware). Arch Linux runs on any x86_64 compatible machine with ~1GiB of RAM, so most modern systems are supported.

✅ Verify that your system is in **UEFI Mode**, not legacy BIOS.

✅ Download the latest Arch Linux **ISO** from the official [Arch Linux website](https://archlinux.org/download/).

✅ Use a tool like [Ventoy](https://ventoy.net/en/download.html) to create a bootable USB.

✅ **Deallocate space for Arch Linux in Windows** via the `Disk Management` app.

✅ **Disable Fast Boot** in Windows: 
   - Go to *Control Panel* > *Power Options* > *Choose What the Power Buttons Do* > *Change settings that are unavailable* 
   - Disable *Turn on fast startup*.

✅ **Disable Secure Boot** in your system's UEFI/BIOS settings.

---

## 2. Boot Into Arch Linux

1. Reboot your PC and enter the *Boot Menu* and select your `USB device`.
2. Choose the `Arch Linux ISO` from Ventoy.
3. Select `Normal mode` in the Ventoy menu.
4. When the Arch Linux menu appears, select `Arch Linux install medium (x86_64, UEFI)`.

---

## 3. Connecting to the Internet

Check if your system already has an internet connection:
> If using **Ethernet**, you should already be connected. 

```sh
$ ping -c 3 cloudflare.com
```

Launch `iwctl` to connect to Wi-Fi:

```sh
$ iwctl
[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
[iwd]# station wlan0 connect <SSID>
[iwd]# <PASSWORD>
[iwd]# exit
```

Verify connectivity:

```sh
$ ping -c 3 cloudflare.com
```

---

# READ THIS WARNING :no_entry:

- Change any `Disk Labels` in this guide such as *nvme0n1* or *nvme1n1* to the correct disk labels on **YOUR** computer as needed.

- Change any `Partition Numbers` in this guide such as *nvme0n1**p2*** or *nvme1n1**p5*** to the correct partition numbers on **YOUR** disk as needed.

- Do not delete the or format the `EFI partition` of type *FAT32* or *vfat*, which is of size *~100-500MiB*.

- Do not delete or format any `Windows partitions` of type *NTFS*, which may vary in size.

Should you not follow the above you **WILL** run into problems sooner or later. Carefully follow the guide to avoid this. 

## 4. Disk Preparation

### Identify Partitions

List the current disk and partition layout for accurate information on your system. 

```sh
$ lsblk
```

### Partition the Disk for Arch Linux

Launch `gdisk` for modifying our *GPT* disk:

```sh
$ gdisk /dev/nvme0n1
[gdisk]# n
[gdisk]# <Enter>
[gdisk]# <Enter>
[gdisk]# <Enter>
[gdisk]# <Enter>        
[gdisk]# w
[gdisk]# y
```

- Creates a new partition
- Uses default partition number
- Uses the default start position for the partition
- Uses the remaining free space we made in windows with the disk management tool
- Uses the default partition type of `8300` which means *Linux Filesystem* for our `Btrfs` filesystem
- We write the changes.

Run the command below to view the changes written. You should see a partition of filesystem type *ext4*. Take note of its label such as *nvme0n1p5* for example.

```sh
lsblk -f
```

**MAKE SURE YOU HAVE THE CORRECT PARTITION LABEL FOR THE COMMANDS BELOW!!!**

Format the new partition as `Btrfs` - *better filsystem*:

```sh
$ mkfs.btrfs -L ArchLinux -f /dev/nvme0n1p5      # Replace nvme0n1p5 with your partition number
```

---

## 5. Create and Mount Btrfs Subvolumes

Mount the new partition temporarily:

```sh
$ mount /dev/nvme0n1p5 /mnt
```

Create subvolumes:

```sh
$ btrfs subvolume create /mnt/@
$ btrfs subvolume create /mnt/@home
$ btrfs subvolume create /mnt/@var
$ btrfs subvolume create /mnt/@log
$ btrfs subvolume create /mnt/@cache
$ btrfs subvolume create /mnt/@tmp
$ btrfs subvolume create /mnt/@snapshots
$ btrfs subvolume create /mnt/@swap
```

Unmount the partition:

```sh
$ umount /mnt
```

Now, mount the subvolumes with optimal options:

```sh
mount -o subvol=@,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvme0n1p5 /mnt
$ mkdir -p /mnt/{home,var,tmp,.snapshots,swap,efi}
$ mount -o subvol=@home,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvme0n1p5 /mnt/home
$ mount -o subvol=@var,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvme0n1p5 /mnt/var
$ mkdir -p /mnt/var/{log,cache}
$ mount -o subvol=@log,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2 /dev/nvme0n1p5 /mnt/var/log
$ mount -o subvol=@cache,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvme0n1p5 /mnt/var/cache
$ mount -o subvol=@tmp,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvme0n1p5 /mnt/tmp
$ mount -o subvol=@snapshots,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvme0n1p5 /mnt/.snapshots
$ mount -o subvol=@swap,nodatacow,ssd,discard=async,space_cache=v2 /dev/nvme0n1p5 /mnt/swap
```

Mount the Windows EFI partition:

```sh
$ mount /dev/nvme0n1p1 /mnt/boot/efi
```

---

## 6. Install the Base System

```sh
$ pacstrap /mnt base linux linux-firmware btrfs-progs grub efibootmgr os-prober networkmanager
```

Generate the `fstab`:

```sh
$ genfstab -U /mnt >> /mnt/etc/fstab
```

Verify the fstab entries:

```sh
$ cat /mnt/etc/fstab
```

---

## 7. Chroot and Configure the System

```sh
$ arch-chroot /mnt
```

### Set Timezone

```sh
$ ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
$ hwclock --systohc
```

### Configure Locale

```sh
$ echo "en_US.UTF-8 UTF-8" > /etc/locale.gen
$ locale-gen
$ echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

### Set Hostname

```sh
$ echo <HOSTNAME> > /etc/hostname
```

---

## 8. Install GRUB Bootloader

```sh
$ grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=ArchLinux
$ sed -i 's/GRUB_DISABLE_OS_PROBER=false/' /etc/default/grub
$ grub-mkconfig -o /boot/grub/grub.cfg
```

Enable NetworkManager:

```sh
$ systemctl enable NetworkManager
```

Set the root password:

```sh
$ passwd
[passwd]# <PASSWORD>
```

Exit chroot:

```sh
exit
```

Unmount everything:

```sh
$ umount -R /mnt
```

Reboot:

```sh
$ reboot
```

---

## 9. Post-Installation Setup

Create a **swap file**:

```sh
$ btrfs filesystem mkswapfile --size 4g --uuid clear /swap/swapfile
$ swapon /swap/swapfile
$ echo "/swap/swapfile none swap defaults 0 0" | tee -a /etc/fstab
```

Enable Btrfs snapshot integration:

```sh
$ systemctl enable grub-btrfsd.service
```

Your dual-boot Arch Linux setup with Btrfs is now complete! 🚀
