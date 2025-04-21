
```markdown
# Comprehensive Guide: System Restoration Workflow

This guide describes how to restore a backed-up version of your system (from a KR260 or similar device) onto a new chip (or SD card). It includes steps for recreating the partition table, formatting partitions, restoring system images using Partclone, and verifying (and repairing) the restored filesystems.

> **Note:** In our example, we assume that:
> - **/dev/sdc1** is a FAT (or FAT32) boot partition.
> - **/dev/sdc2** is an ext4 root filesystem.

---

## 1. Identify the Correct Device

Before you begin, list all block devices to ensure you’re targeting the correct one (e.g. an SD card or chip).  
```bash
lsblk
```
Identify the SD card (for example, **/dev/sdc**) with partitions **/dev/sdc1** and **/dev/sdc2**.

---

## 2. Unmount the Target Partitions

Ensure no partition is in use before applying changes:
```bash
sudo umount /dev/sdc1
sudo umount /dev/sdc2
```

---

## 3. Save or Review the Existing Partition Table (Optional)

If you have a saved partition table from your backup, review it. For example, if you previously backed it up:
```bash
sudo sfdisk -d /dev/sdc > ~/kr260_partition_table.txt
```
*Potential Issue:*  
If the saved table specifies partition sizes that exceed the physical size of your target device, you must adjust them manually. For instance, if the second partition’s end exceeds the device’s total sectors by a small margin, compute the correct size as:  
```
Available sectors for partition 2 = Total sectors - Start sector of partition 2
```
Then update the size field in your partition table file accordingly.

---

## 4. Reapply the Partition Table on the New Chip

Once any necessary adjustments have been made, reapply the partition table:
```bash
sudo sfdisk /dev/sdc < ~/kr260_partition_table.txt
```
Check the output to ensure that the partition table now properly fits within the physical limits of the device.  
*Tip:* Run `sudo fdisk -l /dev/sdc` to verify the updated layout.

---

## 5. Format the Partitions

To ensure a clean slate, format each partition with the correct filesystem type.

### For the Boot Partition (FAT32)
```bash
sudo mkfs.vfat /dev/sdc1
```

### For the Root Partition (ext4)
```bash
sudo mkfs.ext4 /dev/sdc2
```

---

## 6. Restore the Backup Images Using Partclone

Partclone is used to restore system images efficiently (copying only used blocks).

### 6.1. Restore the FAT Partition (/dev/sdc1)
Assuming your backup image is compressed with gzip, restore with:
```bash
sudo gzip -dc ~/kr260_sdc1.img.gz | sudo partclone.fat32 -r -s - -o /dev/sdc1
```
- `-r` tells Partclone to restore.
- `-s -` reads the image from standard input.
- `-o /dev/sdc1` specifies the target partition.

### 6.2. Restore the ext4 Partition (/dev/sdc2)
When restoring the root (ext4) partition, you may encounter an error if the target partition is a few MB smaller than the image source:
```plaintext
Target partition size(61468 MB) is smaller than source(61488 MB). Use option -C to disable size checking(Dangerous).
```
To bypass this check (when the size difference is negligible), use the **-C** option:
```bash
sudo gzip -dc ~/kr260_sdc2.img.gz | sudo partclone.ext4 -C -r -s - -o /dev/sdc2
```
*Potential Issue:*  
Using **-C** skips a safeguard, so ensure that the actual used space on the filesystem is well below the target capacity. Always check `/var/log/partclone.log` for any errors.

---

## 7. Verify and Repair the Restored Filesystem

After restoration, you may get an error when checking the filesystem with `e2fsck` due to a slight size mismatch between the superblock and the partition’s physical size.

### 7.1. Check the Filesystem with e2fsck
```bash
sudo e2fsck -fy /dev/sdc2
```
- The `-f` flag forces a check.
- The `-y` flag automatically fixes errors.

You might see a message such as:
```plaintext
The filesystem size (according to the superblock) is 15011579 blocks
The physical size of the device is 15006720 blocks
Either the superblock or the partition table is likely to be corrupt!
```
This indicates that the superblock still thinks the filesystem is larger than the partition.

### 7.2. Shrink the Filesystem with resize2fs
Adjust the filesystem to match the physical size:
```bash
sudo resize2fs /dev/sdc2 15006720
```
- Here, **15006720** is the number of blocks available on the partition.
- Alternatively, if resize2fs works without the size parameter after a clean e2fsck, you can simply run:
  ```bash
  sudo resize2fs /dev/sdc2
  ```

### 7.3. Verify the New Filesystem Size
Check the block count:
```bash
sudo dumpe2fs /dev/sdc2 | grep 'Block count'
```
and review with:
```bash
df -h /dev/sdc2
```

*Potential Problems & Fixes:*
- **Superblock Mismatch:** If e2fsck continues to report size mismatches, re-run e2fsck after resize2fs.
- **Unrepairable Filesystem Errors:** Consult `/var/log/partclone.log` for specific errors and consider restoring from an earlier backup if repair fails.

---

## 8. Mount the Restored Partitions and Verify the System

After repairs and resizing, mount the partitions to check that everything is in order.

### Mount the Boot Partition:
```bash
sudo mount /dev/sdc1 /mnt/boot
ls -l /mnt/boot
```

### Mount the Root Partition:
```bash
sudo mount /dev/sdc2 /mnt/root
ls -l /mnt/root
```

Examine files and directories to verify that the restored system is intact.

---

## 9. Troubleshooting & Potential Pitfalls

- **Partition Table Mismatch:**  
  - **Issue:** The restored partition table does not match the physical disk size.  
  - **Fix:** Manually calculate the proper size for each partition and update the partition table file before reapplying it.

- **Size Check Error in Partclone:**  
  - **Issue:** When restoring the ext4 partition, Partclone complains about a size mismatch.  
  - **Fix:** Use the `-C` flag if the difference is minor, then immediately run e2fsck and resize2fs to correct the filesystem metadata.

- **Filesystem Corruption After Restore:**  
  - **Issue:** e2fsck detects filesystem errors or superblock mismatches.  
  - **Fix:** Run a forced repair with `sudo e2fsck -fy /dev/sdc2` and adjust the filesystem with `resize2fs`.

- **Syncing Errors:**  
  - **Issue:** After restoration, data might not be completely written to disk.  
  - **Fix:** Always run `sudo sync` after restoration to flush all writes.

- **Log Files:**  
  - Always check `/var/log/partclone.log` for detailed error messages if something goes wrong.

---

# Final Overview

1. **Identify & Unmount:**  
   - Use `lsblk` to verify correct device.
   - Unmount partitions with `sudo umount`.

2. **Partition Table Setup:**  
   - Save and review existing partition table.
   - Adjust and reapply the partition table using sfdisk.

3. **Format Partitions:**  
   - Format boot (mkfs.vfat) and root (mkfs.ext4) partitions.

4. **Restore Images:**  
   - Restore the FAT image using `partclone.fat32`.
   - Restore the ext4 image using `partclone.ext4` with `-C` if necessary.

5. **Verify & Repair Filesystem:**  
   - Run `sudo e2fsck -fy` on ext4 partition.
   - Adjust the filesystem size with `resize2fs` to match the partition.

6. **Mount & Validate:**  
   - Mount restored partitions to check file integrity.

7. **Troubleshooting:**  
   - Refer to log files and adjust partition sizes if errors occur.

By following this guide, you can reliably restore your system backup to a new SD card or chip and ensure that the filesystem metadata matches the physical partition size. Always test the restored system thoroughly before considering the restoration complete.

-