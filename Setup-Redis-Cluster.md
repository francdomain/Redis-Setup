# Set Up Redis on Ubuntu

Redis is an in-memory data structure store that serves as a database, cache, message broker, and streaming engine. Its speed, flexibility, and rich feature set make it a go-to choice for production applications requiring low-latency data access. This guide walks you through setting up Redis on Ubuntu for production environments, covering installation, persistence strategies, security hardening, memory management, high availability with Sentinel, and Redis Cluster for horizontal scaling.

## Prerequisites

**Ensure the following**:

- Ubuntu 22.04 LTS or newer
- Root or sudo access
- At least 2GB of RAM (more for production workloads)
- Basic familiarity with Linux command line

### Installing Redis on Ubuntu

First, update your package index to ensure you get the latest available version:

```bash
sudo apt update && sudo apt upgrade -y
```

Ubuntu's default repositories include Redis, but for the latest stable version, we'll use the official Redis repository:

```bash
# Install prerequisites for adding repositories
sudo apt install -y curl wget gnupg lsb-release

# Add the official Redis GPG key for package verification
wget -qO- https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

# Add the Redis repository to your sources list
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

# Update package index and install Redis
sudo apt update
sudo apt install -y redis-server redis-tools
```

Ubuntu package installs include a systemd unit at `/lib/systemd/system/redis-server.service`.

![alt text](<./images/Screenshot 2026-04-01 at 7.31.10 AM.png>)

### Verify Installation

Confirm Redis is installed and running correctly:

```bash
redis-server --version
redis-cli --version

sudo systemctl status redis-server

# Test connectivity with a simple ping
redis-cli ping
# Expected output: PONG
```

![alt text](<./images/Screenshot 2026-04-01 at 7.40.00 AM.png>)

### Enable Redis to Start on Boot

Ensure Redis starts automatically when the system boots:

```bash
sudo systemctl enable redis-server
sudo systemctl is-enabled redis-server
```

![alt text](<./images/Screenshot 2026-04-01 at 7.42.38 AM.png>)

# Configuring Redis for Production

The default Redis configuration is suitable for development but requires tuning for production. The main configuration file is located at `/etc/redis/redis.conf`

### Create a Backup of the Original Configuration

Always backup configuration files before making changes:

```bash
sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.backup.$(date +%Y%m%d)
```

![alt text](<./images/Screenshot 2026-04-01 at 7.49.15 AM.png>)

## Basic Production Configuration

Edit the Redis configuration file with your preferred editor:

```bash
sudo vim /etc/redis/redis.conf
```

## Apply the following essential production settings:

```bash
# Bind to specific interfaces (localhost only for security)
# Change this if Redis needs to accept connections from other hosts
bind 127.0.0.1 ::1

# Use protected mode for additional security when bound to localhost
protected-mode yes

# Set a custom port (default is 6379)
port 6379

# Set the number of databases (default is 16)
databases 16

# Log level: debug, verbose, notice, warning
# Use 'notice' for production to balance information and performance
loglevel notice

# Specify the log file location
logfile /var/log/redis/redis-server.log

# Set the working directory for RDB and AOF files
dir /var/lib/redis

# Let systemd manage the process lifecycle
daemonize no
supervised systemd

# PID file location
pidfile /run/redis/redis-server.pid

# Timeout for idle client connections (0 = no timeout)
# Set to 300 seconds (5 minutes) for production
timeout 300

# TCP keepalive - send keepalive packets every 300 seconds
tcp-keepalive 300
```

### Apply Configuration Changes

After modifying the configuration, restart Redis to apply changes:

```bash
# Restart Redis to apply new configuration
sudo systemctl restart redis-server

# Verify Redis is running with new configuration
sudo systemctl status redis-server

# Check if Redis is listening on the configured port
sudo ss -tlnp | grep redis
```

![alt text](<./images/Screenshot 2026-04-01 at 8.57.37 AM.png>)

## Persistence Configuration: RDB vs AOF

Redis offers two persistence mechanisms to ensure data durability. Understanding and configuring these correctly is crucial for production environments.

### RDB (Redis Database) Snapshots

