# 📘 สรุปขั้นตอนการติดตั้ง NFS HA Cluster ด้วย DRBD + Keepalived

---

## 🎯 ภาพรวมระบบ

```
┌─────────────────────────────────────────────────────┐
│          NFS HA Cluster Architecture                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────┐         ┌─────────────┐          │
│  │   nfs1      │◄───────►│   nfs2      │          │
│  │ 172.16.xx.71│  DRBD   │172.16.xx.72 │          │
│  └─────────────┘         └─────────────┘          │
│         │                        │                  │
│         └────────────┬───────────┘                  │
│                      │                              │
│              VIP: 172.16.220.73                     │
│                      │                              │
│                 ┌────▼────┐                         │
│                 │ Clients │                         │
│                 └─────────┘                         │
└─────────────────────────────────────────────────────┘

Components:
- DRBD: Real-time block replication (2TB /dev/vdb)
- Keepalived: VIP management & failover detection
- NFS Server: File sharing service
- XFS: Filesystem on DRBD device
```

---

## 📋 ข้อมูลระบบ

| Component | Detail |
|-----------|--------|
| **Node 1** | dc2-nfs-01 (172.16.220.71) |
| **Node 2** | dc2-nfs-02 (172.16.220.72) |
| **VIP** | 172.16.220.73 |
| **Network** | 172.16.220.0/24 (single network) |
| **Storage** | /dev/vdb (2TB) |
| **DRBD Device** | /dev/drbd0 |
| **Mount Point** | /mnt/nfs-data |
| **Filesystem** | XFS |
| **OS** | Ubuntu 24.04 LTS |

---

# 🚀 Part 1: เตรียมระบบ (ทั้ง 2 nodes)

## Step 1.1: ตั้งค่า Hostname

```bash
# บน Node 1
sudo hostnamectl set-hostname nfs1

# บน Node 2
sudo hostnamectl set-hostname nfs2

# ตรวจสอบ
hostnamectl
```

## Step 1.2: เพิ่ม Hosts Entries (ทั้ง 2 nodes)

```bash
# บน nfs1 และ nfs2
sudo tee -a /etc/hosts << 'EOF'

# NFS HA Cluster
172.16.220.71 nfs1 nfs1.local dc2-nfs-01
172.16.220.72 nfs2 nfs2.local dc2-nfs-02
172.16.220.73 nfs-ha nfs-ha.local
EOF

# ทดสอบ connectivity
ping -c 3 nfs2  # จาก nfs1
ping -c 3 nfs1  # จาก nfs2
```

## Step 1.3: Update System (ทั้ง 2 nodes)

```bash
sudo apt update && sudo apt upgrade -y
```

## Step 1.4: ตรวจสอบ Network Interface (ทั้ง 2 nodes)

```bash
# หา interface name ที่ใช้
ip -br addr show | grep 172.16.220

# Output ตัวอย่าง: 
# ens7    UP    172.16.220.71/24 ... 

# จดชื่อ interface ไว้ (เช่น ens7, ens18, eth0)
# จะใช้ใน keepalived config
```

---

# 🔧 Part 2: ติดตั้งและ Configure DRBD

## Step 2.1: ติดตั้ง DRBD (ทั้ง 2 nodes)

```bash
# ติดตั้ง DRBD utilities
sudo apt install -y drbd-utils

# Load DRBD kernel module
sudo modprobe drbd

# เพิ่มให้ load อัตโนมัติตอน boot
echo drbd | sudo tee -a /etc/modules

# ตรวจสอบ
lsmod | grep drbd
drbdadm --version
```

## Step 2.2: เตรียม Disk (ทั้ง 2 nodes)

```bash
# ตรวจสอบ disk
sudo lsblk /dev/vdb

# Output ควรเห็น:
# NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
# vdb  253:16   0    2T  0 disk

# ลบข้อมูลเก่า (ถ้ามี)
sudo wipefs -a /dev/vdb

# ตรวจสอบว่าสะอาดแล้ว
sudo lsblk -f /dev/vdb
# FSTYPE ควรว่างเปล่า
```

## Step 2.3: สร้าง DRBD Global Configuration (ทั้ง 2 nodes - เหมือนกัน)

```bash
sudo tee /etc/drbd.d/global_common.conf << 'EOF'
global {
  usage-count no;
  udev-always-use-vnr;
}

common {
  handlers {
    split-brain "/usr/lib/drbd/notify-split-brain.sh root";
    out-of-sync "/usr/lib/drbd/notify-out-of-sync.sh root";
  }
  
  startup {
    wfc-timeout 15;
    degr-wfc-timeout 15;
  }
  
  options {
    auto-promote yes;
  }
  
  disk {
    on-io-error detach;
  }
  
  net {
    # Split-brain policies
    after-sb-0pri discard-zero-changes;
    after-sb-1pri discard-secondary;
    after-sb-2pri disconnect;
    rr-conflict disconnect;
  }
}
EOF
```

## Step 2.4: สร้าง DRBD Resource Configuration (ทั้ง 2 nodes - เหมือนกัน)

```bash
sudo tee /etc/drbd.d/nfs-data.res << 'EOF'
resource nfs-data {
  device /dev/drbd0;
  disk /dev/vdb;
  meta-disk internal;
  
  # Protocol C = Fully synchronous (safest)
  protocol C;
  
  net {
    # Performance tuning
    max-buffers     8000;
    max-epoch-size  8000;
    sndbuf-size     2M;
    rcvbuf-size     2M;
    
    # Timeout settings
    timeout         60;
    connect-int     10;
    ping-int        10;
    ping-timeout    5;
    ko-count        4;
  }
  
  disk {
    # Disk tuning
    resync-rate     300M;
    c-plan-ahead    20;
    c-fill-target   100M;
    c-max-rate      700M;
    
    # Optimize for virtual disk
    disk-flushes no;
    md-flushes no;
  }
  
  syncer {
    rate 300M;
    verify-alg sha1;
  }
  
  # Node 1 configuration
  on nfs1 {
    device          /dev/drbd0;
    disk            /dev/vdb;
    address         172.16.220.71:7789;
    meta-disk       internal;
    node-id         0;
  }
  
  # Node 2 configuration
  on nfs2 {
    device          /dev/drbd0;
    disk            /dev/vdb;
    address         172.16.220.72:7789;
    meta-disk       internal;
    node-id         1;
  }
  
  connection {
    host nfs1 address 172.16.220.71:7789;
    host nfs2 address 172.16.220.72:7789;
  }
}
EOF

# ตรวจสอบ configuration
sudo drbdadm dump nfs-data
```

