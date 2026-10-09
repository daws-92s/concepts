# Add & Mount an EBS Disk on an EC2 Instance (RHEL / Amazon Linux)

This guide explains how to attach, partition, format, mount, and persist an additional EBS disk on an EC2 instance.

---

### Assumptions

- A new EBS volume is already attached to the EC2 instance.
- The new disk appears as:  
  `/dev/nvme1n1`
- You want to mount it at:  
  `/data`
- OS: RHEL / Amazon Linux  
- Filesystem: XFS (recommended for AWS)

---

### 1. Verify the Disk Is Visible

```bash
lsblk
```

You should see something like:

```text
nvme1n1   259:1   0   25G  0 disk
```

---

## 2. Create a Partition (Non-Interactive – Automation Safe)

```bash
sudo parted /dev/nvme1n1 --script mklabel gpt mkpart primary 0% 100%
```

### What this does

- `--script`  
  Runs non-interactively (no confirmation prompts).  
  Safe for automation (Ansible / Terraform / cloud-init).

- `mklabel gpt`  
  Creates a **GUID Partition Table**.  
  GPT is the modern standard (supports large disks and better reliability).

- `mkpart primary 0% 100%`  
  Creates a single partition using the **entire disk**.

After this, the new partition will appear as:

```text
/dev/nvme1n1p1
```

---

## 3. Create a Filesystem (XFS – Best for AWS)

```bash
sudo mkfs.xfs -f /dev/nvme1n1p1
```

### Why XFS?

- Default filesystem for RHEL  
- Excellent for:
  - Large files  
  - Logs  
  - High I/O workloads  
- Supports online growth  
- Enterprise-grade stability  

> **Warning:** `-f` forces formatting (safe only for new disks).

---

## 4. Create a Mount Directory

```bash
sudo mkdir -p /data
```

---

## 5. Mount the Disk (Temporary Runtime Mount)

```bash
sudo mount /dev/nvme1n1p1 /data
```

Verify:

```bash
df -hT | grep data
```

Expected output:

```text
/dev/nvme1n1p1   xfs   25G   ...   /data
```

---

## 6. Get the Disk UUID (Permanent Identity)

```bash
sudo blkid /dev/nvme1n1p1
```

Example output:

```text
/dev/nvme1n1p1: UUID="d68b606a-4df3-44c2-bd74-418fa36e9fa2" TYPE="xfs"
```

> **Warning:** Never use `/dev/nvme1n1p1` directly in fstab.  
> Device names can change after reboot in AWS.

---

## 7. Add Persistent Mount Entry (`/etc/fstab`)

Open the file:

```bash
sudo vim /etc/fstab
```

Add this line at the bottom:

```text
UUID=d68b606a-4df3-44c2-bd74-418fa36e9fa2   /data   xfs   defaults,nofail   0   2
```

---

## 8. Meaning of the fstab Line

```text
UUID=xxxx   /data   xfs   defaults,nofail   0   2
```

| Field | Meaning |
|-------|--------|
| UUID=xxxx | Unique disk ID (safe, permanent) |
| /data | Mount point |
| xfs | Filesystem type |
| defaults | Standard mount options |
| nofail | Don’t break boot if disk is missing |
| 0 | Don’t include in legacy dump backups |
| 2 | fsck order (after root `/`) |

---

## 9. Test the Configuration (No Reboot Needed)

```bash
sudo mount -a
```

If there are **no errors**, your fstab entry is correct.

Verify:

```bash
df -hT | grep data
```

---

## Optional: Reboot Test (Final Safety Check)

```bash
sudo reboot
```

After reboot:

```bash
df -hT | grep data
```

If `/data` still appears → success.

---

## Key Production Notes

- Always mount using **UUID**, never device names.
- Always use `nofail` for EC2 data disks.
- Never store logs or DB data on the root disk.
- Always test with `mount -a` before reboot.
- Automate these steps using Ansible for consistency.

---

## Summary

```bash
# Create partition
sudo parted /dev/nvme1n1 --script mklabel gpt mkpart primary 0% 100%

# Format disk
sudo mkfs.xfs -f /dev/nvme1n1p1

# Mount disk
sudo mkdir -p /data
sudo mount /dev/nvme1n1p1 /data

# Get UUID
sudo blkid /dev/nvme1n1p1

# Add to fstab
sudo vim /etc/fstab
UUID=xxxx   /data   xfs   defaults,nofail   0   2

# Test mount
sudo mount -a
```

# Resize disk

check the partitions
```
lsblk
```

Use a tool like growpart to resize the existing partition to fill the available space.This will resize the fourth partition (nvme0n1p4) to use the remaining unallocated space on the disk.
```
sudo growpart /dev/nvme0n1 4
```

Extend the Logical Volumes
Decide how much space to allocate to each logical volume. For example, to extend both the root and /var logical volumes:

```
sudo lvextend -L +20G /dev/RootVG/rootVol
sudo lvextend -L +30G /dev/RootVG/varVol
```

After extending the logical volumes, resize the filesystems to utilize the additional space.

For the root filesystem:

```
sudo xfs_growfs /
```

For the /var filesystem:

```
sudo xfs_growfs /var
```