RDB creates point-in-time snapshots of your dataset at specified intervals. It's compact and excellent for backups but may lose data between snapshots.

Configure RDB persistence in `/etc/redis/redis.conf`:

```bash
# RDB Persistence Configuration
# Save the database to disk at specified intervals
# Format: save <seconds> <changes>

# Save after 900 seconds (15 min) if at least 1 key changed
save 900 1

# Save after 300 seconds (5 min) if at least 10 keys changed
save 300 10

# Save after 60 seconds if at least 10000 keys changed
save 60 10000

# Stop accepting writes if RDB snapshot fails (data safety)
stop-writes-on-bgsave-error yes

# Compress RDB files using LZF compression
rdbcompression yes

# Enable RDB checksum for data integrity verification
rdbchecksum yes

# Name of the RDB file
dbfilename dump.rdb

# Directory where RDB file will be stored
dir /var/lib/redis
```

![alt text](<./images/Screenshot 2026-04-01 at 10.35.10 AM.png>)

## AOF (Append Only File) Persistence

AOF logs every write operation, providing better durability at the cost of larger files. This is recommended for production when data loss must be minimized.

Configure AOF persistence in `/etc/redis/redis.conf`:

```bash
# AOF Persistence Configuration
# Enable Append Only File persistence
appendonly yes

# Name of the AOF file
appendfilename "appendonly.aof"

# Fsync policy: always, everysec, or no
# 'everysec' is recommended for production (good balance of safety and speed)
appendfsync everysec

# Don't fsync during background save operations (better performance)
no-appendfsync-on-rewrite no

# Auto-rewrite AOF when it grows by this percentage
auto-aof-rewrite-percentage 100

# Minimum size for AOF rewrite to trigger
auto-aof-rewrite-min-size 64mb

# Load truncated AOF files (safer option)
aof-load-truncated yes

# Enable AOF-RDB preamble for faster loading
aof-use-rdb-preamble yes
```

### Hybrid Persistence (Recommended for Production)

For optimal durability and performance, enable both RDB and AOF:

```
# Enable both persistence mechanisms for maximum durability
# RDB provides fast backups and recovery
save 900 1
save 300 10
save 60 10000
dbfilename dump.rdb

# AOF provides minimal data loss (up to 1 second with everysec)
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
aof-use-rdb-preamble yes
```

## Verify Persistence Settings

Check that persistence is working correctly:

```bash
# Connect to Redis and check persistence info
redis-cli INFO persistence

# Check if RDB file exists
ls -la /var/lib/redis/dump.rdb

# Check if AOF file exists (if enabled)
ls -la /var/lib/redis/appendonly.aof

# Force a manual RDB save to test
redis-cli BGSAVE

# Check the last save timestamp
redis-cli LASTSAVE
```

![alt text](<./images/Screenshot 2026-04-01 at 10.55.13 AM.png>)
![alt text](<./images/Screenshot 2026-04-01 at 11.00.27 AM.png>)

## Security Hardening

Securing Redis is critical for production deployments. By default, Redis has no authentication, making security configuration essential.

### Password Authentication

Set a strong password for Redis authentication:

```bash
# Add to /etc/redis/redis.conf
# Use a strong, random password (at least 32 characters recommended)
requirepass your_very_strong_password_here_at_least_32_chars

# Also set the master password if this is a replica
masterauth your_very_strong_password_here_at_least_32_chars
```

Generate a secure password using OpenSSL:

```
# Generate a 64-character random password
openssl rand -base64 48

# Example output: Xa9K7mN2pL5vB8xC3dF6gH1jQ4rT0wY9zA2eI5oU8uP3sM6nK7lJ0hG=
```

#### Test Password Authentication

After setting the password and restarting Redis:

```bash
# Restart Redis to apply password
sudo systemctl restart redis-server

# This should fail without authentication
redis-cli ping
# Output: (error) NOAUTH Authentication required.

# Authenticate and run commands
redis-cli -a your_password ping
# Output: PONG

# Or authenticate after connecting
redis-cli
AUTH your_password
ping
# Output: PONG
```

![alt text](<./images/Screenshot 2026-04-01 at 11.18.38 AM.png>)

