## The Startup's Guide to GPU Access on Google Cloud - Part 2: Your Step-by-Step Guide to Provisioning GPUs through GCE and MIGs

<img width="1024" height="1024" alt="mig_gce_flex" src="https://gist.github.com/user-attachments/assets/a8e2d07d-109f-46e1-9b3e-27008a6157a3" />

In Part 1, we introduced DWS Flex as a provisioning mode which is now Generally Available (GA) on Google Cloud. We discussed how this innovative capability tackles the common GPU Availability Paradox by allowing AI workloads to efficiently tap into GPU capacity for both short-term experiments and long-running production jobs, all while providing enhanced availability guarantees compared to standard Preemptible VMs. If you haven't read [Part 1](https://gist.github.com/ankdeshm/6dc5d6b2353fca0af1e31821281f6f97), we strongly encourage you to review it. 

In this second part, we are moving from theory to practice. We will walk through the prerequisites and the exact Google Cloud CLI commands needed to provision DWS Flex GPUs, either as a single Compute Engine (GCE) instance or using a Managed Instance Group (MIG) for scalable management.

-----

## Prerequisites & Setup

Before you can start provisioning, ensure your environment is correctly configured.

### IAM & Permissions

To create the necessary Compute Engine resources (VMs, Instance Templates, MIGs), you must have the **Compute Instance Admin (v1)** (`roles/compute.instanceAdmin.v1`) IAM role on your project.

### Tools

You will need a command-line environment with the Google Cloud CLI installed and initialized. Choose one of the following:

  * **Cloud Shell:** An integrated shell environment accessible directly from the Google Cloud Console.
  * **Cloud Workstation:** Offers a fully customizable and more flexible development environment.
  * **Local Shell:** Ensure you have the `gcloud CLI` installed and initialized.

### Preemptible GPU Quota Requirements

DWS Flex consumes your preemptible quotas for the requested machine types. It is critical to check and request sufficient quota in your target region before attempting to provision resources.

| GPU Accelerator Type | Machine Type(s) | Required Preemptible Quota Name |
| :--- | :--- | :--- |
| **NVIDIA B200 GPUs** | `a4-highgpu-8g` | `PREEMPTIBLE_NVIDIA_B200_GPUS` |
| **NVIDIA H200 GPUs** | `a3-ultragpu-8g` | `PREEMPTIBLE_NVIDIA_H200_GPUS` |
| **NVIDIA H100 MEGA GPUs** | `a3-megagpu-8g` | `PREEMPTIBLE_NVIDIA_H100_MEGA_GPUS` |
| **NVIDIA H100 GPUs** | `a3-highgpu-8g`, `a3-highgpu-4g`, etc. | `PREEMPTIBLE_NVIDIA_H100_GPUS` |
| **NVIDIA A100 80GB GPUs** | `a2-ultragpu-8g`, `a2-ultragpu-4g`, etc. | `PREEMPTIBLE_NVIDIA_A100_80GB_GPUS` |
| **NVIDIA A100 GPUs** | `a2-highgpu-8g`, `a2-highgpu-4g`, etc. | `PREEMPTIBLE_NVIDIA_A100_GPUS` |
| **NVIDIA L4 GPUs** | `g2-standard-4` to `g2-standard-96` | `PREEMPTIBLE_NVIDIA_L4_GPUS` |

> **Note:** Your GCP project is likely to have sufficient default preemptible quota in most regions. It is a mandatory prerequisite to check your quota in the target region before running the provisioning commands. If your required quota is higher than the current limit, you must create a quota increase request from the console.

-----

## Networking for Scalability: Why RDMA Matters

For a demanding, multi-node AI workload, you need high-speed, low-latency communication between VMs. This is achieved using Remote Direct Memory Access (RDMA) over Converged Ethernet (RoCE). RDMA allows GPUs on different VMs to exchange data directly, bypassing the CPU and OS, which is critical for distributed training frameworks that rely on collective communication like NCCL.

### Scalability Checkpoint

  * **Single Node:** If you are running a single-node workload for short-term experimentation, you can use the default network and skip the custom VPC and RDMA configuration steps.
  * **Future Scale-Up:** Even for single-node work, if you anticipate scaling up to multiple nodes in the future, it is highly recommended to configure the custom VPC network and enable RDMA now. This prevents complex networking rebuilds later.

You will see the network configuration referred to as (Optional - See Section 1: Network Setup Script) in the provisioning commands below.

-----

## 1\. Network Setup Script (Optional but Recommended)

For multi-node workloads, create a shell script (`create-vpc-rdma.sh`) to provision the required VPC networks, subnets, and firewall rules for gVNIC and MRDMA (RoCEv2) interfaces.

1.  **Create a file** named `create-vpc-rdma.sh`.
2.  **Copy and paste** the script below into the file.
3.  **Update the variables** at the top (e.g., `GVNIC_NAME_PREFIX`, `REGION`, `ZONE`, `IP_RANGE`, etc.).
4.  **Run the script**: `chmod +x create-vpc-rdma.sh && ./create-vpc-rdma.sh`