## Step 2.5: Initialize DRBD Metadata (ทั้ง 2 nodes)

```bash
# สร้าง metadata structures
sudo drbdadm create-md nfs-data

# Output ควรเห็น: 
# initializing activity log
# initializing bitmap (61440 KB) to all zero
# Writing meta data...
# New drbd meta data block successfully created. 
```

## Step 2.6: Start DRBD (ทั้ง 2 nodes)

```bash
# Start DRBD resource
sudo drbdadm up nfs-data

# Enable DRBD service
sudo systemctl enable drbd

# ตรวจสอบ status
sudo drbdadm status nfs-data

# Output:
# nfs-data role: Secondary
#   disk: Inconsistent
#   nfs2 role:Secondary
#     replication:Established peer-disk:Inconsistent
```

## Step 2.7: Initial Synchronization (บน nfs1 เท่านั้น!)

```bash
# ⚠️ รันบน nfs1 เท่านั้น! 
sudo drbdadm primary --force nfs-data

# Monitor sync progress
watch -n 2 'sudo drbdadm status nfs-data'

# Output ระหว่าง sync: 
# nfs-data role:Primary
#   disk: UpToDate
#   nfs2 role:Secondary
#     replication:SyncSource
#     peer-disk: Inconsistent
#     done:15. 3%  ← เพิ่มขึ้นเรื่อยๆ

# รอจนกว่า sync จะเสร็จ (กับ 2TB อาจใช้เวลา 1-3 ชั่วโมง)
# เมื่อเสร็จจะเห็น:
# nfs-data role:Primary
#   disk:UpToDate
#   nfs2 role:Secondary
#     replication: Established peer-disk:UpToDate
```

## Step 2.8: สร้าง Filesystem (บน nfs1 เท่านั้น!)

```bash
# ⚠️ รันบน nfs1 เท่านั้น!  (Primary node)

# Format ด้วย XFS
sudo mkfs.xfs -L nfs-data /dev/drbd0

# สร้าง mount point
sudo mkdir -p /mnt/nfs-data

# Mount filesystem
sudo mount /dev/drbd0 /mnt/nfs-data

# ตรวจสอบ
df -h /mnt/nfs-data

# Output: 
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/drbd0      2.0T   77M  1.9T   1% /mnt/nfs-data

# สร้าง export directories
sudo mkdir -p /mnt/nfs-data/exports/{data,backups,docker}
sudo chmod 755 /mnt/nfs-data/exports/*

# ทดสอบเขียนไฟล์
echo "DRBD is working!" | sudo tee /mnt/nfs-data/test.txt
```

## Step 2.9: Configure Firewall (ทั้ง 2 nodes)

```bash
# เปิด DRBD port
sudo ufw allow from 172.16.220.0/24 to any port 7789 proto tcp

# Reload firewall (ถ้าใช้)
sudo ufw reload

# ตรวจสอบ
sudo ufw status numbered
```

---

# 🔄 Part 3: ติดตั้งและ Configure Keepalived

## Step 3.1: ติดตั้ง Keepalived (ทั้ง 2 nodes)

```bash
sudo apt install -y keepalived
```

## Step 3.2: สร้าง Health Check Script (ทั้ง 2 nodes - เหมือนกัน)

```bash
sudo tee /usr/local/bin/check_nfs_ha. sh << 'EOF'
#!/bin/bash

# ตรวจสอบ NFS server
if !  systemctl is-active --quiet nfs-server; then
    exit 1
fi

# ตรวจสอบ DRBD role
if ! drbdadm role nfs-data | grep -q "Primary"; then
    exit 1
fi

# ตรวจสอบ filesystem mount
if ! mountpoint -q /mnt/nfs-data; then
    exit 1
fi

# ตรวจสอบว่าเขียนไฟล์ได้
if ! touch /mnt/nfs-data/. health_check 2>/dev/null; then
    exit 1
fi

exit 0
EOF

sudo chmod +x /usr/local/bin/check_nfs_ha.sh

# ทดสอบ script
sudo /usr/local/bin/check_nfs_ha.sh
echo $?  # ควรได้ 0 (success) บน nfs1 ที่เป็น Primary
```

## Step 3.3: สร้าง Transition Scripts (ทั้ง 2 nodes - เหมือนกัน)

### Script 1: keepalived_master.sh