### Disable Dangerous Commands

Rename or disable commands that could be dangerous in production:

```bash
# Rename dangerous commands to prevent accidental execution
# Use empty string "" to completely disable the command

# Disable FLUSHDB and FLUSHALL (delete all data)
rename-command FLUSHDB ""
rename-command FLUSHALL ""

# Rename DEBUG command
rename-command DEBUG "DEBUG_a1b2c3d4e5f6"

# Rename CONFIG command to prevent configuration changes
rename-command CONFIG "CONFIG_x9y8z7w6v5u4"

# Rename SHUTDOWN to prevent accidental shutdowns
rename-command SHUTDOWN "SHUTDOWN_m3n4o5p6q7r8"

# Disable KEYS command (can cause performance issues)
rename-command KEYS "KEYS_disabled_s1t2u3v4"
```

# Network Security

Configure network settings to minimize attack surface:

```bash
# Bind only to localhost if Redis doesn't need external access
bind 127.0.0.1

# If external access is needed, bind to specific interface
# bind 192.168.1.100 127.0.0.1

# Enable protected mode (blocks external connections without password)
protected-mode yes

# Disable public access to Redis if not needed
# Only use this in a secured network environment
```

![alt text](<./images/Screenshot 2026-04-01 at 4.46.33 PM.png>)

### Firewall Configuration

Set up UFW (Uncomplicated Firewall) rules for Redis:

```bash
# Enable UFW if not already enabled
sudo ufw enable

# Allow Redis only from specific IP addresses (recommended)
sudo ufw allow from 192.168.1.0/24 to any port 6379 proto tcp comment 'Redis access from local network'

# Or allow from a specific server
sudo ufw allow from 10.0.0.5 to any port 6379 proto tcp comment 'Redis access from app server'

# Deny Redis port from all other sources (default behavior if not explicitly allowed)
sudo ufw status

# Check the rules
sudo ufw status numbered
```

### TLS/SSL Encryption

For production environments with external access, enable TLS encryption:

```bash
# Generate self-signed certificates (use proper CA-signed certs for production)
sudo mkdir -p /etc/redis/tls
cd /etc/redis/tls

# Generate private key and certificate
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout redis.key \
  -out redis.crt \
  -subj "/CN=redis.local"

# Generate DH parameters for additional security
sudo openssl dhparam -out redis-dh.pem 2048

# Set proper permissions
sudo chown -R redis:redis /etc/redis/tls
sudo chmod 600 /etc/redis/tls/redis.key
```

![alt text](<./images/Screenshot 2026-04-01 at 12.56.07 PM.png>)

Add TLS configuration to `/etc/redis/redis.conf`:

```bash
# TLS/SSL Configuration
tls-port 6380
port 0  # Disable non-TLS port

# Certificate and key files
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key

# DH parameters file
tls-dh-params-file /etc/redis/tls/redis-dh.pem

# Require clients to authenticate with certificates (optional)
# tls-auth-clients yes
# tls-ca-cert-file /etc/redis/tls/ca.crt

# TLS protocols (disable older, insecure protocols)
tls-protocols "TLSv1.2 TLSv1.3"

# Prefer server ciphers
tls-prefer-server-ciphers yes
```

![alt text](<./images/Screenshot 2026-04-01 at 2.51.17 PM.png>)

### Connect to Redis using TLS:

**Note:**

1. `--cert/--key` are for client authentication (mTLS). They do not tell redis-cli what CA to trust for the server cert.
2. Since a self-signed server cert is used, you must either:
   provide it as a trusted CA using `--cacert`, or
   bypass verification with `--insecure` (testing only).
3. If Redis config has `tls-auth-clients yes`, then include client `cert/key`:

```bash
# If tls-auth-clients yes
redis-cli --tls --cacert /etc/redis/tls/redis.crt --cert /etc/redis/tls/redis.crt --key /etc/redis/tls/redis.key -p 6380

# If using self-signed certificates and tls-auth-clients is set to no
redis-cli --tls --cacert /etc/redis/tls/redis.crt -p 6380

# For testing only
redis-cli --tls --insecure -p 6380
```

