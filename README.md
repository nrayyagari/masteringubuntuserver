# Mastering Ubuntu Server

Learning materials and practical examples based on **"Mastering Ubuntu Server: Gain expertise in the art of deploying, configuring, managing, and troubleshooting Ubuntu Server"** (3rd Edition) by Jay LaCroix.

A hands-on resource for sysadmins and DevOps professionals to deploy, configure, maintain, and troubleshoot Ubuntu Server in production environments.

## About This Book

This third edition is updated to cover Ubuntu Server 20.04 LTS advancements and covers everything from initial deployment to creating production-ready resources for your network.

Written by Jay LaCroix (LearnLinuxTV), a Solutions Architect with 20+ years of experience and creator of an award-winning Linux education channel.

**Publisher:** Packt Publishing Limited  
**Publication Date:** December 29, 2020  
**Pages:** 702  
**ISBN:** 978-1800564640

## Key Topics

### Core Administration
- Deploying Ubuntu Server
- Managing Users and Permissions
- Managing Software Packages
- Navigating and Essential Commands
- Managing Files and Directories
- Boosting Command-line Efficiency

### System Operations
- Controlling and Managing Processes
- Monitoring System Resources (CPU, Memory, Disk I/O)
- Managing Storage Volumes (Disks, Partitions, Mounting)
- Connecting to Networks
- Setting up Network Services

### File Sharing & Data
- Sharing and Transferring Files (Samba, NFS)
- Managing Databases
- Serving Web Content

### Automation & Infrastructure
- Automating Server Configuration with Ansible
- Automating Cloud Deployments with Terraform
- Virtualization (QEMU & KVM)
- Running Containers (Docker and LXD)
- Container Orchestration with MicroK8s and Kubernetes

### Cloud & Security
- Deploying Ubuntu in the Cloud
- Securing your Server
- Disk Encryption with LUKS
- Secure Shell (SSH) Setup

### Advanced Topics
- Troubleshooting Ubuntu Servers
- Preventing Disasters and Disaster Recovery
- Real-world Production Scenarios

## What You'll Learn

- Deploy and manage Ubuntu Server 20.04 LTS
- Manage users, groups, and file system permissions
- Optimize system resource performance
- Implement disk encryption with LUKS
- Configure SSH for secure remote access
- Share directories using Samba and NFS
- Set up and manage virtual machines and containers
- Automate deployments with Ansible and Terraform
- Implement best practices and troubleshooting techniques

## Prerequisites

- Basic Linux command line knowledge
- IT administration experience
- Shell scripting fundamentals
- Prior Ubuntu knowledge not required

## Target Audience

- System administrators planning Ubuntu/Linux deployments
- DevOps professionals setting up infrastructure
- IT professionals transitioning to Linux servers
- Intermediate-level users seeking practical production knowledge

## Storage Concepts Explained

### What is a Disk vs Volume vs Mount?

**DISK = Physical Hardware**
- `/dev/sda` - First physical hard drive (the actual device)
- `/dev/sdb` - Second physical hard drive
- The raw storage hardware itself

**VOLUME = Logical Partition**
- `/dev/sda1` - First partition (logical section) on disk sda
- `/dev/sda2` - Second partition on the same disk
- Formatted with a filesystem (ext4, XFS, etc.)
- Think: dividing a physical disk into logical sections

**MOUNTING = Access Point**
- The process of attaching a volume to your file system
- Makes a volume accessible at a specific directory
- Example: `mount /dev/sda1 /home` connects sda1 to `/home` directory

**How They Work Together:**
```
Physical Layer:    [/dev/sda - Physical Hard Drive]
                        ↓
Logical Layer:     [/dev/sda1 (ext4)] [/dev/sda2 (ext4)]
                        ↓
Access Layer:      mounted at /home   mounted at /var
                        ↓
You Access:        Files under /home/  Files under /var/
```

### Disk Monitoring (Persistent Storage Only)
When monitoring "disk," you're checking persistent storage, NOT RAM:
- `df` - Shows how full each mounted volume is
- `du` - Shows how much space files/directories use
- `iostat` - Shows disk I/O activity (read/write operations)
- These monitor your actual storage devices, not memory

### Understanding Linux Device Naming

Linux uses a consistent naming scheme inherited from SCSI standards:

**`sda` Breakdown:**
- `sd` = SCSI Disk (historical naming, used for all disk types today)
- `a` = First device (a=1st, b=2nd, c=3rd, etc.)

**`sda1` Breakdown:**
- `sda` = The physical disk
- `1` = Partition number (1st partition on that disk)

