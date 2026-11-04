**HANDS-ON LEARNING GUIDE**

Amazon EBS  
from Console to Linux

A step-by-step volume attachment lab + command reference

|                                                                                                                                                                                      |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Learning outcome:** Understand the storage workflow; identify the correct disk; format only a confirmed blank volume; mount it; test it; and configure persistent mounting safely. |

# Your actual lab environment

| **Setting**                      | **Observed / selected value**                                   |
| -------------------------------- | --------------------------------------------------------------- |
| EC2 instance                     | Inayat | Ubuntu Linux | t3.micro                                |
| AWS Region and Availability Zone | ap-south-2 (Hyderabad) | ap-south-2a                            |
| Existing system disk             | /dev/nvme0n1 | 8 GiB | contains /, /boot, /boot/efi             |
| New attached EBS volume          | /dev/nvme1n1 | 10 GiB | no FSTYPE / UUID / mount point shown    |
| EBS configuration                | gp3 | 3,000 IOPS | 125 MiB/s throughput                         |
| Target Linux folder              | /data (not yet created or mounted in the shown terminal output) |

# Progress at the point of this guide

| **Completed and evidenced**       | **Remaining / not yet evidenced**           |
| --------------------------------- | ------------------------------------------- |
| Volume appears attached in Linux. | Confirm the volume is blank with file -s.   |
| You ran lsblk.                    | Format only if confirmed new and empty.     |
| You ran sudo lsblk -f.            | Create /data, mount, test, configure fstab. |

|                                                                                                                                                                                                                                  |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Most important rule:** Never format /dev/nvme0n1 or its partitions: that is your Ubuntu system drive. The new 10 GiB device appears to be /dev/nvme1n1, but verify device identity and contents again immediately before mkfs. |

# 1 What is EBS, and why add another volume?

Think of the EC2 instance as a rented computer and EBS as a hard drive.
Your Code Combat application can keep its Ubuntu operating system on the
existing root disk and use a separate data volume for uploads, logs, or
files. This separates data from the system disk and lets you expand
storage without replacing the instance.

**What “block storage” means:** AWS exposes a disk-like device; Linux
must provide a file system and a mount point before normal file access
is possible.

**Simple mental model:** AWS creates volume → AWS attaches disk → Linux
sees block device → file system is created (if blank) → Linux mounts it
→ applications read/write files.

**Volume / Snapshot / Lifecycle Manager:** Volumes are usable disks;
snapshots are point-in-time backups; Lifecycle Manager automates
snapshot schedules and retention.

**Important limits:** EBS volumes and instances must be in the same
Availability Zone. Volume data generally survives an EC2 stop; whether a
volume is deleted upon termination depends on its Delete on Termination
setting.

# 2 AWS Console: what you configured

| **Action**                                    | **Why it matters**                                                     |
| --------------------------------------------- | ---------------------------------------------------------------------- |
| EC2 → Instances → inspect Inayat              | Find the instance Availability Zone: ap-south-2a.                      |
| Elastic Block Store → Volumes → Create volume | Create a separate virtual disk, not another EC2 instance.              |
| Choose gp3 and 10 GiB                         | General-purpose SSD; adequate for a small practice lab.                |
| Keep 3,000 IOPS and 125 MiB/s                 | gp3 baseline performance settings; no need to raise for this exercise. |
| Choose ap-south-2a                            | The new disk must match the target EC2 instance’s AZ.                  |
| Don’t create from a snapshot                  | Makes a new empty volume rather than restoring old data.               |
| Enable encryption when creating               | Recommended data-at-rest protection; use an appropriate KMS key.       |
| Actions → Attach volume → Inayat              | Connect the new EBS disk to the running instance.                      |

The console may assign /dev/sdf (or similar) as an attachment name. On
Nitro-based EC2 instances the Ubuntu kernel commonly presents the same
disk as /dev/nvme1n1 or another NVMe name. Do not guess the Linux path
from the AWS attachment name.

# 3 Reading your real terminal output

|       |
| ----- |
| lsblk |

The relevant lines you shared (shortened) were:

<table>
<tbody>
<tr class="odd">
<td>nvme0n1 8G disk<br />
├─nvme0n1p1 6.9G part /<br />
├─nvme0n1p13 1023M part /boot<br />
└─nvme0n1p15 106M part /boot/efi<br />
nvme1n1 10G disk</td>
</tr>
</tbody>
</table>

