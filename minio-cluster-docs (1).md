# Distributed S3-Compatible Object Storage System

## Executive Summary

Implemented a distributed, S3-compatible object storage cluster using MinIO across a multi-node architecture. The project followed a two-phase approach: validation on virtualized infrastructure (Phase 1) and production deployment on physical Raspberry Pi hardware (Phase 2).

**Key Technologies:**
- MinIO Server (distributed mode)
- Ubuntu Server 22.04 LTS (ARM64)
- Raspberry Pi 4 (ARM64 architecture)
- VirtualBox (development/testing)
- Linux networking (bridged, host-only, static IP assignment)

---

## Architecture Overview

### System Design

**Topology:** 4-node distributed cluster
- 1 master node (coordinator)
- 3 worker nodes (storage peers)

**Storage Model:** Distributed erasure coding (EC:2 in 4-drive configuration)
- Minimum 2 drives required for quorum
- Can tolerate up to 2 simultaneous node failures
- Data automatically distributed across all nodes

**Network Architecture:**
- Dual-interface configuration (Phase 1)
  - NAT/Bridged: External access and internet connectivity
  - Host-only: Inter-node cluster communication (192.168.56.0/24)
- Single-interface WiFi (Phase 2)
  - All nodes on shared wireless network (10.118.161.0/24)

---

## Phase 1: Virtual Infrastructure Validation

### Objectives
- Validate distributed MinIO configuration
- Test network isolation and cluster communication
- Verify erasure coding and fault tolerance
- Establish deployment procedures before hardware investment

### Infrastructure Specifications

**Hypervisor:** Oracle VirtualBox on Apple M2
**Guest OS:** Ubuntu Server 22.04 LTS (ARM64)

**VM Configuration (per node):**
```
CPU: 2 cores
Memory: 4GB (master), 2GB (workers)
Storage: 25GB system + 100GB data disk
Network Adapters:
  - Adapter 1: Bridged (access from host/external clients)
  - Adapter 2: Host-only (cluster communication, 192.168.56.x)
```

**Network Design:**
```
Host-only Network (Cluster Communication):
192.168.56.101 - minio-master
192.168.56.102 - minio-worker1
192.168.56.103 - minio-worker2
192.168.56.104 - minio-worker3

Bridged Network (Client Access):
Dynamic DHCP on host network (10.118.161.x range)
```

### Key Learnings from Phase 1

**Network Isolation:**
- Host-only network provides stable IPs for cluster coordination
- Bridged adapter enables flexible external access regardless of host network changes
- Dual-interface design allows cluster to function independently of host connectivity

**Storage Configuration:**
- Each VM configured with dedicated 100GB virtual disk (`/dev/sdb`)
- Formatted as ext4 and mounted at `/mnt/storage`
- MinIO volumes: `http://NODE_IP/mnt/storage/data`

**Challenges Resolved:**
1. **Service Configuration:** Initial hardcoded ExecStart values ignored environment files
   - Solution: Migrated to `EnvironmentFile=/etc/default/minio` pattern
2. **User Permissions:** Permission denied errors with minio-user
   - Solution: Ensured consistent user ownership across all storage paths
3. **Cluster Formation:** Nodes starting in standalone mode
   - Solution: Verified identical MINIO_VOLUMES configuration across all nodes

---

## Phase 2: Raspberry Pi Production Deployment

### Hardware Specifications

**Platform:** Raspberry Pi 4 Model B
**Architecture:** ARM64 (aarch64)
**Connectivity:** IEEE 802.11ac WiFi (2.4GHz/5GHz dual-band)

**Storage Options:**
- Option A: SD card storage (`/mnt/data`)
- Option B: External USB storage (`/mnt/usb/data` or `/mnt/sdc/data`)

### Network Configuration

**Approach:** DHCP with manual IP reservation (preferred) or static assignment

**Rationale for DHCP:**
- Maintains internet connectivity without manual gateway configuration
- Enables remote SSH access without network reconfiguration
- Simplifies deployment across different network environments
- Suitable for demonstration/development scenarios

**Static IP Alternative (production):**
```bash
# /etc/dhcpcd.conf
interface wlan0
static ip_address=192.168.1.201/24
static routers=192.168.1.1
static domain_name_servers=8.8.8.8 8.8.4.4
```

