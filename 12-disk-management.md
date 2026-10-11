# Disk Management, Inodes, and the Linux Directory Layout

## What this is / why it matters
Every file on a Linux server lives on a disk, and the OS tracks it in two separate pieces: the actual content, and the metadata describing it (owner, permissions, size, timestamps). Understanding that split explains how shortcuts (symlinks) and backup-style links (hardlinks) actually work. Knowing how disks actually get attached, partitioned, mounted, and later resized explains why a server's root disk shouldn't be where application data lives — and what to do once that data disk needs more room.

## How it works

### Inodes: a file's metadata, separate from its content
Think of a person's ID record — name, date of birth, Aadhar number — versus the person themselves. An **inode** is that ID record for a file: who owns it, which group has access, what permissions it has, when it was created and last updated, and its size. The file's actual content is stored separately — it isn't part of the inode.

```
ls -li       # shows every file's inode number
stat <file>  # shows full inode metadata for one file
```

### Symlinks (soft links) — a shortcut
A symlink is a pointer to another file or folder, the same idea as a desktop shortcut:
- Has its **own, different inode number** from the file it points to
- If the actual (target) file is removed, the symlink breaks — it was only ever pointing at it, not holding it
- Common uses: backward compatibility, and clean version switching during an upgrade — point a stable path at whichever version is currently active:
```
/home/ec2-user/prometheus  →  /home/ec2-user/prometheus-3.15.0.linux-amd64/prometheus
```
Upgrading later just means re-pointing `prometheus` at a new version's folder — nothing that references the stable `prometheus` path has to change.

### Hard links — another name for the same file
A hard link doesn't point at a file — it *is* the file, under a second name:
- Shares the **exact same inode number** as the original
- Can't be created for directories, only individual files
- Used for backup-style redundancy: deleting one of the names doesn't delete the file's actual content — the content isn't fully removed until every hard-linked reference to that inode is deleted

| | Symlink | Hard link |
|---|---|---|
| Inode number | Different from the target | Same as the target |
| Breaks if target is deleted? | Yes | No — content persists until every link is gone |
| Works on directories? | Yes | No |
| Typical use | Shortcuts, version switching | File-level backup redundancy |

### Adding and managing disks
An EC2 instance comes with one **root disk** by default — and deleting the instance deletes that root disk along with it. That's exactly why application data shouldn't live on the root disk: attach a separate disk (an EBS volume) for it, so the OS and the app's data have independent lifecycles. The root disk holds OS configuration; the extra disk holds the application's own data.

An EBS volume has to be created in the **same Availability Zone** as the instance it attaches to (e.g. `us-east-1d`) — a volume is physically tied to a location, so it has to be near the server using it.

A **filesystem** is what makes a disk usable at all: a disk only stores raw 0s and 1s, and the filesystem is the layer responsible for organizing those bytes into the files and folders a user actually sees.

Once a new EBS volume is attached to the instance (e.g. it shows up as `/dev/nvme1n1`), getting it ready to actually store data is a fixed sequence — skipping or reordering a step here is the most common source of "the disk isn't showing up" problems:

**1. Confirm the disk is visible to the OS:**
```bash
lsblk
```
```text
nvme1n1   259:1   0   25G  0 disk
```

**2. Partition the disk:**
```bash
sudo parted /dev/nvme1n1 --script mklabel gpt mkpart primary 0% 100%
```
- `--script` runs this non-interactively, with no confirmation prompt — needed for automation (Ansible, Terraform, cloud-init).
- `mklabel gpt` creates a GUID Partition Table — the modern standard, which supports large disks and is more reliable than the older MBR format.
- `mkpart primary 0% 100%` creates a single partition spanning the entire disk.

This creates a new device representing that partition: `/dev/nvme1n1p1` (note the `p1` — this is the partition, not the raw disk from step 1).

**3. Create a filesystem on the partition:**
```bash
sudo mkfs.xfs -f /dev/nvme1n1p1
```
XFS is the default filesystem on RHEL and a solid general choice — good for large files, logs, and high-I/O workloads, and it supports growing the filesystem later without unmounting it. The `-f` flag forces formatting: safe on a brand-new, empty disk; destructive on one that already holds data, since formatting wipes it.

**4. Create a mount point, and mount it:**
```bash
sudo mkdir -p /data
sudo mount /dev/nvme1n1p1 /data
```
Verify it worked:
```bash
df -hT | grep data
```

**5. Get the disk's UUID:**
```bash
sudo blkid /dev/nvme1n1p1
```
```text
/dev/nvme1n1p1: UUID="d68b606a-4df3-44c2-bd74-418fa36e9fa2" TYPE="xfs"
```
Device names like `/dev/nvme1n1p1` can shift after a reboot on AWS — the UUID doesn't, which is why the next step uses it instead of the device name.

**6. Make the mount persistent, in `/etc/fstab`:**
```bash
sudo vim /etc/fstab
```
Add a line at the bottom:
```text
UUID=d68b606a-4df3-44c2-bd74-418fa36e9fa2   /data   xfs   defaults,nofail   0   2
```

| Field | Meaning |
|---|---|
| `UUID=...` | The disk's permanent, unique ID — safer than a device name |
| `/data` | Where it gets mounted |
| `xfs` | Filesystem type |
| `defaults` | Standard mount options |
| `nofail` | Don't block the server from booting if this disk is missing or slow to attach |
| `0` | Skip this filesystem in legacy `dump` backups |
| `2` | `fsck` check order — `2` means checked after the root filesystem (`1`) |

