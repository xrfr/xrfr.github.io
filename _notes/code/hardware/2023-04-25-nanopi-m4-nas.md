---
title: NanoPi M4 NAS
feed: show
---

**Table of Contents**

- [Introduction](#Introduction)
- [Hardware](#Hardware)
- [Flashing an OS](#Flashing%20an%20OS)
	- [FriendlyCore](#FriendlyCore)
	- [Armbian](#Armbian)
- [ZFS](#ZFS)
	- [Installing ZFS](#Installing%20ZFS)
	- [ZFS RAIDZ Expansion](#ZFS%20RAIDZ%20Expansion)
	- [Create ZFS Pool](#Create%20ZFS%20Pool)
- [Applications](#Applications)
	- [Backing Up Data](#Backing%20Up%20Data)
	- [Photo Library](#Photo%20Library)

# Introduction

The [NanoPi M4](https://www.friendlyelec.com/index.php?route=product/product&product_id=234) is an ARM board that supports up to 4x SATA drives when enquipped with this [SATA Hat](https://www.friendlyelec.com/index.php?route=product/product&product_id=254). The board also has 4x USB 3.0 ports, meaning we can theorectially have up to 8 drives connected to this little thing. My goal is to set this up with a few HDDs/SSDs in a ZFS RAIDZ pool and use it to backup photos.
# Hardware

- [NanoPi M4](https://www.friendlyelec.com/index.php?route=product/product&product_id=234)
- [SATA Hat](https://www.friendlyelec.com/index.php?route=product/product&product_id=254)
- Power Cable for 4x SATA Hat
- 12V Power Supply (might need this in the future if we get to 4 drives)
- [SATA Power Cable](https://www.amazon.com/Monoprice-108794-24-Inch-15-Pin-Female/dp/B009GULFJ0)
- 2x 8TB HDDs (we can get more when we run out of space)
- 256GB SSD (that I had lying around)
- [3.5" HDD Bracket](https://www.amazon.com/Phanteks-Stackable-Bracket-Cases-PH-HDDKT_03/dp/B07GY2B3WP)

# Flashing an OS

## FriendlyCore

Initially, I tried using one of the OS images provided by FriendlyElec, called [FriendlyCore](https://onedrive.live.com/?authkey=%21AOMCjrhZzok1O%2DY&id=1F5B36BBA3D56743%218027&cid=1F5B36BBA3D56743). This is a Ubuntu-based OS, and comes prepackaged with some NanoPi specific utilities and headers.

However, the image is for an 8GB SD card, and it comes with 2 partitions, one that is 3GB for the base OS, and the remaining 4.5GB for a copy of the OS that is meant to flashed into the eMMC on the NanoPi. The eMMC would be a lot faster to use than an SD card, but alas, I didn't purchase an eMMC module when I bought the card. It's worth trying later, though.

The issue is that 3GB leaves me running out of disk space after any trivial operation. The mount is set up as an `overlayfs`, which I'm not too familiar with. I tried to delete the unnecessary partition, but it just left me unable to boot.

## Armbian

Instead, I decided to try out Armbian, which has prebuilt images for many different SBCs, including the [NanoPi M4](https://www.armbian.com/nanopi-m4/). The setup on Armbian was really smooth.

# ZFS

ZFS is an open-source file system and volume manager with some really cool features. With it, we can make a file system that spans across multiple drives, and extend it  with new drives whenever we need.
## Installing ZFS

Once Armbian is set up, I ran the following to install ZFS.

Typically you can use the `armbian-config` utility to install kernel headers, but it didn't seem to work for me, so I installed them manually.

```
sudo apt install linux-headers-current-rockchip64
sudo apt install zfs-dkms zfsutils-linux 
```

The latter command may take some time to compile the ZFS kernel modules, so take a break, stretch, etc.

Make sure you're able to load the `zfs` kernel module with `sudo modprobe zfs`. If not, something went wrong and you might need to do some googling.


## ZFS RAIDZ Expansion

ZFS offers a number of software based RAID modes for data redundancy. RAIDZ stripes parity information across all disks, letting up to one drive fail without losing data.

As of 2022, you can [expand](https://freebsdfoundation.org/blog/raid-z-expansion-feature-for-zfs/) a RAIDZ group with a new drive without having to recreate the zpool. The caveat is that existing data uses the old data/parity ratio, while new data will use the new data/parity ratio.

For example, if I have 2 disks, then my data/partiy ratio is 1:1, meaning that only 1/2 of my total raw storage is actually usable. If I add another disk to the RAIDZ group, my data/partiy ratio improves to 2:1, meaning 2/3 of my total storage is usable. With this caveat, though, existing data will keep the old data/parity ratio, leaving my usable disk space somewhere between 1/2 and 2/3 of the total.

However, we can get around this by rewriting the old data, giving it the new data/parity ratio.

## Create ZFS Pool

Creating a basic RAIDZ1 pool that allows for one drive to fail without losing data.

```
# find disk by serial
lsblk -o name,model,serial

# add discs by serial instead of `/dev/sda`...
sudo zpool create lake raidz /dev/disk/by-id/XXX /dev/disk/by-id/YYY
```

# Applications

## Backing Up Data

TODO

## Photo Library

There is a plugin for Nextcloud I want to try out called [memories](https://github.com/pulsejet/memories). This would work as a Google Photos replacement.