```bash
sudo tee /usr/local/bin/keepalived_master.sh << 'EOF'
#!/bin/bash

LOGFILE="/var/log/nfs-ha. log"
MAX_RETRIES=5

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') [MASTER] $1" | tee -a "$LOGFILE"
    logger -t nfs-ha "[MASTER] $1"
}

log "===== Transitioning to MASTER ====="
log "Hostname: $(hostname)"

# รอให้ peer node demote DRBD (สำคัญมาก!)
log "Waiting for peer to demote DRBD..."
WAIT_COUNT=0
MAX_WAIT=30

while [ $WAIT_COUNT -lt $MAX_WAIT ]; do
    PEER_ROLE=$(drbdadm role nfs-data 2>&1)
    log "DRBD role: $PEER_ROLE"
    
    if echo "$PEER_ROLE" | grep -q "Secondary/Secondary"; then
        log "Peer has released DRBD (Secondary/Secondary)"
        break
    fi
    
    if echo "$PEER_ROLE" | grep -q "Secondary/Primary"; then
        log "Waiting for peer to demote...  ($WAIT_COUNT/$MAX_WAIT)"
        sleep 1
        WAIT_COUNT=$((WAIT_COUNT + 1))
        continue
    fi
    
    CSTATE=$(drbdadm cstate nfs-data 2>&1)
    log "Connection state: $CSTATE"
    
    if [ "$CSTATE" = "StandAlone" ]; then
        log "WARNING: Running in StandAlone mode (peer down? )"
        break
    fi
    
    sleep 1
    WAIT_COUNT=$((WAIT_COUNT + 1))
done

if [ $WAIT_COUNT -ge $MAX_WAIT ]; then
    log "WARNING: Timeout waiting for peer, will try to promote anyway"
fi

sleep 2

# 1. Promote DRBD to Primary
log "Promoting DRBD to Primary..."
RETRY=0
while [ $RETRY -lt $MAX_RETRIES ]; do
    ERROR_MSG=$(drbdadm primary nfs-data 2>&1)
    PROMOTE_EXIT=$?
    
    if [ $PROMOTE_EXIT -eq 0 ]; then
        log "DRBD promoted successfully"
        break
    else
        RETRY=$((RETRY + 1))
        log "Failed to promote DRBD (exit code: $PROMOTE_EXIT), retry $RETRY/$MAX_RETRIES"
        log "Error: $ERROR_MSG"
        
        if echo "$ERROR_MSG" | grep -q "Multiple primaries"; then
            log "ERROR:  Peer is still Primary!  Cannot promote."
            
            if [ $RETRY -eq $MAX_RETRIES ]; then
                log "CRITICAL: Peer refused to demote after $MAX_WAIT seconds"
                log "Manual intervention required on peer node"
                exit 1
            fi
            
            log "Waiting 5 more seconds for peer..."
            sleep 5
        else
            if [ $RETRY -eq $MAX_RETRIES ]; then
                log "ERROR: Failed to promote DRBD after $MAX_RETRIES attempts"
                exit 1
            fi
            sleep 3
        fi
    fi
done

log "Waiting for DRBD to stabilize..."
sleep 3

DRBD_ROLE=$(drbdadm role nfs-data 2>&1)
log "DRBD Role: $DRBD_ROLE"

if !  echo "$DRBD_ROLE" | grep -q "Primary"; then
    log "ERROR:  DRBD is not Primary!"
    log "Current role: $DRBD_ROLE"
    exit 1
fi

# 2. สร้าง mount point (ถ้ายังไม่มี)
if [ !  -d /mnt/nfs-data ]; then
    log "Creating mount point directory..."
    mkdir -p /mnt/nfs-data
fi

# Mount filesystem
if ! mountpoint -q /mnt/nfs-data; then
    log "Mounting filesystem..."
    RETRY=0
    while [ $RETRY -lt $MAX_RETRIES ]; do
        MOUNT_ERROR=$(mount /dev/drbd0 /mnt/nfs-data 2>&1)
        MOUNT_EXIT=$?
        
        if [ $MOUNT_EXIT -eq 0 ]; then
            log "Filesystem mounted successfully"
            break
        else
            RETRY=$((RETRY + 1))
            log "Failed to mount (exit:  $MOUNT_EXIT), retry $RETRY/$MAX_RETRIES"
            log "Error: $MOUNT_ERROR"
            
            if [ $RETRY -eq $MAX_RETRIES ]; then
                log "ERROR: Failed to mount filesystem"
                drbdadm secondary nfs-data
                exit 1
            fi
            
            sleep 2
        fi
    done
else
    log "Filesystem already mounted"
fi

if !  mountpoint -q /mnt/nfs-data; then
    log "ERROR: Filesystem is not mounted!"
    drbdadm secondary nfs-data
    exit 1
fi

# 3. สร้าง export directories (ถ้ายังไม่มี)
log "Ensuring export directories exist..."
mkdir -p /mnt/nfs-data/exports/{data,backups,docker}
chmod 755 /mnt/nfs-data/exports/*

# 4. Start NFS server
log "Starting NFS server..."
RETRY=0
while [ $RETRY -lt $MAX_RETRIES ]; do
    if systemctl start nfs-server 2>&1 | tee -a "$LOGFILE"; then
        log "NFS server started successfully"
        break
    else
        RETRY=$((RETRY + 1))
        log "Failed to start NFS, retry $RETRY/$MAX_RETRIES..."
        
        if [ $RETRY -eq $MAX_RETRIES ]; then
            log "ERROR: Failed to start NFS server"
            exit 1
        fi
        
        sleep 2
    fi
done

sleep 2

# 5. Export NFS shares
log "Exporting NFS shares..."
if exportfs -ra 2>&1 | tee -a "$LOGFILE"; then
    log "NFS shares exported successfully"
else
    log "WARNING: Failed to export NFS shares"
fi

# 6. ตรวจสอบสถานะสุดท้าย
log "Final status check..."
log "DRBD:  $(drbdadm role nfs-data)"
log "Mount:  $(mount | grep drbd || echo 'Not mounted')"
log "NFS: $(systemctl is-active nfs-server)"
log "Exports: $(exportfs -v 2>/dev/null | wc -l) shares"

log "===== Now serving as MASTER ====="
exit 0
EOF

sudo chmod +x /usr/local/bin/keepalived_master.sh
```

### Script 2: keepalived_backup.sh

```bash
sudo tee /usr/local/bin/keepalived_backup.sh << 'EOF'
#!/bin/bash

LOGFILE="/var/log/nfs-ha.log"
MAX_RETRIES=5

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') [BACKUP] $1" | tee -a "$LOGFILE"
    logger -t nfs-ha "[BACKUP] $1"
}

log "===== Transitioning to BACKUP ====="
log "Hostname: $(hostname)"

# 1. Stop NFS server
log "Stopping NFS server..."
systemctl stop nfs-server 2>&1 | tee -a "$LOGFILE"

if systemctl is-active --quiet nfs-server; then
    log "Force killing NFS server..."
    systemctl kill nfs-server
    sleep 1
fi

log "NFS server stopped"

# 2. Unexport NFS shares
log "Unexporting NFS shares..."
exportfs -ua 2>&1 | tee -a "$LOGFILE"

sleep 3

# 3. Unmount filesystem
if mountpoint -q /mnt/nfs-data; then
    log "Unmounting filesystem..."
    
    log "Killing processes using /mnt/nfs-data..."
    fuser -km /mnt/nfs-data 2>/dev/null
    sleep 2
    
    RETRY=0
    while [ $RETRY -lt $MAX_RETRIES ]; do
        if umount /mnt/nfs-data 2>&1 | tee -a "$LOGFILE"; then
            log "Filesystem unmounted successfully"
            break
        else
            RETRY=$((RETRY + 1))
            log "Failed to unmount, retry $RETRY/$MAX_RETRIES..."
            fuser -km /mnt/nfs-data 2>/dev/null
            sleep 2
            
            if [ $RETRY -eq $MAX_RETRIES ]; then
                log "WARNING: Force lazy unmount..."
                umount -l /mnt/nfs-data
                sleep 1
                break
            fi
        fi
    done
else
    log "Filesystem not mounted"
fi

if mountpoint -q /mnt/nfs-data; then
    log "WARNING: Filesystem still mounted (lazy unmount in progress)"
else
    log "Filesystem unmounted confirmed"
fi

sleep 2

# 4. Demote DRBD to Secondary
log "Demoting DRBD to Secondary..."
RETRY=0
while [ $RETRY -lt $MAX_RETRIES ]; do
    DEMOTE_ERROR=$(drbdadm secondary nfs-data 2>&1)
    DEMOTE_EXIT=$?
    
    if [ $DEMOTE_EXIT -eq 0 ]; then
        log "DRBD demoted successfully"
        break
    else
        RETRY=$((RETRY + 1))
        log "Failed to demote DRBD (exit: $DEMOTE_EXIT), retry $RETRY/$MAX_RETRIES"
        log "Error: $DEMOTE_ERROR"
        
        fuser -km /dev/drbd0 2>/dev/null
        fuser -km /mnt/nfs-data 2>/dev/null
        umount -l /mnt/nfs-data 2>/dev/null
        
        sleep 3
        
        if [ $RETRY -eq $MAX_RETRIES ]; then
            log "ERROR:  Failed to demote DRBD after $MAX_RETRIES attempts"
            log "This will prevent peer from becoming Primary!"
        fi
    fi
done

DRBD_ROLE=$(drbdadm role nfs-data 2>&1)
log "Final DRBD role: $DRBD_ROLE"

if echo "$DRBD_ROLE" | grep -q "Secondary"; then
    log "DRBD is now Secondary - OK"
else
    log "WARNING: DRBD is still $DRBD_ROLE - peer cannot promote!"
fi

log "===== Now in BACKUP state ====="
exit 0
EOF

sudo chmod +x /usr/local/bin/keepalived_backup.sh
```

