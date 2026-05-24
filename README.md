# RHEL10-Logical-Volume-Manager
Here is a complete, step-by-step guide for adding a new disk to a RHEL 10 system. This guide assumes you are using **LVM** (Logical Volume Manager), which is the standard and recommended storage management method for RHEL.

> ⚠️ **Critical Warning:** Double-check device names before running any destructive commands (`wipefs`, `pvcreate`). Selecting the wrong disk will result in permanent data loss. Use `lsblk` frequently to verify.

### Phase 1: Identify and Clean the Disk

1.  **Identify the new disk**
    ```bash
    lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
    ```
    *Look for a disk with no partitions, no FSTYPE, and no MOUNTPOINT. Note the device name (e.g., `/dev/sdb`, `/dev/nvme1n1`).*
    ```
    NAME               SIZE TYPE FSTYPE      MOUNTPOINT MODEL
    nvme0n1             20G disk                        VMware Virtual NVMe Disk
    ├─nvme0n1p1          1M part
    ├─nvme0n1p2          1G part xfs         /boot
    └─nvme0n1p3         19G part LVM2_member
      ├─almalinux-root  17G lvm  xfs         /
      └─almalinux-swap   2G lvm  swap
    nvme0n2              5G disk                        VMware Virtual NVMe Disk
    ```

3.  **Wipe any existing signatures** (old partition tables, RAID metadata, etc.)
    ```bash
    sudo wipefs -a /dev/nvme0n2
    ```
    Replace `/dev/nvme0n2` with your actual new disk. The `-a` flag erases all filesystem/RAID/partition table signatures.

### Phase 2: Create LVM Physical Volume & Extend Volume Group

3.  **Initialize the disk as an LVM Physical Volume (PV)**
    ```bash
    sudo pvcreate /dev/nvme0n2
    ```

4.  **Check your existing Volume Group (VG) name**
    ```bash
    sudo vgs
    ```
    *Note the VG name (commonly `rhel` or `rl10` on RHEL 10).*

5.  **Extend the Volume Group with the new PV**
    ```bash
    sudo vgextend <vg_name> /dev/nvme0n2
    ```

6.  **Verify the VG now has free space**
    Look at the VFree column — it should reflect your new disk's capacity
    ```bash
    sudo vgs
    ```
    
### Mounting a NEW Filesystem (If Not Expanding Existing)

> Skip this phase if you only expanded an existing volume above. Use this if you created a **brand new LV**.

1.  **Create a new Logical Volume**
    ```bash
    sudo lvcreate -n <new_lv_name> -l 100%FREE <vg_name>
    ```

2. **Format with XFS** (RHEL 10 default/recommended)
    ```bash
    sudo mkfs.xfs /dev/<vg_name>/<new_lv_name>
    ```

3. **Create mount point and mount**
    ```bash
    sudo mkdir -p /mnt/newdisk
    sudo mount /dev/<vg_name>/<new_lv_name> /mnt/newdisk
    ```

4. **Make persistent across reboots** — add to `/etc/fstab`
    ```bash
    # Get the UUID
    sudo blkid /dev/<vg_name>/<new_lv_name>

    # Add entry (use UUID, never raw device path)
    echo "UUID=<paste-uuid-here>  /mnt/newdisk  xfs  defaults  0 0" | sudo tee -a /etc/fstab

    # Validate fstab syntax (CRITICAL — prevents unbootable system)
    sudo findmnt --verify --verbose
    ```
### Expand Logical Volume & Filesystem(If Not Mounting a NEW Filesystem)
    The naming convention is always `<VG>-<LV>`. The hyphen separates them:

    | Component | Value | Full Device Path |
    | :--- | :--- | :--- |
    | Volume Group | `rhel` | — |
    | Logical Volume | `root` | `/dev/rehl/root` |
    | Logical Volume | `swap` | `/dev/rehl/swap` |
1.  **Extend the Logical Volume (LV) AND resize the filesystem in one command**
    ```bash
    sudo lvextend -r -l +100%FREE /dev/<vg_name>/<lv_name>
    ```
    -   `-r` (`--resizefs`): Automatically grows the filesystem after extending the LV
    -   `-l +100%FREE`: Uses **all** available free space in the VG
    -   For XFS (RHEL default): uses `xfs_growfs` automatically
    -   For ext4: uses `resize2fs` automatically

2.  **Verify the expansion**
    ```bash
    df -hT /mount/point
    ```



### Simple Partition Non-LVM Alternative

    Only use this if you intentionally do **not** want LVM:

    ```bash
    # Create GPT partition table + single partition
    sudo parted /dev/nvme0n2 mklabel gpt
    sudo parted /dev/nvme0n2 mkpart primary xfs 0% 100%

    # Format and mount
    sudo mkfs.xfs /dev/nvme0n2
    sudo mkdir -p /mnt/newdisk
    sudo mount /dev/nvme0n2 /mnt/newdisk
    ```
Persist via UUID in /etc/fstab
**Make persistent across reboots** — add to `/etc/fstab`

    ```bash
    # Get the UUID
    sudo blkid /dev/<vg_name>/<new_lv_name>

    # Add entry (use UUID, never raw device path)
    echo "UUID=<paste-uuid-here>  /mnt/newdisk  xfs  defaults  0 0" | sudo tee -a /etc/fstab

    # Validate fstab syntax (CRITICAL — prevents unbootable system)
    sudo findmnt --verify --verbose
    ```

### Verification Checklist

| Check | Command | Expected Result |
| :--- | :--- | :--- |
| Disk recognized | `lsblk` | New disk visible under VG/LV |
| PV created | `pvs` | New disk listed as PV |
| VG extended | `vgs` | VFree = 0 (if fully used) |
| LV sized correctly | `lvs` | Size reflects new total |
| Filesystem grown | `df -hT` | Shows expanded size |
| Survives reboot | `sudo mount -a` | No errors returned |

### Important Notes for RHEL 10

-   **XFS cannot shrink.** If you allocate all free space and later need to reduce, you must back up, destroy, and recreate. Consider allocating only what you need now (`-L 50G` instead of `+100%FREE`) if future flexibility matters.
-   **NVMe devices** appear as `/dev/nvme0n1` (no partition suffix needed for whole-disk PV).
-   **SELinux context**: If mounting to a non-standard path for a service, you may need to set SELinux labels:
    ```bash
    sudo semanage fcontext -a -t httpd_sys_content_t "/mnt/newdisk(/.*)?"
    sudo restorecon -Rv /mnt/newdisk
    ```
-   Always run `sudo findmnt --verify` after editing `/etc/fstab`. A malformed fstab entry can make RHEL drop into emergency mode on next boot.
