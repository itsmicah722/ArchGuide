# Arch Linux Install
Personal guide for Arch Linux installation on hardware *(not virtual machines)*, including dual-boot with Windows 11.

## Font

Set a larger font:
```
$ setfont ter-132b
```

## Wi-Fi

If using Ethernet, skip this step.

Start `iwctl`:
```
$ iwctl
```

Check your Wi-Fi device:
```
$ device list
```

Scan and list available networks:
```
$ station wlan0 scan
$ station wlan0 get-networks
```

Connect to Wi-Fi:
```
$ station wlan0 connect "<SSID>" password "<PASSWORD>"
```

Exit `iwctl`:
```
$ exit
```

Confirm connection:
```
$ ip a
$ ping -c 3 cloudflare.com
```

## SSH

Enable SSH (already enabled by default):
```
$ systemctl enable sshd
```

Set a password if using SSH:
```
$ passwd
```

## Partition Info

Check existing partitions:
```
$ lsblk
```

If Windows is installed, an EFI System Partition (ESP) already exists (usually `/dev/nvme0n1p1`). Reuse this partition instead of creating a new one.

Start `gdisk`:
```
$ gdisk /dev/nvme0n1
```

Print the partition table:
```
Command: p
```

## Create partitions:

#### EFI Partition `ef00` (Only if missing)
This partition stores the bootloader and is required for UEFI systems. If you already have one from Windows, do not delete it. If not and one does not already exist, create a new one.
```
Command: n
Partition number: Press Enter
First sector: Press Enter
Last sector: +512M
Hex code: ef00 (EFI partition)
```

#### Swap Partition `8200` (Optional but recommended)
Used when the system runs out of RAM. If you have 16GB of RAM, set this to 16GB (or slightly more if hibernation is required).
```
Command: n
Partition number: Press Enter
First sector: Press Enter
Last sector: Match RAM size (e.g., +16G for 16GB RAM)
Hex code: 8200 (Swap)
```

#### Root Partition `8309` (LUKS Encrypted)
The main system partition where Arch Linux is installed. This will be encrypted for security.
```
Command: n
Partition number: Press Enter
First sector: Press Enter
Last sector: +100G
Hex code: 8309 (Linux LUKS root)
```

#### Home Partition `8309` (LUKS Encrypted)
Stores your personal files, configurations, and user data. Keeping this separate makes future reinstalls easier.
```
Command: n
Partition number: Press Enter
First sector: Press Enter
Last sector: Use remaining space or define a size
Hex code: 8309 (Linux LUKS home)
```

Write the changes:
```
Command: w
Confirm: Y
```

If using a second disk for `/home`:
```
$ gdisk /dev/nvme1n1
```

```
Command: n
Partition number: 1
First sector: Press Enter
Last sector: Use entire disk
Hex code: 8309 (Linux LUKS home)
```

Write the changes:
```
Command: w
Confirm: Y
```

## Encryption (LUKS)

Load encryption modules:
```
$ modprobe dm-crypt
$ modprobe dm-mod
```
#### IMPORTANT NOTE
**MAKE SURE** to change the `/dev/nvme0n1p2` partition labels to the correct ones on your disk. Otherwise these commands may not work and you will encrypt the wrong partitions. 

List the partition labels:
```
$ lsblk
```

Encrypt root partition:
```
$ cryptsetup luksFormat -v -s 512 -h sha512 /dev/nvme0n1p2
```

Encrypt home partition:
```
$ cryptsetup luksFormat -v -s 512 -h sha512 /dev/nvme1n1p1
```

Open encrypted partitions:
```
$ cryptsetup open /dev/nvme0n1p2 luks_root
$ cryptsetup open /dev/nvme1n1p1 luks_home
```

## Volume Setup (LVM)

Once the encrypted partitions have been unlocked, LVM must be set up to manage the storage inside them. First, create a physical volume on the unlocked LUKS partition. This initializes it for use with LVM.
```
$ pvcreate /dev/mapper/luks_root
```

Now, create a volume group to hold logical volumes. The name `arch` is used as an identifier for the group.
```
$ vgcreate arch /dev/mapper/luks_root
```

With the volume group created, logical volumes can now be created within it. These logical volumes will act as individual partitions inside the encrypted container. Create a swap volume sized according to your system's RAM.
```
$ lvcreate -n swap -L <RAM_SIZE>G arch
```

Next, create a root volume. This will store the operating system files. The `-l +60%FREE` option ensures that 60% of the remaining free space in the volume group is used.
```
$ lvcreate -n root -l +60%FREE arch
```

Finally, create a home volume to store user data. The remaining free space is allocated to it.
```
$ lvcreate -n home -l +100%FREE arch
```

Once these volumes are created, they will appear under `/dev/mapper/` as `arch-root`, `arch-home`, and `arch-swap`, and can now be formatted.

## Filesystems

Format partitions before mounting them.
```
$ mkswap /dev/mapper/arch-swap
$ mkfs.btrfs -L root /dev/mapper/arch-root
$ mkfs.btrfs -L home /dev/mapper/arch-home
```

Enable swap:
```
$ swapon /dev/mapper/arch-swap
```

## Mounting

Mount root first:
```
$ mount /dev/mapper/arch-root /mnt
```

Create necessary directories:
```
$ mkdir -p /mnt/{home,boot}
```

Mount home:
```
$ mount /dev/mapper/arch-home /mnt/home
```

Mount boot:
```
$ mount /dev/nvme0n1p2 /mnt/boot
```

Create and mount EFI directory:
```
$ mkdir /mnt/boot/efi
$ mount /dev/nvme0n1p1 /mnt/boot/efi
```

## Install Arch Linux
```
$ pacstrap -K /mnt base linux linux-firmware
```

## Encryption Hooks

Edit mkinitcpio config:
```
$ nvim /etc/mkinitcpio.conf
```

Modify `HOOKS`:
```
HOOKS=(... block encrypt lvm2 filesystems fsck)
```

Regenerate initramfs:
```
$ mkinitcpio -p linux
```

## Bootloader Setup

Install GRUB:
```
$ pacman -S grub efibootmgr
$ grub-install --efi-directory=/boot/efi
```

Modify GRUB config:
```
$ nvim /etc/default/grub
```

Add kernel parameters:
```
root=/dev/mapper/arch-root cryptdevice=UUID=<uuid>:luks_root
```

Generate GRUB config:
```
$ grub-mkconfig -o /boot/grub/grub.cfg
$ grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg
```

## System Configuration (Time, Locale, Users, etc.)

## Reboot
```
$ exit
$ umount -R /mnt
$ reboot now
```