### Script 3: keepalived_fault.sh

```bash
sudo tee /usr/local/bin/keepalived_fault.sh << 'EOF'
#!/bin/bash

LOGFILE="/var/log/nfs-ha.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') [FAULT] $1" | tee -a "$LOGFILE"
    logger -t nfs-ha "[FAULT] $1"
}

log "===== FAULT STATE DETECTED ====="
log "Host: $(hostname)"
log "Reason: $*"

# Send critical alert (uncomment if you have webhook)
# curl -X POST "https://your-webhook-url" \
#   -H 'Content-Type: application/json' \
#   -d "{\"text\": \"🚨 CRITICAL: NFS HA FAULT on $(hostname)\"}"

exit 0
EOF

sudo chmod +x /usr/local/bin/keepalived_fault.sh
```

### Script 4: keepalived_notify.sh

```bash
sudo tee /usr/local/bin/keepalived_notify.sh << 'EOF'
#!/bin/bash

TYPE=$1
NAME=$2
STATE=$3

LOGFILE="/var/log/nfs-ha.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') [$STATE] $1" | tee -a "$LOGFILE"
    logger -t nfs-ha "[$STATE] $1"
}

log "===== Keepalived Notification ====="
log "TYPE: $TYPE"
log "NAME: $NAME"
log "STATE: $STATE"
log "Hostname: $(hostname)"

case $STATE in
    "MASTER")
        /usr/local/bin/keepalived_master.sh
        ;;
    "BACKUP")
        /usr/local/bin/keepalived_backup.sh
        ;;
    "FAULT")
        /usr/local/bin/keepalived_fault. sh "$@"
        ;;
    *)
        log "Unknown state: $STATE"
        ;;
esac

log "===== Notification Complete ====="
EOF

sudo chmod +x /usr/local/bin/keepalived_notify.sh
```

### Script 5: keepalived_cleanup.sh (สำคัญมาก!)

```bash
sudo tee /usr/local/bin/keepalived_cleanup.sh << 'EOF'
#!/bin/bash

LOGFILE="/var/log/nfs-ha.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') [CLEANUP] $1" | tee -a "$LOGFILE"
    logger -t nfs-ha "[CLEANUP] $1"
}

log "===== Keepalived stopping, running cleanup ====="
log "Hostname: $(hostname)"

# ตรวจสอบว่า node นี้เป็น Primary หรือไม่
DRBD_ROLE=$(drbdadm role nfs-data 2>&1 | cut -d/ -f1)
log "Current DRBD role: $DRBD_ROLE"

if [ "$DRBD_ROLE" = "Primary" ]; then
    log "This node is Primary, demoting resources..."
    
    # Stop NFS
    log "Stopping NFS server..."
    systemctl stop nfs-server 2>&1 | tee -a "$LOGFILE"
    systemctl kill nfs-server 2>/dev/null
    
    # Unexport
    log "Unexporting NFS shares..."
    exportfs -ua 2>&1 | tee -a "$LOGFILE"
    
    sleep 2
    
    # Unmount
    if mountpoint -q /mnt/nfs-data; then
        log "Unmounting filesystem..."
        fuser -km /mnt/nfs-data 2>/dev/null
        sleep 1
        
        if umount /mnt/nfs-data 2>&1 | tee -a "$LOGFILE"; then
            log "Unmount successful"
        else
            log "Forcing lazy unmount..."
            umount -l /mnt/nfs-data
        fi
    fi
    
    sleep 2
    
    # Demote DRBD
    log "Demoting DRBD to Secondary..."
    if drbdadm secondary nfs-data 2>&1 | tee -a "$LOGFILE"; then
        log "DRBD demoted successfully"
    else
        log "ERROR: Failed to demote DRBD"
        
        log "Attempting force demote..."
        drbdadm down nfs-data 2>/dev/null
        sleep 1
        drbdadm up nfs-data 2>/dev/null
    fi
    
    FINAL_ROLE=$(drbdadm role nfs-data 2>&1 | cut -d/ -f1)
    log "Final DRBD role: $FINAL_ROLE"
    
    log "Cleanup completed"
else
    log "This node is $DRBD_ROLE, no cleanup needed"
fi

log "===== Cleanup finished ====="
exit 0
EOF

sudo chmod +x /usr/local/bin/keepalived_cleanup.sh

# สร้าง log file
sudo touch /var/log/nfs-ha.log
sudo chmod 644 /var/log/nfs-ha.log
```

## Step 3. 4: สร้าง systemd Override สำหรับ Cleanup (ทั้ง 2 nodes)