|               |
| ------------- |
| sudo lsblk -f |

Your output showed ext4 on the root and /boot partitions, vfat on
/boot/efi, but no FSTYPE, UUID or MOUNTPOINTS for nvme1n1. This is
consistent with a raw, newly created 10 GiB disk. The loop0–loop4
entries are Snap software mount images, not extra EBS disks.

|                                                                                                                                                                                                                                                          |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Evidence versus assumption:** The output supports identifying /dev/nvme1n1 as the new empty volume, but you have not yet shared the file -s check or any formatting result. The instructions below describe next actions, not steps already completed. |

# 4 Command-by-command walkthrough

Follow in order. “When to use it” tells you whether the command is for
discovery, one-time disk setup, normal verification, or restart-proof
configuration.

## 01 List attached disks

|       |
| ----- |
| lsblk |

**What it does:** Lists block devices, sizes, partitions, and mount
points.

**Why we use it:** Distinguishes the existing Ubuntu system disk from
the new disk before any destructive operation.

**When to use it:** After attaching a volume; whenever you need to
inspect disks or check what is mounted.

**In your lab:** Already executed. It showed 8 GiB nvme0n1 and a
separate 10 GiB nvme1n1.

## 02 Display file system details

|               |
| ------------- |
| sudo lsblk -f |

**What it does:** Adds file system types (FSTYPE), labels, UUIDs, space
details and mount points.

**Why we use it:** Helps tell whether a disk has already been formatted,
reducing the risk of overwriting data.

**When to use it:** Before formatting; after formatting; when checking
mounts and UUIDs.

**In your lab:** Already executed. nvme1n1 had no file system type, UUID
or mount point.

## 03 Check the specific disk signature

|                           |
| ------------------------- |
| sudo file -s /dev/nvme1n1 |

**What it does:** Inspects the target device for recognizable
file-system or other data signatures.

**Why we use it:** Provides another check that the 10 GiB disk is truly
blank before creating a new file system.

**When to use it:** Immediately before mkfs on a new disk, and when
assessing whether a disk already holds data.

**In your lab:** If the result is only “/dev/nvme1n1: data”, that
supports a raw volume. If a filesystem or other signature is reported,
STOP and investigate.

|                                                                                                                                                                                                      |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Caution:** A single “data” result does not replace confirming the disk really is the new AWS volume. Never format a snapshot-restored or previously used volume without checking what it contains. |

## 04 Create an ext4 file system — DESTRUCTIVE

|                             |
| --------------------------- |
| sudo mkfs.ext4 /dev/nvme1n1 |

**What it does:** Creates an ext4 filesystem so Linux can organize
directories and files on the new disk.

**Why we use it:** An unformatted raw device cannot be mounted as a
regular ext4 file system.

**When to use it:** Usually ONCE for a genuinely new, empty data volume;
not on every attachment or reboot.

**In your lab:** Only proceed after confirming /dev/nvme1n1 is the new
10 GiB disk and contains no required data.

|                                                                                                                                                           |
| --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Caution:** Formatting overwrites file-system metadata and can destroy data. Never run mkfs on /dev/nvme0n1, /dev/nvme0n1p1, or an existing data volume. |

## 05 Create the /data folder

|                     |
| ------------------- |
| sudo mkdir -p /data |

**What it does:** Creates the /data directory if it does not already
exist; -p avoids an error if it does exist.

**Why we use it:** A mount point is needed to make the file system
reachable in the Linux directory tree.

**When to use it:** Once before the first mount, or again harmlessly if
you need to ensure the empty directory exists.

**In your lab:** /data is a mount point path, not the physical disk
itself.

## 06 Mount the data disk

|                               |
| ----------------------------- |
| sudo mount /dev/nvme1n1 /data |

**What it does:** Connects the ext4 filesystem on that disk to the /data
directory.

**Why we use it:** After mounting, ordinary writes to /data reach the
new EBS volume instead of the root file system.

**When to use it:** After filesystem creation, or for manual mounting
when it is currently unmounted.

**In your lab:** If the file system already exists, do not format it
again; just mount it.

|                                                                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Caution:** Check that /data does not contain files you need to access before mounting: mounting hides underlying files until the volume is unmounted. |

## 07 Verify space, type, and mount

|              |
| ------------ |
| df -hT /data |

**What it does:** Reports the filesystem hosting /data, using
human-readable sizes (-h) and showing file-system type (-T).

