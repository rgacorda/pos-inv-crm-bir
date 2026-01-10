# Infrastructure & Deployment Guide

## Overview

This guide provides step-by-step instructions for setting up and deploying the POS & Inventory Management System using self-hosted infrastructure.

---

## 📋 Prerequisites

### Hardware Requirements

✅ Primary server (16 cores, 128GB RAM, 2TB SSD + 16TB HDD)  
✅ Backup server (8 cores, 64GB RAM, similar storage)  
✅ NAS storage (40TB RAID 6)  
✅ UPS systems for all servers  
✅ Network equipment (router, firewall, switches)  
✅ Redundant internet connections

### Software Requirements

✅ Ubuntu Server 22.04 LTS installation media  
✅ Docker installation files  
✅ WireGuard VPN packages  
✅ Database backup scripts

### Network Requirements

✅ Static IP addresses for all servers  
✅ VPN subnet planned (e.g., 10.0.0.0/24)  
✅ Firewall rules documented  
✅ SSL certificates (Let's Encrypt or commercial)

---

## 🚀 Phase 1: Server Setup

### Step 1: Install Ubuntu Server

```bash
# On primary server
# 1. Boot from Ubuntu Server 22.04 USB
# 2. Follow installation wizard
# 3. Configure:
#    - Hostname: pos-primary
#    - Username: posadmin
#    - Enable OpenSSH server
#    - No automatic updates (manual control)

# After installation, update system
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git vim htop net-tools

# Configure timezone
sudo timedatectl set-timezone Asia/Manila

# Set static IP
sudo vim /etc/netplan/00-installer-config.yaml
```

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    eno1:
      addresses:
        - 192.168.1.10/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

```bash
sudo netplan apply
```

### Step 2: Setup RAID Arrays

```bash
# Install RAID management tools
sudo apt install -y mdadm

# Create RAID 1 for system/database SSDs
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sda /dev/sdb

# Create RAID 10 for file storage HDDs
sudo mdadm --create /dev/md1 --level=10 --raid-devices=4 /dev/sdc /dev/sdd /dev/sde /dev/sdf

# Format and mount
sudo mkfs.ext4 /dev/md0
sudo mkfs.ext4 /dev/md1

sudo mkdir -p /data/ssd
sudo mkdir -p /data/storage

sudo mount /dev/md0 /data/ssd
sudo mount /dev/md1 /data/storage

# Add to /etc/fstab for auto-mount
echo "/dev/md0 /data/ssd ext4 defaults,noatime 0 2" | sudo tee -a /etc/fstab
echo "/dev/md1 /data/storage ext4 defaults,noatime 0 2" | sudo tee -a /etc/fstab

# Save RAID configuration
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

### Step 3: Disk Encryption (Optional but Recommended)

```bash
# Install encryption tools
sudo apt install -y cryptsetup

# Encrypt data partition
sudo cryptsetup luksFormat /dev/md1
# Enter passphrase when prompted

# Open encrypted volume
sudo cryptsetup open /dev/md1 encrypted_storage

# Format and mount
sudo mkfs.ext4 /dev/mapper/encrypted_storage
sudo mount /dev/mapper/encrypted_storage /data/storage

# Setup key file for automatic unlock
sudo dd if=/dev/urandom of=/root/.keyfile bs=1024 count=4
sudo chmod 0400 /root/.keyfile
sudo cryptsetup luksAddKey /dev/md1 /root/.keyfile

# Add to /etc/crypttab
echo "encrypted_storage /dev/md1 /root/.keyfile luks" | sudo tee -a /etc/crypttab

# Update fstab
sudo vim /etc/fstab
# Change: /dev/md1 to /dev/mapper/encrypted_storage
```

### Step 4: Create Directory Structure

```bash
# Create main directories
sudo mkdir -p /data/ssd/{postgres,redis,prometheus}
sudo mkdir -p /data/storage/{minio,backups,wal_archive,logs}
sudo mkdir -p /opt/pos-system

# Set ownership
sudo chown -R posadmin:posadmin /data
sudo chown -R posadmin:posadmin /opt/pos-system

# Create symbolic links for convenience
ln -s /data/ssd /opt/pos-system/data-ssd
ln -s /data/storage /opt/pos-system/data-storage
```

---

## 🐳 Phase 2: Docker Installation

### Step 1: Install Docker

```bash
# Install Docker using official script
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER

# Log out and back in for group changes to take effect
exit
# SSH back in

# Verify installation
docker --version
docker compose version

# Enable Docker service
sudo systemctl enable docker
sudo systemctl start docker

# Test Docker
docker run hello-world
```

### Step 2: Configure Docker

```bash
# Create Docker daemon configuration
sudo mkdir -p /etc/docker
sudo vim /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "data-root": "/data/ssd/docker"
}
```

```bash
# Restart Docker
sudo systemctl restart docker
```

---

## 🔧 Phase 3: Deploy Services

### Step 1: Create Docker Compose File

```bash
cd /opt/pos-system
vim docker-compose.yml
```

```yaml
# docker-compose.yml
version: "3.8"