```bash
# สร้าง directory
sudo mkdir -p /etc/systemd/system/keepalived.service.d

# สร้าง override file
sudo tee /etc/systemd/system/keepalived.service. d/cleanup.conf << 'EOF'
[Service]
# Run cleanup before stopping keepalived
ExecStop=/usr/local/bin/keepalived_cleanup.sh
ExecStop=
ExecStop=/usr/bin/killall -15 keepalived

# Run cleanup after stopping (backup)
ExecStopPost=/usr/local/bin/keepalived_cleanup.sh

# Increase timeout for cleanup
TimeoutStopSec=30
EOF

# Reload systemd
sudo systemctl daemon-reload
```

## Step 3.5: Configure Keepalived - nfs1 (MASTER)

```bash
# บน nfs1
# ⚠️ แก้ interface name ให้ตรงกับของคุณ (ens7, ens18, eth0, etc.)

sudo tee /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
    router_id dc2-nfs-01
    enable_script_security
    script_user root
    vrrp_garp_master_delay 1
    vrrp_startup_delay 5
}

vrrp_script check_nfs_health {
    script "/usr/local/bin/check_nfs_ha.sh"
    interval 2
    timeout 3
    weight -30
    fall 2
    rise 2
}

vrrp_instance NFS_HA {
    state MASTER
    interface ens7                  # ⚠️ เปลี่ยนตาม interface ของคุณ
    virtual_router_id 51
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass NFS_HA24
    }
    
    virtual_ipaddress {
        172.16.220.73/24
    }
    
    track_script {
        check_nfs_health
    }
    
    notify_master "/usr/local/bin/keepalived_master.sh"
    notify_backup "/usr/local/bin/keepalived_backup.sh"
    notify_fault  "/usr/local/bin/keepalived_fault.sh"
    notify_stop   "/usr/local/bin/keepalived_cleanup.sh"
}
EOF

# ตรวจสอบ syntax
sudo keepalived -t -f /etc/keepalived/keepalived.conf

# ควรเห็น:  Configuration is valid
```

## Step 3.6: Configure Keepalived - nfs2 (BACKUP)

```bash
# บน nfs2
# ⚠️ แก้ interface name ให้ตรงกับของคุณ

sudo tee /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
    router_id dc2-nfs-02
    enable_script_security
    script_user root
    vrrp_garp_master_delay 1
    vrrp_startup_delay 5
}

vrrp_script check_nfs_health {
    script "/usr/local/bin/check_nfs_ha.sh"
    interval 2
    timeout 3
    weight -30
    fall 2
    rise 2
}

vrrp_instance NFS_HA {
    state BACKUP
    interface ens7                  # ⚠️ เปลี่ยนตาม interface ของคุณ
    virtual_router_id 51
    priority 90
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass NFS_HA24
    }
    
    virtual_ipaddress {
        172.16.220.73/24
    }
    
    track_script {
        check_nfs_health
    }
    
    notify_master "/usr/local/bin/keepalived_master.sh"
    notify_backup "/usr/local/bin/keepalived_backup. sh"
    notify_fault  "/usr/local/bin/keepalived_fault.sh"
    notify_stop   "/usr/local/bin/keepalived_cleanup.sh"
}
EOF

sudo keepalived -t -f /etc/keepalived/keepalived.conf
```

## Step 3.7: Configure Firewall สำหรับ Keepalived (ทั้ง 2 nodes)

```bash
# เปิด VRRP protocol
sudo ufw allow from 172.16.220.0/24 proto vrrp

# หรือถ้า ufw ไม่รองรับ protocol vrrp
sudo ufw allow from 172.16.220.0/24 to 224.0.0.0/8
sudo ufw allow in on ens7 to 224.0.0.18  # เปลี่ยน ens7 ตาม interface ของคุณ

# Reload
sudo ufw reload
```

---

# 📁 Part 4: ติดตั้งและ Configure NFS Server

## Step 4.1: ติดตั้ง NFS Server (ทั้ง 2 nodes)

```bash
sudo apt install -y nfs-kernel-server
```

## Step 4.2: Configure NFS Exports (ทั้ง 2 nodes - เหมือนกัน)

```bash
sudo tee /etc/exports << 'EOF'
# NFS HA Exports
/mnt/nfs-data/exports/data    172.16.220.0/24(rw,sync,no_subtree_check,no_root_squash)
/mnt/nfs-data/exports/backups 172.16.220.0/24(rw,sync,no_subtree_check,no_root_squash)
/mnt/nfs-data/exports/docker  172.16.220.0/24(rw,sync,no_subtree_check,no_root_squash)
EOF

# ⚠️ อย่า start NFS ตอนนี้ - Keepalived จะจัดการให้
sudo systemctl disable nfs-server
sudo systemctl stop nfs-server
```

## Step 4.3: Configure NFS Tuning (ทั้ง 2 nodes)

```bash
# NFS server configuration
sudo tee /etc/default/nfs-kernel-server << 'EOF'
# Number of NFS server threads
RPCNFSDCOUNT=32

# Options for rpc.mountd
RPCMOUNTDOPTS="--manage-gids"
EOF

# NFS common configuration
sudo tee /etc/default/nfs-common << 'EOF'
# Options for rpc.statd
STATDOPTS=""

# NFSv4 only
NEED_IDMAPD=yes
NEED_GSSD=no
EOF
```

## Step 4.4: Configure Firewall สำหรับ NFS (ทั้ง 2 nodes)

```bash
# เปิด NFS ports
sudo ufw allow from 172.16.220.0/24 to any port nfs
sudo ufw allow from 172.16.220.0/24 to any port 111
sudo ufw allow from 172.16.220.0/24 to any port 2049

# Reload
sudo ufw reload
```

---

# 📊 Part 5: สร้าง Monitoring Scripts

## Step 5.1: สร้าง Status Scripts (ทั้ง 2 nodes)

### Script 1: nfs_ha_status.sh (Full status)

