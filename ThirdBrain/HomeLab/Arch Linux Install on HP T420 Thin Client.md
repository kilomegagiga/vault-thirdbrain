
# Before Starting the Install

## Upgrade harddrive storage:
RAM is soldered on, so fixed at 2GB.  However, the internal flash drive can be successfully increased to at least 128GB.  See: [[Initial Upgrades for HP T420 Thin Client]]

## Update BIOS Settings
The HP T420 is capable of working with some sort of network booting system, which is likely of little use in a small homelab situation.  The following BIOS Settings will prevent attempts to network boot.  Also, disabling "Legacy Support" will prevent attempts to boot using "Legacy" MBR, allowing only the use of UEFI instead.  Since UEFI is used in the scenario described here, booting using MBR can prevent some steps from working properly.
```
Advanced>Device Options:
NIC PXE Option ROM Download: Disable

Security>Network Boot:
Network Boot: Disable

Storage>Storage Options:
Legacy Support: Disable
```

## Create the USB to Use for Installations
Download and appropriately verify the validity of an archlinux-x86_64.iso file from one of the mirror sites at the bottom of the [Arch Linux Downloads](https://archlinux.org/download/) page.  Then, [create the USB for installing Arch Linux](https://wiki.archlinux.org/title/USB_flash_installation_medium) using one of the various methods described on the linked page.  Section 1.1.1 "Using basic command line utilities in Linux" was successful in the scenario described, since a separate Linux machine was already available.

# Arch Linux Install

The steps followed below are almost entirely from the [Installation Guide at archlinux.org](https://wiki.archlinux.org/title/Installation_guide). However, by necessity, the Installation Guide allows for a wide variety of installation paths.  These are the notes that were sufficient to perform identical installs on three separate instances of HP T420 hardware.

## Disk Partitions
In these instructions, /dev/sda is used to indicate the internal flash drive.  However, the internal flash drive can sometimes be /dev/sdb if the installation USB claimed /dev/sda, which happens sometimes, seemingly randomly.  Stay alert and be cautious.

```
fdisk -l
```

```
fdisk /dev/sda
```

The recommended minimum of the UEFI boot partition is 1 GiB. The recommended minimum of the swap partition used to be the greater of double the RAM or 4 GiB, which is 4 GiB either way for the HP T420.  The root partition could contain whatever remains. 

An attempt was made using the previous setup, however /tmp seems to be one of several tmpfs entries using the swap partition, so /tmp ended up with less than 1 GiB of space available and ran out of space during an installation step.  Since the internal flash drive was upgraded to 128 GiB, there is plenty of space for a larger swap partition.

The UEFI boot partition has been fine with 1GiB so far. Swap space is increased to 32 GiB, which may be overkill.  However, the space is available and it allows for options without having reformat everything again.  A single 80 GiB root partition is unnecessary and arguably unwise.  Instead, four separate 20 GiB partitions are created, which again should allow some flexibility for experimentation in usage, if required, such as the possibility of A/B system updates, despite that being thoroughly unnecessary due to running continuously updated Arch Linux. 

Disk usage of /root on the first attempt was less than 10 GiB, so 20 GiB partitions do not seem unreasonable.  One partition is for /root and a one partition is for /home.  The other two can be left blank, although one has been set up as /archive, which is to be used as an overflow space in case /home fills up.  If /home is having no issues, then /archive should remain empty.  Some space on the internal flash drive is left unused, however it is less than 3 GiB. 

boot, swap, root, blank, archive, home: 1+32+20+20+20+20

```
fdisk> p
fdisk> d
fdisk> g
fdisk> l
# aliases: uefi, swap, linux
fdisk> n, default, default, +1G
fdisk> n, default, default, +32G
fdisk> n, default, default, +20G
fdisk> n, default, default, +20G
fdisk> n, default, default, +20G
fdisk> n, default, default, +20G
fdisk> t, 1, uefi
fdisk> t, 2, swap
fdisk> t, 3, linux
fdisk> t, 4, linux
fdisk> t, 5, linux
fdisk> t, 6, linux
fdisk> w
```

## File System Creation and Initial Mounts

```
# mkfs.fat -F 32 /dev/sda1
# mkswap /dev/sda2
# mkfs.ext4 /dev/sda3
# mkfs.ext4 /dev/sda4
# mkfs.ext4 /dev/sda5
# mkfs.ext4 /dev/sda6
```

```
# mount /dev/sda3 /mnt
# mount --mkdir /dev/sda1 /mnt/boot
# swapon /dev/sda2
# mount --mkdir /dev/sda6 /mnt/home
# mount --mkdir /dev/sda5 /mnt/archive
```

## Arch Linux Mirror List

There was advice to adjust the mirrorlist entries for better performance.  The provided entries seem to have been sufficient but best to mention it.

```
# vim /etc/pacman.d/mirrorlist
```

## Packages to include in initial install
When attempting to the pacstrap command minimal, "base linux linux-firmware" was a good start.  However, investigation of boot issues required installation of "grub efibootmgr amd-ucode".  Once booting was successful, further installations via pacman could not happen without "networkmanager dhcpcd".  Setting up sudo requires a reboot if it is not already installed, so including "sudo vi less" avoids that inconvenience.  
I have found the following to be a viable minimal Arch Linux install so far.

```
# pacstrap -K /mnt base linux linux-firmware
```

```
# pacstrap -K /mnt base linux linux-firmware grub efibootmgr amd-ucode
```

```
# pacstrap -K /mnt base linux linux-firmware grub efibootmgr amd-ucode networkmanager dhcpcd sudo vi less
```
## Configure the system
The next several steps are straight from the Arch Linux Installation Guide, so are included without further comment.  Careful if not in Midwestern United States.

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

```
# arch-chroot /mnt
```

```
# ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime
```

```
# hwclock --systohc
```

```
# locale-gen
```

```
# cat > /etc/locale.conf
LANG=en_US.UTF-8
```

```
# localectl list-keymaps
# cat > /etc/vconsole.conf
KEYMAP=us
```

```
# cat > /etc/hostname
thin
```

[[Possible Hostnames]]

```
# passwd
```

No list of possible passwords is provided.  Best of luck to you.

## Configure Boot Loader
An initial attempt to use UEFI only without GRUB failed.  GRUB was used as part of the debugging process and, once working, was not worth the attempt to try UEFI on its own again.  Whether UEFI works without GRUB is still not known nor explored.

During the initial failed attempt, a search for advice yielded the following pages:  [Advice 1](https://bbs.archlinux.org/viewtopic.php?id=269667), [Advice 2](https://www.linuxquestions.org/questions/slackware-14/uefi-boot-problem-on-wyse-3040-thin-client-4175734680/), [Advice 3](https://blog.roberthallam.org/2020/05/psa-dell-wyse-3040-uses-fallback-efi-location/)

Recommendation based on the advice:
- Use Grub rather than trying without it. Some possible solutions are accomplished easiest by moving files generated by Grub into better locations. That requires Grub to have created the files first. 
Additional Realizations:
- Was booting into MBR mode instead of UEFI mode.  Writing UEFI information is not possible if in MBR mode.
- Was leaving out "UUID=" for the syntax "root=UUID={uuid}".

"grub efibootmgr amd-ucode" are now included in pacstrap.  If they were not, the following would be needed at this point:
```
# vim /etc/pacman.d/mirrorlist
```

```
# pacman -Syu
# pacman -S grub efibootmgr amd-ucode
```

List some extra information about the available partitions:
```
# lsblk -f
```

Find the UUID of the root directory. The found UUID replaces {uuid} in the following efibootmgr command:
```
# efibootmgr --create --disk /dev/sda --part 1 --label "Arch Linux" --loader /vmlinuz-linux --unicode 'root=UUID={uuid} rw initrd=\initramfs-linux.img'
```

The boot manager files have been created.  The following command places the generated files in their proper locations:
```
# grub-install --target=x86_64-efi --efi-directory=/boot --removable
```

## Increase Size of /tmp Directory
To permanently increase the size of the /tmp directory, modify the fstab file:
```
# vi /etc/fstab
```
Either add the following item or modify an existing similar item:
```
tmpfs /tmp tmpfs rw,nodev,nosuid,size=16G 0 0
```
 
If only a temporary increase in the size of /tmp is needed, the following can be used instead.  However, this change will not survive the next reboot.
```
# mount -o remount,size=16G,noatime /tmp
```

## Reboot
All necessary installations for the minimal Arch Linux system are complete.

Leave the arch-chroot command:
```
# exit
```

Unmount the created filesystem to confirm all changes have been written to the harddrive:
```
# umount -R /mnt
```

Power off instead of just reboot so that the external USB drive can be safely removed.
```
# shutdown -h now
```

Remove external USB drive then power on again. 
Ideally, a login prompt eventually appears without issues.


## Networking
The following will be needed for proper connections to the network.  Internet access is needed for further pacman installations.
```
# cat /etc/resolv.conf
# systemctl enable --now dhcpcd
# systemctl enable --now NetworkManager
# systemctl status dhcpcd
# systemctl status NetworkManager
# cat /etc/resolv.conf
```
Running these commands while still within arch-chroot might be possible, however it has not yet been attempted.  Running them after the reboot is sufficient for now.

# Troubleshooting

## Returning to arch-chroot
If something goes wrong and returning to the arch-chroot state is required,
insert the required external USB drive again, reboot, then confirm that the internal flash drive is /dev/sda by using lsblk:

```
# lsblk
# mount /dev/sda3 /mnt
# mount /dev/sda1 /mnt/boot
# swapon /dev/sda2
# arch-chroot /mnt
```

## KEYMAP=en
On an initial installation attempt, "KEYMAP=en" was placed into /etc/vconsole.conf instead of the intended "KEYMAP=us".  This caused the hyphen ("-") and equals ("=") keys to not type.  Troubleshooting advice included trying the `showkey` command and checking the file evdev.conf. 

Hyphen was available from the numeric keypad.  However, in this scenario, no editors had been installed, so only cat was available.  Without the equal sign, the file could not be updated by overwriting with the correct contents.

The issue was eventually fixed with following by relying on the hyphen on the numeric keypad:
```
sudo localectl set-keymap --no-convert us
```

## Disconnected Ethernet
Ethernet was no longer connected when returning after several days. Found a recommendation to install networkmanager for access to nmtui command. 

```
# pacman -Syu
# pacman -S networkmanager
# pacman -S dhcpcd
```

```
# systemctl start dhcpcd
# systemctl start NetworkManager
# nmtui
```

Necessary networking related systemctl commands have now been added at the end of the above installation instructions.

---