**Examples:**
```
/dev/sda    = First physical hard drive
/dev/sdb    = Second physical hard drive
/dev/sda1   = First partition on /dev/sda
/dev/sda2   = Second partition on /dev/sda
/dev/sdb1   = First partition on /dev/sdb
/dev/nvme0n1 = NVMe drive (different naming for fast SSDs)
/dev/nvme0n1p1 = First partition on NVMe drive
```

### Is Mounting Mandatory?

**Technically:** No—you can access `/dev/sda1` as raw data without mounting.  
**Practically:** Yes—mounting makes a volume usable. Without mounting, you can't browse files or interact normally.

**What mounting does:**
- Connects a partition to your file system at a mount point
- Allows normal file operations (read, write, browse)
- Makes the volume accessible as part of the directory tree

### Creating Partitions from Scratch

**Tools Available:**
- `parted` - Recommended (works with GPT and MBR)
- `fdisk` - Simpler for MBR, older standard
- `gdisk` - For GPT-only systems

**Complete Workflow: Create sda1 on a new disk**

**Step 1: Verify the disk you're working with**
```bash
lsblk                    # Visual view
sudo parted -l           # Detailed list
sudo fdisk -l /dev/sda   # Info about /dev/sda
```

**Step 2: Create partition table**
```bash
# Choose ONE based on your needs:
sudo parted /dev/sda mklabel gpt    # Modern (supports large disks, UEFI)
# OR
sudo parted /dev/sda mklabel msdos  # Legacy MBR (older but widely compatible)
```

**Step 3: Create partitions**
```bash
# Using parted (recommended):
sudo parted /dev/sda mkpart primary ext4 0% 50%   # Creates sda1 (50% of disk)
sudo parted /dev/sda mkpart primary ext4 50% 100% # Creates sda2 (remaining)

# OR using fdisk (interactive):
sudo fdisk /dev/sda
# Inside fdisk: n (new) → p (primary) → 1 (partition 1) → sizes → w (write)
```

**Step 4: Format the partition**
```bash
sudo mkfs.ext4 /dev/sda1   # Create ext4 filesystem
# OR
sudo mkfs.xfs /dev/sda1    # XFS (better for large files/storage)
```

**Step 5: Mount it**
```bash
# Temporary mount
sudo mkdir -p /mnt/data
sudo mount /dev/sda1 /mnt/data
df -h                       # Verify mount

# Permanent mount (survives reboot)
# Edit /etc/fstab:
sudo nano /etc/fstab

# Add this line:
/dev/sda1  /mnt/data  ext4  defaults  0  2

# Test it:
sudo mount -a               # Mount all entries from fstab
```

### Partition Table: MBR vs GPT

| Feature | MBR | GPT |
|---------|-----|-----|
| Max disk size | 2TB | 8 Zettabytes |
| Max partitions | 4 primary | 128 |
| Boot type | Legacy BIOS | UEFI (modern) |
| Use case | Older servers | Modern systems |

### Safety: Before Partitioning

**CRITICAL:** Partitioning erases all data on a disk. Verify you're targeting the right disk:
```bash
# ALWAYS check disk sizes and labels:
lsblk
sudo fdisk -l /dev/sda

# Wrong disk = data loss
# Example: Never do this without verification:
sudo parted /dev/sda mklabel gpt   # ERASES all data on /dev/sda
```

### Common Mounting Scenarios
```bash
# Mount a volume at boot (persistent)
# Edit /etc/fstab:
/dev/sda1  /home  ext4  defaults  0  2

# Temporary mount
sudo mount /dev/sda1 /mnt/data

# View all mounts
df -h
mount | grep sda

# Unmount (when done)
sudo umount /mnt/data
```

## How to Use This Repository

Each section contains:
- Hands-on configuration examples
- Step-by-step implementation guides
- Real-world troubleshooting scenarios
- Production best practices and tips
- Common pitfalls and how to avoid them

## Resources

- **Author's YouTube Channel:** [LearnLinuxTV](https://www.youtube.com/learnlinux)
- **Book on Amazon:** [Mastering Ubuntu Server, 3rd Edition](https://www.amazon.in/Mastering-Ubuntu-Server-configuring-troubleshooting/dp/1800564643)
- **Publisher:** [Packt Publishing](https://www.packtpub.com/)

## License

This repository contains learning materials and practical examples for educational purposes. Refer to the official book for comprehensive coverage and licensing information.

---

**Note:** This repository supplements the book with hands-on labs, configuration examples, and production scenarios. For complete understanding, refer to the official book by Jay LaCroix.