<!-- end list -->

```bash
# Define variables - UPDATE THESE!
GVNIC_NAME_PREFIX="a3ultra-gvnic" # e.g., 'a4high-gvnic' for A4 VMs
RDMA_NAME_PREFIX="a3ultra-rdma"    # e.g., 'a4high-rdma' for A4 VMs
REGION="us-central1"               # Target region - check availability
ZONE="us-central1-a"               # Target zone - check availability
IP_RANGE="0.0.0.0/0"               # SSH source IP range (e.g., 'YOUR_IP/32' or '0.0.0.0/0' for all)

# Create regular VPC networks and subnets for the gVNICs
echo "Creating gVNIC networks and subnets..."
for N in $(seq 0 1); do
  gcloud compute networks create $GVNIC_NAME_PREFIX-net-$N \
    --subnet-mode=custom \
    --mtu=8896

  gcloud compute networks subnets create $GVNIC_NAME_PREFIX-sub-$N \
    --network=$GVNIC_NAME_PREFIX-net-$N \
    --region=$REGION \
    --range=10.$N.0.0/16

  gcloud compute firewall-rules create $GVNIC_NAME_PREFIX-internal-$N \
    --network=$GVNIC_NAME_PREFIX-net-$N \
    --action=ALLOW \
    --rules=tcp:0-65535,udp:0-65535,icmp \
    --source-ranges=10.0.0.0/8
done

# Create SSH firewall rules (assumes external IP on vNIC 0)
echo "Creating SSH firewall rules..."
gcloud compute firewall-rules create $GVNIC_NAME_PREFIX-ssh \
  --network=$GVNIC_NAME_PREFIX-net-0 \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=$IP_RANGE

gcloud compute firewall-rules create $GVNIC_NAME_PREFIX-allow-ping-net-0 \
  --network=$GVNIC_NAME_PREFIX-net-0 \
  --action=ALLOW \
  --rules=icmp \
  --source-ranges=$IP_RANGE

# List and ensure network profiles exist in the machine type's zone
echo "Checking for network profiles in $ZONE..."
gcloud compute network-profiles list --filter "location.name=$ZONE"

# Create network for CX-7 (MRDMA - RoCEv2)
echo "Creating MRDMA (RoCEv2) network..."
gcloud compute networks create $RDMA_NAME_PREFIX-mrdma \
  --network-profile=$ZONE-vpc-roce \
  --subnet-mode custom \
  --mtu=8896

# Create subnets for MRDMA
echo "Creating MRDMA subnets..."
for N in $(seq 0 7); do
  gcloud compute networks subnets create $RDMA_NAME_PREFIX-mrdma-sub-$N \
    --network=$RDMA_NAME_PREFIX-mrdma \
    --region=$REGION \
    --range=10.$((N+2)).0.0/16 # offset to avoid overlap with gVNICs
done
```

-----

## 2\. Provisioning Modes with DWS Flex

**Before you create instances**

Define the key parameters for your instance template and instance group.

  * Ensure you use an accelerator-optimized image (version `570` or later for B200 compatibility).
  * For duration, you can specify anything between 10 minutes and 7 days (`168h`) for FLEX mode.
  * Choose the right region and zone where the capacity is available.

### **Option A: Single GCE Instance**

Use the `gcloud compute instances create` command. The key flags for DWS Flex are: `--provisioning-model=FLEX_START`, `--request-valid-for-duration`, and `--max-run-duration`.

The network configuration section (in brackets `[]`) is optional for single-node workloads using the default network.

```bash
gcloud compute instances create VM_NAME \
    --machine-type=MACHINE_TYPE \ # e.g. a4-highgpu-8g
    --image=IMAGE \ # e.g. ubuntu-accelerator-2204-amd64-with-nvidia-570-v20250624
    --image-project=IMAGE_PROJECT \ # e.g. ubuntu-os-accelerator-images
    --zone=ZONE \ # e.g. us-central1-b
    --boot-disk-type=hyperdisk-balanced \
    --boot-disk-size=DISK_SIZE \ # the size of the boot disk in GB
    --reservation-affinity=none \ # FLEX_START does not require a reservation
    --provisioning-model=FLEX_START \ # Without the FLEX_START flag, the VMs will be charged as per the on-demand rates.
    --request-valid-for-duration=REQUEST_VALID_FOR_DURATION \ # the duration that the request to create the VM is valid for - `30m` for 30 minutes
    --max-run-duration=MAX_RUN_DURATION \ # the duration you want the requested VMs to run e.g. 1d2h3m4s for one day, two hours, three minutes, and four seconds
    --instance-termination-action=DELETE \
    --maintenance-policy=TERMINATE \
    --scopes=cloud-platform \
    [ # (Optional - See Section 1: Network Setup Script)
    --network-interface=nic-type=GVNIC,network=GVNIC_NAME_PREFIX-net-0,subnet=GVNIC_NAME_PREFIX-sub-0 \
    --network-interface=nic-type=GVNIC,network=GVNIC_NAME_PREFIX-net-1,subnet=GVNIC_NAME_PREFIX-sub-1,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-0,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-1,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-2,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-3,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-4,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-5,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-6,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-7,no-address]
```

