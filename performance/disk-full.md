
# Disk full on EC2 ("No space left on device")

## Symptom
Writing a file fails with "No space left on device" and the root filesystem is at 100%:

```
$  fallocate -l 5G ~/bigfile2.img
fallocate: fallocate failed: No space left on device

$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p1  8.0G  6.6G  1.4G  83% /

```
<img width="447" height="50" alt="image" src="https://github.com/user-attachments/assets/69088098-a45a-4671-b1ff-48cce2c37906" />



*Root volume at 83% usage*

## Environment
- AWS EC2 (t3.micro, Amazon Linux 2023)
- Region: ap-southeast-2 (Sydney)
- 8 GB gp3 root volume (xfs)

## How I reproduced it
Created large files in the home directory until the root volume was full:

```
fallocate -l 5G ~/bigfile1.img
fallocate -l 5G ~/bigfile2.img
```

## Troubleshooting steps
1. Read the error: "No space left on device" points to disk space (or inodes)
2. Ran `df -h`: `/` was at 100%
3. Ran `df -i`: inode usage was low, so the problem was blocks, not inodes
4. Ran `sudo du -xh / --max-depth=1 | sort -rh | head -5`: `/home` was the largest directory
5. Ran `du -ah ~ | sort -rh | head -5`: found the large `bigfile` files

## Root cause
Large files in the home directory used up all the space on the root volume.

## Fix
Deleted the unneeded files:

```
rm ~/bigfile2.img ~/bigfile1.img
```

## Verification
```
$  df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p1  8.0G  1.6G  6.4G  20% /

```
<img width="465" height="45" alt="image" src="https://github.com/user-attachments/assets/73766846-1f0a-4deb-b8c2-fea59ed80279" />



*Disk usage back to normal after deleting the files*

## Key learning
- "No space left on device" can mean **no free blocks** (`df -h`) or **no free inodes** (`df -i`). Check both
- `du` finds **what** is using space, `df` shows **how full** the filesystem is
- A file deleted while a process still holds it open keeps using space. Find it with `sudo lsof +L1` and restart that process
- Log files (`/var/log`) are a very common real-world cause. Fix with log rotation, not just deletion
- CloudWatch does not report disk usage by default (needs the CloudWatch agent)
- If cleanup is not enough, extend the EBS volume (Modify volume, `growpart`, `xfs_growfs`). Volumes can grow but not shrink
- On Amazon Linux 2023, `/tmp` is in RAM (tmpfs), so it does not fill the root disk

## Commands used
```
df -h
df -i
sudo du -xh / --max-depth=1 2>/dev/null | sort -rh | head -5
du -ah ~ | sort -rh | head -5
rm ~/bigfile2.img ~/bigfile1.img
```