**Why we use it:** Proves that /data now resolves to a filesystem; look
for the new disk rather than the root disk.

**When to use it:** Immediately after mounting; any time you want to
monitor available space.

**In your lab:** After success, the device should refer to the new data
volume and type ext4. Before mount, df may report the root filesystem
instead.

## 08 Write a small test file

|                                                   |
| ------------------------------------------------- |
| echo "Hello from EBS\!" | sudo tee /data/test.txt |

**What it does:** echo prints text; the pipe (|) passes it to tee; sudo
tee writes it to the given file as root and echoes it back.

**Why we use it:** Confirms write access and demonstrates that files can
now be created under the mounted data directory.

**When to use it:** After confirming that /data is actually mounted on
the correct EBS volume.

**In your lab:** tee is used with sudo because a plain shell redirect
like sudo echo ... \> file does not run the redirection as root.

|                                                                                                                                              |
| -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Caution:** Do not run this test until df / findmnt confirms /data is on the EBS volume; otherwise the test file may land on the root disk. |

## 09 Read the saved test file

|                    |
| ------------------ |
| cat /data/test.txt |

**What it does:** Prints the file contents to the terminal.

**Why we use it:** Confirms the test file can be read back successfully.

**When to use it:** After creating a test file; when quickly inspecting
small text files.

**In your lab:** Expected test text: Hello from EBS\!

# 5 Keep the mount after a restart (fstab)

Manual mounting is not necessarily remembered after reboot. /etc/fstab
is a system configuration file containing persistent mount rules. Use
the file system UUID rather than the NVMe name because device names can
change across reboots.

## 10 Find the filesystem UUID

|                         |
| ----------------------- |
| sudo blkid /dev/nvme1n1 |

**What it does:** Shows identifying metadata for the disk, including its
UUID and TYPE after formatting.

**Why we use it:** The UUID is a more stable way to identify the target
filesystem in /etc/fstab.

**When to use it:** AFTER mkfs.ext4 and before editing /etc/fstab.

**In your lab:** Copy the entire UUID from your terminal; a
sample/placeholder UUID will not work.

## 11 Back up fstab

|                                   |
| --------------------------------- |
| sudo cp /etc/fstab /etc/fstab.bak |

**What it does:** Copies the existing fstab file to a backup in the same
directory.

**Why we use it:** Lets you restore the earlier configuration if you
make a mistake.

**When to use it:** Immediately before changing /etc/fstab.

**In your lab:** Note: running the same command again overwrites the
previous .bak backup.

## 12 Edit the persistent mount rules

|                      |
| -------------------- |
| sudo nano /etc/fstab |

**What it does:** Opens /etc/fstab in the nano terminal editor with
elevated privileges.

**Why we use it:** Adds the rule that mounts the correct volume at /data
during startup.

**When to use it:** After UUID verification and backup; only when
persistent mounting is required.

**In your lab:** Append ONE line, substituting the actual UUID from
blkid:

|                                                     |
| --------------------------------------------------- |
| UUID=\<ACTUAL-UUID\> /data ext4 defaults,nofail 0 2 |

**Fields explained:** UUID identifies the filesystem; /data is the mount
point; ext4 the filesystem type; defaults,nofail are options (nofail
allows boot with a missing optional disk); 0 disables dump; and 2 sets
non-root filesystem check order. Apps may still fail if storage is
missing.

## 13 Verify the fstab syntax and mounts

|                                 |
| ------------------------------- |
| sudo findmnt --verify --verbose |

**What it does:** Checks the fstab configuration for detectable problems
and reports diagnostics.

**Why we use it:** Catches many syntax and mount specification errors
before reboot.

**When to use it:** Right after saving fstab changes; before attempting
a reboot.

**In your lab:** This is an additional safety check beyond the earlier
basic instructions.

## 14 Apply mount configuration

|               |
| ------------- |
| sudo mount -a |

**What it does:** Attempts to mount entries in fstab that are not
already mounted (subject to options).

**Why we use it:** Tests that the persistent configuration can mount
eligible filesystems without a reboot.

**When to use it:** After editing fstab, followed by inspection of the
actual mount.

**In your lab:** The command may produce no output on success; do not
interpret silence alone as proof.

## 15 Confirm the actual mount

|               |
| ------------- |
| findmnt /data |

**What it does:** Displays the mounted filesystem associated with /data.

