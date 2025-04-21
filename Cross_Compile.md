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

## 
## 9. **Chroot into the Restored Root** (Native‑Style Builds)

This lets you install libraries, run the device’s native toolchain, and test exactly as on hardware.

### 9.1 Prep: QEMU User Emulation

If your target is ARM64 and host is x86‑64:

```bash
sudo apt-get install qemu-user-static
```

### 9.2 (If Needed) Add ConfigFS Overlay Support

If inside chroot you still get  
```
FileNotFoundError: … '/sys/kernel/config/device-tree'
```  
it means your kernel doesn’t expose the configfs device‑tree interface. You must either enable **CONFIG_OF_CONFIGFS**, **CONFIG_OF_OVERLAY**, **CONFIG_OF_DYNAMIC** in your kernel and rebuild **or** build/load the out‑of‑tree `dtbocfg` module:

```bash
git clone https://github.com/ikwzm/dtbocfg.git
cd dtbocfg
make
sudo insmod dtbocfg.ko
```

This registers the “device-tree” directory under configfs so overlays can be loaded.

### 9.3 Copy QEMU Into the Image

```bash
sudo cp /usr/bin/qemu-aarch64-static /mnt/kr260/usr/bin/
```

### 9.4 Mount Pseudo‑Filesystems (Minimal Bind)

```bash
sudo mount -t proc   proc             /mnt/kr260/proc
sudo mount -t sysfs  sysfs            /mnt/kr260/sys
sudo mount -t configfs configfs       /mnt/kr260/sys/kernel/config    # after dtbocfg loaded
sudo mount --rbind  /dev/pts         /mnt/kr260/dev/pts
sudo mount --bind   /run             /mnt/kr260/run
# DNS, if you need networking inside:
sudo cp /etc/resolv.conf /mnt/kr260/etc/resolv.conf
```

### 9.5 Add Essential Device Nodes

```bash
sudo mknod -m 666 /mnt/kr260/dev/null c 1 3
sudo mknod -m 666 /mnt/kr260/dev/zero c 1 5
sudo mknod -m 666 /mnt/kr260/dev/tty  c 5 0
```

### 9.6 Enter the Chroot

```bash
sudo chroot /mnt/kr260 /bin/bash --login
```

Now you can run the target’s package manager or toolchain just like on device:

```bash
# inside chroot
apt-get update
apt-get install libglew-dev
# or build your FYP code…
```

### 9.7 Exit & Clean Up

```bash
exit
sudo umount /mnt/kr260/run
sudo umount /mnt/kr260/dev/pts
sudo umount /mnt/kr260/sys/kernel/config
sudo umount /mnt/kr260/sys
sudo umount /mnt/kr260/proc
sudo umount /mnt/kr260
sudo losetup -d /dev/loop50
```

If any unmount says “busy”, use `umount -l` (lazy) on that single bind; avoid `-f` unless it’s network FS.

---

## 10. Cross‑Compilation Methods

After you’ve got a working sysroot and/or chroot environment, you can choose:

### 10.1 CMake Toolchain File  
Create a **kr260_toolchain.cmake** pointing at `/mnt/kr260` as **CMAKE_SYSROOT** and invoke:

```bash
cmake -DCMAKE_TOOLCHAIN_FILE=kr260_toolchain.cmake \
      -DCMAKE_BUILD_TYPE=Release  \
      -S . -B build-kr260
cmake --build build-kr260
```

### 10.2 Native Build in Chroot  
Simply `chroot /mnt/kr260` and run your usual `./configure && make && make install` there.

