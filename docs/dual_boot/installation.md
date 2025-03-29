# Arch Linux Dual-Boot Install Guide

The installing includes setting up the **Arch Linux Base System** along with configuring dual-boot with **Windows**. Like the previous *pre-installation* guide, this page is designed to be beginner friendly and every step will be explained thoroughly. 

---

## Prerequisites

✅ Complete **ALL** of the steps included in the [Pre-Installation](pre_installation.md) guide before continuing through this guide. After that you will be booted into Arch Linux and ready for installation. 

---

## Connecting to the Internet

The *Arch Linux Installation Environment* in this guide **WILL** require an internet connection in order to download the necessary base packages for the system. Currently I don't have a guide for how to install Arch Linux without an internet connection, but I may make one later.

### Ethernet

Check if you already have an internet connection.
> Note: If you are using **Ethernet**, you should already have internet.

```rs
ping -c 3 cloudflare.com
```

### Wi-Fi

If not using Ethernet, launch the `iwctl` tool to connect to a wireless network:
> Type `exit` if you ever want to abort the tool without applying changes.

```rs
iwctl
```

Scan for networks, get their names, connect to one of them, enter the password, and exit. 

```rs
station wlan0 scan
station wlan0 get-networks
station wlan0 connect <SSID>
<PASSWORD>
exit
```

Ping a reputable website to verify connectivity:

```rs
ping -c 3 cloudflare.com
```

---

# READ THIS WARNING ⛔

Below we will begin *Partitioning* the disks manually. Arch Linux gives us the freedom to setup the OS from scratch to fit whatever your needs are. In this case we will be dealing with *Creating & Formatting* partitions on your hard disks to setup the base system. **Should you not follow the instructions below you WILL run into problems sooner or later. Be very careful in this part of the guide to avoid unrecoverable data loss!** 
> Note: If you don't understand what these mean everything will be explained in the next section. 

- Change the **Disk Label** "*nvmeXnX*" in this guide with the correct disk label on **YOUR** computer such as `nvme0n1`.

- Change the **Partition Number** "nvmeXnXpX" in this guide with the correct partition number on **YOUR** disk such as `nvme0n1p2`.

- DO NOT delete the or format the **EFI System partition** of type "*FAT32*" or "*vfat*" of size *~100-500MiB*. Your computer will not boot if you delete this. 

- DO NOT delete or format any **Windows partitions** of type "*NTFS*" varying in size. You will lose your personal data and operating system files on Windows.

## Disk Preparation

### Get Partition Layout

