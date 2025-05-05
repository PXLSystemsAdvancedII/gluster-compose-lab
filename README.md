# GlusterFS Cluster Lab Guide

## Lab Overview

This lab demonstrates setting up a GlusterFS distributed storage cluster using Docker containers. You'll learn to:
- Deploy GlusterFS nodes using Docker Compose
- Configure trusted server pools 
- Create and manage distributed volumes
- Troubleshoot permission issues
- Use GlusterFS as object storage via S3

## Prerequisites

- Docker and Docker Compose installed
- Basic understanding of storage concepts
- Terminal/CLI access
- At least 4GB RAM and 10GB disk space

## Theory: GlusterFS Fundamentals

### What is GlusterFS?

GlusterFS is a scalable network filesystem. It combines storage servers accessed through Ethernet/TCP/IP to provide a unified storage namespace.

**Key Concepts:**
- **Trusted Server Pool (TSP)**: Group of storage servers
- **Bricks**: Basic storage unit (directory on server)
- **Volumes**: Collection of bricks distributed across servers
- **Replication**: Data mirroring for redundancy
- **Self-healing**: Automatic data repair

## Lab Architecture

```
                    Docker Host
    ┌─────────────────────────────────────────────────┐
    │                                                 │
    │  ┌─────────────┐     ┌─────────────┐            │
    │  │ glusterfs1  │     │ glusterfs2  │            │
    │  │ 172.20.0.101│◄───►│ 172.20.0.102│            │
    │  └─────────────┘     └─────────────┘            │
    │         ▲                    ▲                  │
    │         │                    │                  │
    │         └────────┬───────────┘                  │
    │                  │                              │
    │          ┌───────┴───────┐                      │
    │          │   glusters3   │                      │
    │          │ 172.20.0.103  │                      │
    │          │ (S3 Gateway)  │                      │
    │          └───────────────┘                      │
    │                  │                              │
    └──────────────────┼──────────────────────────────┘
                       │
                 Port 8080
```

## Lab Exercise 1: Cluster Setup

### Step 1: Understand the Docker Compose Configuration

Examine the `compose.yml` file:

```yaml
services:
  glusterfs1:
    container_name: fss-glusterfs1
    image: gluster/gluster-centos
    privileged: true
    networks:
      network_glusterfs:
        ipv4_address: 172.20.0.101
    volumes:
      - glusterfs_etc_volume1:/etc/glusterfs
      - glusterfs_lib_volume1:/var/lib/glusterd
      - glusterfs_log_volume1:/var/log/glusterfs
      - gulsterfs_data_volume1:/data/glusterfs
```

**Analysis:**
- Uses official GlusterFS container image
- Requires privileged mode for filesystem operations
- Persistent volumes for configuration, logs, and data
- Custom network with fixed IP addresses

### Step 2: Start the Cluster

Execute these commands step by step:

```bash
# 1. Start the GlusterFS nodes
docker-compose up -d glusterfs1 glusterfs2 glusters3

# 2. Wait for services to initialize
echo "Waiting for GlusterFS to start..."
sleep 10

# 3. Verify nodes are running
docker ps | grep gluster
```

### Step 3: Create Trusted Server Pool

Establish peer relationship between nodes:

```bash
# Connect to glusterfs1 and probe glusterfs2
docker-compose exec glusterfs1 gluster peer probe glusterfs2

# Verify peer status
docker-compose exec glusterfs1 gluster peer status
docker-compose exec glusterfs1 gluster pool list
```

Expected output:
```
Number of Peers: 1
Hostname: glusterfs2
Uuid: 3c510d0e-bc1d-4bf6-b41b-5bd4632b0d8a
State: Peer in Cluster (Connected)
```

## Lab Exercise 2: Volume Management

### Step 4: Create a Replicated Volume

Create a volume with replica 2 for high availability:

```bash
# Create volume with replica 2
docker-compose exec glusterfs1 gluster volume create test-volume \
  replica 2 transport tcp \
  glusterfs1:/data/glusterfs/test \
  glusterfs2:/data/glusterfs/test

# Start the volume
docker-compose exec glusterfs1 gluster volume start test-volume

# Check volume status
docker-compose exec glusterfs1 gluster volume status
docker-compose exec glusterfs1 gluster volume info test-volume
```

**Volume Creation Parameters:**
- `replica 2`: Data is replicated across 2 nodes
- `transport tcp`: Uses TCP for communication
- Two bricks, one on each node

### Step 5: Configure Volume Permissions

Set proper permissions for user 1000:

```bash
# Set ACL permissions for user 1000
docker-compose exec glusterfs1 setfacl -m u:1000:rwx /data/glusterfs/test
docker-compose exec glusterfs2 setfacl -m u:1000:rwx /data/glusterfs/test

# Verify permissions
docker-compose exec glusterfs1 getfacl /data/glusterfs/test
```

## Lab Exercise 3: Monitoring and Troubleshooting

### Step 6: Monitor Cluster Health

```bash
# Check volume status
docker-compose exec glusterfs1 gluster volume status

# Check peer connectivity
docker-compose exec glusterfs1 gluster peer status

# View brick logs (in real-time)
docker-compose exec glusterfs1 tail -f /var/log/glusterfs/bricks/data-glusterfs-test.log
```

### Step 7: Troubleshoot Permission Issues

If you encounter permission errors:

```bash
# Check brick logs for permission errors
docker-compose exec glusterfs1 cat /var/log/glusterfs/bricks/data-glusterfs-test.log

# Look for entries like:
# client: ..., req(uid:1000,gid:1000,perm:3), ctx(uid:0,gid:0,...) [Permission denied]
```