```bash
sudo tee /usr/local/bin/nfs_ha_status. sh << 'EOF'
#!/bin/bash

echo "=========================================="
echo "NFS HA Cluster Status"
echo "=========================================="
echo "Hostname: $(hostname)"
echo "Date: $(date)"
echo ""

echo "--- Keepalived Status ---"
if systemctl is-active --quiet keepalived; then
    echo "✓ Keepalived:  Running"
    
    if ip addr show | grep -q "172.16.220.73"; then
        echo "✓ Role:  MASTER (VIP Active)"
    else
        echo "○ Role: BACKUP"
    fi
else
    echo "✗ Keepalived:  Stopped"
fi
echo ""

echo "--- DRBD Status ---"
DRBD_ROLE=$(drbdadm role nfs-data 2>/dev/null)
DRBD_CSTATE=$(drbdadm cstate nfs-data 2>/dev/null)
DRBD_DSTATE=$(drbdadm dstate nfs-data 2>/dev/null)

if [ -n "$DRBD_ROLE" ]; then
    echo "Role: $DRBD_ROLE"
    echo "Connection:  $DRBD_CSTATE"
    echo "Disk State: $DRBD_DSTATE"
else
    echo "✗ DRBD: Not configured or not running"
fi
echo ""

echo "--- NFS Status ---"
if systemctl is-active --quiet nfs-server; then
    echo "✓ NFS Server: Running"
    echo "Exports:"
    exportfs -v 2>/dev/null | grep -v "^$" || echo "  No exports"
else
    echo "○ NFS Server:  Stopped"
fi
echo ""

echo "--- Filesystem Status ---"
if mountpoint -q /mnt/nfs-data 2>/dev/null; then
    echo "✓ Filesystem:  Mounted"
    df -h /mnt/nfs-data 2>/dev/null | tail -1
    
    echo ""
    echo "Export directories:"
    ls -la /mnt/nfs-data/exports/ 2>/dev/null || echo "  Export directories not found"
else
    echo "○ Filesystem: Not Mounted"
fi
echo ""

echo "--- Network Status ---"
echo "IP Addresses:"
ip addr show | grep "inet " | grep -v "127.0.0.1"
echo ""

echo "--- Recent Logs (last 10 lines) ---"
if [ -f /var/log/nfs-ha.log ]; then
    tail -10 /var/log/nfs-ha.log
else
    echo "No log file found"
fi
echo ""

echo "=========================================="
EOF

sudo chmod +x /usr/local/bin/nfs_ha_status.sh
```

### Script 2: nfs_ha_quick. sh (Quick status)

```bash
sudo tee /usr/local/bin/nfs_ha_quick.sh << 'EOF'
#!/bin/bash

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo -e "\n=== NFS HA Quick Status on $(hostname) ==="
echo "Time: $(date '+%Y-%m-%d %H:%M:%S')"
echo ""

# Keepalived
if systemctl is-active --quiet keepalived; then
    if ip addr show | grep -q "172.16.220.73"; then
        echo -e "${GREEN}✓ Keepalived:  MASTER${NC} (VIP:  172.16.220.73)"
    else
        echo -e "${YELLOW}○ Keepalived: BACKUP${NC}"
    fi
else
    echo -e "${RED}✗ Keepalived:  STOPPED${NC}"
fi

# DRBD
DRBD_ROLE=$(drbdadm role nfs-data 2>/dev/null | cut -d/ -f1)
case "$DRBD_ROLE" in
    Primary)
        echo -e "${GREEN}✓ DRBD:  Primary${NC} ($(drbdadm cstate nfs-data))"
        ;;
    Secondary)
        echo -e "${YELLOW}○ DRBD:  Secondary${NC} ($(drbdadm cstate nfs-data))"
        ;;
    *)
        echo -e "${RED}✗ DRBD: Unknown${NC}"
        ;;
esac

# Filesystem
if mountpoint -q /mnt/nfs-data 2>/dev/null; then
    USAGE=$(df -h /mnt/nfs-data | tail -1 | awk '{print $5}')
    echo -e "${GREEN}✓ Filesystem: Mounted${NC} (Usage: $USAGE)"
else
    echo -e "${YELLOW}○ Filesystem: Not Mounted${NC}"
fi

# NFS
if systemctl is-active --quiet nfs-server; then
    EXPORTS=$(exportfs -v 2>/dev/null | grep -c "^/")
    echo -e "${GREEN}✓ NFS Server: Running${NC} (Exports: $EXPORTS)"
else
    echo -e "${YELLOW}○ NFS Server: Stopped${NC}"
fi

echo ""
EOF

sudo chmod +x /usr/local/bin/nfs_ha_quick.sh
```

### Script 3: nfs_ha_monitor.sh (Live monitor)

```bash
sudo tee /usr/local/bin/nfs_ha_monitor.sh << 'EOF'
#!/bin/bash

while true; do
    clear
    echo "=========================================="
    echo "NFS HA Live Monitor - Press Ctrl+C to exit"
    echo "=========================================="
    echo "Hostname: $(hostname)"
    echo "Time: $(date '+%Y-%m-%d %H:%M:%S')"
    echo ""
    
    # VIP Status
    if ip addr show | grep -q "172.16.220.73"; then
        echo "🟢 MASTER - VIP Active (172.16.220.73)"
    else
        echo "🟡 BACKUP - VIP Not Active"
    fi
    echo ""
    
    # DRBD Status
    echo "--- DRBD ---"
    DRBD_ROLE=$(drbdadm role nfs-data 2>/dev/null)
    DRBD_CSTATE=$(drbdadm cstate nfs-data 2>/dev/null)
    DRBD_DSTATE=$(drbdadm dstate nfs-data 2>/dev/null)
    
    echo "Role:        $DRBD_ROLE"
    echo "Connection: $DRBD_CSTATE"
    echo "Disk:        $DRBD_DSTATE"
    echo ""
    
    # Mount Status
    echo "--- Filesystem ---"
    if mountpoint -q /mnt/nfs-data 2>/dev/null; then
        df -h /mnt/nfs-data | tail -1
    else
        echo "Not Mounted"
    fi
    echo ""
    
    # NFS Status
    echo "--- NFS Server ---"
    systemctl status nfs-server --no-pager -l 2>/dev/null | head -3
    echo ""
    
    # Active Connections
    echo "--- NFS Connections ---"
    ss -tn | grep ":2049" | wc -l | xargs echo "Active connections:"
    echo ""
    
    # Recent Logs
    echo "--- Recent Log (last 3 lines) ---"
    tail -3 /var/log/nfs-ha.log 2>/dev/null || echo "No logs"
    
    echo ""
    echo "Refreshing in 2 seconds..."
    sleep 2
done
EOF

sudo chmod +x /usr/local/bin/nfs_ha_monitor.sh
```

## Step 5.2: สร้าง Aliases (ทั้ง 2 nodes)

