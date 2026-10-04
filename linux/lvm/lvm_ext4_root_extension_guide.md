# Standard Operating Procedure: Extending ext4 Root Filesystem Using LVM

This document outlines the end-to-end procedure for expanding an existing `ext4` root filesystem (`/`) across a newly attached, uninitialized block device using Logical Volume Manager (LVM) in Linux.

---

## 1. Overview & Architectural Considerations

LVM can consume storage block devices in two modes:
1. **Raw Whole-Disk Mode (`/dev/sdX`):** Direct initialization of the entire block device as an LVM Physical Volume (PV).
2. **Partitioned Mode (`/dev/sdX1`):** Partitioning the disk first and setting the partition type to Linux LVM before initializing the PV.

### Comparison

| Feature | Raw Whole-Disk (`/dev/sdX`) | Partitioned (`/dev/sdX1`) |
| :--- | :--- | :--- |
| **Simplicity** | Faster; zero partition table management. | Requires running `fdisk` / `parted`. |
| **Cloud/Virtual Resizing** | Simpler: running `pvresize /dev/sdX` handles storage expansion immediately without touching partition tables. | Requires growing or rewriting the partition table prior to running `pvresize`. |
| **Visibility / Safety** | Some tools or novice admins might interpret the disk as unallocated/empty. | Clear `Linux LVM` flag in partition table prevents accidental overwrites by third-party tools. |

Both methods are supported in modern enterprise distributions (RHEL, Rocky, Ubuntu, Debian, SLES).

---

## 2. Pre-Flight Inspection & Discovery

Run the following commands to collect configuration details:

```bash
# 1. Identify the newly attached disk and note its identifier (e.g., /dev/sdb, /dev/vdb, /dev/nvme1n1)
lsblk

# 2. Identify the root filesystem mount point, device mapper path, and filesystem type
df -hT /

# 3. List Volume Groups (VG) and Logical Volumes (LV)
sudo vgs
sudo lvs
```

### Variables Used in This Guide
* **Target Disk:** `/dev/sdb`
* **Volume Group (VG):** `vg_root` (replace with your actual VG name, e.g., `ubuntu-vg`)
* **Logical Volume (LV):** `lv_root` (replace with your actual LV name, e.g., `ubuntu-lv`)
* **Device Path:** `/dev/vg_root/lv_root` (or `/dev/mapper/vg_root-lv_root`)

---

## 3. Procedure: Option A (Raw Whole-Disk — Recommended for Virtualized/Cloud Disks)

Use this method to avoid partition table overhead.

### Step 1: Initialize the Disk as a Physical Volume (PV)
```bash
sudo pvcreate /dev/sdb
```
*Verification:*
```bash
sudo pvs /dev/sdb
```

### Step 2: Extend the Volume Group (VG)
Add the newly initialized PV to the target Volume Group:
```bash
sudo vgextend vg_root /dev/sdb
```
*Verification:*
```bash
sudo vgs vg_root
```
Verify that `VFree` reflects the additional capacity.

### Step 3: Extend the Logical Volume and Resize the ext4 Filesystem
Run `lvextend` with the `-r` (`--resizefs`) flag to automatically extend the underlying `ext4` filesystem using `resize2fs` while online:
```bash
sudo lvextend -r -l +100%FREE /dev/vg_root/lv_root
```

---

## 4. Procedure: Option B (Partitioned Disk via `fdisk`)

Use this method if organizational policy mandates explicit partition tables.

### Step 1: Create the Partition
Launch `fdisk` on the target disk:
```bash
sudo fdisk /dev/sdb
```

Execute the following interactive sequence:
1. Press `n` to create a new partition.
2. Press `Enter` to accept defaults for partition number (1), first sector, and last sector (uses 100% of the disk).
3. Press `t` to change the partition type.
   * If using **MBR**: Enter hex code `8e` (Linux LVM).
   * If using **GPT**: Enter partition type `30` (or type `lvm` / select the alias for `Linux LVM`).
4. Press `w` to write the table to disk and exit.

Inform the kernel of partition changes if necessary:
```bash
sudo partprobe /dev/sdb
```

### Step 2: Initialize the Partition as a Physical Volume (PV)
```bash
sudo pvcreate /dev/sdb1
```

### Step 3: Extend the Volume Group (VG)
```bash
sudo vgextend vg_root /dev/sdb1
```

### Step 4: Extend the Logical Volume and Resize the ext4 Filesystem
```bash
sudo lvextend -r -l +100%FREE /dev/vg_root/lv_root
```

---

## 5. Manual Fallback: Filesystem Extension

If the `-r` option was omitted during `lvextend`:

```bash
# 1. Extend the LV allocate-only:
sudo lvextend -l +100%FREE /dev/vg_root/lv_root

# 2. Resize the ext4 filesystem online:
sudo resize2fs /dev/vg_root/lv_root
```

---

## 6. Post-Extension Verification

Validate the new storage capacity:

```bash
# Check updated filesystem size
df -h /

# Verify LVM state
sudo pvs
sudo vgs
sudo lvs
```