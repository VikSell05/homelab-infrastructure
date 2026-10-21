# 🛠️ Homelab Maintenance & Operations Cheatsheet

**Document Version:** 1.1  
**Target Host:** Ubuntu Server 24.04 LTS (`192.168.1.252`) / Proxmox VE (`192.168.1.254`)  

---

## 1. 🐋 Docker & Container Lifecycle

### Update Stack / Containers
```bash
# Navigate to service directory and pull latest images
cd ~/homelab-infrastructure/services/<service-folder>
docker compose pull
docker compose up -d --remove-orphans

# Check status and running logs
docker compose ps
docker compose logs -f --tail=100
```
### System Cleanup & Storage Recovery
```Bash
# Display disk space used by Docker
docker system df

# Safe Prune: Remove stopped containers, unused networks, and dangling images
docker system prune -f

# Deep Prune: Remove ALL unused images and build cache (frees NVMe space)
docker system prune -a --volumes -f
```

--- 

## 2. ☁️ Nextcloud Operations (occ CLI)
All commands must be run as `www-data` inside the `nextcloud-app` container:

```Bash
# Helper alias (optional: add to ~/.bashrc)
alias nc-occ="docker exec -it --user www-data nextcloud-app php occ"
```

### Essential Tasks
```Bash
# 1. Rescan all user files (if files were added directly to storage)
docker exec -it --user www-data nextcloud-app php occ files:scan --all

# 2. Toggle Maintenance Mode (On / Off)
docker exec -it --user www-data nextcloud-app php occ maintenance:mode --on
docker exec -it --user www-data nextcloud-app php occ maintenance:mode --off

# 3. Database maintenance & repair
docker exec -it --user www-data nextcloud-app php occ db:add-missing-indices
docker exec -it --user www-data nextcloud-app php occ db:add-missing-columns
docker exec -it --user www-data nextcloud-app php occ maintenance:repair

# 4. Check Cron background tasks status
docker exec -it --user www-data nextcloud-app php occ status

# 5. Clear Nextcloud memory cache
docker exec -it --user www-data nextcloud-app php occ memcache:clear
```

---

## 3. 📷 Immich Operations & Administration

### Admin Password Reset & User Management
```Bash
# Reset Admin Password via CLI
docker exec -it immich_server immich-admin reset-password

# List all users in database
docker exec -it immich_postgres psql -U postgres -d immich -c "SELECT id, email, \"isAdmin\" FROM users;"
```

### Database Maintenance
```Bash
# Vacuum PostgreSQL DB to reclaim space
docker exec -it immich_postgres vacuumdb -U postgres --all --analyze
```
---

## 4. :shield: CrowdSec WAF & Intrusion Prevention

### Active Decisions & Threat Analysis
```Bash
# View all active IP bans / blocks
sudo cscli decisions list

# Inspect recent alerts and threat triggers
sudo cscli alerts list

# View detailed alert details by Alert ID
sudo cscli alerts inspect <ALERT_ID>

# Manually ban an abusive IP address (1 day duration)
sudo cscli decisions add --ip 1.2.3.4 --duration 24h --reason "Manual Block"

# Manually unban an IP address (Whitelist/Unban)
sudo cscli decisions delete --ip 1.2.3.4
```

### Security Hub & Bouncers
```Bash
# Check status of CrowdSec Firewall / NPM Bouncer
sudo cscli bouncers list

# Update CrowdSec scenario rules and parser collections
sudo cscli hub update
sudo cscli hub upgrade
sudo systemctl restart crowdsec
```

---


## 5. 💽 Storage & MergerFS Health Checks

### Pool & Drive Space Monitoring
```Bash
# View storage usage per physical mount and MergerFS pool
df -h /mnt/hdd1 /mnt/hdd2 /mnt/storage

# Verify MergerFS options and mount status
mount | grep mergerfs
```

### SMART HDD Health Inspection
```Bash
# Check S.M.A.R.T. health status of physical drives
sudo smartctl -H /dev/sdb
sudo smartctl -H /dev/sdc

# Run a quick SMART self-test
sudo smartctl -t short /dev/sdb
```

---

## 6. 🏗️ Host System & Infrastructure Maintenance

### :package: Ubuntu Server Maintenance (192.168.1.252)
```Bash
# System updates
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y

# Check system logs for I/O errors or kernel issues
sudo dmesg -T --level=err,warn
```
### :strawberry: Pi-hole LXC Maintenance (192.168.1.253)
```Bash
# Update Pi-hole Ad-lists (Gravity)
pihole -g

# Update Pi-hole core and web interface (run inside LXC terminal)
pihole -up
```
### :computer: Proxmox VE Host (192.168.1.254)
```Bash
# Check Proxmox storage pools & ZFS/LVM status
pvesm status

# Check running VMs and LXC status via CLI
qm list
pct list
```