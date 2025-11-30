# Ch4: Navigating and Essential Commands - Filesystems in Linux

## What is a Filesystem in Linux?

A **filesystem** is an organizational system that stores, retrieves, and manages data on storage devices (disks, SSDs, etc.). It's essentially a hierarchical database that answers three core questions:

1. **Where is the data?** (physical location on the disk)
2. **What is the data?** (files, directories, metadata)
3. **Who can access it?** (permissions and ownership)

## Why "Filesystem"?

The name is literal: it's a **system for managing files**. The term emerged because the alternative (storing raw bytes without organization) is useless. The filesystem imposes structure on chaotic storage.

### Analogy: Filing Cabinet
- **Without filesystem**: Bytes dumped randomly on disk = useless noise
- **With filesystem**: Organized structure with addresses, names, permissions = usable data

## Core Linux Principle: "Everything is a File"

This is the Unix/Linux foundational philosophy. A filesystem isn't just for files—it abstracts **everything** as a file-like interface:

```
/dev/sda           → Hardware disk (character/block device)
/proc/cpuinfo      → Running kernel data (virtual filesystem)
/sys/class/net     → System hardware info (virtual filesystem)
/var/log/auth.log  → Text log file (regular file)
/home/user/        → Directory (special file)
/tmp/socket        → Inter-process communication (socket)
```

This unified abstraction is powerful: same tools (`cat`, `grep`, `dd`) work across hardware, processes, and data because everything follows the file interface.



## Production Relevance

**Why this matters at 2AM in production:**

- **Inode Exhaustion**: Understanding inodes helps diagnose "disk full" errors that might be caused by inode exhaustion rather than actual byte usage
- **Filesystem Choice**: Different filesystems have different performance characteristics
  - ext4: Reliable, journaled, widely adopted
  - XFS: Better for large files and database workloads
  - btrfs: Modern with snapshots and RAID capabilities
- **Mount Options**: Security and reliability (noexec, nosuid, nodev for /tmp)
- **Journaling**: Prevents corruption on crashes
- **Snapshots**: Enable backups without downtime

## Filesystems Across Operating Systems

### Comparison Table

