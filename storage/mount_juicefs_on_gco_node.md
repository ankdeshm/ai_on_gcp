# Guide to Mounting JuiceFS on Google Cloud GPU Nodes

## Why JuiceFS and Redis?
JuiceFS is a high-performance, distributed POSIX file system designed to address the latency and consistency challenges of object storage. It splits file storage into two components:

**Data Blocks**: Stored reliably and cheaply in Google Cloud Storage (GCS).

**Metadata (Filenames, directories, permissions)**: Stored in a fast, low-latency database like Redis.

**Why Memorystore for Redis?** For AI/ML workloads, file operations (looking up files, checking attributes) must be near-instantaneous. Standard GCS cannot provide sub-millisecond metadata latency. Memorystore for Redis (a fully managed service) provides the necessary speed, stability, and guaranteed low-latency connection required by JuiceFS to perform metadata lookups, ensuring high I/O throughput for your GPU applications.


## Objective

The goal is to provision **JuiceFS** on a **Google Compute Engine (GCE) GPU node** by separating file **Metadata** (in Memorystore for Redis) and file **Data** (in Google Cloud Storage/GCS).

-----

## Pre-requisites & Key Components

This guide assumes your GPU node (`a3megad-a3meganodeset-3`) is running in the **`us-east4-b`** zone and is attached to a VPC network: **`a3mega-sys-net-v2`**. Please modify accordingly.

| Component | Detail | Description |
| :--- | :--- | :--- |
| **GPU Node VPC** | `a3mega-sys-net-v2` | Same VPC as your GPU node |
| **GCS Bucket** | `gs://ankita-jfs-test-gs` | Your bucket |
| **Redis Instance** | `ankita-jfs-metadata-redis` | Unique name for metadata store |

### Initial GCP Steps (Run from Cloud Shell/Workstation/IDE with gcloud plugin)

1.  **Set Project Variables:** (Replace accordingly)
    ```bash
    export PROJECT_ID=northam-ce-mlai-tpu
    export REGION=us-east4
    export ZONE=us-east4-b
    export NODE_VPC_NAME=a3mega-sys-net-v2 #same as your GPU's VPC
    export REDIS_INSTANCE_NAME=ankita-jfs-metadata-redis
    export GCS_BUCKET=ankita-jfs-test-gs
    ```
2.  **Enable APIs:**
    ```bash
    gcloud services enable redis.googleapis.com compute.googleapis.com servicenetworking.googleapis.com
    ```
3.  **Create GCS Bucket:**
    ```bash
    gcloud storage buckets create gs://${GCS_BUCKET} --location=${REGION} --uniform-bucket-level-access
    ```

-----

## Stage 1: Provisioning Memorystore

We must ensure the Redis instance is created on the same VPC network as the GPU node to allow private IP communication.

### 1.1 Allocate IP Range for Service Peering

This reserves an IP range for Google's internal service network to peer with your custom VPC.

```bash
gcloud compute addresses create memorystore-redis-jfs-range-a3mega-ankita \
    --global \
    --prefix-length=24 \
    --network=${NODE_VPC_NAME} \
    --purpose=vpc_peering
```

### 1.2 Create Redis Instance on the same VPC as GPU Node

```bash
gcloud redis instances create ${REDIS_INSTANCE_NAME} \
    --region=${REGION} --zone=${ZONE} \
    --size=1 \
    --network=${NODE_VPC_NAME} \
    --connect-mode=DIRECT_PEERING \
    --tier=BASIC
```

**Wait for Operation to Complete:** The instance must reach the **`READY`** state (approx. 5-10 minutes).

### 1.3 Validate Redis Network Alignment

Ensure that the Redis instance is on the same VPC network as your GPU node.

```bash
gcloud redis instances describe ${REDIS_INSTANCE_NAME} --region=${REGION} --format="value(authorizedNetwork)"
```

**Expected Result:** The output must be the full URI containing your custom VPC name:
`projects/.../global/networks/a3mega-sys-net-v2` (Success).


### 1.4. Get the Connection Details (The Redis URL)
Wait for the instance to finish provisioning. Then, describe the instance to get its private IP address (the Host) and Port.

Make a note of the host and the port described in the output of this command

```bash
gcloud redis instances describe ${REDIS_INSTANCE_NAME} --region=${REGION}
```

-----

## Stage 2: Establish Connectivity and Final Validation

We will require a firewall rule to allow traffic on port 6379 (check the right port from step 1.4).

### 2.1 Apply Egress Firewall Rule

You may get an **`Connection timed out`** error without explicitly opening the right port outbound on your VM's network to the Redis IP.