```bash
cat >> ~/. bashrc << 'EOF'

# NFS HA Aliases
alias nfs-status='sudo /usr/local/bin/nfs_ha_status. sh'
alias nfs-quick='sudo /usr/local/bin/nfs_ha_quick. sh'
alias nfs-monitor='sudo /usr/local/bin/nfs_ha_monitor. sh'
alias nfs-log='sudo tail -f /var/log/nfs-ha.log'
alias nfs-drbd='sudo drbdadm status nfs-data'
alias nfs-failover='sudo systemctl stop keepalived'
alias nfs-recover='sudo systemctl start keepalived'
alias nfs-vip='ip addr show | grep 172.16.220.73'

# NFS HA Manual Control
alias nfs-start-master='sudo drbdadm primary nfs-data && sudo mount /dev/drbd0 /mnt/nfs-data && sudo systemctl start nfs-server && sudo exportfs -ra && sudo systemctl start keepalived'
alias nfs-start-backup='sudo systemctl start keepalived'
alias nfs-stop-all='sudo systemctl stop keepalived && sudo systemctl stop nfs-server && sudo umount -l /mnt/nfs-data && sudo drbdadm secondary nfs-data'
EOF

source ~/.bashrc
```

---

# 🎬 Part 6: Start Cluster

## Step 6.1: Prepare nfs1 (MASTER)

```bash
# บน nfs1
# ตรวจสอบว่า DRBD sync เสร็จแล้ว
sudo drbdadm status nfs-data

# ควบเห็น: 
# nfs-data role: Primary
#   disk: UpToDate
#   peer role:Secondary
#     replication: Established peer-disk: UpToDate

# ตรวจสอบว่า mount แล้ว
df -h | grep drbd

# ตรวจสอบ export directories
ls -la /mnt/nfs-data/exports/

# ถ้ามี Primary และ mount แล้ว ให้ start NFS
sudo systemctl start nfs-server
sudo exportfs -ra
```

## Step 6.2: Start Keepalived บน nfs1

```bash
# บน nfs1
sudo systemctl enable keepalived
sudo systemctl start keepalived

# Monitor logs
sudo tail -f /var/log/nfs-ha.log

# ควรเห็น:
# [MASTER] ===== Transitioning to MASTER =====
# ...  (หรือไม่เห็นอะไรเลยถ้า state ไม่เปลี่ยน)

# ตรวจสอบ status
nfs-quick

# ควบเห็น:
# ✓ Keepalived:  MASTER (VIP:  172.16.220.73)
# ✓ DRBD: Primary (Connected)
# ✓ Filesystem: Mounted
# ✓ NFS Server: Running
```

## Step 6.3: Start Keepalived บน nfs2 (รอ 10 วินาที)

```bash
# บน nfs2
sudo systemctl enable keepalived
sudo systemctl start keepalived

# Monitor logs
sudo tail -f /var/log/nfs-ha.log

# ควบเห็น:
# [BACKUP] ===== Transitioning to BACKUP =====
# [BACKUP] Hostname: dc2-nfs-02
# [BACKUP] Stopping NFS server... 
# [BACKUP] NFS server stopped
# ...  (demote process)
# [BACKUP] ===== Now in BACKUP state =====

# ตรวจสอบ status
nfs-quick

# ควบเห็น:
# ○ Keepalived: BACKUP
# ○ DRBD: Secondary (Connected)
# ○ Filesystem: Not Mounted
# ○ NFS Server: Stopped
```

## Step 6.4: ตรวจสอบ Cluster Status

```bash
# บน nfs1
nfs-status

# บน nfs2
nfs-status

# ตรวจสอบ VIP จาก client
ping -c 3 172.16.220.73

# ตรวจสอบ NFS exports
showmount -e 172.16.220.73

# ควบเห็น:
# Export list for 172.16.220.73:
# /mnt/nfs-data/exports/data    172.16.220.0/24
# /mnt/nfs-data/exports/backups 172.16.220.0/24
# /mnt/nfs-data/exports/docker  172.16.220.0/24
```

---

# 🧪 Part 7: ทดสอบ Failover

## Step 7.1: Setup Monitoring (บน nfs2)

```bash
# Terminal 1: NFS HA logs
sudo tail -f /var/log/nfs-ha.log

# Terminal 2: Live monitor
nfs-monitor

# Terminal 3:  Keepalived logs
sudo journalctl -u keepalived -f
```

## Step 7.2:  Trigger Failover (บน nfs1)

```bash
# Stop keepalived
sudo systemctl stop keepalived

# รอ 10 วินาที
sleep 10

# ตรวจสอบ cleanup
sudo grep "$(date '+%Y-%m-%d')" /var/log/nfs-ha.log | grep CLEANUP | tail -20

# ควบเห็น:
# [CLEANUP] ===== Keepalived stopping, running cleanup =====
# [CLEANUP] Current DRBD role: Primary
# [CLEANUP] This node is Primary, demoting resources...
# [CLEANUP] Stopping NFS server...
# [CLEANUP] Unexporting NFS shares...
# [CLEANUP] Unmounting filesystem...
# [CLEANUP] Unmount successful
# [CLEANUP] Demoting DRBD to Secondary... 
# [CLEANUP] DRBD demoted successfully
# [CLEANUP] Final DRBD role: Secondary
# [CLEANUP] Cleanup completed
# [CLEANUP] ===== Cleanup finished =====

# ตรวจสอบ DRBD
sudo drbdadm status nfs-data

# ควบเห็น:
# nfs-data role:Secondary
#   peer role:Primary
```

## Step 7.3: ตรวจสอบ Failover (บน nfs2)

```bash
# ดู logs
sudo grep "$(date '+%Y-%m-%d %H:%M')" /var/log/nfs-ha.log | grep MASTER | tail -30

# ควบเห็น:
# [MASTER] ===== Transitioning to MASTER =====
# [MASTER] Waiting for peer to demote DRBD...
# [MASTER] DRBD role: Secondary/Secondary
# [MASTER] Peer has released DRBD (Secondary/Secondary)
# [MASTER] Promoting DRBD to Primary...
# [MASTER] DRBD promoted successfully
# [MASTER] Mounting filesystem...
# [MASTER] Filesystem mounted successfully
# [MASTER] Starting NFS server...
# [MASTER] NFS server started successfully
# [MASTER] ===== Now serving as MASTER =====

# ตรวจสอบ status
nfs-quick

# ควบเห็น: 
# ✓ Keepalived: MASTER (VIP:  172.16.220.73)
# ✓ DRBD: Primary (Connected)
# ✓ Filesystem: Mounted
# ✓ NFS Server:  Running
```

## Step 7.4: ทดสอบ VIP

