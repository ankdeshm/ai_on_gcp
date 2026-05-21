## The Startup's Guide to GPU Access on Google Cloud - Part 4: Provisioning GPUs with DWS Flex-Start on Slurm


<img width="977" height="977" alt="slurm_flex" src="https://gist.github.com/user-attachments/assets/00b79555-17ad-4864-a203-037a760c8fc8" />

**Dynamic Workload Scheduler (DWS) ** continues to be the strategic advantage for startups needing high-demand GPU resources with enhanced availability.

In [Part 1](https://gist.github.com/ankdeshm/6dc5d6b2353fca0af1e31821281f6f97) of this series, we introduced the core principles and financial implications of DWS Flex, highlighting how it tackles the GPU Availability Paradox. In [Part 2](https://gist.github.com/ankdeshm/4d752cd1a4e5f69f24a382ef1de9b79c), we provided a hands-on guide for utilizing DWS Flex with standalone Compute Engine (GCE) instances and Managed Instance Groups (MIGs). Building on that foundation, [Part 3](https://gist.github.com/ankdeshm/2f7ddefc174d02145ca5bec2bd38dd73) introduced the sophisticated, cloud-native approach of integrating DWS Flex with Google Kubernetes Engine (GKE) for containerized workloads.

In this fourth and final part, we shift our focus to High-Performance Computing (HPC) environments, detailing how to provision DWS Flex GPUs using **Slurm** on Google Cloud.

#### Why Slurm and the Cluster Toolkit?

For engineers, data scientists, and academics already familiar with traditional HPC setups, Slurm is the dominant and often preferred workload manager. It is essential for orchestrating large-scale, highly parallel training jobs.

While setting up a custom, best-practices-compliant Slurm cluster on Google Cloud using traditional infrastructure-as-code (IaC) can be time-consuming, the Google Cloud Cluster Toolkit offers a fast-track solution for POCs.

The Cluster Toolkit provides abstracted and extensible blueprints for deploying complex HPC environments, pre-configured with Google Cloud best practices for networking, storage, and image selection. For smaller startup teams with minimal infrastructure support, the Cluster Toolkit is a great way to set up a managed Slurm environment.

We will use this tool to quickly and efficiently deploy a DWS Flex-enabled Slurm environment. If you are a Slurm expert, you are welcome to adapt this guide to your own custom deployment blueprint.

-----

## Implementation Guide: DWS Flex on Slurm

### Prerequisites & Setup

#### 1\. API and Tool Requirements

Before proceeding, ensure you have the following APIs enabled and tools installed:

  * **APIs to Enable:**

      * **Compute Engine API**
      * **Cloud Identity and Access Management (IAM) API**
      * **Cloud Storage API**
      * **Service Usage API**
      * **Cloud Resource Manager API**

  * **CLI Tools:**

      * **Google Cloud SDK (`gcloud` CLI):** Ensure you have the latest version installed and configured.
      * **`terraform`:** The Cluster Toolkit relies on Terraform for provisioning infrastructure.
      * **Cluster Toolkit:**

#### 2\. Installing and Setting up the Cluster Toolkit

To begin, you must install and set up the Google Cloud HPC Cluster Toolkit. This is mandatory for leveraging the pre-configured blueprints in this guide.

> **Recommended Action:** Please follow the official [Cluster Toolkit Installation and Setup Guide](https://cloud.google.com/cluster-toolkit/docs/setup/configure-environment) to clone the repository and complete the initial setup steps before proceeding.

#### 3\. GPU Quota and Permissions Check

It is crucial to confirm you have sufficient GPU quota in your chosen region/zone *before* starting the setup.

  * **Quota:** Please refer back to [Part 2](https://gist.github.com/ankdeshm/4d752cd1a4e5f69f24a382ef1de9b79c) of this blog series for details on checking and requesting the necessary DWS Flex-compatible GPU quota.
  * **Permissions:** Ensure the service account used by Terraform (often the Compute Engine default service account) has the necessary roles to create and manage Compute resources, IAM roles, and Storage Buckets in your project.

-----


## Step-by-Step Implementation

Now that your prerequisites are met, we will focus on configuring and deploying your DWS Flex-enabled Slurm cluster using the Cluster Toolkit. This setup is optimized for high-performance computing (HPC) with A3/H100 GPUs.

### 1\. Configure Core Cloud Resources

First, we will set up the essential supporting resources for the Slurm cluster's data and state management, primarily a Cloud Storage bucket. This bucket is used by **Terraform** to store the state file.

#### 1\. Create a Dedicated Cloud Storage Bucket

The bucket must be created with uniform bucket-level access (UBLA) and versioning enabled for state integrity.

```bash
# Create a GCS bucket for your project. Choose a globally unique name.
gcloud storage buckets create gs://BUCKET_NAME \
    --project=PROJECT_ID \
    --default-storage-class=STANDARD --location=REGION \
    --uniform-bucket-level-access

# Example: gs://slurm-toolkit-state-ankita-123

# NOTE: Replace BUCKET_NAME, PROJECT_ID, and REGION with your actual values.
```

#### 2\. Enable Bucket Versioning and Grant Permissions

It is highly recommended to enable versioning for the state bucket.

```bash
# Enable versioning on the bucket
gcloud storage buckets update gs://BUCKET_NAME --versioning

# Grant the Storage Admin role to your user account (replace USERNAME)
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member=user:USERNAME \
    --role=roles/storage.admin
```

-----

### 2\. Update Slurm Blueprint Files

Since we are using the Cluster Toolkit, we need to modify two core files in the repository clone to enable DWS Flex and define our deployment parameters: the **Deployment YAML** (which sets variables) and the **Blueprint YAML** (which defines the infrastructure structure).

> **IMPORTANT NOTE TO AVOID RESOURCE CONFLICTS:** To ensure a clean deployment and avoid conflicts with old resources, it is crucial to update the unique variables in these files. This is essential for preventing errors, especially if you have prior failed deployments.


> **IMPORTANT NOTE ON NETWORK OPTIMIZATION:**
> This configuration is optimized for single-node workloads or multiple independent single-node jobs. For optimizing communication in multi-node, high-bandwidth workloads (e.g., distributed training) on A3 Mega VMs, [GPUDirect-TCPXO](https://cloud.google.com/cluster-toolkit/docs/machine-learning/a3-mega-enable-gpudirect-tcpxo) is required to be enabled on your Slurm cluster.

#### 1\. Update the Deployment YAML (`a3mega-slurm-deployment.yaml`)

Edit the file located at `examples/machine-learning/a3-megagpu-8g/a3mega-slurm-deployment.yaml`.

This file sets the Terraform backend, project variables, and key Slurm cluster parameters. The values below are generic placeholders.

| Variable Field | Action Required | Note |
| :--- | :--- | :--- |
| `terraform_backend_defaults.configuration.bucket` | Set to **`BUCKET_NAME`**. | Connects to the GCS bucket created in Step 1. |
| `vars.deployment_name` | Set to **`DEPLOYMENT_NAME`**. | Unique name for this Cluster Toolkit deployment (e.g., `slurm-dws-flex-poc`). |
| `vars.project_id` | Set to **`PROJECT_ID`**. | Your Google Cloud Project ID. |
| `vars.region` | Set to **`REGION`**. | The primary region for your resources (e.g., `us-east4`). |
| `vars.zone` | Set to **`ZONE`**. | The zone for your resources (e.g., `us-east4-b`). |
| `vars.slurm_cluster_name` | Set to **`SLURM_CLUSTER_NAME`**. | Slurm cluster name (must be $\le 9$ characters). |
| `vars.a3mega_cluster_size` | Set to **`MAX_SLURM_NODES`**. | Your desired maximum cluster size in nodes (e.g., `16`). |
| `vars.a3mega_dws_flex_enabled` | Set to **`true`**. | **Crucially enables DWS Flex provisioning.** |
| `vars.max_run_duration` | Set to **`MAX_RUN_DURATION`**. | Hard limit for VM termination in seconds (e.g., `7200`s = 2 hours). |

**Example of Updated Variables Section (within `a3mega-slurm-deployment.yaml`):**

```yaml
terraform_backend_defaults:
  type: gcs
  configuration:
    bucket: BUCKET_NAME # Example: slurm-toolkit-state-ankita-123

vars:
  deployment_name: DEPLOYMENT_NAME # Example: slurm-dws-flex-poc
  project_id: PROJECT_ID
  region: REGION
  zone: ZONE
  # ... network and other generic variables
  disk_size_gb: DISK_SIZE_GB # Example: 200
  final_image_family: FINAL_IMAGE_FAMILY # Example: slurm-dws-flex-image
  slurm_cluster_name: SLURM_CLUSTER_NAME # Example: slurmdws
  a3mega_cluster_size: MAX_SLURM_NODES # Example: 16
  a3mega_dws_flex_enabled: true # DWS FLEX ENABLED
  max_run_duration: MAX_RUN_DURATION # Example: 7200
  # ... other variables
```

#### 2\. Update the Blueprint YAML (`a3mega-slurm-blueprint.yaml`)

Edit the file located at `examples/machine-learning/a3-megagpu-8g/a3mega-slurm-blueprint.yaml`.

This file requires specific modifications in the compute node and partition definitions to fully enable and configure DWS Flex. The values reference the variables defined in the Deployment YAML (using `$(vars.VARIABLE_NAME)` syntax).

| Blueprint Field | Action Required | Note |
| :--- | :--- | :--- |
| `a3mega_nodeset.settings.node_count_static` | Set to **`0`**. | **Essential DWS Flex Setting:** This prevents nodes from being provisioned at deployment time and relies entirely on Slurm to request them dynamically. |
| `a3mega_nodeset.settings.machine_type` | Set to **`MACHINE_TYPE`**. | Machine type for compute nodes (e.g., `a3-megagpu-8g`). |
| `a3mega_nodeset.settings.dws_flex.enabled` | Set to **`true`**. | Activates DWS Flex provisioning for the compute node pool. |
| `a3mega_nodeset.settings.dws_flex.max_run_duration` | Set to **`$(vars.max_run_duration)`**. | Inherits the hard VM termination limit set in the deployment file. |
| `a3mega_partition.settings.suspend_time` | Set to **`$(vars.max_run_duration)`**. | **Recommended for Experimentation:** Setting this equal to the max run duration prevents Slurm from suspending nodes due to idle state before the DWS Flex termination limit is reached. |

**Example of Updated Blueprint Section (within `a3mega-slurm-blueprint.yaml`):**

```yaml
- id: a3mega_nodeset
    # ... other settings
    settings:
      # Key Change 1: Set static nodes to 0 to enable DWS Flex
      node_count_static: 0 
      # Set max dynamic nodes based on the deployment variable
      node_count_dynamic_max: $(vars.a3mega_cluster_size) 
      disk_type: pd-ssd
      machine_type: MACHINE_TYPE # Example: a3-megagpu-8g
      # ... other settings
      dws_flex:
        # Key Change 2: Enable DWS Flex on the nodeset
        enabled: true
        # Key Change 3: Set hard limit for the node VM lifetime
        max_run_duration: $(vars.max_run_duration) 
      # ... other settings
      
- id: a3mega_partition
    # ... other settings
    settings:
      # ... partition name and other settings
      # Key Change 4: Set Slurm suspend time equal to max run duration
      suspend_time: $(vars.max_run_duration) 
      partition_conf:
        OverSubscribe: EXCLUSIVE
        ResumeTimeout: 900
        SuspendTimeout: 600
```

-----

### 3\. Provision the Slurm Cluster

With the deployment files correctly configured, use the `gcluster deploy` command from the Cluster Toolkit to provision the infrastructure using **Terraform**.

```bash
# Execute the deployment command from the Cluster Toolkit directory
./gcluster deploy -d examples/machine-learning/a3-megagpu-8g/a3mega-slurm-deployment.yaml \
examples/machine-learning/a3-megagpu-8g/a3mega-slurm-blueprint.yaml \
--auto-approve
```

> **NOTE:** Building the custom compute node image is the most time-consuming part of this process, typically taking **30-40 minutes**. This step creates the controller, login, and debug nodes, as well as the custom image that your dynamic compute nodes will use.

-----

## Cluster Interaction and Job Management

Once the Cluster Toolkit deployment command completes, your Slurm control plane (controller, login, and debug nodes) is running, and the DWS Flex-enabled compute node pool is configured but empty (set to `node_count_static: 0`). The next step is to interact with the scheduler to trigger dynamic provisioning.

### 1\. Access the Slurm Cluster

SSH into the **login node** to interact with the Slurm scheduler and submit jobs. The login node is the gateway to your compute resources.

```bash
# Example command to SSH into the Slurm login node
gcloud compute ssh SLURM_CLUSTER_NAME-login --zone=ZONE
```

### 2\. Verify Node Configuration (`sinfo`)

Use the `sinfo` command to check the current status of the compute nodes.

> **`sinfo`** is a Slurm utility used to view information about Slurm partitions and nodes.

```bash
sinfo
```

| Expected Output | Description |
| :--- | :--- |
| `a3mega` | The partition name defined in the blueprint. |
| `idle~` | The state of the dynamic nodes. The **tilde (`~`)** indicates that the nodes are **powered down** (DWS Flex is active) and waiting for a job request to provision them. |
| `0/MAX_SLURM_NODES` | Shows that 0 nodes are currently allocated out of the maximum configured. |

### 3\. Submitting Jobs and Triggering Dynamic Scaling

When DWS Flex is enabled, **any job submission command** (`sbatch` or `salloc`) that requests resources Slurm does not currently have will automatically trigger the creation of new VMs. The job will simply wait until the requested resources are provisioned and ready.

#### Option A: Non-Interactive Job Submission (`sbatch`)

This is the standard and recommended way to submit batch workloads.

> **`sbatch`** submits a script for later execution, allowing the job to run in the background while you exit the shell. **If no nodes are available, `sbatch` will trigger dynamic scaling to provision them.**

**Process:**

1.  Create a job submission script (`test_job.sh`).
2.  Run `sbatch test_job.sh`. Slurm returns a **Job ID**.
3.  Slurm checks for available nodes. Finding none, it sets the job state to **Pending (PD)** or **Configuring (CF)**.
4.  The Slurm controller sends a request to DWS Flex to provision the required VMs.
5.  The job waits until the VMs are booted and registered with Slurm, then transitions to the **Running (R)** state.

#### Option B: Interactive Allocation (`salloc`)

This is useful for debugging, testing, or interactively running code on a provisioned compute node.

> **`salloc`** requests a resource allocation for a job in real-time, typically providing an interactive shell on a compute node once allocated.

```bash
# Request 2 nodes (-N 2) from the a3mega partition (-p a3mega) 
# for a 24-hour time limit (-t 1-00:00:00) with exclusive access (--exclusive)
salloc -p a3mega -N 2 -t 1-00:00:00 --exclusive
```

| Process Monitoring | Description |
| :--- | :--- |
| **Login Node Output** | The command will block and show `salloc: Waiting for resource configuration`. This is the wait time for DWS Flex to provision and boot the VMs. Once ready, the shell will switch to one of the newly provisioned **compute nodes**. |
| **Controller Node Monitoring** | Meanwhile, SSH into the **controller node** and monitor the logs for real-time updates on VM provisioning: `sudo tail -f /var/log/slurm/slurmctld.log`. |

### 4\. Run a Job (`sbatch`)

Here is an example of the job script and submission.

#### Example Job Submission Script (`test_job.sh`)

Create a script file (e.g., `test_job.sh`) with your job parameters and commands:

```bash
#!/bin/bash
#SBATCH --job-name=quick-test-job
#SBATCH --nodes=1
#SBATCH --time=00:03:00

echo "Starting job on allocated node..."


# Example workload command
sleep 120 

echo "Job finished successfully."
```

#### Submit the Job

```bash
sbatch test_job.sh
# Slurm will return a Job ID, e.g., Submitted batch job 123
```

### 5\. Check Job and GPU Status

Use the following commands from the login node to manage your job.

| Command | Purpose | Notes |
| :--- | :--- | :--- |
| `squeue -u USERNAME` | **Check running/pending jobs.** | States: **R** (Running), **PD** (Pending), **CF** (Configuring - provisioning DWS Flex VM). Replace `USERNAME` with your GCP username. |
| `sacct -j JOB_ID` | **View accounting details.** | Provides historical and status information for a specific job, including start and end times. |
| `nvidia-smi` | **Verify GPUs on compute node.** | Run this directly from the allocated compute node shell (if you used `salloc`) or include it in your `sbatch` script. |

### 6\. Manage and Destroy the Cluster

#### Cancel a Running Job (`scancel`)

To stop a running job and allow the Slurm scheduler to trigger dynamic descaling/termination of the associated nodes, use `scancel`.

```bash
scancel JOB_ID
```

#### Redeploy the Cluster

To update the cluster's configuration **without rebuilding the custom compute node image** (which saves significant time), use the `--only` flag. This is useful for minor configuration changes.

```bash
./gcluster deploy -d \
examples/machine-learning/a3-megagpu-8g/a3mega-slurm-deployment.yaml \
examples/machine-learning/a3-megagpu-8g/a3mega-slurm-blueprint.yaml \
--only primary,cluster --auto-approve -w
```

#### Destroy the Cluster

To avoid incurring charges for idle resources, **always destroy your cluster** when you are finished. Replace `DEPLOYMENT_FOLDER` with your specific path (e.g., `examples/machine-learning/a3-megagpu-8g`).

```bash
./gcluster destroy DEPLOYMENT_FOLDER --auto-approve
```


---

## Troubleshooting and Important Considerations

Understanding compute node behavior is key to successful experimentation with Slurm on DWS Flex.

* **Node Down Error: Understanding VM Termination**
    Depending on your blueprint configuration, you might observe compute nodes going down after a period of time. This can be attributed to two main settings:

    * **`max_run_duration` Exceeded:** This is a hard limit set in the blueprint file for DWS Flex-provisioned GPUs. Once this duration is exceeded, the underlying VM will be returned to the resource pool **irrespective of its current job state**. If your longer job is terminated due to `max_run_duration`, Slurm's scheduler will detect the incomplete job and automatically restart it on a newly provisioned VM. The job will maintain the same job ID and name and will be scheduled to run (from the beginning) on a new instance that is given the same node name. This is the intended behavior, ensuring that jobs interrupted by node de-provisioning are resumed to completion.

    * **`suspend_time`:** If a compute node remains idle for more than the configured `SuspendTime` (default 300 seconds), it will be placed into a power-save mode by Slurm's `SuspendProgram`. While this is by design for cost optimization, during experimentation, you might prefer to keep nodes active. To prevent nodes from being suspended due to idle time before the DWS Flex `max_run_duration` limit, you can set `suspend_time` to **`-1`** (never suspend) or **equal to `max_run_duration`** in your blueprint's partition configuration.
---

This series has demonstrated that DWS Flex provides a superior balance between cost and availability for AI and HPC workloads on Google Cloud. By enabling reliable, queue-managed access to scarce Cloud accelerators through tools like Slurm and GKE, DWS Flex fundamentally accelerates your experimentation and production deployment.