![alt text](<./images/Screenshot 2026-04-01 at 5.19.28 PM.png>)

# Memory Management

Proper memory management prevents Redis from consuming all available system memory and ensures stable operation.

### Configure Memory Limits

**Set memory limits and eviction policies in `/etc/redis/redis.conf`:**

```bash
# Maximum memory Redis can use (adjust based on your server)
# Set to 75% of available RAM, leaving room for OS and other processes
maxmemory 2gb

# Memory eviction policy when maxmemory is reached
# Options:
# - volatile-lru: Evict keys with expiration set, least recently used first
# - allkeys-lru: Evict any key, least recently used first (recommended for cache)
# - volatile-lfu: Evict keys with expiration, least frequently used first
# - allkeys-lfu: Evict any key, least frequently used first
# - volatile-random: Evict random keys with expiration set
# - allkeys-random: Evict random keys
# - volatile-ttl: Evict keys with shortest TTL first
# - noeviction: Return errors when memory limit is reached
maxmemory-policy allkeys-lru

# Number of samples to check for LRU/LFU eviction (higher = more accurate but slower)
maxmemory-samples 10
```

![alt text](<./images/Screenshot 2026-04-01 at 5.34.22 PM.png>)

## Monitor Memory Usage

Use Redis commands to monitor memory:

```bash
# Check memory usage summary
redis-cli INFO memory

# Get detailed memory statistics
redis-cli MEMORY STATS

# Check memory usage of a specific key
redis-cli MEMORY USAGE mykey

# Get memory doctor recommendations
redis-cli MEMORY DOCTOR
```

![alt text](<./images/Screenshot 2026-04-01 at 5.45.38 PM.png>)

![alt text](<./images/Screenshot 2026-04-01 at 5.50.50 PM.png>)

![alt text](<./images/Screenshot 2026-04-01 at 6.10.22 PM.png>)

### Memory Optimization Tips

Configure additional memory optimization settings:

```bash
# Enable active memory defragmentation (Redis 4.0+)
activedefrag yes

# Start defragmentation when fragmentation exceeds this percentage
active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100

# CPU effort for defragmentation (1-100)
active-defrag-cycle-min 1
active-defrag-cycle-max 25

# Use jemalloc memory allocator (usually default on Linux)
# Check with: redis-cli INFO memory | grep mem_allocator

# Lazy freeing - asynchronous deletion to prevent blocking
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
replica-lazy-flush yes
```

![alt text](<./images/Screenshot 2026-04-01 at 6.30.14 PM.png>)

![alt text](<./images/Screenshot 2026-04-01 at 6.32.43 PM.png>)

# System Memory Configuration

Optimize system settings for Redis:

```bash
# Configure kernel settings in a dedicated file
sudo tee /etc/sysctl.d/99-redis.conf >/dev/null << 'EOF'
vm.overcommit_memory = 1
net.core.somaxconn = 65535
EOF

# Load sysctl settings
sudo sysctl --system

# Disable Transparent Huge Pages immediately
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag

# Persist THP settings with systemd
sudo tee /etc/systemd/system/disable-thp.service >/dev/null << 'EOF'
[Unit]
Description=Disable Transparent Huge Pages (THP)
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now disable-thp
```

![alt text](<./images/Screenshot 2026-04-01 at 6.41.28 PM.png>)

![alt text](<./images/Screenshot 2026-04-01 at 6.44.41 PM.png>)

```bash
sudo vim /etc/systemd/system/disable-thp.service
```

![alt text](<./images/Screenshot 2026-04-01 at 6.45.18 PM.png>)

# Redis Sentinel for High Availability

Redis Sentinel provides high availability through automatic failover, monitoring, and configuration provider services.

## Sentinel Architecture Overview

A typical Sentinel setup includes:

- 1 Master Redis instance
- 2+ Replica Redis instances
- 3+ Sentinel instances (odd number for quorum)

## Configure Redis Master

**On the master server, ensure basic configuration:**

