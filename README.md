# Linux Disk Setup for Linux

## Phase 1: Identify and Verify the New Disk

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
```

> **Note:** Visually identify your new disk. It will have no FSTYPE, no MOUNTPOINT, and no child partitions. Common names are `/dev/sdb`, `/dev/nvme1n1`, or `/dev/vdb`.

```bash
# REPLACE <DEVICE> with actual disk name (e.g., /dev/sdb)
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT <DEVICE>
```

> **Note:** Safety check. This must show a single line with empty FSTYPE and MOUNTPOINT columns. If it shows partitions or a filesystem, you have selected the wrong disk. Stop immediately.

## Phase 2: Wipe and Format

```bash
sudo wipefs -a <DEVICE>
```

> **Note:** Erases all partition tables, RAID metadata, and filesystem signatures from the entire disk. The `-a` flag is required to remove all signature types. This is irreversible.

```bash
sudo mkfs.xfs <DEVICE>
```

> **Note:** Creates an XFS filesystem directly on the whole disk without a partition table. Do not append `p1` or any partition suffix. XFS is the default and recommended filesystem for RHEL/AlmaLinux.

## Phase 3: Prepare Mount Point and Capture UUID

```bash
# REPLACE <MOUNT_PATH> with desired directory (e.g., /mnt/mydisk)
sudo mkdir -p <MOUNT_PATH>
```

> **Note:** Creates the mount directory before modifying fstab. The `-p` flag prevents errors if parent directories don't exist or if the directory already exists. This step must happen before `findmnt --verify` or validation will fail with "target does not exist."

```bash
UUID=$(sudo blkid -s UUID -o value <DEVICE>)
echo "Detected UUID: ${UUID}"
test -n "$UUID" || echo "ERROR: UUID is empty. Stop now."
```

> **Note:** Extracts only the UUID string from the whole disk device and validates it. Never query a partition like `<DEVICE>p1` when using the whole-disk method — that device does not exist and will return an empty string. If the UUID is blank, do not proceed. An empty `UUID=` in fstab causes emergency mode on next boot.

## Phase 4: Persist Mount and Validate

```bash
echo "UUID=${UUID}  <MOUNT_PATH>  xfs  defaults  0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
```

> **Note:** Forces systemd to re-read fstab. Without this, `findmnt --verify` may report stale warnings about modified fstab even though the file is correct.

```bash
sudo findmnt --verify --verbose
```

> **Note:** Validates every fstab entry for syntax, source existence, target existence, and filesystem type match. You must see 0 parse errors and 0 errors before proceeding. If any error appears, fix it now — do not run `mount -a`.

## Phase 5: Mount and Confirm

```bash
sudo mount -a
```

> **Note:** Mounts all unmounted fstab entries. Returns silently on success. If it prints an error, the fstab entry is still broken despite passing `findmnt` — investigate before rebooting.

```bash
df -hT <MOUNT_PATH>
```

> **Note:** Final confirmation. Should show your new disk's size, XFS as the type, and the correct mount point. After a successful reboot test, this setup is permanent and automatic.

---

## Undo and Clean Section

```bash
sudo umount <MOUNT_PATH>
```

> **Note:** Unmounts the filesystem. Must be done before editing fstab or wiping the disk. If it reports "target is busy," run `sudo lsof +D <MOUNT_PATH>` to find processes using the mount and stop them first.

```bash
sudo sed -i '\|<MOUNT_PATH>|d' /etc/fstab
sudo systemctl daemon-reload
```

> **Note:** Reloads systemd after fstab modification. Prevents stale mount unit warnings and ensures the system won't attempt to mount the removed entry on next boot.

```bash
sudo wipefs -a <DEVICE>
sudo rmdir <MOUNT_PATH>
```

> **Note:** Removes the now-empty mount directory. Only works if the directory is truly empty. If it fails with "Directory not empty," data was written to the mount before unmounting — that data is gone after `wipefs`, so you can safely force-remove with `sudo rm -rf <MOUNT_PATH>` if you are certain.
