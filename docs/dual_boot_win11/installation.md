<div align="center">

<img src="../../assets/arch_linux_logo.svg" alt="Arch Linux Logo" width="200" height="200">

# Arch Linux & Win11 Dual-Boot Install Guide

In this guide, we’ll be working from within the **Live Arch Linux Installation Environment** to set up your partitions, install the base system, configure the bootloader, and handle essential user settings. Just like the *Pre-Installation* guide, this walkthrough is designed to be beginner-friendly, with each step clearly explained from start to finish.

</div>

---

## Prerequisites

✅ Complete **ALL** of the steps included in the [Pre-Installation](pre_installation.md) guide before continuing through this guide. After that you will be booted into Arch Linux and ready for installation. 

---

## Connecting to the Internet

This guide **WILL** require an internet connection for downloading the essential base system packages and utils. Currently I don't have a guide for how to install Arch Linux without an internet connection. If you do not have an available network to connect to, go [here](https://wiki.archlinux.org/title/Offline_installation).

### Connect to Ethernet

If you are using *Ethernet*, you should already have an internet connection. Ping a website to check:

```rs
ping -c 3 cloudflare.com
```

### Connect to Wi-Fi

Launch the built-in [iwctl](https://wiki.archlinux.org/title/Iwd) tool to connect to a wireless network:
> **Note:** Type `exit` at any point if you want to abort the tool.

```rs
iwctl
```

List available *Wireless Interfaces* *(stations)*. Usually wireless stations are called: `wlan0`, `wlp2s0`, `wlan1`.

```rs
device list
```

Start a scan to find nearby networks:

```rs
station <STATION_NAME> scan
```

Show the list of networks:

```rs
station <STATION_NAME> get-networks
```

Connect to the SSID *(Wi-Fi Name)* of your network:
> **Note:** You will be prompted to enter the *Wi-Fi Password*.

```rs
station <STATION_NAME> connect <SSID>
```

Exit the tool once you connected:

```rs
exit
```

Ping a reputable website to verify connectivity:

```rs
ping -c 3 cloudflare.com
```

---

## READ THIS WARNING ⛔

The guide will walk you through every step. In this case we will be dealing with *Creating & Formatting* partitions and subvolumes on your hard disks to setup the base system. 

### SHOULD YOU NOT FOLLOW THE INSTRUCTIONS BELOW YOU (WILL) HAVE ISSUES! BE VERY CAREFUL IN THIS NEXT PART OF THE GUIDE TO AVOID UNRECOVERABLE DATA LOSS.

- **MAKE SURE YOU CHANGE:** the *Disk Label:* "*nvmeXnX*" in this guide with the correct disk label on **YOUR** computer such as `nvme0n1` or `sda`.

- **MAKE SURE YOU CHANGE:** the *Partition Number:* "*nvmeXnX(pX)*" in this guide with the correct partition number on **YOUR** disk such as `nvme0n1p2` or `sda2`.

- **DO NOT:** delete the or format the *EFI partition* of type "*FAT32*" or "*vfat*" of size *~100-500MiB*. Your computer will not boot if you delete this. 

- **DO NOT:** delete or format any *Windows partitions* of type "*NTFS*" varying in size. You will lose your personal data and operating system files on Windows.

## Disk Preparation

### Get Partition Layout

​A *Hard Disk Drive* [(HDD)](https://en.wikipedia.org/wiki/Hard_disk_drive) is the device inside your computer which stores all your data; including the system files, applications, and personal data. All computers have them and each one contains a unique *Disk Layout*. The disk layout includes different [Partitions](https://simple.wikipedia.org/wiki/Disk_partition), which are sections of the HDD treated as a separate unit of the OS. On the other hand, [Volumes](https://en.wikipedia.org/wiki/Volume_(computing)) are storage areas for a single filesystem, typically stored within the partitions. The terms *Volume* and *Partition* are often used interchangeably but [Are Not](https://blog.purestorage.com/purely-educational/partition-vs-volume-whats-the-difference/) exactly the same. 
> **Note:** For the sake of this guide, the terms "Drive", "Hard Disk/HDD", "NVME SSD", and "SATA HDD", all mean the same thing. All of them are simply physical data storage devices you could hold in your hand located in the PC.

To get your current *Disk Layout*, you'll need to run the `lsblk` command:

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

- The *nvme0n1* label is an example of a [NVME SSD](https://en.wikipedia.org/wiki/Solid-state_drive) hard disk. Most modern PC's will have these. 

- The *sda* label is an example of a legacy [SATA HDD](https://drivesaversdatarecovery.com/everything-you-need-to-know-about-sata-hard-drives/) hard disk. These are found in older computers.

- The *nvme0n1"p1"* is a partition number telling us this is the first partition inside the NVME disk.

- The *sda"3"* is also a partition number telling us this is the third partition inside the SDA disk.

- At the top, the *NAME*, *SIZE*, and *TYPE* categories are self explanatory, but the *mount-points* category will be covered later in the installation. 

### Get Filesystem Layout

Every computer contains partitions on its disk/disk(s), and each of these partitions will contain [Filesystems](https://wiki.archlinux.org/title/File_systems). A *filesystem* is a method used by operating systems to store, organize, and manage data on its hard drives. When a hard drive is divided into partitions, each partition can be formatted with a specific filesystem. 

To get accurate information on the Filesystem layout, run the `lsblk -f` command: 

```rs
lsblk -f
```

This will output something similar to this: 

```rs
NAME        FSTYPE LABEL      UUID                                 MOUNTPOINTS
nvme0n1
├─nvme0n1p1 vfat   SYSTEM     A1B2-C3D4                            /efi
├─nvme0n1p2
├─nvme0n1p3 ntfs   Windows    1234567890ABCDEF
├─nvme0n1p4 ntfs   Recovery   FEDCBA0987654321
└─nvme0n1p5 ext4   ArchRoot   1122aabb-3344-cc55-dd66-778899aabbcc /
sda
├─sda1      vfat   ESP        B2C3-D4E5
├─sda2
├─sda3      ntfs   Data       23456789ABCDEF01
├─sda4      ntfs   WinRE      CDEF0123456789AB
└─sda5
```

- The **FSTYPE** category tells us what type of filesystem the partition uses.

- The **LABEL** category tells us what that particular filesystem means to the OS.

- The **UUID** *(Universally Unique Identifier)* is the assigned label of each filesystem on a partition by the system. 

Each filesystem serves a unique purpose in the OS. We'll cover them more in depth when we format the partitions.

- [FAT32/vfat](https://en.wikipedia.org/wiki/File_Allocation_Table): The *(File Allocation Table 32)* is an ancient filesystem developed in 1977 and is compatible with both Windows and Linux. FAT32 aligns with the *UEFI* specification. This means you can have one EFI system partition formatted as FAT32 storing both Linux and Windows boot files. 

- [NTFS](https://en.wikipedia.org/wiki/NTFS) (New Technology File System): Also an old filesystem which serves as the primary filesystem for Windows OS's, but does also work with Linux and macOS. *(Though other filesystems are preferred here in Linux)*

### Create Linux Filesystem Partition

[UEFI](https://en.wikipedia.org/wiki/UEFI) *(Unified Extensible Firmware Interface)* and [BIOS](https://en.wikipedia.org/wiki/BIOS) *(Basic Input/Output System)* are two primary types of firmware interfaces that your computer uses to initialize hardware and boot into your operating system. BIOS is an older, legacy standard typically found in earlier hardware. UEFI is the modern replacement, commonly used in newer hardware. Each firmware interface corresponds to a specific partitioning scheme; BIOS systems usually work with the older MBR *(Master Boot Record)* format, while UEFI systems utilize the modern GPT *(GUID Partition Table)* format.

Since you are using dual-booting Arch Linux with Windows 11, we will use the modern [gdisk](https://wiki.archlinux.org/title/GPT_fdisk) *(GPT fdisk)* tool instead of the legacy *fdisk*. Launch the `gdisk` tool:
> **Note:** Type `q` if you ever want to abort the tool without applying changes.

```rs
gdisk /dev/nvmeXnX
```

Once inside gdisk, type `n` to create a new **Linux Filesystem** partition, and press `Enter` to give it the default partition number:

```rs
[gdisk]# n
Press Enter for default
```

Press `Enter` for the default first sector available for the partition on the hard drive:

```rs
Press Enter for default
```

Next `gdisk` will ask you how much space you want to give this partition. Previously in the *Pre-Installation* you freed up space on your windows volume + space for an extra EFI partition. In this case the guide said **500GiB** for the main partition. *`+500G`*

```rs
[gdisk]# +<SIZE>G
```

Finally, specify the type you want the partition to be with a `GUID/Hex` code. Press `Enter` to use the default option of *Linux Filesystem*. 

```rs
Press Enter for default
```

### Create EFI Partition

The [ESP](https://wiki.archlinux.org/title/EFI_system_partition) *(EFI System Partition)* is a special partition required by modern computers that use *UEFI Firmware* with [GPT](https://en.wikipedia.org/wiki/GUID_Partition_Table) partition tables. Older computers use *BIOS* with [MBR](https://en.wikipedia.org/wiki/Master_boot_record) partition tables. Since we're dual-booting Arch Linux with Windows 11, the system already has an ESP containing the [Windows Boot Manager](https://en.wikipedia.org/wiki/Windows_Boot_Manager). In this guide, we will create a second ESP dedicated to Linux Distributions. The 2 major reasons for this: 

1. The Windows Boot Manager is located in the default *100MiB* ESP created by windows. This is not enough space for a second GRUB bootloader for linux.
2. When Windows is updated or reinstalled, theres a high probability Windows will detect the foreign GRUB bootloader in it's same partition and delete it. This will completely prevent you from booting into Linux, which is not good. 

After lots of research, its apparent that creating a separate EFI partition for bootloaders other than Windows Boot Manager is the best way to prevent the issues above.

Like before, type `n` to create a new partition and press `Enter` to give it the default partition number. 

```rs
[gdisk]# n
Press Enter for default
```

Press `Enter` for the default first sector available for the partition on the hard drive:

```rs
Press Enter for default
```

Previously in the *Pre-Installation* you freed up 500MiB on Windows 11 for an extra EFI partition. Now we will use it:

```rs
[gdisk]# +500M
```

Use the `ef00` code to make the created partition an ESP:

```rs
[gdisk]# ef00
```

Write the changes to your disk:

```rs
[gdisk]# w
[gdisk]# y
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

[Btrfs](https://en.wikipedia.org/wiki/Btrfs) *(Better Filesystem)* is a modern filesystem with advanced features like copy-on-write, snapshots, and built-in support for [RAID](https://en.wikipedia.org/wiki/RAID) *(multiple storage devices as one)*. One of its most useful features is [Subvolumes](https://btrfs.readthedocs.io/en/latest/Subvolumes.html), which are independent filesystems within a single Btrfs partition; each serving a different purpose. Think of Subvolumes as partitions for partitions. You can separate your system files, personal files, logs, temporary files, into different Subvolumes. This makes it extremely easy to manage backups, rollbacks, system organization, or OS reinstallations without losing data or needing more partitions.

### Mount Root Temporarily

In order to create the **Btrfs Subvolumes**, we first temporarily mount the root partition to `/mnt` *(common mount point)*. We mount here since the subvolumes will exist at the *(Root)* of the whole filesystem. 

```rs
mount /dev/nvmeXnXpX /mnt
```

### Creating The Subvolumes

Next we create the Btrfs Subvolumes; each of which will be mounted in the coming section. 

The [Root](https://en.wikipedia.org/wiki/Root_directory) `/` directory is the top level of the Linux filesystem hierarchy. In Arch Linux, all partitions, subvolumes, and filesystems are mounted somewhere under "/", making them part of a single unified directory structure. The subvolume name is `@`:

```rs
btrfs subvolume create /mnt/@
```

The [Home](https://en.wikipedia.org/wiki/Home_directory#Unix) `/home` directory contains the user's personal files such as settings, downloads, documents, configurations, etc. The subvolume name is `@home`:

```rs
btrfs subvolume create /mnt/@home
```

The [Logs](https://www.loggly.com/ultimate-guide/linux-logging-basics/) `var/logs` directory is used to isolate system debug output from other parts of */var*. The subvolume name is `@log`:

```rs
btrfs subvolume create /mnt/@log
```

The [Cache](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch05s05.html) `/var/cache` directory stores temporary downloaded package files (like pacman cache). The subvolume name is `@cache`:

```rs
btrfs subvolume create /mnt/@cache
```

The [Temporaries](https://en.wikipedia.org/wiki/Temporary_folder) `/tmp` directory is used for deposable files that don’t need to be backed up. The subvolume name is `@tmp`:

```rs
btrfs subvolume create /mnt/@tmp
```

The [Swap](https://en.wikipedia.org/wiki/Memory_paging) `/swap` directory is used for storing the swapfile, isolated to avoid copy-on-write issues. The subvolume name is `@swap`:

```rs
btrfs subvolume create /mnt/@swap
```

### Unmount Root

Once the subvolumes are created, unmount the root partition so we can remount each subvolume individually later with proper options:

```rs
umount -Rlf /mnt
```

---

## Mounting

After creating our subvolumes, we need to [Mount](https://en.wikipedia.org/wiki/Mount_(computing)) them in the locations we want them to be in when we boot. *Mounting* means attaching a storage location (e.g., a disk, partition or subvolume) to a specific directory. This makes the mounted files and directories available to the user via the subvolume's filesystem. Each subvolume gets mounted to its own mount point (e.g. `/mnt`, `/mnt/home`, etc.). This allows us to control how each part of the system behaves. For instance we can enable *compression*, reduce disk writes with *noatime*, or disable *copy-on-write* for swap.

### Mount Btrfs Subvolumes

Mount the root subvolume ***"@"*** to `/mnt` with compression, SSD optimizations, and reduced write frequency.

```rs
mount -o subvol=@,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt
```

Create the *Mount Points* (directories) for each subvolume so we can mount them there:

```rs
mkdir -p /mnt/{home,tmp,swap,efi}
```

Mount home subvolume ***"@home"*** to `mnt/home` for user data like documents and config files in: 

```rs
mount -o subvol=@home,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt/home
```

Mount temporaries subvolume ***"@tmp"*** to `/mnt/tmp` for unimportant temporary files; it’s reset often:

```rs
mount -o subvol=@tmp,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt/tmp
```

Create subdirectories inside `mnt/var` for log files and cached data:

```rs
mkdir -p /mnt/var/{log,cache}
```

Mount log subvolume ***"@log"*** to `/mnt/var/log` for system logs and debugging:

```rs
mount -o subvol=@log,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt/var/log
```

Mounts cache subvolume ***"@cache"*** to `/mnt/var/cache`, used by pacman and other tools to store downloaded packages:

```rs
mount -o subvol=@cache,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt/var/cache
```

### Setup SWAP (Optional)

The [SWAP](https://wiki.archlinux.org/title/Swap) acts as *virtual memory* stored on your hard disk, providing additional space when your physical RAM is fully utilized. This is especially useful if your computer has limited RAM installed or when enabling power-saving [Hibernation](https://knowledgebase.frame.work/hibernation-on-linux-BkL1N5ffJg).

If you **DO** plan to use Hibernation:
- Swap size must be at least equal to your RAM (e.g. 16 GiB RAM → 16 GiB swap).

If you **DO NOT** plan to use Hibernation:
- 8–16 GiB RAM: Use 2–4 GiB swap.
- 32 GiB or more: Use 1–2 GiB, or skip it entirely if you're sure.

Mount swap subvolume ***"@swap"*** to `/mnt/swap` with copy-on-write disabled (nodatacow) to safely hold a swap file:

```rs
mount -o subvol=@swap,nodatacow,ssd,discard=async,space_cache=v2 /dev/nvmeXnXpX /mnt/swap
```

Disable Btrfs's *copy-on-write* feature for the `/mnt/swap` directory. This is required for swap files to work correctly on Btrfs.

```rs
chattr +C /mnt/swap
```

| Swap Size | dd Count Value (MB) | Explanation         |
|-----------|---------------------|---------------------|
| 1 GiB     | 1024                | 1 GiB = 1024 MB     |
| 2 GiB     | 2048                | 2 × 1024 = 2048 MB  |
| 4 GiB     | 4096                | 4 × 1024 = 4096 MB  |
| 8 GiB     | 8192                | 8 × 1024 = 8192 MB  |
| 16 GiB    | 16384               | 16 × 1024 = 16384 MB|
| 32 GiB    | 32768               | 32 × 1024 = 32768 MB|
| 64 GiB    | 65536               | 64 × 1024 = 65536 MB|

To create the swap file, use the chart above to calculate the `SIZE` you want to allocate for the file. Put that number in the `dd` command below:

```rs
dd if=/dev/zero of=/mnt/swap/swapfile bs=1M count=<SIZE> status=progress
```

Sets permissions so only the root user can read and write to the swapfile. This keeps it secure from other users.

```rs
chmod 600 /mnt/swap/swapfile
```

Formats the file to be used as swap space. It marks the file as usable memory overflow storage.

```rs
mkswap /mnt/swap/swapfile
```

Activates the swapfile so your system can start using it immediately. You’ll now have extra *“RAM”* available from disk.

```rs
swapon /mnt/swap/swapfile
```

Ensure the swap file is activated: 

```rs
swapon --show
```

### Mount EFI System Partition

The **ESP** *(EFI System Partition)* is where bootloaders live. We created it earlier and it should be formatted as *FAT32*, around *500 MiB* in size. Mount it to `/mnt/efi`:

```rs
mount /dev/nvmeXnXpX /mnt/efi
```

## Base Arch Linux System

[Pacstrap](https://wiki.archlinux.org/title/Pacstrap) is a tool used in the Arch Linux Install Environment to copy essential packages onto your new system (usually at `/mnt`). It installs the base system, which includes core utilities, and other packages you specify such as *firmware*, *microcode*, *bootloaders*, *development utilities*, etc. This forms the minimal working Arch system you'll boot into.

```rs
pacstrap /mnt base linux linux-firmware efibootmgr os-prober grub btrfs-progs grub-btrfs timeshift base-devel 
git openssh man sudo nano networkmanager pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber 
reflector inotify-tools
```

> **Note:** I prefer [nvim](https://wiki.archlinux.org/title/Neovim) (*NeoVim*) over Nano for text editing, but this guide will use Nano since its more beginner-friendly for Windows users. 

### 📦 General Arch Linux Packages

- **base**  
  The essential package group required to boot and operate Arch Linux. It includes the core userland utilities like `bash`, `coreutils`, `file`, `fsck`, etc.

- **linux**  
  The Arch Linux default kernel. This is the core of your operating system that manages hardware and system processes.

- **linux-firmware**  
  Contains firmware for various devices (CPUs, GPUs, Wi-Fi, Bluetooth, etc.). Necessary for full hardware support.

- **efibootmgr**  
  A tool to manage UEFI boot entries. Needed when installing GRUB on systems using UEFI.

- **os-prober**  
  Detects other installed operating systems (like Windows) during GRUB installation. Essential for dual-boot setups.

- **grub**  
  The GRUB bootloader lets you choose between multiple operating systems or kernels at boot.

- **btrfs-progs**  
  Tools for managing Btrfs filesystems. Includes utilities like `mkfs.btrfs`, `btrfs balance`, and snapshotting tools.

- **grub-btrfs**  
  A script that integrates Btrfs snapshots with GRUB so you can boot into Timeshift snapshots directly from the GRUB menu.

- **timeshift**  
  A system restore utility using Btrfs or rsync. Great for taking system snapshots before upgrades or risky changes.

- **git**  
  A version control system. Essential for downloading dotfiles, cloning repos, and interacting with the AUR.

- **openssh**  
  Provides the `ssh` command and SSH server/client tools. Useful for remote access, Git over SSH, etc.

- **man**  
  The manual page system. `man <command>` gives you documentation for commands and tools on your system.

- **sudo**  
  Allows regular users to run commands as root by prefixing them with `sudo`. Important for secure system administration.

- **networkmanager**  
  Manages network connections automatically. Great for desktops/laptops—works with both Wi-Fi and Ethernet.

- **pipewire**  
  A modern audio and video server for Linux. Replaces PulseAudio and JACK with better performance and compatibility.

- **pipewire-alsa**  
  ALSA plugin to route audio through PipeWire, enabling compatibility with ALSA-based applications.

- **pipewire-pulse**  
  PulseAudio compatibility layer for PipeWire. Allows apps expecting PulseAudio to work seamlessly with PipeWire.

- **pipewire-jack**  
  JACK compatibility layer for PipeWire. Lets professional audio applications that use JACK work with PipeWire.

- **wireplumber**  
  A session manager for PipeWire that handles audio/video routing and device policy. Required for PipeWire to function fully.

- **reflector**  
  Automatically fetches the latest and fastest Arch mirror list based on location and speed. Useful for speeding up package downloads.

### Microcode Firmware

[Microcode](https://wiki.archlinux.org/title/Microcode) is low-level firmware for your **CPU** that addresses **security vulnerabilities**, **stability issues**, and **performance bugs**. These updates are loaded **early during boot**, allowing Linux to apply critical fixes **even if your motherboard BIOS isn’t up to date**.

On Arch Linux, simply install the appropriate microcode package based on your CPU manufacturer:

For Intel CPUs:

```rs
pacstrap /mnt intel-ucode
```

For AMD CPUs:

```rs
pacstrap /mnt amd-ucode
```

> **Note:** These packages will generate an additional boot initrd (like `intel-ucode.img`) that gets loaded before the Linux kernel by your bootloader (e.g., GRUB).

---

### Video Drivers

Your **GPU (Graphics Processing Unit)** needs dedicated drivers for 2D/3D acceleration, Vulkan support, and hardware video decoding. Arch Linux provides vendor-supported drivers for [AMD GPU](https://wiki.archlinux.org/title/AMDGPU), [NVIDIA](https://wiki.archlinux.org/title/NVIDIA), and [Intel Graphics](https://wiki.archlinux.org/title/Intel_graphics). We install these video drivers with pacstrap so we don't have to later.

For AMD GPUs:

```rs
pacstrap /mnt mesa mesa-vdpau vulkan-radeon libva-mesa-driver
```

- `mesa`: Core open-source 3D graphics library
- `mesa-vdpau`: Enables hardware video decoding (VDPAU)
- `vulkan-radeon`: Vulkan driver for AMD GPUs (RADV)
- `libva-mesa-driver`: VA-API support for video acceleration

---

For NVIDIA GPUs (Proprietary):

```rs
pacstrap /mnt nvidia nvidia-utils nvidia-settings egl-wayland
```

- `nvidia`: Proprietary NVIDIA kernel module (GPU driver)
- `nvidia-utils`: User-space utilities and libraries (OpenGL, CUDA, Vulkan)
- `nvidia-settings`: GUI tool for fan control, resolution, etc.
- `egl-wayland`: Enables hardware acceleration for Wayland compositors (e.g., GNOME, Hyprland)

---

For Intel Integrated Graphics:

```rs
pacstrap /mnt mesa xf86-video-intel vulkan-intel lib32-mesa lib32-vulkan-intel
```

- `mesa`: Core graphics library (OpenGL, GLES)
- `xf86-video-intel`: X11 driver (optional; some users prefer to omit it)
- `vulkan-intel`: Vulkan driver for Intel GPUs
- `lib32-*`: 32-bit support (useful for compatibility with games or Wine)

---

### Filesystem Table (fstab)

[fstab](https://wiki.archlinux.org/title/Fstab) *(Filesystem Table)* located in `/etc/fstab` is a config file that tells Linux which partitions, subvolumes, or drives to mount automatically at boot, where to mount them, and with what options. Each line in the file represents a storage device and its mount configuration. The `genfstab` command scans your currently mounted filesystems (like `/mnt`, `/mnt/home`, etc.) and generates the appropriate entries for fstab, using stable identifiers like *UUIDs* so mounts stay consistent even if device names change.

```rs
genfstab -U /mnt >> /mnt/etc/fstab
```

Append the swap file entry:

```rs
echo '/swap/swapfile none swap defaults 0 0' >> /mnt/etc/fstab
```

> **Note:** The `-U` flag ensures it uses *UUIDs* (unique disk identifiers), which are more stable than device names like `/dev/nvmeXnX`.

### Change Root

In Linux, the ***"Root Environment"*** refers to the top-level filesystems the entire OS operates in. During installation, you're working from the arch install environment *Live Arch ISO*; so the root is still in a temporary location. To configure your newly installed system as if you had booted into it, you *Change Root* or [Chroot](https://wiki.archlinux.org/title/Chroot) this root location to the place you mounted your partitions. To do this, use `arch-chroot /mnt`, which will switch the root directory (`/`) to your new system at `/mnt`. This lets you run commands like setting the *timezone*, *locale*, *bootloader* etc. inside your new Arch setup. When you reboot, you will boot to this new *"root"* location every time.

```rs
arch-chroot /mnt
```

### GRUB Installation & Config

[​GRUB](https://wiki.archlinux.org/title/GRUB), *(GNU GRand Unified Bootloader)*, is a powerful boot manager widely used in Unix-like systems. It initializes hardware components and loads the operating system kernel during the boot process, allowing users to select from multiple installed operating systems or kernel configurations. This is useful for multi-boot setups like this guide, as it enables seamless switching between OS's; For instance, Windows 11 & Arch Linux.

Install GRUB for UEFI systems, specifying the target architecture, EFI system partition, and assigning the bootloader identifier: `GRUB`:

```rs
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
```

[os-prober](https://wiki.archlinux.org/title/GRUB#Detecting_other_operating_systems) is a utility that detects other operating systems installed on your machine by scanning disk partitions. When integrated with GRUB, it allows the bootloader to include these detected systems in its boot menu, for selection at startup. In some distributions, os-prober is disabled by default. To enable it, you may need to set `GRUB_DISABLE_OS_PROBER=false` in your GRUB configuration. To enable it: 

```rs
sed -i '/^GRUB_DISABLE_OS_PROBER=/d' /etc/default/grub && 
echo 'GRUB_DISABLE_OS_PROBER=false' >> /etc/default/grub
```

Create a temporary directory for the Windows ESP:

```rs
mkdir -p /mnt/windows_esp
```

Temporarily mount the Windows ESP so `os-prober` can detect it:

***MAKE SURE YOU MOUNT THE (WINDOWS) EFI PARTITION, NOT THE ONE WE CREATED IN THIS GUIDE***

```rs
mount /dev/nvmeXnXpX /mnt/windows_esp
```

The *GRUB Configuration File*, typically located at `/boot/grub/grub.cfg`, is crucial for defining the bootloader's behavior, including available operating systems, default selection, and timeout settings. Properly configuring this file ensures that all desired operating systems are accessible from the GRUB menu, providing a smooth multi-boot experience. To make the configuration, use `grub-mkconfig`:

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

> **Note:** The Windows Boot Manager needs to be found so we can boot into Windows 11 with GRUB. If it is not detected, dual boot will not work. Make sure you followed previous steps!

## Essential Configurations Before Reboot

These last configurations set essential **System Settings** required for your Arch Linux system to function properly after rebooting. They ensure the correct system *timezone*, *locale* (language settings), *network connectivity*, *hostname*, and *user security* settings are configured.

### Set the Timezone

The [System Time](https://wiki.archlinux.org/title/System_time) controls the clock synced with your local region's timezone. To get a timezones list:

```rs
timedatectl list-timezones
```

Set the timezone. For example, United States Eastern time is: *"America/New_York"*

```rs
timedatectl set-timezone <Your/Timezone>
```

```rs
hwclock --systohc
```

### Configure Locale

Your [Locale](https://wiki.archlinux.org/title/Locale) is the language of the region your in and is not set by default. 

Open `/etc/locale.gen` with the *nano* text editor:
> **Note:** Mouse & Touchpad DO NOT work in the Arch TTY! Use the *Arrow Keys* instead. Press *Ctrl + O* to save changes, and *Ctrl + X* to exit nano.

```rs
nano /etc/locale.gen
```

In the file, uncomment the locales you wish to generate by removing the `#` at the beginning of the respective lines.

For example, to enable *US English UTF-8*:

```rs
en_US.UTF-8 UTF-8
```

Similarly, for *British English UTF-8*:

```rs
en_GB.UTF-8 UTF-8
```

***UNCOMMENT THE LOCALE FOR YOUR REGION AS NEEDED***

Generates the locale data based on your selections.

```rs
locale-gen
```

### Set Hostname

The [Hostname](https://en.wikipedia.org/wiki/Hostname) is your computer's name on the network, which helps identify your machine on local networks.

```rs
echo <HOSTNAME> > /etc/hostname
```

### Enable NetworkManager

The [NetworkManager](https://wiki.archlinux.org/title/NetworkManager) package automatically manages your network connections, ensuring internet connectivity on reboot.

```rs
systemctl enable NetworkManager
```

### Set Root Password

Establishes a password for the root user, **CRITICAL** for system security and for logging back in.

```rs
passwd
```

> **Note:** You will be prompted to enter your new *Root Password*.

### Create a Regular User

Creating a regular (non-root) user for everyday use enhances security by reducing root privileges.

```rs
useradd -m -G wheel <USERNAME>
```

Set a password for this new user:

```rs
passwd <USERNAME>
```

> **Note:** You will be prompted to enter your new *User Password*.

### Enable `sudo` for Regular User

The [Sudo](https://wiki.archlinux.org/title/Sudo) tool allows the regular user to perform administrative tasks securely when given permission.

To set a user as `sudo`:

```rs
echo "%wheel ALL=(ALL) ALL" >> /etc/sudoers
```

### Check FSTAB

The [fstab section](#generate-filesystem-table-fstab) from earlier will cover the generated filesystem table at `/etc/fstab`. Ensure the boot entries for partitions and subvolumes are correctly set to mount in the right places.

```rs
cat /etc/fstab
```

Should look something like this: 

```rs
# /etc/fstab: static file system information.
# <file system>                         <mount point>  <type>   <options>                                                                                      <dump>  <pass>

# Btrfs root subvolume
# /etc/fstab: static file system information.
# <file system>                                 <mount point>  <type>   <options>                                                                                      <dump>  <pass>

# Btrfs root subvolume
UUID=550e8400-e29b-41d4-a716-446655440000       /             btrfs    subvol=@,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2                               0       0

# Home subvolume
UUID=550e8400-e29b-41d4-a716-446655440000       /home         btrfs    subvol=@home,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2                           0       0

# Log subvolume
UUID=550e8400-e29b-41d4-a716-446655440000       /var/log      btrfs    subvol=@log,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2                            0       0

# Cache subvolume
UUID=550e8400-e29b-41d4-a716-446655440000       /var/cache    btrfs    subvol=@cache,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2                          0       0

# Temporary files subvolume
UUID=550e8400-e29b-41d4-a716-446655440000       /tmp          btrfs    subvol=@tmp,compress=zstd:3,noatime,ssd,discard=async,space_cache=v2                            0       0

# Swap subvolume
UUID=550e8400-e29b-41d4-a716-446655440000       /swap         btrfs    subvol=@swap,nodatacow,ssd,discard=async,space_cache=v2                                         0       0

# EFI System Partition
UUID=123e4567-e89b-12d3-a456-426614174000       /efi          vfat     umask=0077                                                                                     0       2
```

- **UUIDs:** The UUIDs will be different for you, this is normal. 
- **Swap:** The swap subvolume may or may not exist for you, depending on if you chose to create one or not.

***EVERYTHING ELSE IN THIS EXAMPLE OUTPUT SHOULD BE THE SAME FOR YOU, OTHERWISE YOU HAVE A PROBLEM***

### Reboot Commands

Exit from chroot back into the arch install environment:

```rs
exit
```

Disable the swap file:

```rs
swapoff /mnt/swap/swapfile
```

Force all filesystems to unmount: 

```rs
umount -Rlf /mnt
```

Reboot the system, you will be greeted with the GRUB menu. If you did everything right, windows 11 and arch linux will appear in the boot options. 
> **Note:** While the computer is rebooting, unplug the installation media USB

```rs
reboot
```

### Btrfs Snapshots

Btrfs [Snapshots](https://wiki.archlinux.org/title/Btrfs#Snapshots) are like lightweight *Restore Points* that capture the current state of a subvolume. Snapshots do this without hard copying data from the subvolume, which takes up very little space on disk; done via [Copy-On-Write](https://en.wikipedia.org/wiki/Copy-on-write) *(COW)* feature. We will setup Btrfs snapshots to work directly with GRUB so we can boot into them without needing the *Arch Linux ISO Environment*.

Start the `grub-btrfsd` daemon to update GRUB when snapshots are created or deleted:

```rs
sudo systemctl start grub-btrfsd
```

Activate grub-btrfsd on system startup: 

```rs
sudo systemctl enable grub-btrfsd
```

Make grub-btrfs work with `timeshift` correctly:

```rs
sudo systemctl edit --full grub-btrfsd
```

Go to the line that contains: 

```rs
ExecStart=/usr/bin/grub-btrfsd --syslog
```

Edit that line to look like this: 

```rs
ExecStart=/usr/bin/grub-btrfsd --syslog --timeshift-auto
```

The modified file should look like this:

```rs
[Unit]
Description=Regenerate grub-btrfs.cfg

[Service]
Type=simple
LogLevelMax=notice
# Set the possible paths for `grub-mkconfig`
Environment="PATH=/sbin:/bin:/usr/sbin:/usr/bin"
# Load environment variables from the configuration
EnvironmentFile=/etc/default/grub-btrfs/config
# Start the daemon, usage of it is:
# grub-btrfsd [-h, --help] [-t, --timeshift-auto] [-l, --log-file LOG_FILE] SNAPSHOTS_DIRS
# SNAPSHOTS_DIRS         Snapshot directories to watch, without effect when --timeshift-auto
# Optional arguments:
# -t, --timeshift-auto  Automatically detect Timeshifts snapshot directory
# -l, --log-file        Specify a logfile to write to
# -v, --verbose         Let the log of the daemon be more verbose
# -s, --syslog          Write to syslog
ExecStart=/usr/bin/grub-btrfsd --syslog --timeshift-auto

[Install]
WantedBy=multi-user.target
```

Restart the grub-btrfsd daemon: 

```rs
sudo systemctl restart grub-btrfsd 
```

Here, we will create a snapshot of the root subvolume (`@`). Root stores the exact current state of the Arch Linux Base System (Root Directory `/`). This will backup system configs, installed packages, root files, etc. Later, if something breaks the OS such as an unstable update or misconfiguration, we simply *Roll-Back* to the snapshot.
> **Note:** We do not snapshot the other subvolumes, because we do not want to roll-back our personal files, system logs, etc to a previous point; because that would lose our data. 

```rs
sudo timeshift --create --comments "archlinux-base-install"
```

After creating a snapshot, update the GRUB config and ensure it detects the snapshot. 

```rs
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

## Finally Done 🎉

**Congratulations!!! You have finally booted into Arch Linux!!** 🎆

So after the reboot your back in Arch Linux and just see a black TTY with a login prompt and no internet connection? Yeah, that's because Arch Linux really does give you the freedom to set ***Everything*** up from scratch just the way you want it. In the [Post-Installation](post_installation.md), we will make it look like an actual operating system.