### Deployment Differences from Phase 1

| Aspect | Phase 1 (VMs) | Phase 2 (Raspberry Pi) |
|--------|---------------|------------------------|
| Network | Dual interface (bridged + host-only) | Single interface (wlan0) |
| IP Strategy | Static host-only for cluster | DHCP or router-reserved IPs |
| Storage | Virtual disk (100GB VDI) | Physical SD/USB storage |
| Binary | x86_64/ARM64 via VirtualBox | Native ARM64 |
| Cluster Stability | Isolated host-only network | Dependent on WiFi reliability |

---

## MinIO Configuration

### Distributed Mode Setup

**Volume Configuration Pattern:**
```bash
MINIO_VOLUMES="http://NODE1_IP/path http://NODE2_IP/path http://NODE3_IP/path http://NODE4_IP/path"
```

**Single-Drive per Node (Current Implementation):**
```bash
MINIO_VOLUMES="http://10.118.161.53/mnt/data http://10.118.161.54/mnt/data http://10.118.161.55/mnt/data http://10.118.161.56/mnt/data"
```

**Multi-Drive per Node (Optional):**
```bash
MINIO_VOLUMES="http://NODE1/mnt/data{1...4} http://NODE2/mnt/data{1...4} ..."
```
- Provides higher erasure coding parity (EC:8 with 16 drives)
- Increases fault tolerance to 8 drive failures

### Service Configuration

**SystemD Unit (`/etc/systemd/system/minio.service`):**
```ini
[Unit]
Description=MinIO
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
User=minio-user
Group=minio-user
EnvironmentFile=/etc/default/minio
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
Restart=always
LimitNOFILE=65536
TasksMax=infinity
TimeoutStopSec=infinity
SendSIGKILL=no

[Install]
WantedBy=multi-user.target
```

**Environment File (`/etc/default/minio`):**
```bash
MINIO_VOLUMES="http://NODE1_IP/storage ..."
MINIO_OPTS="--console-address :9001"
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin123
MINIO_SERVER_URL="http://MASTER_IP:9000"
```

### Startup Sequence

**Critical:** Master node must start first
```bash
# On master node:
sudo systemctl start minio

# Wait 10-15 seconds for cluster coordination

# On worker nodes (sequential or parallel):
sudo systemctl start minio
```

**Verification:**
```bash
sudo journalctl -u minio -n 50
# Expected: "Status: N Online, 0 Offline"
# Where N = number of drives (4 for single-drive, 16 for quad-drive)
```

---

## Data Flow and Consistency Model

### Write Path
1. Client sends PUT request to any node (S3 API)
2. MinIO coordinator calculates erasure coding shards
3. Data distributed across available drives
4. Quorum write confirmation (>50% nodes)
5. Object metadata replicated cluster-wide

### Read Path
1. Client sends GET request to any node
2. MinIO retrieves object from any available shard set
3. Automatic reconstruction if drives unavailable
4. Direct data stream to client

### Consistency Guarantees
- **Write Consistency:** Read-after-write consistency within quorum
- **Eventually Consistent:** Cross-cluster replication (if configured)
- **Erasure Coding:** Automatic data repair and rebalancing

---

## Failure Scenarios and Recovery

### Single Node Failure
**Behavior:**
- Cluster remains operational with N-1 nodes
- Read/write operations continue
- Automatic failover to available shards

**Recovery:**
- Restart failed node
- MinIO automatically reintegrates
- Data heals from parity information

### Network Partition
**Behavior:**
- Cluster maintains quorum if >50% nodes accessible
- Isolated nodes become read-only or unavailable
- No split-brain (MinIO uses distributed locking)

**Recovery:**
- Network restoration triggers automatic reconciliation
- No manual intervention required

### Data Corruption
**Behavior:**
- BitRot protection via erasure coding checksums
- Automatic detection during read operations
- Silent data corruption prevented

**Recovery:**
- Corrupted shards reconstructed from parity
- Self-healing without administrator intervention

---

## API and Client Access

### Endpoints
- **S3 API:** `http://NODE_IP:9000`
- **Web Console:** `http://NODE_IP:9001`

