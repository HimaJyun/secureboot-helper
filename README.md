WARNING: WIP, beta, testing, researching, incomplete documentation, ...and other all.  
contains a risk of break the bootloader.

# secureboot-helper
some scripts for own-keys secure boot, LUKS full disk encryption, TPM auto unlock, over SSH manually unlock.  
BitLocker-like FDE for Linux servers, auto rebootable for auto updates, data protection from theft, anti mischief.

WARNING: 3-letter government agency maybe able unlock your device.  
It is intended to protect against theft and attacks by ordinary citizens.  
may not be possible to protect against forensic technology or equipment.  (eg: cold boot attack, DMA attack)  
Don't keeping sensitive data in plain text on **any** servers.  
these data must be encrypted on a secure local device before being upload to the server, and keeping password to your brain.

# Usage 
## Ubuntu 22.04 Server
LUKS without LVM.

1. `Enther shell` from `Help` menu. (over SSH recommended)
2. clone and enter this git repository.
3. `./fde-partition -l -r -10G /dev/<device>` (`-r -10G` for temporary partition, sizes is depend to additional software)
4. `sgdisk -n "0::" -p /dev/<device> && partprobe` (It is temporary partition [^1])
5. `exit` -> Installing Ubuntu with 'Custom storage layout'
    - EFI for `/boot/efi`
    - 1GB ext4 for `/boot/` (need format, it is not problem)
    - temporary partition for `/`
6. don't reboot, re-enter shell.
7. manually copy from temporary partition to LUKS partition using rsync. [^1]
    1. `mount /dev/mapper/<name> /mnt`
    2. `rsync -aAXH --numeric-ids --info=progress2 --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/boot/*","/lost+found"} /target/ /mnt/`
    3. `umount /target/{boot/efi,boot,/} /mnt`
    4. `mount /dev/mapper/<name> /target/`
    5. `blkid` (copy `/dev/mapper/<name>` UUID)
    6. `nano /target/etc/fstab` (fix root partition UUID)
    7. `dd if=/dev/zero of=/target/swap.img bs=$(stat -c %s /target/swap.img) count=1 conv=sync && mkswap /target/swap.img`
    8. delete temporary partition.
8. `mount /dev/<boot> /target/boot && mount /dev/<EFI> /target/boot/efi && mount -t proc none /target/proc && mount -o bind /sys /target/sys && mount -o bind /dev /target/dev && mount -o bind /dev/pts /target/dev/pts`
9. chroot /target
10. `update-initramfs -u && update-grub`
11. reboot
12. clone and enter this git repository.
13. `sudo ./unified-installer -b "4096" -l symlink -r -x --ms-uefi`  
    if you having error in enroll keys step, use other methods.
    - Use UEFI settings menu.
    - Use efi-updatevar (eg: `sudo efi-updatevar -a -f PK.auth PK`)
    - Use KeyTool.efi
14. `sudo ./dropbear-installer`
15. `sudo ./clevis-installer -s "0,1,2,3,7,8,9,14"`
16. delete boot partition.
    1. `sudo umount /boot{/efi,/}`
    2. `sudo mount /dev/<boot> /mnt`
    3. `sudo rsync -aAXH --numeric-ids --info=progress2 /mnt/ /boot/`
    4. `sudo umount /mnt`
    5. `sudo nano /etc/fstab` (delete boot partition entry)
    6. `sudo gdisk /dev/<device>` (delete boot partition and expand root partition)
    7. `sudo systemctl reboot` (can't use `partprobe` for root expand)
    8. grow root fs (eg: `sudo xfs_growfs /`)

Hint: keep grub. If disable it, automatic updates may re-enable it in a higher boot order. In that case, manual operation is required. (now researching to uninstall grub)

[^1]: It is workaround, current version Ubuntu installer doesn't support installing to LUKS partitions.

## Debian
Installing Debian to crypt partion. Debian installer supports it.  
Partitioning example:
- 500MiB: ESP
- any size: LUKS root
- 1GB: /boot (delete after, recommended to make it as the last partition)

0. `sed -i -e "s/^deb cdrom/#deb cdrom/" /etc/apt/sources.list && apt update`
1. `apt install git systemd-boot-efi curl rsync gdisk`
2. Same as from step 12 on Ubuntu.
3. adjust `/etc/kernel/cmdline` like as `root=/dev/mapper/root ro panic=0`

Hint: Debian is easy to uninstall grub.

```
apt purge grub-common
apt autoremove
rm -rfv /boot/grub /boot/efi/EFI/debian
```

## Proxmox
Install as Debian and convert to Proxmox.  
follow official documentation: https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_12_Bookworm

<!-- Not tried, Might be better than the conversion method.
https://forum.proxmox.com/threads/adding-full-disk-encryption-to-proxmox.137051/
-->

Hint: Proxmox 8.1 needs to install proxmox-ve packages without rebooting.  
if rebooted, it is stuck at "Loading initial ramdisk ...".

<!-- TODO
- rewrite shell's to python
-->
