
# Usage 
## Ubuntu 22.04 Server
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
    4. `sudo nano /etc/fstab` (delete boot partition entry)
    5. `sudo gdisk /dev/<device>` (delete boot partition and expand root partition)
    6. `sudo reboot` (can't use `partprobe` for root expand)
    7. `sudo xfs_growfs /`

[^1]: It is workaround, current version Ubuntu installer doesn't support installing to LUKS partitions.

<!-- TODO
- rewrite shell's to python
-->