```bash
# จาก client หรือ node อื่น
ping -c 3 172.16.220.73

# ทดสอบ NFS
showmount -e 172.16.220.73

# Mount NFS (ถ้ายังไม่ได้ mount)
sudo mount -t nfs -o nfsvers=4 172.16.220.73:/exports/data /mnt/nfs/data

# ทดสอบเขียนไฟล์
echo "Test after failover - $(date)" | sudo tee /mnt/nfs/data/failover-test.txt
```

## Step 7.5: Failback (Optional)

```bash
# บน nfs1 - start keepalived กลับมา
sudo systemctl start keepalived

# nfs1 จะกลับมาเป็น MASTER (เพราะ priority สูงกว่า)
# รอ 10-20 วินาที

# ตรวจสอบ
nfs-quick

# ควบเห็น:
# ✓ Keepalived: MASTER (VIP: 172.16.220.73)
# ✓ DRBD: Primary (Connected)
# ✓ Filesystem:  Mounted
# ✓ NFS Server: Running
```

---

# 🐳 Part 8: Docker Swarm Integration (Optional)

## Step 8.1: ติดตั้ง NFS Client บน Docker Nodes

```bash
# บนทุก Docker Swarm nodes
sudo apt install -y nfs-common

# เพิ่ม hosts entry
echo "172.16.220.73 nfs-ha" | sudo tee -a /etc/hosts

# สร้าง mount points
sudo mkdir -p /mnt/nfs/{data,backups,docker}
```

## Step 8.2: Mount NFS Shares

```bash
# บนทุก Docker Swarm nodes

# Mount shares
sudo mount -t nfs -o nfsvers=4,soft,timeo=100,retrans=2 \
  nfs-ha:/exports/data /mnt/nfs/data

sudo mount -t nfs -o nfsvers=4,soft,timeo=100,retrans=2 \
  nfs-ha:/exports/backups /mnt/nfs/backups

sudo mount -t nfs -o nfsvers=4,soft,timeo=100,retrans=2 \
  nfs-ha:/exports/docker /mnt/nfs/docker

# เพิ่มใน /etc/fstab เพื่อ auto-mount
sudo tee -a /etc/fstab << 'EOF'
nfs-ha:/exports/data    /mnt/nfs/data    nfs4 soft,timeo=100,retrans=2,_netdev 0 0
nfs-ha:/exports/backups /mnt/nfs/backups nfs4 soft,timeo=100,retrans=2,_netdev 0 0
nfs-ha:/exports/docker  /mnt/nfs/docker  nfs4 soft,timeo=100,retrans=2,_netdev 0 0
EOF

# ตรวจสอบ
df -h | grep nfs
```

## Step 8.3: Docker Stack Example

```yaml
# docker-stack.yml
version: '3.8'

services:
  web:
    image: nginx:latest
    deploy:
      replicas: 3
      restart_policy: 
        condition: on-failure
    volumes:
      - /mnt/nfs/docker/nginx/html:/usr/share/nginx/html: ro
      - /mnt/nfs/docker/nginx/conf:/etc/nginx/conf.d:ro
    ports:
      - "80:80"
    networks:
      - webnet

  app:
    image: myapp: latest
    deploy:
      replicas: 5
    volumes:
      - /mnt/nfs/data/app:/app/data
      - /mnt/nfs/backups/app:/app/backups
    networks:
      - webnet

networks:
  webnet:
    driver: overlay
```

```bash
# Deploy stack
docker stack deploy -c docker-stack. yml myapp
```

---

# 📚 Part 9: Maintenance และ Troubleshooting

## 9.1: ตรวจสอบ Status

```bash
# Quick status
nfs-quick

# Full status
nfs-status

# Live monitor
nfs-monitor

# DRBD status
nfs-drbd

# View logs
nfs-log
```

## 9.2: Manual Failover

```bash
# Trigger failover
nfs-failover

# หรือ
sudo systemctl stop keepalived
```

## 9.3: Manual Recovery

```bash
# Start keepalived กลับมา
nfs-recover

# หรือ
sudo systemctl start keepalived
```

## 9.4: Manual Cleanup

```bash
# Stop ทุกอย่าง
nfs-stop-all

# หรือ run cleanup script
sudo /usr/local/bin/keepalived_cleanup.sh
```

## 9.5: DRBD Commands

```bash
# ดู status
sudo drbdadm status nfs-data

# ดู detailed info
cat /proc/drbd

# ดู connection state
sudo drbdadm cstate nfs-data

# ดู disk state
sudo drbdadm dstate nfs-data

# ดู role
sudo drbdadm role nfs-data

# Promote to Primary
sudo drbdadm primary nfs-data

# Demote to Secondary
sudo drbdadm secondary nfs-data

# Verify sync
sudo drbdadm verify nfs-data

# Disconnect/Reconnect
sudo drbdadm disconnect nfs-data
sudo drbdadm connect nfs-data
```

## 9.6: Troubleshooting Split-Brain

```bash
# ถ้าเกิด split-brain (ไม่ควรเกิดถ้า setup ถูกต้อง)

# บน victim node (ข้อมูลจะถูก discard)
sudo drbdadm disconnect nfs-data
sudo drbdadm secondary nfs-data
sudo drbdadm connect --discard-my-data nfs-data

# บน survivor node (ข้อมูลที่ถูกต้อง)
sudo drbdadm connect nfs-data
```

## 9.7: View Logs

```bash
# NFS HA logs
sudo tail -f /var/log/nfs-ha.log
sudo grep CLEANUP /var/log/nfs-ha.log | tail -20
sudo grep ERROR /var/log/nfs-ha.log | tail -20

# Keepalived logs
sudo journalctl -u keepalived -n 50 --no-pager
sudo journalctl -u keepalived -f

# NFS logs
sudo journalctl -u nfs-server -n 50 --no-pager

# System logs
sudo tail -f /var/log/syslog | grep -E "nfs|drbd|keepalived"
```

---

# 🔄 Part 10: Startup Procedure

## 10.1: Normal Startup (หลัง Reboot ทั้ง 2 nodes)

### Start nfs1 (MASTER) ก่อนเสมอ: 

```bash
# บน nfs1
# 1. ตรวจสอบ DRBD
sudo drbdadm status nfs-data

# 2. Promote to Primary
sudo drbdadm primary nfs-data

# 3. Mount filesystem
sudo mount /dev/drbd0 /mnt/nfs-data

# 4. Start NFS
sudo systemctl start nfs-server
sudo exportfs -ra

# 5. Start keepalived
sudo systemctl start keepalived