### MinIO Client (mc) Usage
```bash
# Configure alias
mc alias set mycluster http://MASTER_IP:9000 minioadmin minioadmin123

# Create bucket
mc mb mycluster/my-bucket

# Upload object
mc cp local-file.txt mycluster/my-bucket/

# List objects
mc ls mycluster/my-bucket/

# Cluster health
mc admin info mycluster

# Expected output: "Status: N Online, 0 Offline"
```

### S3 SDK Compatibility
Compatible with AWS SDK for:
- Python (boto3)
- JavaScript (aws-sdk)
- Java (AWS SDK for Java)
- Go (aws-sdk-go)

Configuration:
```python
import boto3

s3 = boto3.client('s3',
    endpoint_url='http://MASTER_IP:9000',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadmin123'
)
```

---

## Security Considerations

### Current Implementation (Development)
- Default credentials (`minioadmin:minioadmin123`)
- Unencrypted HTTP transport
- No network segmentation beyond host-only (Phase 1)

### Production Hardening Recommendations
1. **TLS/SSL Encryption:**
   ```bash
   minio server --certs-dir /path/to/certs ...
   ```

2. **IAM Policies:**
   - Bucket-level access controls
   - User-specific policies
   - Service account separation

3. **Network Security:**
   - Firewall rules (UFW configured for ports 9000, 9001)
   - VPN access for management interfaces
   - Internal-only cluster communication

4. **Credential Management:**
   - Rotate access keys
   - Use strong passwords (min 16 characters)
   - Environment-based secrets injection

---

## Performance Characteristics

### Observed Behavior (Phase 1 - VirtualBox)
- **Throughput:** Limited by hypervisor overhead and virtual networking
- **Latency:** ~5-10ms inter-node (host-only network)
- **Bottleneck:** Virtual disk I/O (VDI format)

### Expected Behavior (Phase 2 - Raspberry Pi)
- **WiFi Throughput:** 
  - 2.4GHz: ~50 Mbps
  - 5GHz: ~200 Mbps
- **Storage I/O:**
  - SD Card: ~20-40 MB/s sequential
  - USB 3.0 Drive: ~100-150 MB/s
- **Network Latency:** 2-5ms (local WiFi)

### Optimization Opportunities
1. **Ethernet over WiFi:** Gigabit Ethernet reduces latency and increases throughput
2. **SSD over SD Card:** NVMe over USB 3.0 improves I/O
3. **Dedicated VLAN:** Isolates cluster traffic from general network

---

## Monitoring and Observability

### Built-in Metrics
```bash
# Prometheus metrics endpoint
curl http://NODE_IP:9000/minio/v2/metrics/cluster

# Admin API
mc admin prometheus metrics mycluster
```

### Key Metrics to Monitor
- **Drive Status:** Online/offline count
- **Network Throughput:** Bytes in/out per node
- **API Latency:** Request response times
- **Object Count:** Total objects and buckets
- **Storage Utilization:** Used/available capacity

### Log Analysis
```bash
# Service logs
sudo journalctl -u minio -f

# Look for:
# - "Status: N Online, 0 Offline"
# - "All MinIO sub-systems initialized successfully"
# - Warning/error messages
```

---

## Deployment Checklist

### Pre-Deployment
- [ ] OS installed and updated (`apt update && apt upgrade`)
- [ ] Static IPs assigned or DHCP reservations configured
- [ ] `/etc/hosts` populated with all node hostnames
- [ ] Storage devices formatted and mounted
- [ ] Firewall rules configured (ports 22, 9000, 9001)

### MinIO Installation (per node)
- [ ] MinIO binary downloaded and installed (`/usr/local/bin/minio`)
- [ ] `minio-user` system user created
- [ ] Storage directories created with correct ownership
- [ ] `/etc/default/minio` configured (identical on all nodes)
- [ ] SystemD service file created
- [ ] Service enabled (`systemctl enable minio`)

### Cluster Initialization
- [ ] Start master node first
- [ ] Verify master node logs show waiting for peers
- [ ] Start worker nodes sequentially
- [ ] Verify cluster status: "Status: N Online, 0 Offline"
- [ ] Test web console access
- [ ] Test MinIO client operations