```bash
# /etc/redis/redis.conf on master (e.g., 192.168.1.10)
bind 192.168.1.10 127.0.0.1
port 6379
requirepass your_redis_password
masterauth your_redis_password

# Enable persistence
appendonly yes
appendfsync everysec
```

## Configure Redis Replicas

On each replica server, configure replication:

```bash
# /etc/redis/redis.conf on replica (e.g., 192.168.1.11, 192.168.1.12)
bind 192.168.1.11 127.0.0.1
port 6379
requirepass your_redis_password
masterauth your_redis_password

# Configure replication to master
replicaof 192.168.1.10 6379

# Make replica read-only (recommended)
replica-read-only yes

# Enable persistence on replicas too
appendonly yes
appendfsync everysec
```

## Configure Sentinel

Create a Sentinel configuration file on each Sentinel node:

```bash
# Create Sentinel configuration directory
sudo mkdir -p /etc/redis

# Create the Sentinel configuration file
sudo nano /etc/redis/sentinel.conf
```

**Add the following Sentinel configuration:**

```bash
# /etc/redis/sentinel.conf
# Sentinel instance port
port 26379

# Let systemd manage process lifecycle
daemonize no
supervised systemd

# PID file
pidfile /run/redis/redis-sentinel.pid

# Log file
logfile /var/log/redis/redis-sentinel.log

# Working directory
dir /var/lib/redis

# Monitor the master named "mymaster" at specified address
# The last number (2) is the quorum - number of Sentinels needed to agree on failure
sentinel monitor mymaster 192.168.1.10 6379 2

# Authentication for connecting to master and replicas
sentinel auth-pass mymaster your_redis_password

# Time in milliseconds to consider master down (30 seconds)
sentinel down-after-milliseconds mymaster 30000

# Number of replicas that can be reconfigured simultaneously during failover
sentinel parallel-syncs mymaster 1

# Failover timeout in milliseconds
sentinel failover-timeout mymaster 180000

# Deny scripts execution (security)
sentinel deny-scripts-reconfig yes

# Announce IP and port (useful in NAT environments)
# sentinel announce-ip 192.168.1.10
# sentinel announce-port 26379
```

Create a systemd unit for Sentinel:

```bash
sudo tee /etc/systemd/system/redis-sentinel.service >/dev/null << 'EOF'
[Unit]
Description=Redis Sentinel
After=network.target

[Service]
Type=notify
User=redis
Group=redis
ExecStart=/usr/bin/redis-server /etc/redis/sentinel.conf --sentinel
ExecStop=/bin/kill -s TERM $MAINPID
Restart=always
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
```

### Start Sentinel Services

Start Sentinel on each node:

```bash
# Allow Sentinel port from trusted nodes only
sudo ufw allow from 192.168.1.0/24 to any port 26379 proto tcp comment 'Redis Sentinel'

# Start and enable Sentinel service
sudo systemctl start redis-sentinel
sudo systemctl enable redis-sentinel

# Check Sentinel status
redis-cli -p 26379 INFO sentinel

# Check master status
redis-cli -p 26379 SENTINEL master mymaster

# List all replicas
redis-cli -p 26379 SENTINEL replicas mymaster

# List all Sentinels
redis-cli -p 26379 SENTINEL sentinels mymaster
```

### Sentinel Client Configuration

Applications should connect through Sentinel for automatic failover support:

```bash
# Python example using redis-py with Sentinel
from redis.sentinel import Sentinel

# Connect to Sentinel nodes
sentinel = Sentinel([
    ('192.168.1.10', 26379),
    ('192.168.1.11', 26379),
    ('192.168.1.12', 26379)
], socket_timeout=0.5)

# Get master connection (automatically follows failovers)
master = sentinel.master_for('mymaster', socket_timeout=0.5, password='your_redis_password')

# Get replica connection for read operations
replica = sentinel.slave_for('mymaster', socket_timeout=0.5, password='your_redis_password')

# Use the connections
master.set('key', 'value')
value = replica.get('key')
```

## Test Failover

Test the failover mechanism:

```bash
# Trigger manual failover
redis-cli -p 26379 SENTINEL failover mymaster

# Watch failover progress
redis-cli -p 26379 SENTINEL master mymaster

# Check who is the new master
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
```

