# popos_btrfs-subvolumes_legacy-installation

## Installation guide for Pop!_OS in the legacy mode with btrfs subvolumes (tested in version 21.04)

1. Detect whether you are in UEFI / Legacy mode

   * Open a terminal

```
ek@efraimkaov:~$ [ -d /sys/firmware/efi ] && echo "Installed in UEFI mode" || echo "Installed in Legacy mode"
```

2. Switch to an interactive root session

   * If you are in Legacy mode, you are good to go

```
ek@efraimkaov:~$ sudo -i
```

3. Prepare partitions manually

   * You can configure your partitions now direct from terminal with tools like `fdisk` or `parted` or with `gparted` when you use the grafical installer to install the system

   * If you have a swap partition like /dev/sda1 Pop!_OS will turn it on this automatic, just make sure you will turn it off

```
ek@efraimkaov:~# lsblk
ek@efraimkaov:~# swapoff /dev/sda1
```

   * Example of partitioning scheme

| DEVICE | SIZE | PTYPE | FSTYPE | MOUNTPOINT |
| --- | --- | --- | --- | --- |
| /dev/sda1 | 2 GiB - 16 GiB | 'Linux swap' | swap | [SWAP] |
| /dev/sda2 | 30 GiB - 100% | 'Linux' | btrfs | / |

4. Installing the system

   * Just use the graphical installer

   * After installation go back to the terminal

5. Mount the btrfs top-level root filesystem

   * **VERY IMPORTANT**, make sure you read carefully

      * `nossd` - By default, BTRFS will enable or disable SSD optimizations, but with regular SSD your system will complety freeze occasionally with `ssd` option enabled. I recommend you to use `nossd` option for both HDD or SSD.

      * `noatime` - Under read intensive work-loads, specifying `noatime` significantly improves performance because no new access time information needs to be written. Note that `noatime` may break applications that rely on `atime` uptimes like the venerable Mutt (unless you use maildir mailboxes).

      * `space_cache` - The free `space_cache` greatly improves performance when reading block group free space into memory. However, managing the `space_cache` consumes some resources, including a small amount of disk space.

      * `autodefrag` - **WARNING:** Defragmenting will break up the reflinks of COW data (for example files copied with cp --reflink, snapshots or de-duplicated data). This may cause considerable increase of space usage depending on the broken up reflinks.

      * `compress=zstd` - zstd expose the compression level as a tunable knob with higher levels trading speed and memory for higher compression ratios.

      * `discard=async` - **WARNING:** BTRFS documentation recommend you to use `fstrim.timer` service over `discard=async` option. By default many distributions have enabled `fstrim.timer` service. I highly don't recommend you to have enabled both `fstrim.timer` service and `discard=async` option. With `discard=async` option enabled your system will complety freeze occasionally.

```
ek@efraimkaov:~# mount -o subvolid=5,nossd,noatime,space_cache,compress=zstd /dev/sda2 /mnt
```

6. Create subvolume @ for /

   * By default Pop!_OS don't make btrfs subvolumes at all, so you need to create them manually

```
ek@efraimkaov:~# btrfs subvolume create /mnt/@
```

7. Move all files and folders from the top-level filesystem into @

```
ek@efraimkaov:~# cd /mnt
ek@efraimkaov:~# ls | grep -v @ | xargs mv -t @
```

8. Create subvolume @home for /home

```
ek@efraimkaov:~# btrfs subvolume create /mnt/@home
```

9. Move all files and folders from the /mnt/@/home/ into @home

```
ek@efraimkaov:~# mv /mnt/@/home/* /mnt/@home/
```

10. Changes to fstab

```
ek@efraimkaov:~# sed -i 's/btrfs  defaults/btrfs  defaults,subvol=@,nossd,noatime,space_cache,compress=zstd/' /mnt/@/etc/fstab
ek@efraimkaov:~# echo "UUID=$(blkid -s UUID -o value /dev/sda2)  /home  btrfs  defaults,subvol=@home,nossd,noatime,space_cache,compress=zstd 0 0" >> /mnt/@/etc/fstab
```

11. Create a chroot environment

```
ek@efraimkaov:~# cd /
ek@efraimkaov:~# umount -l /mnt
ek@efraimkaov:~# mount -o defaults,subvol=@,nossd,noatime,space_cache,compress=zstd /dev/sda2 /mnt
ek@efraimkaov:~# for i in /dev /dev/pts /proc /sys /run; do mount -B $i /mnt$i; done
ek@efraimkaov:~# cp -n /etc/resolv.conf /mnt/etc/
ek@efraimkaov:~# chroot /mnt
ek@efraimkaov:~# mount -av
```

12. Post-Installation steps

```
ek@efraimkaov:~# grub-install /dev/sda
ek@efraimkaov:~# update-grub
```

13. Reboot

```
ek@efraimkaov:~# exit
ek@efraimkaov:~# reboot
```