```bash
NEW_REDIS_IP=$(gcloud redis instances describe ${REDIS_INSTANCE_NAME} --region=${REGION} --format="value(host)")

gcloud compute firewall-rules create allow-outbound-redis-jfs-v2 \
    --network=${NODE_VPC_NAME} \
    --action=ALLOW \
    --direction=EGRESS \
    --rules=tcp:6379 \ # confirm the port number received from the outout of step 1.4
    --destination-ranges=${NEW_REDIS_IP}
```

### 2.2 Final Connectivity Validation (Success Checkpoint 2)

SSH into your GPU node and run the test.

**On GPU Node:**

```bash
redis-cli -h ${NEW_REDIS_IP} -p 6379 PING
```

**Expected Result:** **`PONG`** (This confirms full connectivity to the metadata engine).

-----

## Stage 3: Install and Mount JuiceFS

### 3.1 Install the Full Client Binary

**On GPU Node (as any user):**

```bash
# Installs and places the full binary in /usr/local/bin/
curl -sSL https://d.juicefs.com/install | sudo sh -
```

### 3.2 Format the Filesystem
This is a one-time initialization step that establishes the core of your distributed file system. The command writes the file system structure (the Volume Name) into the Redis metadata engine and permanently links it to the GCS bucket where the data will reside.

You must run this command as the root user on one of the compute nodes.
```bash
# 1. Switch to the root user for administrative permissions
sudo su

# 2. Define the variables for clarity and ease of use
export GCS_BUCKET_NAME="YOUR_GCS_BUCKET_NAME"
export REDIS_IP="10.71.97.195" 
export VOLUME_NAME="my-gpu-test-volume"

# 3. Execute the format command
/usr/local/bin/juicefs format \
  --storage gs \
  --bucket https://${GCS_BUCKET_NAME} \
  redis://${REDIS_IP}/1 \
  ${VOLUME_NAME}
```

**Validation**: The command is successful when it prints the final configuration block:
Volume is formatted as { "Name": "my-gpu-test-volume", ... }

-----

### 3.3 Mount the Filesystem Permanently

This step starts the JuiceFS client (the daemon) on the compute node. We use the nohup command to ensure the mount persists even after your SSH session or any Slurm job that executes this command is terminated.

```bash
# 1. Create the local directory where the file system will be mounted
mkdir -p /jfs

# 2. Execute the persistent mount command
# 'nohup' detaches the process from the current shell, making it persistent.
# 'redis://...' tells the client where the metadata is located.
# '> /var/log/juicefs.log 2>&1 &' redirects all output to a log file and runs in the background.

sudo nohup /usr/local/bin/juicefs mount redis://10.71.97.195/1 /jfs > /var/log/juicefs.log 2>&1 &
```

### 3.4 Final Mount Verification

This command confirms that the operating system recognizes the new file system and displays its massive, elastic capacity.

```bash
df -h | grep /jfs
```

**Expected Result:** You should see the mounted volume listing its virtual capacity (1.0P):
JuiceFS:my-gpu-test-volume 1.0P 0 1.0P 0% /jfs

The file system is now ready for use. Exit the root shell (`exit`) and begin running your applications.

-----

## FAQ & Troubleshooting Guide

| Issue | Cause from Trace | Resolution in this Guide |
| :--- | :--- | :--- |
| **`Could not connect to Redis: Connection timed out`** | The Redis instance was initially created in the **`default`** VPC, while the GPU node was on the custom **`a3mega-sys-net-v2`** network. | **Solution:** Delete and recreate the Redis instance, explicitly setting `--network=a3mega-sys-net-v2` (Stage 1). |
| **`Connection timed out` (after network fix)** | VPC Peering was correct, but the VPC's general **Egress** firewall rules blocked port 6379. | **Solution:** Create an explicit **EGRESS firewall rule** allowing TCP traffic on port 6379 to the Redis IP address (Stage 2.1). |
| **`sudo: 3 incorrect password attempts`** | The Linux user's password on the Slurm node is unknown or disabled for `sudo`. | **Solution:** Switch to the root user (`sudo su`) to perform installation and mounting (Stage 3). |
| **`No such file or directory` (for `juicefs`)** | The initial small download was a placeholder, not the full binary. | **Solution:** Use the robust installer: `curl -sSL https://d.juicefs.com/install | sh -` to get the correct, full binary (Stage 3.1). |
| **`AOF is not enabled` / `ERR unknown command 'config'`** | Memorystore Basic Tier does not support Append-Only File (AOF) persistence or manual `CONFIG SET` commands. | **Ignore:** These are diagnostic warnings for managed Redis services and do not prevent the JuiceFS format operation from succeeding. |  |

## Useful Resources

[1] https://github.com/juicedata/juicefs

[2] https://juicefs.com/docs/community/reference/how_to_set_up_object_storage