**Fix permission errors:**
```bash
# Grant proper ACL permissions
docker-compose exec glusterfs1 setfacl -m u:1000:rwx /data/glusterfs/test
docker-compose exec glusterfs2 setfacl -m u:1000:rwx /data/glusterfs/test
```

## How to Use This GlusterFS Cluster

### Method 1: Direct Volume Mount

Mount the GlusterFS volume on a client system:

```bash
# On a client machine (not in the lab containers)
# Install GlusterFS client
sudo apt-get install glusterfs-client  # Ubuntu/Debian
sudo yum install glusterfs-client      # CentOS/RHEL

# Create mount point
sudo mkdir -p /mnt/glusterfs

# Mount the volume
sudo mount -t glusterfs glusterfs1:/test-volume /mnt/glusterfs

# Or using IP address
sudo mount -t glusterfs 172.20.0.101:/test-volume /mnt/glusterfs

# Add to /etc/fstab for persistent mounting
echo "glusterfs1:/test-volume /mnt/glusterfs glusterfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
```

### Method 2: NFS Access

GlusterFS can be accessed via NFS protocol:

```bash
# Enable NFS on the volume
docker-compose exec glusterfs1 gluster volume set test-volume nfs.disable off

# Mount via NFS (from client)
sudo mount -t nfs glusterfs1:/test-volume /mnt/glusterfs
```

### Method 3: Application Integration

Applications can use GlusterFS libraries directly:

```python
# Python example using gfapi
from glusterfs import gfapi

# Connect to volume
vol = gfapi.Volume("glusterfs1", "test-volume")
vol.mount()

# File operations
with vol.open("testfile.txt", "w") as f:
    f.write("Hello GlusterFS!")

# Read file
with vol.open("testfile.txt", "r") as f:
    content = f.read()
    print(content)
```

### Method 4: Container Volume

Use as Docker volume for containers:

```yaml
# In docker-compose.yml
services:
  app:
    image: myapp
    volumes:
      - type: volume
        source: gluster-vol
        target: /data
        volume:
          driver: local
          driver_opts:
            type: glusterfs
            o: addr=glusterfs1,volfile=/test-volume
            device: glusterfs1:/test-volume
```

### Method 5: S3 Gateway (via glusters3)

Access as object storage:

```bash
# S3 credentials (from the lab setup)
S3_ACCOUNT: test
S3_USER: admin
S3_PASSWORD: tests3passwd
S3_ENDPOINT: http://localhost:8080

# Using boto3 (Python)
import boto3

s3 = boto3.client('s3',
    endpoint_url='http://localhost:8080',
    aws_access_key_id='admin',
    aws_secret_access_key='tests3passwd'
)

# Create bucket
s3.create_bucket(Bucket='mybucket')

# Upload file
s3.upload_file('local_file.txt', 'mybucket', 'remote_file.txt')
```

### Use Cases

1. **Shared Storage**: Multiple servers accessing the same files
2. **Container Persistence**: Persistent volumes for containerized applications  
3. **Backup Storage**: Centralized backup destination
4. **Media Storage**: Video, image, and document repositories
5. **Database Storage**: MySQL, PostgreSQL data directories
6. **Application Data**: Shared application state across clusters

## Common Commands Reference

### Peer Management:
```bash
gluster peer probe <server>    # Add server to TSP
gluster peer detach <server>   # Remove server from TSP
gluster peer status            # Check peer status
gluster pool list              # List all nodes
```

### Volume Management:
```bash
gluster volume create <name> [options] <bricks>
gluster volume start <name>
gluster volume stop <name>
gluster volume info [name]
gluster volume status [name]
gluster volume list
```

## Cleanup

To remove the lab environment:

```bash
# Stop and remove containers
docker-compose down

# Remove volumes (optional)
docker-compose down -v
```

## Additional Resources

- [GlusterFS Documentation](https://gluster.readthedocs.io/)
- [GlusterFS CLI Reference](https://gluster.readthedocs.io/en/latest/CLI-Reference/cli-main/)
- [GlusterFS Admin Guide](https://gluster.readthedocs.io/en/latest/Administrator%20Guide/overview/)
- [Docker Container Source](https://github.com/gluster/gluster-containers)

## Troubleshooting Guide

### Issue 1: Containers Won't Start

**Symptoms:** Containers exit immediately
**Solution:**
```bash
# Check logs
docker logs fss-glusterfs1

# Ensure privileged mode is set
# Check compose.yml for `privileged: true`
```

### Issue 2: Peer Probe Fails

**Symptoms:** `peer probe` command fails
**Solution:**
```bash
# Check network connectivity
docker network inspect glusterfs_network_glusterfs

# Verify IP addresses
docker inspect fss-glusterfs1 | grep IPAddress
```

### Issue 3: Volume Creation Fails

**Symptoms:** Volume creation returns error
**Solution:**
```bash
# Check brick directories exist
docker-compose exec glusterfs1 ls -la /data/glusterfs/

# Create directory if missing
docker-compose exec glusterfs1 mkdir -p /data/glusterfs/test
```

### Issue 4: Permission Denied

**Symptoms:** Client cannot access volume
**Solution:**
```bash
# Check brick logs
docker-compose exec glusterfs1 tail /var/log/glusterfs/bricks/data-glusterfs-test.log

# Apply correct ACLs
docker-compose exec glusterfs1 setfacl -m u:1000:rwx /data/glusterfs/test
```
