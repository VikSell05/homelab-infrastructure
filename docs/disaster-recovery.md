# 🚑 Disaster Recovery & Business Continuity Plan (DRP)

**Document Version:** 1.0  
**Target Environment:** Proxmox VE Homelab (VM 101 - `docker-host`)  
**Storage Architecture:** Local SSD NVMe + MergerFS Pool (`/mnt/storage`)  

---

## 1. Overview & Objectives

This runbook defines the procedures required to recover the homelab infrastructure from data corruption, drive failures, VM crashes, or catastrophic bare-metal loss.

### Recovery Targets
* **Recovery Point Objective (RPO):** < 24 hours (Daily automated database dumps and state backups).
* **Recovery Time Objective (RTO):** < 2 hours for single service/DB recovery; < 4 hours for complete bare-metal host rebuild.

---

## 2. Infrastructure Backup Strategy

The infrastructure is categorized into four distinct backup Tiers:

| Tier | Category | Component / Path | Backup Method | Destination |
| :--- | :--- | :--- | :--- | :--- |
| **Tier 0** | Infrastructure as Code | Docker Compose files, `.env.example`, Scripts | Git Repository | GitHub (Remote) |
| **Tier 1** | Databases & State | PostgreSQL (Immich), MariaDB (Nextcloud), NPM Data | Automated Nightly `cron` Dumps | `/mnt/storage/backups/db/` |
| **Tier 2** | Heavy User Data | `/mnt/storage/immich-data`, `/mnt/storage/nextcloud-data` | MergerFS Storage + Cold Copy | External Storage / Offsite |
| **Tier 3** | Virtual Machine / LXC | Proxmox VE VM 101 (`docker-host`), LXC 100 (`pihole`) | Proxmox VZDump Scheduled Backups | Secondary Storage |

---

## 3. Automated Backup Procedures

### 3.1 Nightly Database Backup Script
A shell script running on the Ubuntu host executes every night at 02:00 AM to generate compressed SQL dumps.

**Script location on Ubuntu Host:** `/usr/local/bin/homelab-backup.sh`

```bash
#!/usr/bin/env bash
set -e

BACKUP_DIR="/mnt/storage/backups/db"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7

mkdir -p "${BACKUP_DIR}"

echo "[+] Starting Nightly Homelab Backup: ${TIMESTAMP}"

# 1. PostgreSQL Backup (Immich)
echo "[+] Dumping Immich PostgreSQL Database..."
docker exec -t immich_postgres pg_dumpall -c -U postgres | gzip > "${BACKUP_DIR}/immich_postgres_${TIMESTAMP}.sql.gz"

# 2. MariaDB Backup (Nextcloud)
echo "[+] Dumping Nextcloud MariaDB Database..."
docker exec -t nextcloud-db mariadb-dump -u root -pYourSecureRootPassword --all-databases | gzip > "${BACKUP_DIR}/nextcloud_mariadb_${TIMESTAMP}.sql.gz"

# 3. Nginx Proxy Manager Data Archive
echo "[+] Archiving Nginx Proxy Manager Configuration..."
tar -czf "${BACKUP_DIR}/npm_config_${TIMESTAMP}.tar.gz" -C /home/dockeradmin/npm-app data letsencrypt

# 4. Prune Old Backups
echo "[+] Cleaning backups older than ${RETENTION_DAYS} days..."
find "${BACKUP_DIR}" -type f -mtime +${RETENTION_DAYS} -delete

echo "[+] Backup Completed Successfully!"
```
---

## 4. Disaster Recovery Runbooks

### Scenario A: Nextcloud Database Corruption or File Inconsistency

### Symptoms
- Nextcloud displays database connection errors, HTTP 500 internal errors, or corrupted file indexes.

### Recovery Steps
1. **Stop Nextcloud Application Containers:**

    ```bash
    cd ~/homelab-infrastructure/services/nextcloud
    docker compose stop app redis
    ```
2. **Restore MariaDB Database from Last Known Good Dump:**
    ```Bash
    # Decompress latest backup
    gunzip -c /mnt/storage/backups/db/nextcloud_mariadb_YYYYMMDD_HHMMSS.sql.gz > /tmp/nextcloud_restore.sql

    # Restore into MariaDB container
    docker exec -i nextcloud-db mariadb -u root -pYourSecureRootPassword nextcloud < /tmp/nextcloud_restore.sql

    # Clean up temporary file
    rm /tmp/nextcloud_restore.sql
    
    ```
3. **Restart Application Containers:**
    ```Bash
    docker compose start app redis
    ```
4. **Rescan File System & Repair Indexes:**

    ```Bash
    docker exec -it --user www-data nextcloud-app php occ files:scan --all
    docker exec -it --user www-data nextcloud-app php occ db:add-missing-indices
    ```


### Scenario B: Immich Database Corruption or Failed Upgrade

