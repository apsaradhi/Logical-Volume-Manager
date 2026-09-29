# Logical-Volume-Manager

# Linux LVM Storage Management Lab

## 📌 Project Overview

This project demonstrates how to configure and manage **Linux Logical Volume Management (LVM)** using a newly attached VMware virtual NVMe disk.

In this lab, a **1 GB virtual disk** was partitioned, configured as an LVM Physical Volume (PV), added to a Volume Group (VG), and used to create a Logical Volume (LV). The LV was then formatted with the **XFS filesystem**, mounted to `/newdir`, and verified using Linux storage commands.

### Storage Architecture

```
VMware Virtual NVMe Disk
        │
        ▼
/dev/nvme0n2
        │
        ▼
Partition
/dev/nvme0n2p1
        │
        ▼
Physical Volume (PV)
        │
        ▼
Volume Group
new-vg
        │
        ▼
Logical Volume
new-lv
        │
        ▼
XFS Filesystem
        │
        ▼
Mount Point
/newdir
```

---

# 🎯 Learning Objectives

By completing this lab, you will learn how to:

* Identify newly attached disks in Linux.
* Inspect disk and partition information.
* Create partitions using `fdisk`.
* Change a partition type to **Linux LVM**.
* Create an LVM Physical Volume.
* Create and manage a Volume Group.
* Create a Logical Volume.
* Create an XFS filesystem.
* Create a mount point.
* Mount an LVM logical volume.
* Verify filesystem and disk utilization.
* Understand the relationship between **Disk → Partition → PV → VG → LV → Filesystem → Mount Point**.

---

# 🛠️ Prerequisites

Before performing this lab, you should have:

* A Linux virtual machine.
* VMware Workstation/ESXi or another virtualization platform.
* Root or `sudo` privileges.
* Basic knowledge of Linux commands.
* A newly attached disk of approximately **1 GB**.
* Familiarity with Linux filesystems and storage concepts.

### Environment Used

```
Operating System : RHEL 10
Virtualization   : VMware
Disk             : /dev/nvme0n2
Disk Size        : 1 GiB
Filesystem       : XFS
LVM Tools        : lvm2
```



---

# 1. Identify the New Disk

First, identify the newly attached disk.

```
fdisk -l
```


```
Disk /dev/nvme0n2: 1 GiB, 1073741824 bytes, 2097152 sectors
Disk model: VMware Virtual NVMe Disk
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```


The command displays:

* Disk device name
* Disk size
* Sector size
* Partition information
* Partition table type

Our newly attached disk is:

```
/dev/nvme0n2
```

It is approximately **1 GB** in size and does not yet contain a recognized partition table.

---

# 2. Create a Partition Using fdisk

Open the disk using `fdisk`:

```
fdisk /dev/nvme0n2
```

We then created a new partition.

### Create partition

```
Command (m for help): n
```

Select:

```
p
```

This creates a primary partition.

We accepted the default partition number:

```
1
```

We also accepted the default first and last sectors so that almost the entire disk was used.

### Result

```
Created a new partition 1 of type 'Linux' and of size 1023 MiB.
```

The partition created was:

```
/dev/nvme0n2p1
```

---

# 3. Verify the Partition

Inside `fdisk`, use:

```text
p
```



```
Device         Boot Start     End Sectors  Size Id Type
/dev/nvme0n2p1       2048 2097151 2095104 1023M 83 Linux
```

At this stage, the partition type is:

```
Linux
```

---

# 4. Change Partition Type to Linux LVM

We changed the partition type using:

```text
t
```

Then selected partition:

```text
1
```

And entered:

```
8e
```

`8e` represents **Linux LVM** for an MBR partition table.

The result became:

```
/dev/nvme0n2p1    1023M    8e Linux LVM
```

### Why do this?

Changing the partition type identifies the partition as intended for LVM.

> Note: Modern LVM systems can often work without changing the partition type, particularly with GPT. The important step is creating the partition correctly for your chosen disk layout.

---

# 5. Write the Partition Table

To save the changes:

```
w
```



```
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

The new partition is now available to Linux.

---

# 6. Create the Physical Volume

Now convert the partition into an LVM Physical Volume:

```
pvcreate /dev/nvme0n2p1
```



```
Physical volume "/dev/nvme0n2p1" successfully created.
```

### What is a Physical Volume?

A **Physical Volume (PV)** is an LVM-managed storage device or partition.

Our structure is now:

```
/dev/nvme0n2
      │
      └── /dev/nvme0n2p1
                │
                ▼
             PV
```

---

# 7. Verify the Physical Volume

Run:

```
pvs
```



```
PV             VG   Fmt  Attr PSize    PFree
/dev/nvme0n1p3 rhel lvm2 a--    27.41g       0
/dev/nvme0n2p1      lvm2 ---  1023.00m 1023.00m
```

Our new PV is:

```
/dev/nvme0n2p1
```

It has approximately:

```
1023 MB
```

of free space.

---

# 8. Create a Volume Group

Create a new Volume Group named `new-vg`:

```
vgcreate new-vg /dev/nvme0n2p1
```



```text
Volume group "new-vg" successfully created
```

### What is a Volume Group?

A **Volume Group (VG)** combines one or more Physical Volumes into a storage pool from which Logical Volumes can be created.

Our structure is now:

```
/dev/nvme0n2
      │
      └── /dev/nvme0n2p1
                │
                ▼
               PV
                │
                ▼
             new-vg