**7. Test the fstab entry without rebooting:**
```bash
sudo mount -a
df -hT | grep data
```
If this runs with no errors, the entry is correct. It's still worth doing a one-time reboot test (`sudo reboot`, then confirming `/data` is still mounted) before trusting this in production — catching an fstab mistake at a planned reboot beats discovering it during an unplanned one.

**Rules of thumb for production disks:**
- Always mount by UUID, never by device name.
- Always include `nofail` for a data disk — losing one non-critical disk shouldn't take the whole server down at boot.
- Never store logs or database data on the root disk.
- Always run `mount -a` to test a new fstab entry before rebooting.

### Resizing an existing disk
Growing an existing disk, instead of adding a new one, is a layered operation — the EBS volume, the partition on it, the logical volume (if LVM is used), and the filesystem each have to be grown in turn. Stopping partway through (e.g. growing the volume in AWS but nothing else) leaves the extra space allocated but completely unusable.

**1. Check the existing partitions:**
```bash
lsblk
```

**2. Grow the partition** to fill space that's newly available (e.g. after increasing the EBS volume's size in AWS) — this example grows partition 4 on `/dev/nvme0n1`:
```bash
sudo growpart /dev/nvme0n1 4
```

**3. Extend the logical volume(s)** — on an LVM setup, a bigger partition doesn't automatically mean a bigger logical volume on top of it:
```bash
sudo lvextend -L +20G /dev/RootVG/rootVol
sudo lvextend -L +30G /dev/RootVG/varVol
```

**4. Grow the filesystem itself**, so it actually uses the newly extended space:
```bash
sudo xfs_growfs /
sudo xfs_growfs /var
```

### Checking disk and memory usage
```
df -hT     # disk space and filesystem type, per mounted disk
free -m    # RAM usage
du -sh *   # disk usage of each item in the current directory
```

### The Linux directory layout
Every one of these exists for a specific reason — none of it is arbitrary:

| Directory | What it's for |
|---|---|
| `/bin` | The commands used every day (`ls`, `cp`, etc.) |
| `/boot` | Files needed to boot the server — the kernel (`vmlinuz-*`), the initial RAM disk (`initramfs`), and the boot loader (GRUB2) |
| `/dev` | Devices — disks, keyboard, mouse, and everything else the kernel exposes as a device |
| `/etc` | System-wide configuration — `/etc/passwd`, `/etc/group`, `/etc/nginx`, `/etc/yum.repos.d`, `/etc/systemd`, `/etc/sudoers`, and more |
| `/home` | Each regular user's personal space, e.g. `/home/ec2-user` |
| `/lib` | Shared system libraries |
| `/media` | Where removable media (USB drives, CDs) auto-mounts on a desktop system |
| `/mnt` | The conventional place to manually mount an extra disk |
| `/opt` | Where third-party software gets installed, by convention, when it's not managed by the package manager |
| `/proc` | A virtual directory the kernel uses to expose live process and system info, e.g. `/proc/cpuinfo`, `/proc/meminfo` |
| `/root` | The root user's home directory |
| `/run` | Where running programs track their own process state |
| `/sbin` | System administration commands |
| `/srv` | Originally intended for data served by services on the box (e.g. `/usr/share/nginx/html`); rarely used directly today |
| `/tmp` | Temporary files |
| `/var` | Variable data that changes constantly — most importantly, logs (`/var/log`) |

Swap space (virtual RAM backed by disk) kicking in is a signal worth knowing on its own — if swap usage is climbing, it means physical RAM is already full.

## Common problems and how to solve them
- **A symlink pointing at nothing** — if the target file was deleted or moved, the symlink still exists but is now "dangling." Fix: re-create the target, or re-point the symlink.
- **Trying to hard-link a directory** — not supported by design. Use a symlink instead if a directory needs a shortcut.
- **A mounted disk disappears after a reboot** — it was mounted manually with `mount` but never added to `/etc/fstab`, so the mount didn't persist. Fix: add the proper `/etc/fstab` entry, using the disk's UUID.
- **Server fails to boot after adding a disk entry to `/etc/fstab`** — usually means the `nofail` option was left out and that disk wasn't available in time at boot. Add `nofail` so a missing/slow disk doesn't block the entire boot.
- **Growing an EBS volume in AWS but the extra space never shows up on the server** — resizing the volume itself doesn't automatically resize anything on top of it. Each layer needs its own resize step: the partition (`growpart`), the logical volume if LVM is used (`lvextend`), and finally the filesystem (`xfs_growfs`). Skipping any one of them leaves the space allocated but inaccessible.

## Key takeaways
- An inode holds a file's metadata (owner, permissions, timestamps, size) — never the file's actual content.
- A symlink points at a file (different inode, breaks if the target disappears); a hard link *is* the file under another name (same inode, survives until every link referencing it is gone, and only works on files, not directories).
- Keep application data on a separate disk from the root disk — the root disk disappears with the instance; a separate EBS volume doesn't, and it has to live in the same AZ as the instance using it.
- `df -hT`, `free -m`, and `du -sh *` are the quick checks for disk space, RAM, and per-directory usage, respectively.
- Adding a disk is a fixed sequence — partition it, put a filesystem on it, mount it, then make that mount permanent in `/etc/fstab` by UUID. Resizing one is a layered sequence instead — the EBS volume, partition, logical volume, and filesystem each need their own resize step.
- Every top-level Linux directory has one specific job — `/etc` for config, `/var` for logs and changing data, `/opt` for third-party installs, `/mnt` for manually mounted disks, and so on.

See also: [06-linux-admin.md](06-linux-admin.md), [04-linux.md](04-linux.md)