| Aspect | Linux | Unix | Windows |
|--------|-------|------|---------|
| **Common filesystems** | ext4, btrfs, XFS | UFS, FFS, ZFS | NTFS, ReFS |
| **Path separator** | `/` | `/` | `\` |
| **Case sensitivity** | Yes (by default) | Yes (by default) | No (case-insensitive) |
| **Permissions model** | POSIX (rwx for owner/group/other) | POSIX (rwx) | ACLs (Access Control Lists) |
| **Hard links** | Yes | Yes | Limited |
| **Symlinks** | Yes, native | Yes, native | Yes (but different) |
| **File locking** | Advisory | Advisory | Mandatory |
| **Journaling** | Yes (ext4, XFS) | Optional (modern) | Yes (NTFS) |

### Design Philosophy Differences

**Linux/Unix**: "Everything is a File"
- Unified abstraction across hardware, processes, and data
- Same tools work on files, devices, pipes, sockets, and processes

**Windows**: Layered Abstraction
- Specialized interfaces for different resources
- Hardware, processes, and network are separate abstractions

### Key Architectural Differences

#### 1. Permissions Model
- **Linux/Unix (POSIX)**: Simple (`rwx` for owner/group/others) but coarse-grained
- **Windows (ACLs)**: Fine-grained, complex, more powerful

#### 2. Case Sensitivity
- **Linux/Unix**: `file.txt ≠ FILE.TXT` (three different files possible)
- **Windows**: `file.txt == FILE.TXT` (preserves case but ignores it)

#### 3. Hard Links
- **Linux/Unix**: Full support, multiple directory entries point to same inode
- **Windows**: Poorly supported, applications often don't expect them

#### 4. File Locking
- **Linux/Unix (Advisory)**: Process locks are suggestions, cooperative locking
- **Windows (Mandatory)**: When a file is locked, other processes cannot access it

#### 5. Path Semantics
- **Linux/Unix**: Single root filesystem (`/`)
- **Windows**: Multiple roots (C:, D:, etc.), drive-letter dependent

### Production Implications

- **Permission Complexity**: Windows ACLs are powerful but opaque
- **Case Sensitivity**: Docker containers (Linux) on Windows cause path issues
- **File Locking**: Windows locks exclusively, Linux can rename/delete open files
- **Symlinks**: Linux symlinks are native and fast; Windows symlinks work differently
- **Performance**: Different optimization targets (servers vs. consumer/office)
- **Consistency Model**: Different journaling and failure modes

---

## Understanding Inodes in Linux

### What is an Inode?

An **inode** (index node) is a data structure that stores all metadata about a file **EXCEPT its name**.

```
inode 12345 {
  type: regular file
  owner (uid): 1000
  group (gid): 1000
  permissions: 644 (rw-r--r--)
  size: 4096 bytes
  
  timestamps:
    created (ctime): 2024-01-15 10:30:00
    modified (mtime): 2024-01-20 14:22:00
    accessed (atime): 2024-11-30 16:45:00
  
  link_count: 1
  block_pointers: [87654, 87655, 87656, ...]
}
```

**Key insight**: The filename is just a pointer to the inode number. The inode is where the actual metadata lives.

### Directory Structure: A Lookup Table

```
Directory /home/alice/
├─ file1.txt      → inode 1001
├─ document.pdf   → inode 1002
├─ notes/         → inode 1003 (special inode, it's a directory)
└─ .config        → inode 1004
```

When you access `/home/alice/file1.txt`:
1. Kernel reads root directory (inode 2)
2. Finds "home" → inode 500
3. Reads inode 500 (directory)
4. Finds "alice" → inode 501
5. Reads inode 501 (directory)
6. Finds "file1.txt" → inode 1001
7. Reads inode 1001 (file metadata)
8. Uses block_pointers to fetch actual data from disk

### Viewing Inode Information

```bash
# See inode number and file info
ls -i /home/user/file.txt
1048576 /home/user/file.txt

# Get detailed inode data
stat /home/user/file.txt
  File: /home/user/file.txt
  Inode: 1048576
  Links: 1
  Access: (0644/-rw-r--r--)
  Uid: ( 1000/user)  Gid: ( 1000/user)
  Size: 4096
  Blocks: 8
```

### The Block Pointers

Inodes don't store file data directly. They store **pointers to data blocks** on disk:

```
inode 1001 {
  direct[0]: block 87654      ← First 4KB of file
  direct[1]: block 87655      ← Second 4KB of file
  ...
  indirect: inode 2000        ← Points to more blocks (for large files)
  double_indirect: inode 2001 ← Points to indirects (for huge files)
}
```

This allows:
- **Small files**: Direct pointers are fast
- **Large files**: Indirect pointers scale efficiently

### Hard Links vs Symlinks

**Hard link**: Another directory entry pointing to the same inode
```bash
ln file.txt link.txt
ls -i
1048576 file.txt
1048576 link.txt     ← Same inode number

# Delete original, hard link still works
rm file.txt
cat link.txt         ← Data still there
```

**Symlink**: A directory entry pointing to a filename (different inode)
```bash
ln -s file.txt sym.txt
ls -i
1048576 file.txt
2048000 sym.txt      ← Different inode

# Delete original, symlink breaks
rm file.txt
cat sym.txt          ← Broken! No such file or directory
```

### Link Count

Inodes track how many directory entries point to them:

```bash
touch file.txt
stat file.txt | grep Links
Links: 1

# Create a hard link
ln file.txt link.txt
stat file.txt | grep Links
Links: 2

# File data is only deleted when link_count reaches 0
rm file.txt  # Decrements link_count
# Data still exists because link_count is 2
```

### How Inodes Enable Fast Operations

| Operation | Impact |
|-----------|--------|
| **Rename file** | Only update directory entry, inode never moves (O(1)) |
| **Change permissions** | Only update inode bits, no data copy (milliseconds) |
| **Create hard link** | Just add directory entry, no data copy (instant) |
| **Delete file** | Decrement link_count, data freed when count reaches 0 (O(1)) |

### Inode Exhaustion: The 2AM Disaster

You can run out of inodes before running out of disk space:

```bash
# Plenty of space, but inode problem
df -h /var
Filesystem      Size  Used Avail Use%
/dev/sda2       100G   50G   50G  50%    ← 50GB free!

# But inodes exhausted
df -i /var
Filesystem     Inodes IUsed IFree IUse%
/dev/sda2     1000000 999999 1    100%  ← 100% inodes used!

# Try to create a file
touch /var/log/newfile.log
touch: cannot touch '/var/log/newfile.log': No space left on device
# Misleading error - it's not bytes, it's inodes!

# Find inode consumers
find /var/log -type f | wc -l
999999
# Ah, old log files never deleted

# Clean up
find /var/log -type f -mtime +30 -delete
```

### Inode Table (The Database)

Every filesystem has an inode table stored on disk:

```
ext4 filesystem:
┌─────────────────────┐
│  Superblock         │ ← Metadata about filesystem
│  Group Descriptors  │ ← Info about inode groups
│  Inode Table        │ ← All inodes (database)
│  Data Blocks        │ ← Actual file content
└─────────────────────┘
```

Check inode count:
```bash
df -i /home
Filesystem     Inodes IUsed IFree IUse%
/dev/sda1    1000000 50000 950000   5%

tune2fs -l /dev/sda1 | grep -i inode
Inode count: 1024000
Free inodes: 950000
```

---

## Key Takeaway

The filesystem is where the rubber meets the road—it's the contract between the application and the hardware. Understand inodes, permissions models, and filesystem design choices, and you'll troubleshoot disk-related issues with confidence. From cross-platform compatibility (case sensitivity, paths) to capacity planning (inode exhaustion), filesystem knowledge is essential for production systems.
