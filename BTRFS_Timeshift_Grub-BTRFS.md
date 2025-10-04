# Fresh Install Debian 13 Trixie manually with BTRFS, Timeshift, and GRUB-BTRFS

## Why BTRFS?

`btrfs`, often referred to as "BetterFS" or "ButterFS", is a file system that boasts many advanced features such as data integrity checksums, flexible subvolume creation, RAID support, and a lightweight system snapshot footprint. Its primary appeal to desktop users is its ability to take full system snapshots, making it easier to roll back to previous system states. `btrfs` is a *COW* (Copy-On-Write) file system: instead of overwriting data, it creates a copy of only the changed items as a backup, rather than full copies like `rsync` or other solutions would otherwise do.

## Does Debian 13 not offer BTRFS as an option?

Yes, technically. The catch is that Debian 13 does not provide an out-of-the-box option for `btrfs` subvolume creation during installation. It only creates a `/` and `/home` subvolume when `btrfs` is selected, which is a totally viable option when paired with `btrfs-assistant` for snapshots, but we want additional subvolumes created. Additionally we want to use `Timeshift` as our snapshots scheduler, which requires some further tweaks, so we will be performing a manual installation.

It is still common for Debian desktop installs to go with an `ext4` filesysem, which is a rocksolid choice in its own right due to it being well tested and optimized. Since `btrfs` is newer in comparison, and [still has certain features that are unstable or unrefined](https://btrfs.readthedocs.io/en/latest/Status.html), it is still not the ideal choice for what the Debian project predominately serves. This is why there aren't feature rich options for a `btrfs` subvolume setup in their installer yet. However, by sticking to the stable features of `btrfs`, we can still enjoy the benefits of this file system without issues.

## What is Timeshift?

`Timeshift` is a simple-to-configure system backup tool maintained by the Linux Mint team. It offers an easy interface for configuring which directories to back up, when to schedule backups, and how many backups to retain. `Timeshift` supports both `rsync` and `btrfs`, making it a perfect companion for our setup. It ensures at least one snapshot is automatically created, providing a backup in case of emergencies.

## References

This guide is a combination of instructions and sources I found useful while doing my first manual installation of Debian 13 with BTRFS and snapshot support.

- [How Linux Works (3rd Edition)](https://www.amazon.com/How-Linux-Works-Brian-Ward/dp/1718500408) is a great all encompassing guide on the inner workings of Linux. Chapter 4 was particularly helpful here for going over partitioning and the fstab file layout.
- Special thanks to "JustAGuy Linux" on YouTube for his great video on setting up BTRFS on Debian. It was an especially helpful walkthrough for the manual partitioning. [Debian 13 Trixie Minimal Install w/BTRFS](https://www.youtube.com/watch?v=_zC4S7TA1GI)
- The [Debian Wiki](https://wiki.debian.org/NvidiaGraphicsDrivers#Wayland) was invaluable for setting up the NVIDIA proprietary drivers and resolving Wayland compatibility issues.
- The [Grub-BTRFS GitHub](https://github.com/Antynea/grub-btrfs) page contains all the information you need to install and configure Grub-BTRFS, which we will touch on here but is explained in greater detail there.

---

## Partition Disks and Create Directories

1. **Start the installation** in "Expert" mode. This will take you to the disk setup, where we will manually partition the drive.

2. **Partitioning the Disk**:
    - Select the disk you want to use, whether it's a new or existing disk, and choose the GPT partition type.
    - **EFI Partition**: Create a 1GB partition for the EFI system. Select "EFI" as the partition type.
    - **BTRFS Partition**: Create a partition for the main BTRFS system. In my case, I reserved some space for a swap partition (e.g., leave 20GB for swap). Choose "BTRFS" as the partition type.
    - **Swap Partition**: Create a swap partition for additional memory management. If using `zram`, this step is optional, although both together would be optional.

3. **Creating the Partitions**: After selecting partition sizes and types, confirm and return to the main installation screen.

---

## Configure Subvolumes

This part requires manual configuration via the terminal. Here's how:

1. **Access Terminal**: Use `Ctrl+Alt+F2` to open the `Box Buddy` terminal view.

2. **List Available Drives**: Use the `df -h` command to list available drives and partitions.

3. **Unmount Partitions**:
    ```bash
    umount /target/boot/efi
    umount /target
    ```

4. **Mount the Largest Partition**: Replace `/your/drivepath` with the actual partition path.
    ```bash
    umount /your/drivepath /mnt
    ```

5. **Rename the Root Volume**: Change `@rootfs` to `@`, which is the format `Timeshift` needs to utilize snapshots.
    ```bash
    mv @rootfs @
    ```

6. **Create Subvolumes**:
    At minimum, you need `@` (root) and `@home`. You can also create `@snapshots`, `@log`, `@var`, and any additional subvolumes you wish:
    ```bash
    btrfs su cr @home
    btrfs su cr @snapshots
    btrfs su cr @log
    btrfs su cr @var
    ```

7. **Mount Subvolumes**: Mount the root subvolume first, and then other subvolumes:
    ```bash
    #Remember to replace /your/drivepath/ with your actual path
    mount -o noatime,compress=zstd,subvolume=@ /your/drivepath /target
    mount -o noatime,compress=zstd,subvolume=@home /your/drivepath /target/home
    mount -o noatime,compress=zstd,subvolume=@snapshots /your/drivepath /target/.snapshots
    mount -o noatime,compress=zstd,subvolume=@log /your/drivepath /target/var/log
    mount -o noatime,compress=zstd,subvolume=@var /your/drivepath /target/var/cache
    ```
    Here I used `zstd` compression since it has been the most tested, but you have other options like `lzo` and `zlib`.

---

## FSTAB File Configuration

1. **Edit the FSTAB File**: Open the `fstab` file to configure the mounts using `nano` text editor:
    ```bash
    nano /target/etc/fstab
    ```

2. **Modify Mount Entries**: For each mount point (e.g., `/home`, `/`, etc.), replace the default options with:
    ```bash
    /            btrfs  noatime,compress=zstd,subvol=@   0    0
    /home        btrfs  noatime,compress=zstd,subvol=@home   0    0
    /.snapshots  btrfs  noatime,compress=zstd,subvol=@snapshots   0    0
    /var/log     btrfs  noatime,compress=zstd,subvol=@log   0    0
    /var/cache   btrfs  noatime,compress=zstd,subvol=@var   0    0
    ```
   Here we didn't add a level for `zstd` to compress to, because leaving out any indicator will default it to `zstd:3`, which is a good middle ground of cpu performance and amount of compression. If you want to manually adjust this, the range can go from `-7` for the least but fastest compression, or to `22` for the slowest but most compression.
   
   The last digit at the end of each line is for the filesystem integrity test level setting that `fsck` runs on boot. We have them all set to `0` to ommit `fsck` from checking the subvolumes, as the integrity checks on boot are only needed on other filesystems. The `btrfs` filesystem already has integrity checks built into it, and not just on boot, so running `fsck` isn't necessary and doesn't even function the same.
   
   We have `noatime` so there isn't updates to file metadata everytime a file is accessed, which increases performance. Additionally, you can add`ssd` if you are using one which will provide ssd specific performance boosts. The filesystem can normally detect and enable `ssd` without you putting it into your `fstab` file, but it doesn't hurt to explicitly add it. If you want to know all the options you have available, you can read about them on the [BTRFS Administration Documentation](https://btrfs.readthedocs.io/en/latest/Administration.html#mount-options).

4. **Save and Exit**: Press `Ctrl+O`, hit `Enter`, and then `Ctrl+X` to exit Nano.

---

## Timeshift Configuration

1. **Install Timeshift**:
    ```bash
    sudo apt install timeshift
    ```

2. **Configure Timeshift**: During setup, select `btrfs` as the backup method, set up automated snapshot creation (e.g., on boot and daily), and configure snapshot retention.

---

## GRUB-BTRFS Configuration

Grub-BTRFS integrates snapshots into the `Grub` bootloader, allowing you to select system snapshots at boot time.

1. **Install Dependencies**:
    ```bash
    sudo apt install git inotify-tools
    ```

2. **Clone Grub-BTRFS Repository**:
    ```bash
    cd ~/Downloads
    git clone https://github.com/Antynea/grub-btrfs
    cd grub-btrfs
    ```

3. **Install Grub-BTRFS**:
    ```bash
    sudo make install
    sudo grub-mkconfig
    ```

4. **Start and Enable the Grub-BTRFS Service**:
    ```bash
    sudo systemctl start grub-btrfsd
    sudo systemctl enable grub-btrfsd
    ```

5. **Edit Grub-BTRFS Config**:
    ```bash
    sudo systemctl edit --full grub-btrfsd
    ```
    Change the `ExecStart` line to point to `Timeshift` snapshots:
    ```bash
    ExecStart=/usr/bin/grub-btrfsd --syslog --timeshift-auto
    ```

6. **Restart the Service**:
    ```bash
    sudo systemctl restart grub-btrfs
    sudo grub-mkconfig
    ```

---

## Disclaimer

While `btrfs` snapshots are an excellent tool for quick system recovery, they should not replace full system backups. Always ensure you create backups to an additional hard drive using `rsync` or other methods. This is especially crucial in the ever-looming threat of hard drive failure, since snapshots won't help if the hard drive you have is now a paperweight. `btrfs` is also a newer filesystem in comparison to others like `ext4`, and still has a bunch of unstable features, so use at your own risk. You can learn more on the [Debian Wiki page for Btrfs](https://wiki.debian.org/Btrfs) and the [Btrfs documentation status page](https://btrfs.readthedocs.io/en/latest/Status.html).