## Redis Cluster for Horizontal Scaling

Redis Cluster provides automatic sharding across multiple nodes, enabling horizontal scaling beyond a single server's capacity.

### Cluster Architecture

Redis Cluster:

- Automatically shards data across multiple nodes using hash slots (16384 total)
- Provides high availability with automatic failover
- Requires a minimum of 6 nodes (3 masters + 3 replicas)

#### Prepare Cluster Nodes

Create configuration for each cluster node (repeat for nodes on ports 7000-7005):

```bash
# Create directories for each cluster node
sudo mkdir -p /etc/redis/cluster
sudo mkdir -p /var/lib/redis/cluster/{7000,7001,7002,7003,7004,7005}

# Ensure Redis user can read/write cluster files
sudo chown -R redis:redis /etc/redis/cluster /var/lib/redis/cluster
```

**Create a configuration file for each node:**

```bash
# /etc/redis/cluster/7000.conf
# Node-specific port
port 7000

# Enable cluster mode
cluster-enabled yes

# Cluster configuration file (auto-generated, do not edit manually)
cluster-config-file /var/lib/redis/cluster/7000/nodes.conf

# Node timeout in milliseconds
cluster-node-timeout 5000

# Enable cluster replica validity factor
cluster-replica-validity-factor 10

# Require full cluster coverage (all slots must be covered)
cluster-require-full-coverage yes

# Authentication
requirepass your_cluster_password
masterauth your_cluster_password

# Persistence
appendonly yes
appendfilename "appendonly-7000.aof"
dir /var/lib/redis/cluster/7000

# Bind address
bind 192.168.1.10 127.0.0.1

# Keep protected mode enabled
protected-mode yes

# Required when nodes communicate across hosts/NAT
cluster-announce-ip 192.168.1.10
cluster-announce-port 7000
cluster-announce-bus-port 17000

# Log file
logfile /var/log/redis/redis-7000.log

# PID file
pidfile /run/redis/redis-7000.pid

# Let systemd manage process lifecycle
daemonize no
supervised systemd
```

Create files `7001.conf` to `7005.conf` from the same template and change:

- `port`
- `cluster-config-file`
- `appendfilename`
- `dir`
- `logfile`
- `pidfile`
- `cluster-announce-port`
- `cluster-announce-bus-port` (port + 10000)

Open firewall ports on each node for both client and cluster bus traffic:

```bash
sudo ufw allow 7000:7005/tcp comment 'Redis cluster data ports'
sudo ufw allow 17000:17005/tcp comment 'Redis cluster bus ports'
```

Create a systemd template service for cluster instances:

```bash
sudo tee /etc/systemd/system/redis-cluster@.service >/dev/null << 'EOF'
[Unit]
Description=Redis Cluster Node %i
After=network.target

[Service]
Type=notify
User=redis
Group=redis
ExecStart=/usr/bin/redis-server /etc/redis/cluster/%i.conf
ExecStop=/bin/kill -s TERM $MAINPID
Restart=always
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
```

### Start Cluster Nodes

Start each Redis instance:

```bash
# Start and enable each node
sudo systemctl enable --now redis-cluster@7000
sudo systemctl enable --now redis-cluster@7001
sudo systemctl enable --now redis-cluster@7002
sudo systemctl enable --now redis-cluster@7003
sudo systemctl enable --now redis-cluster@7004
sudo systemctl enable --now redis-cluster@7005

# Verify all nodes are running
sudo systemctl status redis-cluster@7000
redis-cli -p 7000 -a your_cluster_password PING
```

## Create the Cluster

Use redis-cli to create the cluster:

```bash
# Create cluster with 3 masters and 3 replicas
# The --cluster-replicas 1 option assigns one replica to each master
redis-cli --cluster create \
  192.168.1.10:7000 192.168.1.10:7001 192.168.1.10:7002 \
  192.168.1.11:7003 192.168.1.11:7004 192.168.1.11:7005 \
  --cluster-replicas 1 \
  -a your_cluster_password

# When prompted, type 'yes' to accept the configuration
```

### Verify Cluster Status