> **Crucial Flags Explained:**
>
>   * `--provisioning-model=FLEX_START`: This is the key flag that designates the VM for the DWS Flex provisioning system.
>   * `--max-run-duration`: Defines the maximum billable time you commit to.

-----

### **Option B: Managed Instance Group (MIG)**

MIGs are recommended for multi-VM management and easier scaling.

#### **Step 1: Create an Instance Template**

This template defines the configuration for all VMs in your group.

```bash
gcloud compute instance-templates create INSTANCE_TEMPLATE_NAME \
    --machine-type=MACHINE_TYPE \ # e.g. a4-highgpu-8g
    --image=IMAGE \ # e.g. ubuntu-accelerator-2204-amd64-with-nvidia-570-v20250624
    --image-project=IMAGE_PROJECT \ # e.g. ubuntu-os-accelerator-images
    --boot-disk-type=hyperdisk-balanced \
    --boot-disk-size=DISK_SIZE \ # the size of the boot disk in GB
    --reservation-affinity=none \ # FLEX_START does not require a reservation
    --instance-termination-action=DELETE \
    --maintenance-policy=TERMINATE \
    --max-run-duration=RUN_DURATION \ # the duration you want the requested VMs to run e.g. 1d2h3m4s for one day, two hours, three minutes, and four seconds or 20h for 20 hours
    --provisioning-model=FLEX_START \ # Without the FLEX_START flag, the VMs will be charged as per the on-demand rates
    --scopes=cloud-platform \
    [ # (Optional - See Section 1: Network Setup Script)
    --network-interface=nic-type=GVNIC,network=GVNIC_NAME_PREFIX-net-0,subnet=GVNIC_NAME_PREFIX-sub-0 \
    --network-interface=nic-type=GVNIC,network=GVNIC_NAME_PREFIX-net-1,subnet=GVNIC_NAME_PREFIX-sub-1,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-0,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-1,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-2,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-3,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-4,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-5,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-6,no-address \
    --network-interface=nic-type=MRDMA,network=RDMA_NAME_PREFIX-mrdma,subnet=RDMA_NAME_PREFIX-mrdma-sub-7,no-address]
```

#### **Step 2: Create a Zonal MIG**

Crucially, start with a size of `0` for DWS Flex MIGs. This ensures no VMs are created until you explicitly request capacity in the next step.

```bash
gcloud compute instance-groups managed create MIG_NAME \
    --template=INSTANCE_TEMPLATE_URL \
    --size=0 \
    --default-action-on-vm-failure=do-nothing \
    --zone=ZONE
```

#### **Step 3: Create a Resize Request in the Zonal MIG**

This single command triggers the DWS Flex provisioning system, queuing a request for `COUNT` number of instances for the specified duration defined in the instance template.

```bash
gcloud compute instance-groups managed resize-requests create MIG_NAME \
    --resize-request=RESIZE_REQUEST_NAME \
    --resize-by=COUNT \
    --zone=ZONE
```

#### **Step 4: Monitor the Status of the Instances in Your Managed Instance Group**

After submitting the resize request, the VMs may take some time to provision, especially for high-demand resources like H100, H200 or B200 GPUs. Use this command to track their status:

```bash
gcloud compute instance-groups managed list-instances ${INSTANCE_GROUP_NAME} --zone=${ZONE}
```

> **Important Note on Capacity:**
> On running this command, you may temporarily see a `LAST_ERROR` such as `ZONE_RESOURCE_POOL_EXHAUSTED_WITH_DETAILS: Waiting for resources. Currently there are not enough resources available to fulfill the request.` This is expected and indicates that the DWS Flex Start system is actively queuing your request and waiting for the full capacity to become available. The VM's `STATUS` will change to `RUNNING` once the provisioning is successful.

#### **Step 5: Access the Provisioned VM and Verify Configuration**

Once the VM's status changes to `RUNNING`, the instance is fully available. You can grab its generated name from the `list-instances` output and connect to it using SSH.

```bash
# Replace VM_NAME with the actual instance name from the list-instances output
gcloud compute ssh VM_NAME --zone=ZONE
```

#### **Step 6: Verify GPU Drivers and Readiness**

After SSH'ing into the VM instance, the final step is to verify that the NVIDIA GPU drivers are correctly installed and the accelerators are detected.

```bash
nvidia-smi
```

This command should output information about the detected GPUs, their status, driver version, and CUDA version, confirming your GPUs are ready to run your AI workloads.

-----

## What's Next?

You have successfully learned how to provision DWS Flex GPUs using GCE and MIGs.

In the next parts of this series, we will elevate this by exploring more sophisticated orchestration mechanisms, focusing on how to integrate DWS Flex with Google Kubernetes Engine (GKE) to handle complex, distributed AI workflows at scale.