services:
  postgres:
    image: postgres:16-alpine
    container_name: pos-postgres
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: pos_system
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - /data/ssd/postgres:/var/lib/postgresql/data
      - /data/storage/backups:/backups
      - /data/storage/wal_archive:/wal_archive
      - ./init-scripts:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    command: >
      postgres
      -c shared_buffers=16GB
      -c effective_cache_size=48GB
      -c work_mem=64MB
      -c maintenance_work_mem=2GB
      -c max_connections=500
      -c random_page_cost=1.1
      -c effective_io_concurrency=200
      -c wal_level=replica
      -c max_wal_senders=3
      -c wal_keep_size=1GB
      -c archive_mode=on
      -c archive_command='test ! -f /wal_archive/%f && cp %p /wal_archive/%f'
      -c logging_collector=on
      -c log_directory=/var/log/postgresql
      -c log_filename=postgresql-%Y-%m-%d.log
      -c log_statement=mod
      -c log_min_duration_statement=1000
    networks:
      - pos-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  pgbouncer:
    image: edoburu/pgbouncer:latest
    container_name: pos-pgbouncer
    restart: always
    environment:
      DATABASE_URL: postgres://postgres:${POSTGRES_PASSWORD}@postgres:5432/pos_system
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 1000
      DEFAULT_POOL_SIZE: 25
      MIN_POOL_SIZE: 10
      RESERVE_POOL_SIZE: 5
    ports:
      - "6432:6432"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - pos-network

  redis:
    image: redis:7-alpine
    container_name: pos-redis
    restart: always
    command: >
      redis-server
      --maxmemory 8gb
      --maxmemory-policy allkeys-lru
      --save 900 1
      --save 300 10
      --save 60 10000
      --appendonly yes
      --appendfsync everysec
      --requirepass ${REDIS_PASSWORD}
    volumes:
      - /data/ssd/redis:/data
    ports:
      - "6379:6379"
    networks:
      - pos-network
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  minio:
    image: minio/minio:latest
    container_name: pos-minio
    restart: always
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
      MINIO_BROWSER_REDIRECT_URL: http://minio.example.com:9001
    volumes:
      - /data/storage/minio:/data
    ports:
      - "9000:9000"
      - "9001:9001"
    networks:
      - pos-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  prometheus:
    image: prom/prometheus:latest
    container_name: pos-prometheus
    restart: always
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - /data/ssd/prometheus:/prometheus
    ports:
      - "9090:9090"
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=90d"
    networks:
      - pos-network

  node-exporter:
    image: prom/node-exporter:latest
    container_name: pos-node-exporter
    restart: always
    command:
      - "--path.rootfs=/host"
    pid: host
    volumes:
      - "/:/host:ro,rslave"
    ports:
      - "9100:9100"
    networks:
      - pos-network

  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    container_name: pos-postgres-exporter
    restart: always
    environment:
      DATA_SOURCE_NAME: postgresql://postgres:${POSTGRES_PASSWORD}@postgres:5432/pos_system?sslmode=disable
    ports:
      - "9187:9187"
    depends_on:
      - postgres
    networks:
      - pos-network

  redis-exporter:
    image: oliver006/redis_exporter:latest
    container_name: pos-redis-exporter
    restart: always
    environment:
      REDIS_ADDR: redis:6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    ports:
      - "9121:9121"
    depends_on:
      - redis
    networks:
      - pos-network