```

---

# 9. Verify the Volume Group

Run:

```
vgs
```


```
VG      #PV #LV #SN Attr   VSize    VFree
new-vg    1   0   0 wz--n- 1020.00m 1020.00m
rhel      1   2   0 wz--n-  27.41g       0
```

### Important values

```
VG       = new-vg
#PV      = 1
#LV      = 0
VSize    = approximately 1020 MB
VFree    = approximately 1020 MB
```

The entire new PV is available to the `new-vg` storage pool.

---

# 10. Create the Logical Volume

Create a 1000 MB Logical Volume named `new-lv`:

```
lvcreate -n new-lv --size 1000m new-vg
```



```
Logical volume "new-lv" created.
```

### What is a Logical Volume?

A **Logical Volume (LV)** is a virtual block device created from space inside a Volume Group.

Our architecture is now:

```text
/dev/nvme0n2
      │
      └── /dev/nvme0n2p1
                │
                ▼
          Physical Volume
                │
                ▼
             new-vg
                │
                ▼
             new-lv
```

---

# 11. Verify the Logical Volume

Run:

```
lvs
```

You can also use:

```
lvdisplay
```

These commands display information about the logical volume, including its size, attributes, and volume group.

The logical volume can be accessed as:

```
/dev/new-vg/new-lv
```

---

# 12. Create an XFS Filesystem

Format the Logical Volume with XFS:

```
mkfs.xfs /dev/new-vg/new-lv
```



The output contains information similar to:

```
meta-data=/dev/new-vg/new-lv
data     = bsize=4096
log      =internal log
realtime =none
```

### What happened?

The Logical Volume previously contained raw block storage.

`mkfs.xfs` created an **XFS filesystem** on that block device.

The storage hierarchy is now:

```
Disk
 │
 └── Partition
       │
       └── PV
            │
            └── VG
                 │
                 └── LV
                      │
                      └── XFS
```

---

# 13. Create a Mount Point

Create a directory where the filesystem will be mounted:

```
mkdir /newdir
```

The directory:

```
/newdir
```

will act as the access point for the new filesystem.

---

# 14. Mount the Logical Volume

Mount the XFS filesystem:

```
mount /dev/new-vg/new-lv /newdir/
```

If the command produces no output, the mount was successful.

---

# 15. Verify the Mount

Run:

```
df -h
```



```
Filesystem                   Size  Used Avail Use% Mounted on
/dev/mapper/new--vg-new--lv  936M   51M  887M   6% /newdir
```

Your original output showed:

```
/dev/mapper/new--vg-new--lv    958464    51424    907040   6% /newdir
```

### What this confirms

The output confirms that:

* The LV exists.
* XFS is mounted.
* The filesystem is accessible.
* `/newdir` is using the newly created LVM storage.

---

# 🔍 Final Storage Architecture

After completing the lab, the storage structure is:

```
VMware Virtual NVMe Disk
        │
        ▼
/dev/nvme0n2
        │
        ▼
/dev/nvme0n2p1
        │
        ▼
Physical Volume (PV)
        │
        ▼
Volume Group (VG)
new-vg
        │
        ▼
Logical Volume (LV)
new-lv
        │
        ▼
XFS Filesystem
        │
        ▼
/newdir
```

---

# 📋 Command Summary

| Step | Command                                  | Purpose                   |
| ---- | ---------------------------------------- | ------------------------- |
| 1    | `fdisk -l`                               | List disks and partitions |
| 2    | `fdisk /dev/nvme0n2`                     | Create/manage partition   |
| 3    | `pvcreate /dev/nvme0n2p1`                | Create Physical Volume    |
| 4    | `pvs`                                    | Display Physical Volumes  |
| 5    | `vgcreate new-vg /dev/nvme0n2p1`         | Create Volume Group       |
| 6    | `vgs`                                    | Display Volume Groups     |
| 7    | `lvcreate -n new-lv --size 1000m new-vg` | Create Logical Volume     |
| 8    | `lvs`                                    | Display Logical Volumes   |
| 9    | `mkfs.xfs /dev/new-vg/new-lv`            | Create XFS filesystem     |
| 10   | `mkdir /newdir`                          | Create mount point        |
| 11   | `mount /dev/new-vg/new-lv /newdir`       | Mount filesystem          |
| 12   | `df -h`                                  | Verify filesystem usage   |

---

# 🧪 Verification Commands

After completing the lab, these commands can be used to inspect the complete configuration:

```
lsblk
```

Displays the block-device hierarchy.

```
sudo pvs
```

Displays Physical Volumes.

```
sudo vgs
```

Displays Volume Groups.

```
sudo lvs
```

Displays Logical Volumes.

```
df -h
```

Displays mounted filesystem usage.

```
mount | grep newdir
```

Confirms that the filesystem is mounted on `/newdir`.

---