### Symptoms
- Immich web interface fails to load media, throws PostgreSQL connection errors, or migration errors after container updates.

## Recovery Steps
1. **Stop Immich App Engine:**

    ```Bash
    cd ~/homelab-infrastructure/services/immich
    docker compose stop immich-server immich-machine-learning
    ```
2. **Restore PostgreSQL Database:**

    ```Bash
    # Decompress latest backup
    gunzip -c /mnt/storage/backups/db/immich_postgres_YYYYMMDD_HHMMSS.sql.gz > /tmp/immich_restore.sql

    # Restore into PostgreSQL container
    docker exec -i immich_postgres psql -U postgres < /tmp/immich_restore.sql

    # Clean up temporary file
    rm /tmp/immich_restore.sql
    ```
3. **Restart Immich Stack:**

    ```Bash
    docker compose start immich-server immich-machine-learning
    ```

### Scenario C: Single USB Drive Failure in MergerFS Pool

### Symptoms
- Operating system logs I/O errors on `/dev/sdb` or `/dev/sdc`.

- `/mnt/hdd1` or `/mnt/hdd2` becomes read-only or disappears from `lsblk`.

### Resilience Note
Because MergerFS is a JBOD/pooling abstraction (not RAID 0 stripe), all data stored on the surviving hard drive remains **100% intact and readable**.

### Recovery Steps
1. **Identify the Failing Drive:**

    ```Bash
    lsblk -f
    dmesg | grep -iE 'error|sdb|sdc'
    ```

2. **Unmount the Failing Drive:**

    ```Bash
    sudo umount -l /mnt/hdd1
    ```
3. **Update /etc/fstab:**
    Remove the failed drive's mount entry and update the MergerFS line to pool only surviving drives:

    ```Plaintext
    # MergerFS line with surviving drive
    /mnt/hdd2  /mnt/storage  mergerfs  defaults,nonempty,allow_other,use_ino,cache.files=off,drop_privs=true,category.create=mfs  0  0
    ```

4. **Remount MergerFS:**

    ```Bash
    sudo mount -o remount /mnt/storage
    ```
5. **Replace Drive & Restore Missing Files:**
    - Insert new HDD, format to `ext4` following [`docs/storage-management.md`](storage-management.md).
    - Restore missing files from `offsite/cold` backup directly into `/mnt/storage/`.

### Scenario D: Total Host VM Crash / Bare-Metal Rebuild

### Objective
Rebuild Ubuntu VM 101 from scratch on Proxmox VE and restore all services.

### Recovery Sequence
1. **Provision New Ubuntu 24.04 VM in Proxmox VE:**
    + Assign 4 Cores, 8GB RAM, 40GB NVMe Disk.
    + Assign Static IP: `192.168.1.252/24`, Gateway: `192.168.1.1`, DNS: `192.168.1.253`.

2. **Mount USB Storage & MergerFS:**
    + Pass through USB HDDs in Proxmox.
    + Install MergerFS: `sudo apt update && sudo apt install -y mergerfs`
    + Configure `/etc/fstab` and mount all drives: `sudo mount -a`.

3. **Install Docker Engine & Git:**

    ```Bash
    curl -fsSL https://get.docker.com | sh
    sudo usermod -aG docker $USER
    ```
4. **Clone Infrastructure Repository:**

    ```Bash
    git clone https://github.com/VikSell05/homelab-infrastructure.git
    cd homelab-infrastructure
    ```
5. **Recreate Production `.env` Files:**
    + Copy `.env.example` to `.env ` in `core/proxy`, `services/immich`, `services/nextcloud`.
    + Populate production passwords/keys from password manager.

6. **Restore Databases:**
    + Start database containers only (`docker compose up -d db` / `docker compose up -d database`).
    + Restore latest `.sql.gz` backups into MariaDB and PostgreSQL using Scenario A & B commands.

7. **Deploy All Container Stacks:**

    ```Bash
    cd ~/homelab-infrastructure/core/proxy && docker compose up -d
    cd ~/homelab-infrastructure/services/immich && docker compose up -d
    cd ~/homelab-infrastructure/services/nextcloud && docker compose up -d
    ```
8. **Re-attach CrowdSec IPS:**
    + Install CrowdSec on host OS and configure the Nginx Proxy Manager log acquisition plugin.

---

## 5. Verification & DR Testing Protocol
To ensure disaster recovery procedures remain valid, perform this check quarterly:

1. **Backup File Verification:** Verify nightly `.sql.gz` dump files exist in `/mnt/storage/backups/db/` and have non-zero file sizes.

2. **Dump Integrity Test:** Test parsing compressed database dumps without modifying production data:

    ```Bash
    gunzip -t /mnt/storage/backups/db/nextcloud_mariadb_*.sql.gz
    gunzip -t /mnt/storage/backups/db/immich_postgres_*.sql.gz
    ```

3. **Log Audit:** Check /var/log/homelab-backup.log for any exit status errors.