​A *Hard Disk Drive* [(HDD)](https://www.geeksforgeeks.org/hard-disk-drive-hdd-secondary-memory/) is a device inside a computer that stores all your data, including the operating system/system(s), applications, and personal files. All computers have them and every computer has a unique *Disk Layout*. This includes different [Partitions](https://www.techtarget.com/searchstorage/definition/partition) which are sections of the HDD which is treated as a separate unit of the OS. On the other hand, [Volumes](https://en.wikipedia.org/wiki/Volume_(computing)) are storage areas for a single filesystem, typically stored inside partitions. The terms *Volume* and *Partition* are often used interchangeably but [Are Not](https://blog.purestorage.com/purely-educational/partition-vs-volume-whats-the-difference/) exactly the same. 
> Note: The terms "Drive", "Hard Disk/HDD", "NVME SSD", and "SATA HDD", all mean the same thing for the purpose of this guide. All of them are simply physical devices you could hold in your hand located in the PC for data storage.

To get accurate information on how your particular computer's *Disk Layout* is structured, you'll need to run the `lsblk` command:

```rs
lsblk
```

This will output something similar to this: 

```rs
NAME          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1       259:0    0 476.9G  0 disk
├─nvme0n1p1   259:1    0   100M  0 part /boot/efi
├─nvme0n1p2   259:2    0    16M  0 part
├─nvme0n1p3   259:3    0 250.0G  0 part
├─nvme0n1p4   259:4    0   500M  0 part
└─nvme0n1p5   259:5    0 226.3G  0 part /
sda             8:0    0 931.5G  0 disk
├─sda1          8:1    0   500M  0 part
├─sda2          8:2    0   128M  0 part
├─sda3          8:3    0 500.0G  0 part
├─sda4          8:4    0   450M  0 part
├─sda5          8:5    0    50G  0 part /mnt/data
└─sda6          8:6    0 380.4G  0 part
```

- The *nvme0n1* is a modern [NVME SSD](https://www.seagate.com/blog/what-is-an-nvme-ssd/) disk storing 476.9GiB of data on your computer. 

- The *sda* is an old [SATA HDD](https://drivesaversdatarecovery.com/everything-you-need-to-know-about-sata-hard-drives/) disk storing 941.5GiB of data in your computer.

- The *nvme0n1"p1"* is a partition label telling us this is a partition inside the NVME disk.

- The *sda"1"* is also a partition label telling us this is a partition inside the SDA disk.

- At the top the *name*, *size*, and *type* categories are self explanatory, but the *mount-points* category will be discussed later. 

### Get Filesystem Layout

Every computer contains partitions on its disk/disk(s), and each of these partitions will contain [Filesystems](). A filesystem is a method used by operating systems to store, organize, and manage data on its hard drives. When a hard drive is divided into sections called partitions, each partition can be *Formatted* with a specific filesystem. 

To get accurate information on the Filesystem layout, run the `lsblk -f` command: 

```rs
lsblk -f
```

This will output something similar to this: 

```rs
NAME        FSTYPE LABEL      UUID                                 MOUNTPOINTS
nvme0n1
├─nvme0n1p1 vfat   SYSTEM     A1B2-C3D4                            /boot/efi
├─nvme0n1p2
├─nvme0n1p3 ntfs   Windows    1234567890ABCDEF
├─nvme0n1p4 ntfs   Recovery   FEDCBA0987654321
├─nvme0n1p5 ext4   ArchRoot   1122aabb-3344-cc55-dd66-778899aabbcc /
sda
├─sda1      vfat   ESP        B2C3-D4E5
├─sda2
├─sda3      ntfs   Data       23456789ABCDEF01
├─sda4      ntfs   WinRE      CDEF0123456789AB
├─sda5      ext4   LinuxData  2233bbcc-4455-dd66-ee77-8899aabbccdd /mnt/data
└─sda6
```

- The *FSTYPE* category tells us what type of filesystem the partition uses.

- The *LABEL* category tells us what that particular filesystem means to the OS.

- The *UUID* or (Universally Unique Identifier) is the assigned label of each filesystem on a partition by the system. 

Each filesystem serves a unique purpose in the OS. FAT32 & NTFS:

- [FAT32/vfat](https://en.wikipedia.org/wiki/File_Allocation_Table) (File Allocation Table 32): A filesystem compatible with both Windows and Linux aligning with the UEFI specification. You can have one EFI system partition formatted as FAT32 storing both Linux and Windows boot files. 

- [NTFS](https://en.wikipedia.org/wiki/NTFS) (New Technology File System): The primary filesystem for Windows. *(ext4 and btrfs are much better honestly)*

### Create Linux Filesystem Partition

*UEFI* and *BIOS* are the two primary types of [Firmware Interfaces](https://www.geeksforgeeks.org/uefiunified-extensible-firmware-interface-and-how-is-it-different-from-bios/) that initialize hardware and boot your operating system. BIOS is the legacy standard seen in older hardware, while UEFI is its modern replacement seen in newer hardware. Different [Partitioning Styles](https://www.easeus.com/partition-master/mbr-vs-gpt.html#part1) work with different firmware interfaces. *MBR* (Master Boot Record) is the older standard used by older hard disks and works with BIOS. *GPT* (GUID Partition Table) is the modern replacement used by UEFI systems. 

The `fdisk` tool can be used to handle both MBR and GPT disks alike, so we will use that tool for partitioning:
> Type `exit` if you ever want to abort the tool without applying changes.

```rs
fdisk /nvmeXnX
```

Once inside fdisk, type `n` to create a new **Linux Filesystem** partition, and press `Enter` to give it the default partition number:

```rs
[fdisk]# n
[fdisk]# <ENTER>
```

Press `Enter` for the default first sector location for the partition on the hard drive:

```rs
[fdisk]# <ENTER>
```

Next `fdisk` will ask you how much space you want to give this partition. Previously in the *Pre-Installation* you freed up space on your windows volume + space for an extra EFI partition. In this case the guide said **500GiB** for the main partition:

```rs
[fdisk]# +500G
```

### Create EFI System Partition

Now we will create the **EFI partition** for storing GRUB files for booting Arch Linux. Like before, type `n` to create a new partition and press `Enter` to give it the default partition number. 

```rs
[fdisk]# n
[fdisk]# <ENTER>
```

Press `Enter` for the default first sector location for the partition on the hard drive:

```rs
[fdisk]# <ENTER>
```

Previously in the *Pre-Installation* you freed up space on windows for an extra EFI partition. In this case this guide we used **500MiB** for the EFI partition:

```rs
[fdisk]# +500M
```

`fdisk` defaults to making new partitions of type *Linux Filesystem*, however this is an EFI partition. So we will type `t` to change the type, and enter the partition number it has. If you don't know what partition number the EFI partition we created is, type `p` to print the layout:

```rs
[fdisk]# p
[fdisk]# t
[fdisk]# <PARTITION_NUMBER>
```

Apply the changes to your disk:

```rs
[fdisk]# w
[fdisk]# y
```

### Format The Partitions

Format the Linux Filesystem to use Btrfs:

```rs
    mkfs.btrfs -L ArchLinux -f /dev/nvmeXnXpX
```

Format the EFI partition to use FAT32:

```rs
mkfs.fat -F32 /dev/nvmeXnXpX
```

## Create Btrfs Subvolumes

[Btrfs](https://btrfs.readthedocs.io/en/latest/Introduction.html) (Better File System) is a modern Linux filesystem with advanced features like copy-on-write, snapshots, and built-in support for [RAID](https://en.wikipedia.org/wiki/RAID) (multiple storage devices). One of its most useful features is [Subvolumes](https://btrfs.readthedocs.io/en/latest/Subvolumes.html), which are independent filesystems within a single Btrfs partition; each serving a different purpose. Subvolumes let you separate parts of your system, such as the system files, personal files, logs, temporary files, and more. This makes it MUCH easier to manage backups, rollbacks, and system organization without needing multiple partitions.

### Mount Root Temporarily

In order to create the *Btrfs Subvolumes*, we first temporarily mount the **Root** partition to `/mnt` (common mount point). We mount here since the subvolumes will exist at the *(Root)* of the whole filesystem. 

```rs
mount /dev/nvmeXnXpX /mnt
```

### Creating The Subvolumes

Next we create the Btrfs Subvolumes; each of which will be mounted in the coming section. 

This is the root filesystem `/`. It will contain the core operating system: 

```rs
btrfs subvolume create /mnt/@
```

Holds user's personal files, settings, downloads, and documents in `/home`:

```rs
btrfs subvolume create /mnt/@home
```

Contains variable data like system logs, caches, databases, and package info in `/var`:

```rs
btrfs subvolume create /mnt/@var
```

Used to isolate system logs `/var/log` from other parts of */var*:

```rs
btrfs subvolume create /mnt/@log
```

Stores temporary downloaded package files (like pacman cache) in `/var/cache`:
> Note: `pacman` is the package manager where you download and install most of your software in Arch Linux. 

```rs
btrfs subvolume create /mnt/@cache
```

Used for temporary files in `/tmp` that don’t need to be backed up:

```rs
btrfs subvolume create /mnt/@tmp
```

Holds snapshots taken by tools like Snapper or Btrfs Assistant in `/snapshots`:

```rs
btrfs subvolume create /mnt/@snapshots
```

Used for placing a swapfile in `/swap`, isolated to avoid copy-on-write issues:

```rs
btrfs subvolume create /mnt/@swap
```

### Unmount Root

Once the subvolumes are created, unmount the root partition so we can remount each subvolume individually later with proper options:

```rs
$ umount /mnt
```

---

## Mounting

After creating our subvolumes, we need to *Mount* them so the Arch Linux installer knows where to put the system files. [Mounting](https://man.archlinux.org/man/mount.8) in Linux means attaching a storage location (like a disk, partition or subvolume) to a specific directory in the filesystem. Each subvolume gets mounted to its own location (e.g. `/mnt`, `/mnt/home`, `/mnt/var`, etc.). This allows us to control how each part of the system behaves — such as enabling *compression*, reducing disk writes with *noatime*, or disabling *copy-on-write*.

### Mount Btrfs Subvolumes

Mount the root subvolume **"@"** to `/mnt` with compression, SSD optimizations, and reduced write frequency.

```rs
mount -o subvol=@,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt
```

Create the *Mount Points* (directories) for each subvolume so we can mount them next:

```rs
mkdir -p /mnt/{home,var,tmp,.snapshots,swap,efi}
```

Mount subvolume **"@home"** to directory `mnt/home` for user data like documents and config files in: 

```rs
mount -o subvol=@home,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt/home
```

Mount subvolume **"@var"** to directory `/mnt/var`, which holds package data, logs, and system state:

```rs
mount -o subvol=@var,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt/var
```

Create subdirectories inside `mnt/var` for log files and cached data:
> ⚠️ This is done **AFTER** mounting **"@var"** to `/var` because if not the mount will overwrite any existing subdirectories we create.  

```rs
mkdir -p /mnt/var/{log,cache}
```

Mount subvolume **"@log"** to `/mnt/var/log` for system logs separately, making them easier to manage or exclude from snapshots:

```rs
mount -o subvol=@log,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt/var/log
```

Mounts subvolume **"@cache"** to directory `/mnt/var/cache`, used by pacman and other tools to store downloaded packages:

```rs
mount -o subvol=@cache,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt/var/cache
```

Mount **"@tmp"** to directory `/mnt/tmp` for temporary files; it’s reset often and doesn’t need to be backed up:

```rs
mount -o subvol=@tmp,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt/tmp
```

Mount subvolume **"@snapshots"** to directory `/mnt/.snapshots`, where snapshot tools can store system restore points:

```rs
mount -o subvol=@snapshots,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt/.snapshots
```

### SWAP (Optional)

If you plan **DO** to use Hibernation:
- Swap size must be at least equal to your RAM (e.g. 16 GiB RAM → 16 GiB swap).

If you **DO NOT** use Hibernation:
- 8–16 GiB RAM: Use 2–4 GiB swap.
- 32 GiB or more: Use 1–2 GiB, or skip it entirely if you're sure.

[Hibernation](https://knowledgebase.frame.work/hibernation-on-linux-BkL1N5ffJg) requires enough swap to store the full RAM contents. If you're not using it, swap just acts as a backup when memory is tight. A [SWAP](https://wiki.archlinux.org/title/Swap) file is a space on your disk that acts like extra RAM when your physical memory (RAM) runs out. 

Mount subvolume **"@swap"** to directory `/mnt/swap` with copy-on-write disabled (nodatacow) to safely hold a swapfile:

```rs
mount -o subvol=@swap,nodatacow,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt/swap
```

### Mount EFI System Partition

The EFI System Partition (ESP) is where bootloaders live. We created it earlier and it should be formatted as *FAT32*, around *500 MiB* in size. Mount it to `/mnt/efi`:

```rs
mount /dev/nvmeXnXpX /mnt/efi
```

## Setup Base Arch Linux System

Install the base system along with essential tools and packages:

```rs
pacstrap /mnt base linux linux-firmware sudo grub efibootmgr os-prober btrfs-progs networkmanager nmcli nvim
```

### Package Breakdown (Short Overview):

- **base:** Core Arch Linux system files.
- **linux:** The Linux kernel.
- **linux-firmware:** Firmware for CPUs, Wi-Fi, GPUs, etc.
- **sudo:** Lets users run commands as root.
- **grub:** The GRUB bootloader.
- **efibootmgr:** Manages UEFI boot entries.
- **os-prober:** Detects Windows or other OSes during bootloader setup.
- **btrfs-progs:** Tools for working with the Btrfs filesystem.
- **networkmanager:** Automatically manages network connections.
- **nmcli:** Command-line interface for NetworkManager.
- **nvim:** Lightweight text editor (NeoVim).

### Microcode (CPU-Specific Fixes)

Install one of the following depending on your CPU. These are considered [Microcode](https://wiki.archlinux.org/title/Microcode) packages are low-level CPU drivers loaded at boot to fix bugs or security issues.

If you have an *Intel* CPU:  

```rs
pacman -S intel-ucode
```

If you have an *AMD* CPU:  

```rs
pacman -S amd-ucode
```

### Generate Filesystem Table (fstab)

This step scans your mounted partitions and writes them to `/etc/fstab` so that they’re mounted automatically on boot:

```rs
genfstab -U /mnt >> /mnt/etc/fstab
```

> Note: The `-U` flag ensures it uses *UUIDs* (unique disk identifiers), which are more stable than device names like `/dev/sda1`.

### Change Root into Your New System

This replaces the current root environment with `/mnt`, so any commands you run from this point on will affect your new Arch Linux install — ***(NOT the live ISO system)***:

```rs
arch-chroot /mnt
```

### Create the Swap File

This creates a swap file on your mounted @swap Btrfs subvolume. The chattr +C disables copy-on-write (required for swap to work on Btrfs):

chattr +C /mnt/swap

dd if=/dev/zero of=/swap/swapfile bs=1M count=8192 status=progress  
chmod 600 /swap/swapfile  
mkswap /swap/swapfile  
swapon /swap/swapfile

This example creates an 8GiB swap file. Adjust the count= value if you want more or less swap.

Once done, don’t forget to add the swapfile to /etc/fstab so it gets used on boot:

echo '/swap/swapfile none swap defaults 0 0' >> /etc/fstab

### Setup GRUB

Install GRUB

```rs
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
```

Enable os-prober in GRUB: 

```rs
sed -i '/^GRUB_DISABLE_OS_PROBER=/d' /etc/default/grub
echo 'GRUB_DISABLE_OS_PROBER=false' >> /etc/default/grub
```

Temporarily mount the windows ESP so os-prober can find it:

```rs
mkdir -p /mnt/windows_esp
mount /dev/nvmeXnXpX /mnt/windows_esp
```

Update GRUB config

```rs
grub-mkconfig -o /boot/grub/grub.cfg
```

You should now see something like this: 

```rs
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-linux
Found initrd image: /boot/initramfs-linux.img
Found fallback initrd image(s) in /boot:  initramfs-linux-fallback.img
Warning: os-prober will be executed to detect other bootable partitions.
Its output will be used to detect bootable binaries on them and create new boot entries.
Found Windows Boot Manager on /dev/nvme0n1p1@/EFI/Microsoft/Boot/bootmgfw.efi
Adding boot menu entry for UEFI Firmware Settings ...
done
```

Make sure you see the **Windows Boot Manager** line and the linux kernel files. 

Configure timezone: 

```rs
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
hwclock --systohc
```

Configure Locale: 

```rs
$ echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
$ locale-gen
$ echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Set a HostName for your computer:

```rs
echo <HOSTNAME> > /etc/hostname
```

Enable NetworkManager: 

```rs
systemctl enable NetworkManager
```

Set root password: 

```rs
# passwd
[passwd]# <PASSWORD>
```

Exit chroot, unmount everything and reboot:

```rs
exit
swapoff /mnt/swap/swapfile
umount -R /mnt
reboot
```

---