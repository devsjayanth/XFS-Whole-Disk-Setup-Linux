# Linux Whole-Disk XFS Setup

> ⚠️ **Warning:** This formats an entire disk without a partition table. XFS cannot be shrunk. Never run this on your OS/boot disk.

### 1. Identify & Safety Check
```bash
# Find your disk. Cross-reference SIZE and SERIAL to be certain.
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,SERIAL

# REPLACE <DEVICE> (e.g., /dev/sdb). Must return exactly "1".
# If it returns anything else, STOP — you selected the wrong device.
lsblk -no TYPE <DEVICE> | grep -c '^disk$'
```

### 2. Wipe & Format with Unique UUID
```bash
sudo wipefs -a <DEVICE>

# -f forces overwrite; explicit uuidgen prevents duplicate-UUID mount failures
sudo mkfs.xfs -f -m uuid=$(uuidgen) <DEVICE>
```

### 3. Create Mount Point & Validate UUID
```bash
sudo mkdir -p /mnt/mydisk

UUID=$(sudo blkid -s UUID -o value <DEVICE>)
# Halt if UUID is missing or malformed — empty UUID = emergency mode on boot
[[ "$UUID" =~ ^[0-9a-f-]{36}$ ]] || { echo "ERROR: Invalid UUID"; exit 1; }
echo "UUID: $UUID"
```

### 4. Update fstab Safely & Verify
```bash
# Backup fstab FIRST
sudo cp /etc/fstab /etc/fstab.bak.$(date +%s)

# Append entry with noatime (reduces write amplification on data disks)
echo "UUID=$UUID  /mnt/mydisk  xfs  defaults,noatime  0 0" | sudo tee -a /etc/fstab

sudo systemctl daemon-reload

# MUST show 0 errors before proceeding
sudo findmnt --verify --verbose
```

### 5. Mount & Confirm Writability
```bash
sudo mount -a

# Verify type, options, AND that the disk is actually writable
findmnt -n -o FSTYPE,OPTIONS /mnt/mydisk
sudo touch /mnt/mydisk/.write_test && sudo rm /mnt/mydisk/.write_test && echo "Ready"
```

---

### Undo / Remove Disk
```bash
sudo umount /mnt/mydisk

# Match on UUID, NOT mount path — prevents accidentally deleting other fstab entries
sudo sed -i "\|^UUID=$UUID|d" /etc/fstab
sudo systemctl daemon-reload

sudo wipefs -a <DEVICE>
sudo rmdir /mnt/mydisk   # Fails safely if not empty — inspect before force-deleting
```
