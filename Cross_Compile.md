# Comprehensive Guide: Backing Up and Setting Up a Root Partition for Cross Compilation

This guide describes how to:
1. Identify the correct devices.
2. Safely back up the SD card’s partitions using Partclone.
3. Verify and, if needed, decompress and restore the backup images.
4. Create a raw image from the ext4 backup.
5. Mount the restored image and set it up as a sysroot for cross-compilation.(2 methods)

> **Note:** In our example, we assume that:
> - **/dev/sdc1** is a FAT (or FAT32) boot partition.
> - **/dev/sdc2** is an ext4 root filesystem.

---

## 1. Identify the Correct Device

List available block devices to verify which one is your SD card:

```bash
lsblk
```

Check the output and identify the SD card device (for example, **/dev/sdc** with partitions **/dev/sdc1** and **/dev/sdc2**).

---

## 2. Unmount the SD Card Partitions

Before performing any block-level operations, ensure that the partitions are not mounted:

```bash
sudo umount /dev/sdc1
sudo umount /dev/sdc2
```

---

## 3. Save the Partition Table

It’s critical to save the partition layout so that you can recreate it later if needed:

```bash
sudo sfdisk -d /dev/sdc > ~/kr260_partition_table.txt
```

---

## 4. Check Filesystem Types

Verify the filesystem types and details of each partition:

```bash
lsblk -f
```

Ensure that you correctly identify **/dev/sdc1** (FAT) and **/dev/sdc2** (ext4).

---

## 5. Backup Each Partition Using Partclone

### For the FAT Partition (/dev/sdc1)

Use the FAT-specific version of Partclone (it may be named `partclone.fat32` or similar depending on your installation):

```bash
sudo partclone.fat32 -c -s /dev/sdc1 | gzip -6 > ~/kr260_sdc1.img.gz
```

- `-c`: Tells Partclone to clone (create an image).
- Output is piped through `gzip` with compression level 6.

### For the ext4 Partition (/dev/sdc2)

Backup the root (ext4) filesystem with Partclone:

```bash
sudo partclone.ext4 -c -s /dev/sdc2 | gzip -6 > ~/kr260_sdc2.img.gz
```

*Partclone smartly skips free blocks, making the backup faster and saving disk space.*

---

## 6. Verify the Backups

After creating the backups, verify their integrity using gzip’s testing mode:

```bash
gzip -t ~/kr260_sdc1.img.gz
gzip -t ~/kr260_sdc2.img.gz
```

If no errors are reported, your backups are intact.

---

## 7. Setting Up the Cross-Compilation Environment

For cross-compilation, you’ll need a sysroot that mirrors the target environment. This guide uses the ext4 image for the root filesystem.

### 7.1. Decompress the Image (if necessary)

If you need to work with an uncompressed image, decompress it:

```bash
gunzip -c ~/kr260_sdc2.img.gz > ~/kr260_sdc2.img
```

### 7.2. Create a Destination Backing File for the Raw Restored Image

Since your root ("/") may not have enough space, use a partition with sufficient free space (for instance, one mounted at **/mnt/sda1**). Create a sparse file that is large enough to hold the restored filesystem. If `fallocate` isn’t supported, use `truncate`:

```bash
sudo truncate -s 70G /mnt/sda1/FYP_2024/Ruchith/cross_compile/playground/kr260_sdc2.new.raw
```

This creates a sparse file of 70 GB that will eventually hold the raw ext4 filesystem restored from the Partclone image.

### 7.3. Attach the Sparse File to a Loop Device

Bind the sparse file to an available loop device:

```bash
sudo losetup --find --show /mnt/sda1/FYP_2024/Ruchith/cross_compile/playground/kr260_sdc2.new.raw
```

This command will output a loop device name (e.g., **/dev/loop50**). Make note of it.

### 7.4. Restore the ext4 Partition Image to the Loop Device

Now, restore the Partclone image to the loop device. Use the appropriate Partclone binary (here we use the extfs variant):

```bash
sudo partclone.extfs -r -s ~/kr260_sdc2.img -o /dev/loop50
```

- `-r`: Tells Partclone to restore (reverse of clone).
- The source image is the decompressed ext4 image.
- The destination is the loop device we set up.

Check `/var/log/partclone.log` if errors occur.

### 7.5. Create a Mount Point and Mount the Restored Filesystem

Once the restoration is successful, create a mount point and mount the loop device:

```bash
sudo mkdir -p /mnt/kr260
sudo mount -t ext4 /dev/loop50 /mnt/kr260
```

Now the restored root filesystem is accessible at **/mnt/kr260**.

### 7.6. Navigate into the Mounted Partition

Change to the mount point to explore the filesystem:

```bash
cd /mnt/kr260
ls -l
```

---

## 8. Automate Mounting at Startup

If you need the cross-compilation sysroot to be available at boot, you can add an entry in **/etc/fstab**. For example, if you plan to use the raw image file directly (instead of manually attaching to a loop device), add an entry such as:

```fstab
/mnt/sda1/FYP_2024/Ruchith/cross_compile/playground/kr260_sdc2.new.raw  /mnt/kr260  ext4  loop,nofail  0  2
```

Place this line in **/etc/fstab** (with the full absolute paths) so that the system automatically mounts the image at startup using a loop device.

---

## 9. Cross-Compilation Methods

There are several approaches to cross-compile for the KR260 target system. Here are the main methods:

### 9.1. Using CMake Toolchain File

This method uses a CMake toolchain file to specify cross-compilation settings:

<!-- 1. Create a toolchain file (e.g., `kr260_toolchain.cmake`):
 -->

 2.Chrooting into the mounted board image
