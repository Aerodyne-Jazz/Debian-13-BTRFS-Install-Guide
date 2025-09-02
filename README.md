# Fresh Install Debian 13 Trixie manually with BTRFS, Timeshift, and GRUB-BTRFS

## Why BTRFS?

BTRFS, often referred to as "ButterFS", is a file system that boasts many advanced features such as data integrity via checksums, flexible subvolume creation, RAID support, and a lightweight system snapshot footprint. Its primary appeal to desktop users is its ability to take full system snapshots, making it easier to roll back to previous system states. BTRFS is a *COW* (Copy-On-Write) file system: instead of overwriting data, it creates a copy as a backup, keeping the system efficient by only backing up the changes rather than full copies, like RSYNC or other solutions would otherwise do.

## Does Debian 13 not offer BTRFS by default?

Yes, technically. Unfortunately Debian 13 does not provide an out-of-the-box option for BTRFS during installation. As a result, we will be performing a manual installation. Debian still defaults to EXT4, which is a very stable filesystem. Since BTRFS is relatively new in comparison and still evolving, Debian has not yet deemed it stable enough for default inclusion. However, by sticking to the more stable features of BTRFS, we can still enjoy the benefits of system snapshots without issues.

## What is Timeshift?

Timeshift is a simple-to-configure system backup tool maintained by the Linux Mint team. It offers an easy interface for configuring which directories to back up, when to schedule backups, and how many backups to retain. Timeshift supports both RSYNC and BTRFS, making it a perfect companion for our BTRFS Debian setup. It ensures at least one snapshot is automatically created, providing a backup in case of emergencies.

## References

This guide is a combination of instructions and sources I found useful while doing my first manual installation of Debian 13 with BTRFS and snapshot support.

- Special thanks to "JustAGuy Linux" on YouTube for his great video on setting up BTRFS on Debian. It was an especially helpful reference for the manual partitioning process. [Debian 13 Trixie Minimal Install w/BTRFS](https://www.youtube.com/watch?v=_zC4S7TA1GI)
- The [Debian Wiki](https://wiki.debian.org/NvidiaGraphicsDrivers#Wayland) was invaluable for setting up the NVIDIA proprietary drivers and resolving Wayland compatibility issues.
- The [Grub-BTRFS GitHub](https://github.com/Antynea/grub-btrfs) page contains all the information you need to install and configure Grub-BTRFS, which we will touch on here but is explained in greater detail there.

---

## Partition Disks and Create Directories

1. **Start the installation** in "Expert" mode. This will take you to the disk setup, where we will manually partition the drive.

2. **Partitioning the Disk**:
    - Select the disk you want to use, whether it's a new or existing disk, and choose the GPT partition type.
    - **EFI Partition**: Create a 1GB partition for the EFI system. Select "EFI" as the partition type.
    - **BTRFS Partition**: Create a partition for the main BTRFS system. In my case, I reserved some space for a swap partition (e.g., leave 20GB for swap). Choose "BTRFS" as the partition type.
    - **Swap Partition**: Create a swap partition for additional memory management. If using zRAM, this step is optional.

3. **Creating the Partitions**: After selecting partition sizes and types, confirm and return to the main installation screen.

---

## Configure Subvolumes

This part requires manual configuration via the terminal. Here's how:

1. **Access Terminal**: Use `Ctrl+Alt+F2` to open the Box Buddy terminal view.

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

5. **Rename the Root Volume**: Change `@rootfs` to `@`, which is the format for BTRFS and Timeshift need to utilize snapshots.
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

---

## FSTAB File Configuration

1. **Edit the FSTAB File**: Open the FSTAB file to configure the mounts:
    ```bash
    nano /target/etc/fstab
    ```

2. **Modify Mount Entries**: For each mount point (e.g., `/home`, `/`, etc.), replace the default options with:
    ```bash
    /home    btrfs    noatime,compress=zstd,subvol=@home   0    1
    ```

3. **Save and Exit**: Press `Ctrl+O`, hit `Enter`, and then `Ctrl+X` to exit Nano.

---

## Timeshift Configuration

1. **Install Timeshift**:
    ```bash
    sudo apt install timeshift
    ```

2. **Configure Timeshift**: During setup, select BTRFS as the backup method, set up automated snapshot creation (e.g., on boot and daily), and configure snapshot retention.

---

## GRUB-BTRFS Configuration

Grub-BTRFS integrates snapshots into the GRUB bootloader, allowing you to select system snapshots at boot time.

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
    Change the `ExecStart` line to point to Timeshift snapshots:
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

While BTRFS snapshots are an excellent tool for quick system recovery, they should not replace full system backups. Always ensure you have additional backups using RSYNC or other methods, especially in case of hardware failure. BTRFS snapshots can't help if your entire hard drive fails.
