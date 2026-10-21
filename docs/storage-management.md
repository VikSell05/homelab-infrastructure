# 💾 Storage Management Runbook

Guide on how to manage USB storage drives, format new partitions, and expand the MergerFS logical volume `/mnt/storage`.

## Adding a New Hard Drive to MergerFS

### Step 1: Identify and Format the New Disk
Connect the new USB drive to the IdeaPad host and pass it through to the Ubuntu VM.
Run the following commands inside the Ubuntu VM'(192.168.1.252):

```bash
# List connected disks to identify the new device (e.g., /dev/sdc)
lsblk

# Create a new partition table and partition using fdisk
sudo fdisk /dev/[DISK]
# Press 'n' for new partition, 'p' for primary, accept defaults, then 'w' to write changes.

# Format the partition with ext4 file system
sudo mkfs.ext4 /dev/[DISK]
```



### Step 2: Create Mount Point

```bash
# Create a mount point (e.g., hdd3)
sudo mkdir -p /mnt/[HDD]
```


### Step 3: Update /etc/fstab
Add the new physical disk entry using its UUID:

```bash
echo "UUID=$(sudo blkid -s UUID -o value /dev/[DISK]) /mnt/[HDD] ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
```

Update the existing MergerFS mount line to include the new drive (`/mnt/hdd3`):

```bash
sudo nano /etc/fstab
# MergerFS Pool Configuration
/mnt/hdd1:/mnt/hdd2:/mnt/[HDD] /mnt/storage mergerfs defaults,allow_other,use_ino,cache.files=off,dropcacheonclose=true,category.create=mfs 0 0
```


### Step 4: Mount and Verify
```bash
# Reload all mount points
sudo mount -a

# Verify that the total capacity of /mnt/storage has increased
df -h /mnt/storage
```