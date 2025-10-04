# Fresh Install Debian 13 Trixie manually with BTRFS, Timeshift, and GRUB-BTRFS

## Why BTRFS?

`btrfs`, often referred to as "BetterFS" or "ButterFS", is a file system that boasts many advanced features such as data integrity checksums, flexible subvolume creation, RAID support, and a lightweight system snapshot footprint. Its primary appeal to desktop users is its ability to take full system snapshots, making it easier to roll back to previous system states. `btrfs` is a *COW* (Copy-On-Write) file system: instead of overwriting data, it creates a copy of only the changed items as a backup, rather than full copies like `rsync` or other solutions would otherwise do.

## Does Debian 13 not offer BTRFS as an option?

Yes, technically. The catch is that Debian 13 does not provide an out-of-the-box option for `btrfs` subvolume creation during installation. It only creates a `/` and `/home` subvolume when `btrfs` is selected, which is a totally viable option when paired with `btrfs-assistant` for snapshots, but we want additional subvolumes created. Additionally we want to use `Timeshift` as our snapshots scheduler, which requires some further tweaks, so we will be performing a manual installation.

It is still common for Debian desktop installs to go with an `ext4` filesysem, which is a rocksolid choice in its own right due to it being a well tested and optimized. Since `btrfs` is newer in comparison, and [still has certain features that are unstable or unrefined](https://btrfs.readthedocs.io/en/latest/Status.html), it is still not the ideal choice for what the Debian project predominately serves. This is why there aren't feature rich options for a `btrfs` subvolume setup in their installer yet. However, by sticking to the stable features of `btrfs`, we can still enjoy the benefits of this file system without issues.

## What is Timeshift?

`Timeshift` is a simple-to-configure system backup tool maintained by the Linux Mint team. It offers an easy interface for configuring which directories to back up, when to schedule backups, and how many backups to retain. `Timeshift` supports both `rsync` and `btrfs`, making it a perfect companion for our setup. It ensures at least one snapshot is automatically created, providing a backup in case of emergencies.

## References

This guide is a combination of instructions and sources I found useful while doing my first manual installation of Debian 13 with BTRFS and snapshot support.

- [How Linux Works (3rd Edition)](https://www.amazon.com/How-Linux-Works-Brian-Ward/dp/1718500408) is a great all encompassing guide on the inner workings of Linux. Chapter 4 was particularly helpful here for going over partitioning and the fstab file layout.
- Special thanks to "JustAGuy Linux" on YouTube for his great video on setting up BTRFS on Debian. It was an especially helpful walkthrough for the manual partitioning. [Debian 13 Trixie Minimal Install w/BTRFS](https://www.youtube.com/watch?v=_zC4S7TA1GI)
- The [Debian Wiki](https://wiki.debian.org/NvidiaGraphicsDrivers#Wayland) was invaluable for setting up the NVIDIA proprietary drivers and resolving Wayland compatibility issues.
- The [Grub-BTRFS GitHub](https://github.com/Antynea/grub-btrfs) page contains all the information you need to install and configure Grub-BTRFS, which we will touch on here but is explained in greater detail there.