Check that the cluster is functioning correctly:

```bash
# Connect to any node and check cluster info
redis-cli -c -p 7000 -a your_cluster_password CLUSTER INFO

# Check cluster nodes and their roles
redis-cli -c -p 7000 -a your_cluster_password CLUSTER NODES

# Check cluster slots distribution
redis-cli -c -p 7000 -a your_cluster_password CLUSTER SLOTS

# Use cluster check for comprehensive health check
redis-cli --cluster check 192.168.1.10:7000 -a your_cluster_password
```

## Working with Redis Cluster

When connecting to a cluster, use the `-c` flag for automatic redirection:

```bash
# Connect in cluster mode (handles redirects automatically)
redis-cli -c -p 7000 -a your_cluster_password

# Set a key (will be redirected to correct node automatically)
SET mykey "myvalue"

# Get a key
GET mykey

# Check which slot a key belongs to
CLUSTER KEYSLOT mykey
```

## Cluster Maintenance Operations

Common cluster maintenance tasks:

```bash
# Add a new node to the cluster
redis-cli --cluster add-node 192.168.1.12:7006 192.168.1.10:7000 \
  -a your_cluster_password

# Add a new replica to a specific master
redis-cli --cluster add-node 192.168.1.12:7007 192.168.1.10:7000 \
  --cluster-slave --cluster-master-id <master-node-id> \
  -a your_cluster_password

# Reshard slots between nodes
redis-cli --cluster reshard 192.168.1.10:7000 \
  -a your_cluster_password

# Rebalance slots across all nodes
redis-cli --cluster rebalance 192.168.1.10:7000 \
  -a your_cluster_password

# Remove a node from cluster (migrate slots first if it's a master)
redis-cli --cluster del-node 192.168.1.10:7000 <node-id> \
  -a your_cluster_password

# Fix cluster issues automatically
redis-cli --cluster fix 192.168.1.10:7000 \
  -a your_cluster_password
```

## Performance Tuning

Optimize Redis for maximum performance in production.

### Connection and Client Settings

```bash
# Maximum number of connected clients
maxclients 10000

# TCP backlog (connections waiting to be accepted)
tcp-backlog 511

# TCP keepalive interval
tcp-keepalive 300

# Close connection after client is idle for N seconds (0 = disabled)
timeout 0
```

### I/O Threading (Redis 6.0+)

Enable I/O threading for better performance on multi-core systems:

```bash
# Enable I/O threads for reads (recommended for high traffic)
io-threads 4

# Enable I/O threading for both reads and writes
io-threads-do-reads yes
```

### Slow Log Configuration

Monitor slow commands to identify performance issues:

```bash
# Log commands taking longer than 10000 microseconds (10ms)
slowlog-log-slower-than 10000

# Keep last 128 slow commands in memory
slowlog-max-len 128
```

Check slow log:

```bash
# View slow log entries
redis-cli SLOWLOG GET 10

# Get slow log length
redis-cli SLOWLOG LEN

# Reset slow log
redis-cli SLOWLOG RESET
```

### Latency Monitoring

Enable latency monitoring to diagnose issues:

```bash
# Enable latency monitoring (threshold in milliseconds)
redis-cli CONFIG SET latency-monitor-threshold 100

# Check latency history
redis-cli LATENCY HISTORY command

# Get latency diagnosis
redis-cli LATENCY DOCTOR

# View latest latency spikes
redis-cli LATENCY LATEST
```

## Monitoring and Logging

Essential Monitoring Commands

```bash
# Real-time statistics
redis-cli INFO

# Memory statistics
redis-cli INFO memory

# Replication status
redis-cli INFO replication

# Client connections
redis-cli CLIENT LIST

# Current operations per second
redis-cli INFO stats | grep instantaneous_ops_per_sec

# Monitor all commands in real-time (use carefully in production)
redis-cli MONITOR
```

#### Log Management

Configure proper log rotation:

```bash
# Create logrotate configuration for Redis
sudo vim /etc/logrotate.d/redis
```

Add the following content:

```txt
/var/log/redis/*.log {
    weekly
    rotate 12
    compress
    delaycompress
    notifempty
    missingok
    create 640 redis redis
    postrotate
        /usr/bin/redis-cli ping > /dev/null 2>&1 || true
    endscript
}
```

## Backup and Recovery

**Automated Backups**
Create a backup script:

```bash
#!/bin/bash
# /usr/local/bin/redis-backup.sh
# Redis backup script with rotation

BACKUP_DIR="/var/backups/redis"
REDIS_DIR="/var/lib/redis"
RETENTION_DAYS=7
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory if it doesn't exist
mkdir -p $BACKUP_DIR

# Trigger a background save
redis-cli -a your_password BGSAVE

# Wait for background save to complete
LASTSAVE_BEFORE=$(redis-cli -a your_password LASTSAVE)
while [ "$(redis-cli -a your_password LASTSAVE)" = "$LASTSAVE_BEFORE" ]; do
    sleep 1
done

# Copy RDB file
cp $REDIS_DIR/dump.rdb $BACKUP_DIR/dump_$DATE.rdb

# Copy AOF file if it exists
if [ -f "$REDIS_DIR/appendonly.aof" ]; then
    cp $REDIS_DIR/appendonly.aof $BACKUP_DIR/appendonly_$DATE.aof
fi

# Compress backups
gzip $BACKUP_DIR/dump_$DATE.rdb
gzip $BACKUP_DIR/appendonly_$DATE.aof 2>/dev/null || true

# Remove old backups
find $BACKUP_DIR -name "*.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $DATE"
```

#### Schedule the backup:

```bash
# Make script executable
sudo chmod +x /usr/local/bin/redis-backup.sh

# Add to crontab (daily at 2 AM)
echo "0 2 * * * /usr/local/bin/redis-backup.sh >> /var/log/redis-backup.log 2>&1" | sudo crontab -
```

#### Recovery Procedure

Restore from backup:

```bash
# Stop Redis
sudo systemctl stop redis-server

# Replace RDB file
sudo gunzip -c /var/backups/redis/dump_YYYYMMDD_HHMMSS.rdb.gz > /var/lib/redis/dump.rdb
sudo chown redis:redis /var/lib/redis/dump.rdb

# Start Redis
sudo systemctl start redis-server

# Verify data restoration
redis-cli -a your_password DBSIZE
```

## Production Checklist

**Before going to production, verify the following:**

```bash
# Security checklist
# [ ] Strong password configured (requirepass)
# [ ] Protected mode enabled
# [ ] Dangerous commands renamed/disabled
# [ ] Firewall rules configured
# [ ] TLS enabled (if external access needed)

# Persistence checklist
# [ ] Persistence method chosen (RDB, AOF, or both)
# [ ] Backup system in place
# [ ] Recovery procedure tested

# Memory checklist
# [ ] maxmemory configured
# [ ] Eviction policy set
# [ ] Overcommit memory enabled
# [ ] Transparent Huge Pages disabled

# High availability checklist
# [ ] Sentinel or Cluster configured (if needed)
# [ ] Failover tested
# [ ] Monitoring in place

# Performance checklist
# [ ] Slow log configured
# [ ] Latency monitoring enabled
# [ ] Connection limits set appropriately
```

### Conclusion

**You now have a production-ready Redis installation on Ubuntu with:**

- Secure installation with password authentication
- Proper persistence configuration for data durability
- Memory management to prevent resource exhaustion
- High availability options with Sentinel and Cluster
- Performance tuning for optimal operation
- Monitoring and backup strategies

Remember to regularly update Redis, monitor its performance, and test your backup and recovery procedures. For critical production environments, consider using Redis Enterprise or managed Redis services for additional features and support.

### Additional Resources

- [Official Redis Documentation](https://redis.io/docs/latest/)
- [Redis Security Guidelines](https://redis.io/docs/latest/develop/)
- [Redis Cluster Tutorial](https://redis.io/docs/latest/develop/)
- [Redis Sentinel Documentation](https://redis.io/docs/latest/develop/)
- [Redis Performance Optimization](https://redis.io/docs/latest/develop/)

<!-- sudo grep -n 'appendname' /etc/redis/redis.conf -->