networks:
  pos-network:
    driver: bridge
```

### Step 2: Create Environment File

```bash
vim .env
```

```bash
# .env
POSTGRES_PASSWORD=your_super_secure_password_here
REDIS_PASSWORD=your_redis_password_here
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=your_minio_password_here
```

```bash
# Secure the env file
chmod 600 .env
```

### Step 3: Create Prometheus Configuration

```bash
vim prometheus.yml
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: "pos-primary"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "postgres"
    static_configs:
      - targets: ["postgres-exporter:9187"]

  - job_name: "redis"
    static_configs:
      - targets: ["redis-exporter:9121"]
```

### Step 4: Create Database Initialization Scripts

```bash
mkdir -p init-scripts
vim init-scripts/01-create-tenant-function.sql
```

```sql
-- init-scripts/01-create-tenant-function.sql

-- Create tenant management functions
CREATE OR REPLACE FUNCTION create_tenant_schema(tenant_code VARCHAR)
RETURNS VOID AS $$
BEGIN
  -- Create schema
  EXECUTE format('CREATE SCHEMA IF NOT EXISTS %I', tenant_code);

  -- Create transactions table
  EXECUTE format('
    CREATE TABLE %I.transactions (
      id BIGSERIAL PRIMARY KEY,
      invoice_number VARCHAR(50) UNIQUE NOT NULL,
      transaction_date TIMESTAMPTZ NOT NULL,
      amount DECIMAL(15, 2) NOT NULL,
      vat_amount DECIMAL(15, 2),
      vat_exempt_amount DECIMAL(15, 2) DEFAULT 0,
      zero_rated_amount DECIMAL(15, 2) DEFAULT 0,
      discount_amount DECIMAL(15, 2) DEFAULT 0,
      discount_type VARCHAR(50),
      cashier_id INTEGER,
      terminal_id VARCHAR(50),
      payment_method VARCHAR(50),
      status VARCHAR(20) DEFAULT ''completed'',
      voided BOOLEAN DEFAULT FALSE,
      voided_at TIMESTAMPTZ,
      voided_by INTEGER,
      void_reason TEXT,
      created_at TIMESTAMPTZ DEFAULT NOW(),
      updated_at TIMESTAMPTZ DEFAULT NOW()
    )', tenant_code);

  -- Create transaction items table
  EXECUTE format('
    CREATE TABLE %I.transaction_items (
      id BIGSERIAL PRIMARY KEY,
      transaction_id BIGINT REFERENCES %I.transactions(id),
      product_id INTEGER NOT NULL,
      product_name VARCHAR(255) NOT NULL,
      sku VARCHAR(100),
      quantity INTEGER NOT NULL,
      unit_price DECIMAL(10, 2) NOT NULL,
      discount DECIMAL(10, 2) DEFAULT 0,
      vat_rate DECIMAL(5, 4) DEFAULT 0.12,
      subtotal DECIMAL(15, 2) NOT NULL,
      created_at TIMESTAMPTZ DEFAULT NOW()
    )', tenant_code, tenant_code);

  -- Create products table
  EXECUTE format('
    CREATE TABLE %I.products (
      id SERIAL PRIMARY KEY,
      sku VARCHAR(100) UNIQUE NOT NULL,
      barcode VARCHAR(100),
      name VARCHAR(255) NOT NULL,
      description TEXT,
      category VARCHAR(100),
      price DECIMAL(10, 2) NOT NULL,
      cost DECIMAL(10, 2),
      quantity INTEGER DEFAULT 0,
      reorder_level INTEGER DEFAULT 10,
      vat_type VARCHAR(20) DEFAULT ''vatable'',
      active BOOLEAN DEFAULT TRUE,
      created_at TIMESTAMPTZ DEFAULT NOW(),
      updated_at TIMESTAMPTZ DEFAULT NOW()
    )', tenant_code);

  -- Create inventory movements table
  EXECUTE format('
    CREATE TABLE %I.inventory_movements (
      id BIGSERIAL PRIMARY KEY,
      product_id INTEGER REFERENCES %I.products(id),
      movement_type VARCHAR(50) NOT NULL,
      quantity INTEGER NOT NULL,
      reference_id VARCHAR(100),
      reference_type VARCHAR(50),
      notes TEXT,
      user_id INTEGER,
      created_at TIMESTAMPTZ DEFAULT NOW()
    )', tenant_code, tenant_code);

  -- Create daily summaries table (for Z-reading)
  EXECUTE format('
    CREATE TABLE %I.daily_summaries (
      id SERIAL PRIMARY KEY,
      date DATE UNIQUE NOT NULL,
      terminal_id VARCHAR(50),
      opening_invoice VARCHAR(50),
      closing_invoice VARCHAR(50),
      transaction_count INTEGER DEFAULT 0,
      gross_sales DECIMAL(15, 2) DEFAULT 0,
      vat_sales DECIMAL(15, 2) DEFAULT 0,
      vat_amount DECIMAL(15, 2) DEFAULT 0,
      vat_exempt_sales DECIMAL(15, 2) DEFAULT 0,
      zero_rated_sales DECIMAL(15, 2) DEFAULT 0,
      discount_total DECIMAL(15, 2) DEFAULT 0,
      void_count INTEGER DEFAULT 0,
      void_amount DECIMAL(15, 2) DEFAULT 0,
      net_sales DECIMAL(15, 2) DEFAULT 0,
      z_counter INTEGER,
      finalized BOOLEAN DEFAULT FALSE,
      finalized_at TIMESTAMPTZ,
      finalized_by INTEGER,
      created_at TIMESTAMPTZ DEFAULT NOW()
    )', tenant_code);

  -- Create e-journal table
  EXECUTE format('
    CREATE TABLE %I.ejournal (
      id BIGSERIAL PRIMARY KEY,
      event_type VARCHAR(50) NOT NULL,
      event_data JSONB NOT NULL,
      user_id INTEGER,
      terminal_id VARCHAR(50),
      ip_address VARCHAR(50),
      created_at TIMESTAMPTZ DEFAULT NOW()
    )', tenant_code);

  -- Create indexes
  EXECUTE format('CREATE INDEX idx_%I_transactions_date ON %I.transactions(transaction_date DESC)', tenant_code, tenant_code);
  EXECUTE format('CREATE INDEX idx_%I_transactions_invoice ON %I.transactions(invoice_number)', tenant_code, tenant_code);
  EXECUTE format('CREATE INDEX idx_%I_transactions_cashier ON %I.transactions(cashier_id)', tenant_code, tenant_code);
  EXECUTE format('CREATE INDEX idx_%I_products_sku ON %I.products(sku)', tenant_code, tenant_code);
  EXECUTE format('CREATE INDEX idx_%I_products_barcode ON %I.products(barcode)', tenant_code, tenant_code);
  EXECUTE format('CREATE INDEX idx_%I_inventory_product ON %I.inventory_movements(product_id, created_at DESC)', tenant_code, tenant_code);
  EXECUTE format('CREATE INDEX idx_%I_ejournal_date ON %I.ejournal(created_at DESC)', tenant_code, tenant_code);

  RAISE NOTICE 'Tenant schema % created successfully', tenant_code;
END;
$$ LANGUAGE plpgsql;

-- Create shared tables
CREATE TABLE IF NOT EXISTS public.tenants (
  id SERIAL PRIMARY KEY,
  tenant_id VARCHAR(50) UNIQUE NOT NULL,
  business_name VARCHAR(255) NOT NULL,
  business_address TEXT,
  tin VARCHAR(20) NOT NULL,
  ptu_number VARCHAR(50),
  vat_registered BOOLEAN DEFAULT TRUE,
  plan_type VARCHAR(50) DEFAULT 'basic',
  status VARCHAR(20) DEFAULT 'active',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS public.users (
  id SERIAL PRIMARY KEY,
  tenant_id VARCHAR(50) REFERENCES public.tenants(tenant_id),
  username VARCHAR(100) NOT NULL,
  email VARCHAR(255) NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  role VARCHAR(50) NOT NULL,
  permissions JSONB,
  active BOOLEAN DEFAULT TRUE,
  last_login TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(tenant_id, username),
  UNIQUE(tenant_id, email)
);

CREATE TABLE IF NOT EXISTS public.terminals (
  id SERIAL PRIMARY KEY,
  tenant_id VARCHAR(50) REFERENCES public.tenants(tenant_id),
  terminal_id VARCHAR(50) NOT NULL,
  terminal_name VARCHAR(100),
  machine_serial VARCHAR(100),
  branch_code VARCHAR(50),
  active BOOLEAN DEFAULT TRUE,
  last_seen TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(tenant_id, terminal_id)
);

-- Create indexes for shared tables
CREATE INDEX idx_users_tenant ON public.users(tenant_id);
CREATE INDEX idx_terminals_tenant ON public.terminals(tenant_id);
```

### Step 5: Deploy Services

```bash
# Start all services
cd /opt/pos-system
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs -f

# Check specific service
docker compose logs -f postgres
docker compose logs -f redis
docker compose logs -f minio
```

### Step 6: Verify Installation

```bash
# Test PostgreSQL
docker exec -it pos-postgres psql -U postgres -c "SELECT version();"

# Test Redis
docker exec -it pos-redis redis-cli -a ${REDIS_PASSWORD} ping

# Test MinIO
curl http://localhost:9000/minio/health/live

# Check Prometheus
curl http://localhost:9090/-/healthy
```

---

## 🔒 Phase 4: Security Configuration

### Step 1: Configure Firewall

```bash
# Install UFW
sudo apt install -y ufw

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (change default port for security)
sudo vim /etc/ssh/sshd_config
# Change: Port 22 to Port 2222
# Change: PermitRootLogin no
# Add: AllowUsers posadmin

sudo systemctl restart sshd
sudo ufw allow 2222/tcp

# Allow services only from VPN subnet (configure after VPN setup)
# sudo ufw allow from 10.0.0.0/24 to any port 5432
# sudo ufw allow from 10.0.0.0/24 to any port 6379
# sudo ufw allow from 10.0.0.0/24 to any port 9000

# Enable firewall
sudo ufw enable
sudo ufw status verbose
```

### Step 2: Setup WireGuard VPN

```bash
# Install WireGuard
sudo apt install -y wireguard wireguard-tools

# Generate server keys
cd /etc/wireguard
umask 077
wg genkey | tee privatekey | wg pubkey > publickey

# Create configuration
sudo vim /etc/wireguard/wg0.conf
```

```ini
# /etc/wireguard/wg0.conf
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = YOUR_PRIVATE_KEY_HERE
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eno1 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eno1 -j MASQUERADE

# App Server 1
[Peer]
PublicKey = APP_SERVER_1_PUBLIC_KEY
AllowedIPs = 10.0.0.10/32

# App Server 2
[Peer]
PublicKey = APP_SERVER_2_PUBLIC_KEY
AllowedIPs = 10.0.0.11/32

# Backup Server
[Peer]
PublicKey = BACKUP_SERVER_PUBLIC_KEY
AllowedIPs = 10.0.0.2/32
```

```bash
# Enable IP forwarding
sudo vim /etc/sysctl.conf
# Uncomment: net.ipv4.ip_forward=1
sudo sysctl -p

# Start VPN
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

# Check status
sudo wg show

# Allow VPN port through firewall
sudo ufw allow 51820/udp

# Now allow database access only from VPN
sudo ufw allow from 10.0.0.0/24 to any port 5432
sudo ufw allow from 10.0.0.0/24 to any port 6379
sudo ufw allow from 10.0.0.0/24 to any port 9000
sudo ufw allow from 10.0.0.0/24 to any port 9090
```

### Step 3: SSL/TLS Configuration

```bash
# Generate SSL certificates for internal use
sudo apt install -y certbot

# For internal services, use self-signed certificates
sudo mkdir -p /etc/ssl/pos-system
cd /etc/ssl/pos-system

# Generate CA
sudo openssl genrsa -out ca.key 4096
sudo openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt

# Generate server certificate
sudo openssl genrsa -out server.key 4096
sudo openssl req -new -key server.key -out server.csr
sudo openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 1825 -sha256

# Set permissions
sudo chmod 600 *.key
sudo chmod 644 *.crt
```

---

## 💾 Phase 5: Backup Configuration

### Step 1: Create Backup Scripts

```bash
sudo mkdir -p /usr/local/bin/pos-backups
sudo vim /usr/local/bin/pos-backups/backup-databases.sh
```

```bash
#!/bin/bash
# /usr/local/bin/pos-backups/backup-databases.sh

set -e  # Exit on error

# Configuration
DATE=$(date +%Y%m%d_%H%M%S)
DAY=$(date +%Y%m%d)
BACKUP_DIR="/data/storage/backups"
NAS_DIR="/mnt/nas/backups"
LOG_FILE="/var/log/pos-backups.log"
RETENTION_DAYS=30
POSTGRES_CONTAINER="pos-postgres"
REDIS_CONTAINER="pos-redis"

# Logging function
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "=== Starting backup process ==="

# Create backup directory
mkdir -p "$BACKUP_DIR/$DAY"

# 1. Backup PostgreSQL - Full database
log "Backing up PostgreSQL (full database)..."
docker exec $POSTGRES_CONTAINER pg_dumpall -U postgres | gzip > "$BACKUP_DIR/$DAY/postgres_full_$DATE.sql.gz"

if [ $? -eq 0 ]; then
    log "PostgreSQL full backup completed successfully"
else
    log "ERROR: PostgreSQL full backup failed"
    exit 1
fi

# 2. Backup each tenant separately
log "Backing up individual tenant schemas..."
TENANTS=$(docker exec $POSTGRES_CONTAINER psql -U postgres -d pos_system -t -c "SELECT tenant_id FROM public.tenants WHERE status='active'" | xargs)

for tenant in $TENANTS; do
    if [ ! -z "$tenant" ]; then
        log "Backing up tenant: $tenant"
        docker exec $POSTGRES_CONTAINER pg_dump -U postgres -d pos_system -n "$tenant" | gzip > "$BACKUP_DIR/$DAY/tenant_${tenant}_$DATE.sql.gz"
    fi
done

# 3. Backup Redis
log "Backing up Redis..."
docker exec $REDIS_CONTAINER redis-cli --no-auth-warning -a "$REDIS_PASSWORD" BGSAVE
sleep 5  # Wait for BGSAVE to complete
cp /data/ssd/redis/dump.rdb "$BACKUP_DIR/$DAY/redis_$DATE.rdb"

# 4. Backup MinIO metadata (data already replicated)
log "Backing up MinIO configuration..."
tar -czf "$BACKUP_DIR/$DAY/minio_config_$DATE.tar.gz" -C /data/storage/minio .minio.sys/ 2>/dev/null || true

# 5. Create checksums
log "Creating checksums..."
cd "$BACKUP_DIR/$DAY"
sha256sum * > checksums.txt

# 6. Copy to NAS
if [ -d "$NAS_DIR" ]; then
    log "Copying to NAS..."
    rsync -avz --delete "$BACKUP_DIR/$DAY/" "$NAS_DIR/$DAY/"
    if [ $? -eq 0 ]; then
        log "NAS sync completed successfully"
    else
        log "WARNING: NAS sync failed"
    fi
else
    log "WARNING: NAS directory not mounted, skipping NAS backup"
fi

# 7. Clean old backups
log "Cleaning old backups..."
find "$BACKUP_DIR" -type d -mtime +$RETENTION_DAYS -exec rm -rf {} \; 2>/dev/null || true

# 8. Backup sizes
BACKUP_SIZE=$(du -sh "$BACKUP_DIR/$DAY" | cut -f1)
log "Backup size: $BACKUP_SIZE"

log "=== Backup process completed successfully ==="
```

```bash
# Make executable
sudo chmod +x /usr/local/bin/pos-backups/backup-databases.sh

# Test backup script
sudo /usr/local/bin/pos-backups/backup-databases.sh
```

### Step 2: Setup Cron Jobs

```bash
# Edit crontab
sudo crontab -e
```

```bash
# PostgreSQL and Redis backup - Daily at 2 AM
0 2 * * * /usr/local/bin/pos-backups/backup-databases.sh >> /var/log/pos-backups.log 2>&1

# Weekly full backup - Sunday at 3 AM
0 3 * * 0 /usr/local/bin/pos-backups/full-backup.sh >> /var/log/pos-backups.log 2>&1

# Monthly archive - 1st of month at 4 AM
0 4 1 * * /usr/local/bin/pos-backups/monthly-archive.sh >> /var/log/pos-backups.log 2>&1

# Check disk space - Every 6 hours
0 */6 * * * /usr/local/bin/pos-backups/check-disk-space.sh >> /var/log/disk-space.log 2>&1

# Health check - Every hour
0 * * * * /usr/local/bin/pos-backups/health-check.sh >> /var/log/health-check.log 2>&1
```

### Step 3: Mount NAS for Backups

```bash
# Install NFS client
sudo apt install -y nfs-common

# Create mount point
sudo mkdir -p /mnt/nas

# Mount NAS
sudo mount -t nfs NAS_IP:/volume1/backups /mnt/nas

# Add to /etc/fstab for automatic mount
echo "NAS_IP:/volume1/backups /mnt/nas nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab

# Test mount
df -h /mnt/nas
```

---

## 📊 Phase 6: Monitoring Setup

### Step 1: Access Prometheus

```bash
# Prometheus should be running at:
http://YOUR_SERVER_IP:9090

# Check targets
# Navigate to Status > Targets
# All targets should be "UP"
```

### Step 2: Setup Grafana (on app server or separate machine)

```bash
# On application server or monitoring machine
docker run -d \
  --name=grafana \
  -p 3000:3000 \
  -v grafana-storage:/var/lib/grafana \
  --restart=always \
  grafana/grafana-oss:latest

# Access Grafana
# http://YOUR_SERVER_IP:3000
# Default login: admin/admin
```

### Step 3: Configure Grafana

1. Add Prometheus as data source:

   - URL: `http://PROMETHEUS_IP:9090`
   - Access: Server

2. Import dashboards:

   - Node Exporter Full (ID: 1860)
   - PostgreSQL Database (ID: 9628)
   - Redis Dashboard (ID: 11835)

3. Create custom dashboards for business metrics

---

## ✅ Phase 7: Verification & Testing

### Step 1: Test Database Connection

```bash
# Test PostgreSQL
docker exec -it pos-postgres psql -U postgres -d pos_system

# Create test tenant
INSERT INTO public.tenants (tenant_id, business_name, tin, ptu_number)
VALUES ('test_001', 'Test Store', '123-456-789-000', 'PTU-TEST-001');

SELECT create_tenant_schema('test_001');

# Verify schema was created
\dn

# Test tenant schema
\c pos_system
SET search_path TO test_001;
\dt

# Insert test data
INSERT INTO test_001.products (sku, name, price, cost, quantity)
VALUES ('TEST001', 'Test Product', 100.00, 50.00, 10);

SELECT * FROM test_001.products;

\q
```

### Step 2: Test Redis

```bash
# Test Redis connection
docker exec -it pos-redis redis-cli -a YOUR_REDIS_PASSWORD

# Test commands
SET test:key "Hello World"
GET test:key
DEL test:key
QUIT
```

### Step 3: Test MinIO

```bash
# Install MinIO client
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Configure MinIO client
mc alias set pos-minio http://localhost:9000 YOUR_MINIO_USER YOUR_MINIO_PASSWORD

# Create test bucket
mc mb pos-minio/test-bucket

# Upload test file
echo "Test file" > test.txt
mc cp test.txt pos-minio/test-bucket/

# List files
mc ls pos-minio/test-bucket/

# Download file
mc cp pos-minio/test-bucket/test.txt downloaded.txt
cat downloaded.txt

# Clean up
rm test.txt downloaded.txt
mc rb --force pos-minio/test-bucket
```

### Step 4: Test Backups

```bash
# Run backup manually
sudo /usr/local/bin/pos-backups/backup-databases.sh

# Check backup files
ls -lh /data/storage/backups/$(date +%Y%m%d)/

# Verify checksums
cd /data/storage/backups/$(date +%Y%m%d)/
sha256sum -c checksums.txt
```

### Step 5: Test Restore

```bash
# Test PostgreSQL restore
gunzip -c /data/storage/backups/YYYYMMDD/tenant_test_001_*.sql.gz | \
  docker exec -i pos-postgres psql -U postgres -d pos_system

# Verify data
docker exec -it pos-postgres psql -U postgres -d pos_system -c "SELECT * FROM test_001.products;"
```

---

## 🚀 Phase 8: Application Deployment

### Deploy Backend API (on app server via VPN)

See separate application deployment documentation.

---

## 📝 Post-Installation Checklist

### Security

- [ ] Firewall configured and enabled
- [ ] SSH on non-standard port
- [ ] VPN configured and tested
- [ ] SSL certificates installed
- [ ] Database passwords changed from defaults
- [ ] Disk encryption enabled (optional)
- [ ] Fail2ban installed and configured

### Monitoring

- [ ] Prometheus collecting metrics
- [ ] Grafana dashboards created
- [ ] Alerts configured
- [ ] Health checks running
- [ ] Log rotation configured

### Backups

- [ ] Backup scripts tested
- [ ] Cron jobs configured
- [ ] NAS mounted and accessible
- [ ] Restore procedure tested
- [ ] Off-site backup configured

### Performance

- [ ] PostgreSQL tuned for workload
- [ ] PgBouncer connection pooling working
- [ ] Redis caching tested
- [ ] Disk I/O performance acceptable
- [ ] Network bandwidth tested

### Documentation

- [ ] All passwords documented in secure location
- [ ] Network diagram created
- [ ] Recovery procedures documented
- [ ] Contact information updated
- [ ] Maintenance schedule planned

---

## 🆘 Troubleshooting

### Docker Containers Won't Start

```bash
# Check logs
docker compose logs SERVICE_NAME

# Check disk space
df -h

# Check Docker status
sudo systemctl status docker

# Restart Docker
sudo systemctl restart docker
```

### Database Connection Issues

```bash
# Check PostgreSQL is running
docker exec pos-postgres pg_isready -U postgres

# Check PostgreSQL logs
docker logs pos-postgres --tail 100

# Test connection
docker exec -it pos-postgres psql -U postgres -c "SELECT 1"
```

### Backup Failures

```bash
# Check backup logs
tail -f /var/log/pos-backups.log

# Check disk space
df -h /data/storage

# Check NAS mount
df -h /mnt/nas
mount | grep nas
```

### Performance Issues

```bash
# Check system resources
htop
iotop
iftop

# Check PostgreSQL stats
docker exec -it pos-postgres psql -U postgres -c "
SELECT * FROM pg_stat_activity WHERE state != 'idle';
"

# Check slow queries
docker exec -it pos-postgres psql -U postgres -c "
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
"
```

---

## 📚 Next Steps

1. **Deploy Application Servers** - Setup Node.js backend and Next.js frontend on VPS
2. **Configure Load Balancing** - Setup Nginx or HAProxy
3. **Setup Domain & SSL** - Configure domain names and Let's Encrypt SSL
4. **Create First Tenant** - Onboard first customer
5. **BIR Accreditation** - Begin PTU application process

---

**Your self-hosted infrastructure is now ready! 🎉**