**Why we use it:** Confirms the directory is really backed by the
intended filesystem and shows its source.

**When to use it:** After mount, after mount -a, and after a later
reboot.

**In your lab:** Use alongside df -hT /data to avoid writing to the root
disk by mistake.

# 6 A safe execution checklist for your next terminal session

| **Order** | **Action / command**                                  | **Decision**                                        |
| --------- | ----------------------------------------------------- | --------------------------------------------------- |
| 1         | lsblk / sudo lsblk -f                                 | Already done; re-check if devices changed.          |
| 2         | sudo file -s /dev/nvme1n1                             | Confirm no existing filesystem or data to preserve. |
| 3         | sudo mkfs.ext4 /dev/nvme1n1                           | ONLY if truly blank and correctly identified.       |
| 4         | sudo mkdir -p /data                                   | Prepare the mount directory.                        |
| 5         | sudo mount /dev/nvme1n1 /data                         | Mount the EBS disk.                                 |
| 6         | df -hT /data / findmnt /data                          | Verify the new disk is the backing filesystem.      |
| 7         | echo … | sudo tee /data/test.txt / cat /data/test.txt | Test write/read after verified mount.               |
| 8         | sudo blkid / backup and edit /etc/fstab               | Configure UUID-based persistent mounting.           |
| 9         | sudo findmnt --verify --verbose / sudo mount -a       | Check the configuration before any reboot.          |
| 10        | findmnt /data                                         | Verify again, including after any planned reboot.   |

# 7 Common questions and mistakes

### “I attached the volume, but df -h does not show 10 GiB.”

This is normal until you mount a file system. lsblk shows devices; df
reports mounted file systems.

### “Why is /dev/sdf not visible inside Ubuntu?”

The EC2 attachment name is not always the Linux device name. Nitro
instances commonly expose EBS disks as /dev/nvme\*n\*.

### “Do I format the disk every time I restart?”

No. mkfs is an initial setup operation for a genuinely empty disk;
repeat formatting may erase data.

### “What if I used a snapshot to create the volume?”

A restored volume can already have a file system and data. Identify and
mount it; do NOT run mkfs just because it is newly attached.

### “Why do we use sudo?”

Device inspection, file-system creation, system mounting, and fstab
editing commonly require administrator privileges.

### “Why use a UUID instead of /dev/nvme1n1 in fstab?”

The NVMe name can change; the filesystem UUID normally remains stable
for that filesystem.

### “Can I write application files under /data immediately?”

After mounting, yes, subject to file ownership and permissions. Use
least-privilege permissions for your application; do not blindly chmod
777.

### “Does stopping EC2 remove the volume or avoid its cost?”

EBS provisioned storage is generally billed independently of instance
uptime. Termination behavior depends on Delete on Termination; check the
actual attachment settings.

# 8 Optional cleanup after a practice lab

If you no longer need the data volume, first back up any wanted files or
take a snapshot. Stop processes using /data. Remove/comment the /data
entry from /etc/fstab so the next boot will not expect this volume. Then
unmount it cleanly:

|                   |
| ----------------- |
| sudo umount /data |

**What / why / when:** umount detaches the filesystem from the Linux
directory tree and helps flush outstanding writes. Use before detaching
an EBS volume through AWS; not while applications are writing to it.

Finally use EC2 → Volumes → Actions → Detach volume; wait for Available;
delete the volume only if its contents are no longer needed. Deleting
the volume permanently removes its data unless you can recover it from a
snapshot. Deleting a volume or taking snapshots may have billing
implications.

# 9 Quick knowledge check

  - 1\. What is the difference between a block device (lsblk) and a
    mounted file system (df)?

  - 2\. Why should /dev/nvme0n1 never be used as the formatting target
    in this lab?

  - 3\. Under what condition is mkfs.ext4 appropriate, and when should
    it be skipped?

  - 4\. What will happen if /data is not in /etc/fstab when Ubuntu
    reboots?

  - 5\. Why is a UUID usually safer than a transient NVMe name for
    fstab?

# References for verification

AWS — Create an Amazon EBS volume —
https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-volume.html

AWS — Attach an EBS volume —
https://docs.aws.amazon.com/ebs/latest/userguide/ebs-attaching-volume.html

AWS — Make an EBS volume available for use on Linux —
https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html

AWS — General Purpose SSD gp3 —
https://docs.aws.amazon.com/ebs/latest/userguide/general-purpose.html