### Post-Deployment Validation
- [ ] Create test bucket
- [ ] Upload test objects
- [ ] Verify object retrieval from different nodes
- [ ] Simulate node failure (stop service on one node)
- [ ] Verify cluster remains operational
- [ ] Restart failed node and verify reintegration

---

## Technical Debt and Future Work

### Current Limitations
1. **Network Dependency:** WiFi reliability affects cluster stability (Phase 2)
2. **No Encryption:** HTTP-only transport (development acceptable)
3. **Single Storage Device:** Limited to SD card or single USB drive per node
4. **Manual IP Management:** DHCP requires tracking IP changes

### Potential Enhancements
1. **Multi-Drive Configuration:** 4 drives per node (16 total) for EC:8
2. **TLS Implementation:** Encrypted transport with self-signed or Let's Encrypt certificates
3. **External Load Balancer:** HAProxy or Nginx for round-robin client requests
4. **Automated Monitoring:** Prometheus + Grafana dashboards
5. **Backup Strategy:** S3 replication to external MinIO cluster or cloud provider
6. **Terraform/Ansible Deployment:** Infrastructure as Code for reproducible setup

---

## Lessons Learned

### Phase 1 Insights
- **Virtualization validates architecture:** Cheaper and faster than hardware iteration
- **Host-only networking is stable:** Isolated cluster communication independent of host network
- **Configuration consistency is critical:** Identical `/etc/default/minio` across nodes is non-negotiable
- **Service file patterns matter:** Environment file approach superior to hardcoded values

### Phase 2 Insights
- **WiFi is sufficient for demos:** Adequate throughput for small-scale testing
- **DHCP simplifies deployment:** Avoids static IP conflicts and gateway misconfigurations
- **USB storage recommended:** SD card wear concerns for write-heavy workloads
- **Raspberry Pi is viable for MinIO:** ARM64 support is first-class

### General Observations
- **Distributed systems are complex:** Network issues manifest as subtle failures
- **Logs are essential:** `journalctl -u minio` reveals 90% of issues
- **Start simple, scale up:** Single-drive worked; multi-drive adds complexity
- **Documentation matters:** Reproducibility requires detailed notes

---

## References and Resources

### Official Documentation
- MinIO Documentation: https://docs.min.io
- MinIO GitHub: https://github.com/minio/minio
- S3 API Reference: https://docs.aws.amazon.com/AmazonS3/latest/API/

### Deployment Guides
- MinIO Distributed Setup: https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-multi-node-multi-drive.html
- Erasure Coding Explained: https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html

### Related Technologies
- Raspberry Pi OS: https://www.raspberrypi.org/software/
- Ubuntu Server ARM: https://ubuntu.com/download/server/arm
- SystemD Service Management: https://www.freedesktop.org/software/systemd/man/systemd.service.html

---

## Appendix: Command Reference

### Quick Start Commands
```bash
# Check cluster status
mc admin info mycluster

# Create bucket
mc mb mycluster/bucket-name

# Upload file
mc cp file.txt mycluster/bucket-name/

# List buckets
mc ls mycluster

# Monitor logs
sudo journalctl -u minio -f

# Restart service
sudo systemctl restart minio

# Check service status
sudo systemctl status minio
```

### Network Diagnostics
```bash
# Test connectivity
ping NODE_IP

# Check port availability
nc -zv NODE_IP 9000

# View current IP
hostname -I

# Check DNS resolution
cat /etc/hosts
```

### Storage Management
```bash
# Check disk usage
df -h

# List block devices
lsblk

# Check mount points
mount | grep minio

# Verify permissions
ls -la /mnt/storage/data
```

---

## Project Metadata

**Implementation Timeline:** Two-phase approach
- Phase 1 (VirtualBox): Development and validation
- Phase 2 (Raspberry Pi): Production deployment

**Total Nodes:** 4 (1 master, 3 workers)

**Storage Architecture:** Distributed erasure coding

**Compatibility:** S3 API standard (AWS SDK compatible)

**Target Use Cases:**
- Local S3-compatible storage for development
- Learning distributed systems concepts
- Private cloud storage alternative
- Educational demonstration of object storage

**Skills Demonstrated:**
- Distributed systems architecture
- Linux system administration
- Network configuration and troubleshooting
- Storage system design
- Infrastructure documentation
- Virtualization and containerization concepts