# Setting up Virtual machines on Macbook Pro via OrbStack to run a TiDB cluster
# TiDB Cluster Setup with OrbStack

A comprehensive guide for setting up TiDB cluster on Apple Silicon using OrbStack.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Quick Start (Single-Node)](#quick-start-single-node)
- [Multi-Node Cluster Setup](#multi-node-cluster-setup)
- [Accessing TiDB](#accessing-tidb)
- [OrbStack Management](#orbstack-management)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- Apple Silicon Mac (M1/M2/M3/M4)
- macOS 12.0 or later
- Homebrew installed

## Installation

### Step 1: Install OrbStack

```bash
brew install orbstack
```

### Make sure you're using ARM Homebrew
```bash
which brew
```
### Must show: /opt/homebrew/bin/brew

### Install OrbStack (ARM version)
```bash
brew install --cask orbstack
```
### It should now download: arm64.dmg (not amd64.dmg)

### Step 2: Create Ubuntu VM

#### Single VM (Development)

```bash
# Create and start Ubuntu 22.04 VM
orb create ubuntu:22.04 tidb-node1

# SSH into the VM
orb -m tidb-node1
```

#### Multiple VMs (Production-like Cluster)

```bash
orb create ubuntu:22.04 tidb-node1
orb create ubuntu:22.04 tidb-node2
orb create ubuntu:22.04 tidb-node3
```

### Step 3: Install TiUP

Inside each VM, run:

```bash
# Install TiUP (TiDB deployment tool)
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh

# Add to PATH
source ~/.bashrc

# Verify installation
tiup --version

# Install cluster component
tiup cluster
```

## Quick Start (Single-Node)

Perfect for development and testing.

### Launch TiDB Playground

```bash
# Default version
tiup playground

# Specific version
tiup playground v8.5.0

# With custom configuration
tiup playground v8.5.0 --db 1 --pd 1 --kv 3
```

This starts:
- 1 TiDB instance (SQL layer) - Port 4000
- 1 PD instance (Placement Driver) - Port 2379
- 1 TiKV instance (Storage layer) - Port 20160
- Prometheus (Monitoring) - Port 9090
- Grafana (Visualization) - Port 3000

### Connect to TiDB

From within the VM:

```bash
# Using MySQL client
mysql -h 127.0.0.1 -P 4000 -u root

# Test connection
mysql -h 127.0.0.1 -P 4000 -u root -e "SELECT 'Hello TiDB';"
```

From your Mac:

```bash
# Install MySQL client
brew install mysql-client

# Connect (replace with your VM IP)
mysql -h <vm-ip> -P 4000 -u root
```

## Multi-Node Cluster Setup

For production-like environments with high availability.

### Step 1: Get VM IP Addresses

```bash
# On your Mac terminal
orb list
```

Note the IP addresses of your VMs (e.g., 192.168.194.2, 192.168.194.3, 192.168.194.4).

### Step 2: Create Topology Configuration

Create a file named `topology.yaml`:

```yaml
global:
  user: "tidb"
  ssh_port: 22
  deploy_dir: "/tidb-deploy"
  data_dir: "/tidb-data"

pd_servers:
  - host: 192.168.194.2
  - host: 192.168.194.3
  - host: 192.168.194.4

tidb_servers:
  - host: 192.168.194.2
  - host: 192.168.194.3

tikv_servers:
  - host: 192.168.194.2
  - host: 192.168.194.3
  - host: 192.168.194.4

monitoring_servers:
  - host: 192.168.194.2

grafana_servers:
  - host: 192.168.194.2
```

### Step 3: Prepare Environment

On each VM:

```bash
# Create tidb user
sudo useradd -m -d /home/tidb tidb

# Set up SSH keys for passwordless access
ssh-keygen -t rsa -b 4096
ssh-copy-id tidb@<vm-ip>

# Disable firewall or configure ports
sudo ufw disable  # Ubuntu
```

### Step 4: Deploy Cluster

From one of your VMs (or your Mac if you have SSH access):

```bash
# Check topology configuration
tiup cluster check topology.yaml --user root

# Automatically fix issues (optional)
tiup cluster check topology.yaml --apply --user root

# Deploy the cluster
tiup cluster deploy tidb-prod v8.5.0 topology.yaml --user root

# Start the cluster
tiup cluster start tidb-prod

# Check cluster status
tiup cluster display tidb-prod
```

### Step 5: Verify Cluster

```bash
# Check cluster health
tiup cluster display tidb-prod

# View detailed status
tiup cluster status tidb-prod

# Access TiDB Dashboard
# Open browser: http://<pd-ip>:2379/dashboard
```

## Accessing TiDB

### MySQL Client Connection

```bash
# From Mac
mysql -h <tidb-server-ip> -P 4000 -u root

# From VM
mysql -h 127.0.0.1 -P 4000 -u root
```

### TiDB Dashboard

The dashboard provides cluster monitoring and management:

```
http://<pd-server-ip>:2379/dashboard
```

Default login: `root` (no password initially)

### Grafana Monitoring

```
http://<grafana-server-ip>:3000
```

Default credentials:
- Username: `admin`
- Password: `admin`

### Prometheus

```
http://<monitoring-server-ip>:9090
```

## OrbStack Management

### Common Commands

```bash
# List all VMs
orb list

# Start VM
orb start tidb-node1

# Stop VM
orb stop tidb-node1

# Restart VM
orb restart tidb-node1

# Delete VM
orb delete tidb-node1

# SSH into VM
orb shell tidb-node1

# Execute command in VM
orb run tidb-node1 -- <command>

# Get VM IP
orb list | grep tidb-node1
```

### Resource Management

```bash
# View VM resource usage
orb info tidb-node1

# Adjust VM resources (if needed)
# Edit ~/.orbstack/machines/<vm-name>/config.json
```

## TiDB Cluster Management

### Cluster Operations

```bash
# Start cluster
tiup cluster start tidb-prod

# Stop cluster
tiup cluster stop tidb-prod

# Restart cluster
tiup cluster restart tidb-prod

# Scale out (add nodes)
tiup cluster scale-out tidb-prod scale-out.yaml

# Scale in (remove nodes)
tiup cluster scale-in tidb-prod --node <node-id>

# Upgrade cluster
tiup cluster upgrade tidb-prod v8.5.1

# Destroy cluster
tiup cluster destroy tidb-prod
```

### Backup and Restore

```bash
# Install BR (Backup & Restore tool)
tiup install br

# Backup database
tiup br backup full --pd "<pd-address>:2379" --storage "local:///tmp/backup"

# Restore database
tiup br restore full --pd "<pd-address>:2379" --storage "local:///tmp/backup"
```

## Troubleshooting

### Check Logs

```bash
# View TiDB logs
tiup cluster display tidb-prod
# Then SSH to the node and check /tidb-deploy/tidb-<port>/log/

# View all cluster logs
tiup cluster logs tidb-prod

# View specific component logs
tiup cluster logs tidb-prod --role tidb
```

### Common Issues

#### Connection Refused

```bash
# Check if services are running
tiup cluster display tidb-prod

# Check firewall
sudo ufw status

# Check if ports are listening
netstat -tlnp | grep -E '4000|2379|20160'
```

#### Out of Memory

```bash
# Check system resources
free -h
top

# Adjust TiKV memory settings in topology.yaml
# Then reload configuration
tiup cluster reload tidb-prod
```

#### Disk Space Issues

```bash
# Check disk usage
df -h

# Clean up old logs
tiup cluster clean tidb-prod --log

# Clean up old data (careful!)
tiup cluster clean tidb-prod --data
```

### Health Checks

```bash
# Check cluster status
tiup cluster check tidb-prod --cluster

# Check system requirements
tiup cluster check topology.yaml --user root

# View cluster metrics
curl http://<pd-ip>:2379/pd/api/v1/health
```

## Best Practices

### For Development

- Use `tiup playground` for quick testing
- Single VM is sufficient
- Default configurations work well

### For Production-like Testing

- Use at least 3 nodes for high availability
- Separate TiDB, PD, and TiKV nodes
- Enable monitoring (Prometheus + Grafana)
- Regular backups using BR tool
- Test failover scenarios

### Performance Tuning

```yaml
# Example optimized topology.yaml
tikv_servers:
  - host: 192.168.194.2
    config:
      storage.block-cache.capacity: "8GB"
      raftstore.apply-pool-size: 4
      raftstore.store-pool-size: 4
```

## Resources

- [TiDB Official Documentation](https://docs.pingcap.com/)
- [TiUP Documentation](https://docs.pingcap.com/tidb/stable/tiup-overview)
- [OrbStack Documentation](https://docs.orbstack.dev/)
- [TiDB GitHub](https://github.com/pingcap/tidb)
