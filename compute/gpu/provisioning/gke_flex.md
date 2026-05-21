## The Startup's Guide to GPU Access on Google Cloud - Part 3: Provisioning GPUs with DWS Flex-Start on GKE


<img width="977" height="977" alt="gke-flex" src="https://gist.github.com/user-attachments/assets/93f7ae9f-d1c1-4d3c-a3f0-0d16ea55b6ac" />


Welcome back to our series focused on helping startups efficiently spin up and utilize powerful GPUs using Dynamic Workload Scaling (DWS) Flex. In [Part 1](https://gist.github.com/ankdeshm/6dc5d6b2353fca0af1e31821281f6f97), we introduced the general guidelines and core concepts of DWS Flex. In [Part 2](https://gist.github.com/ankdeshm/4d752cd1a4e5f69f24a382ef1de9b79c), we covered the setup process using standalone Compute Engine (GCE) instances and Managed Instance Groups (MIGs).

Now, in this third part, we introduce one of the most sophisticated and native approaches on Google Cloud: integrating DWS Flex with Google Kubernetes Engine (GKE).

Why GKE is Ideal for DWS Flex
While GCE instances are perfect for simple, one-off experiments, GKE is the superior choice for production or complex, multi-stage workflows. GKE abstracts away underlying infrastructure, offering features like automatic scaling, self-healing, and declarative resource management through Kubernetes manifests. When combined with DWS Flex, GKE enables startups to manage their containerized GPU workloads with robust scalability and high availability, effortlessly scaling from zero nodes to handle bursts of demand and back down again to save cost.


## Implementation Guide: DWS Flex on GKE

We will now walk through the essential prerequisites and the step-by-step process to provision DWS Flex GPUs within a **GKE Standard cluster**.

> **Note on Hardware Selection:** For this guide, we are focusing on the high-performance **A3 Mega machine series**, specifically the `a3-megagpu-8g` type, which features **8 NVIDIA H100 80GB GPUs** per node. If you are targeting a different accelerator (e.g., A100 or L4), please ensure you adjust the `MACHINE_TYPE`, `ACCELERATOR_TYPE`, and `GPU_COUNT` environment variables below to match your chosen hardware and quota.

### Prerequisites & Setup

#### 1\. API and Tool Requirements

You must ensure the following are enabled and installed:

  * **APIs to Enable:**
      * **Kubernetes Engine API** (Essential for GKE cluster management).
      * **Compute Engine API** (Required for underlying VMs and GPUs).
  * **CLI Tools:**
      * **Google Cloud SDK (`gcloud` CLI):** Ensure you have the latest version configured.
      * **`kubectl`:** Ensure `kubectl` is installed to interact with your GKE cluster.
  * **GKE Version:**
      * **Standard Cluster:** Version 1.28.3-gke.1098000 or later.
      * **Autopilot Cluster:** Version 1.30.3-gke.1451000 or later.
      * *(For more details, consult the GKE Release Notes.)*

#### 2\. GPU Quota Check

It is critical to confirm you have sufficient GPU quota in your chosen region/zone before starting the setup.

> **Note:** For a detailed guide on checking and requesting quota, please refer back to [Part 2](https://gist.github.com/ankdeshm/4d752cd1a4e5f69f24a382ef1de9b79c) of this blog series.

-----

### Step 1: Set Environment Variables

Define the variables that will be used for your cluster, node pool, and workload configuration according to your requirements.

```bash
# UPDATE THESE!

# Project and Location variables
export PROJECT_ID="$(gcloud config get-value project)"
export CLUSTER_NAME="a3mega-poc" 
export REGION="us-east4"
export COMPUTE_ZONE="us-east4-a" # Capacity can vary by zone.

# Hardware and Scaling variables (Targeting A3 Mega / H100)
export NODEPOOL_NAME="h100-flex-pool"
export MACHINE_TYPE="a3-megagpu-8g"
export ACCELERATOR_TYPE="nvidia-h100-80gb"
export GPU_COUNT=8 # 8 GPUs per node for a3-megagpu-8g
export MAX_NODES=4 # Max number of nodes the pool can scale up to (32 GPUs total)
export MAX_RUN_DURATION="24h" # Max time for DWS Flex provisioning (e.g., min 10m, max 168h)
```

### Step 2: Create the GKE Standard Cluster

Create the GKE cluster:

```bash
gcloud container clusters create "${CLUSTER_NAME}" \
    --zone "${COMPUTE_ZONE}" \
    --project="${PROJECT_ID}" \
    --release-channel=REGULAR
```

### Step 3: Get Cluster Credentials

Configure `kubectl` to connect to your newly created cluster:

```bash
gcloud container clusters get-credentials "${CLUSTER_NAME}" \
    --zone="${COMPUTE_ZONE}" \
    --project="${PROJECT_ID}"
```

-----

### Step 4: Create the DWS Flex Node Pool

The node pool creation is where we enable the core DWS Flex functionality. By setting `--num-nodes=0` and enabling autoscaling, the node pool can scale from zero to the maximum configured node count based on demand.

The key flag here is `--flex-start`. This flag enables DWS Flex for the entire node pool, allowing nodes to be provisioned with enhanced availability guarantees compared to standard Preemptible VMs.

#### Option A: DWS Flex Start (Single Node/Independent Workloads)

This option is suitable for:

  * Experimentation

  * Workloads that fit entirely on a single node

  * Workloads that do not require an "All-or-Nothing" resource guarantee

<!-- end list -->

```bash
gcloud container node-pools create "${NODEPOOL_NAME}" \
    --cluster="${CLUSTER_NAME}" \
    --zone="${COMPUTE_ZONE}" \
    --accelerator type=${ACCELERATOR_TYPE},count=${GPU_COUNT},gpu-driver-version=latest \
    --machine-type="${MACHINE_TYPE}" \
    --flex-start \
    --max-run-duration=${MAX_RUN_DURATION} \
    --enable-autoscaling \
    --num-nodes=0 \
    --total-max-nodes "${MAX_NODES}" \
    --reservation-affinity=none \
    --no-enable-autorepair \
    --enable-gvnic \
    --project="${PROJECT_ID}"
```

#### Option B (Optional): DWS Flex with Queued Provisioning (All-or-Nothing Approach)

For large-scale, distributed training jobs where the entire workload must start simultaneously, you require an "All-or-Nothing" resource allocation. This means you would rather wait for all required nodes to become available than start with only a partial set.

To achieve this, you must enable the optional `--enable-queued-provisioning` flag on your node pool, in addition to `--flex-start`.

  * **Command:**

<!-- end list -->

```bash
gcloud container node-pools create "${NODEPOOL_NAME}-qp" \
    --cluster="${CLUSTER_NAME}" \
    --location="${COMPUTE_ZONE}" \
    --enable-queued-provisioning \
    --accelerator type=${ACCELERATOR_TYPE},count=${GPU_COUNT},gpu-driver-version=latest \
    --machine-type="${MACHINE_TYPE}" \
    --flex-start \
    --max-run-duration=${MAX_RUN_DURATION} \
    --enable-autoscaling  \
    --num-nodes=0   \
    --total-max-nodes "${MAX_NODES}"  \
    --location-policy=ANY  \
    --reservation-affinity=none  \
    --no-enable-autorepair
```

### Step 5: Verify DWS Flex Start Configuration (Optional)

You can confirm that `flexStart` is enabled on your node pool:

```bash
gcloud container node-pools describe "${NODEPOOL_NAME}" \
    --cluster="${CLUSTER_NAME}" \
    --zone="${COMPUTE_ZONE}" \
    --format="get(config.flexStart)"
```

*Expected output:* `True`


### Step 6: Deploying Workloads

The final step is submitting your GPU workload. Regardless of the DWS Flex option you chose, you must ensure your Kubernetes manifest correctly targets the specialized node pool using **Taints, Labels, and Annotations**.

This sample single Pod deployment will trigger the GKE cluster autoscaler to provision a DWS Flex node from zero, as needed.

This YAML uses specific Kubernetes fields to ensure the Pod is scheduled on the correct node pool:

1.  **`nodeSelector`**: This is used to target nodes with the specific attributes required by the workload.
      * `cloud.google.com/gke-flex-start: "true"`: Crucially, this directs the Pod to the node pool where DWS Flex has been enabled.
      * `cloud.google.com/gke-accelerator: nvidia-h100-80gb`: This ensures the Pod only lands on nodes with the specified GPU type.
2.  **`resources.limits` & `requests`**: The values of `"8"` for `nvidia.com/gpu` indicate that the Pod requires all 8 GPUs available on a single node (matching the `a3-megagpu-8g` machine type used in the node pool).
3.  **`tolerations`**: This allows the Pod to be scheduled onto GPU nodes, which are typically tainted to prevent general workloads from consuming expensive resources.

Create a file named `single-node-deploy.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: h100-pod-a
spec:
  nodeSelector:
    cloud.google.com/gke-flex-start: "true" # Targets DWS Flex enabled nodes (Option A)
    cloud.google.com/gke-accelerator: nvidia-h100-80gb # Ensures it goes to H100 GPU nodes
  containers:
  - name: cuda-container
    image: nvidia/cuda:12.9.1-base-ubuntu22.04 # A CUDA-enabled image
    command: ["sleep", "3600"] # Keeps the container running for 3600 seconds
    resources:
      limits:
        nvidia.com/gpu: "8" # Requests 8 GPU, matching the node configuration
      requests:
        nvidia.com/gpu: "8"
  tolerations: # Allows scheduling on tainted GPU nodes
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

**Apply the Pod:**

```bash
kubectl apply -f single-node-deploy.yaml
```


Once applied, the GKE cluster autoscaler will detect the pending Pod and provision a new DWS Flex-enabled node to satisfy the request.


#### Deploying a Multi-Node Workload (Targeting Option B)

To successfully target the Queued Provisioning node pool, the following fields are essential:

1.  **Label (`kueue.x-k8s.io/queue-name`)**: This label must be set to `dws-local-queue`. This is a mandatory requirement that signals the workload should be handled by the DWS provisioning queue.
2.  **Annotation (`provreq.kueue.x-k8s.io/maxRunDurationSeconds`)**: This annotation specifies the maximum amount of time the workload is allowed to run once it is scheduled. It is a key parameter for DWS Flex Queued Provisioning. We will set this to **86400** seconds (24 hours), though you can adjust this as needed.
3.  **`parallelism` and `completions`**: Setting these to `2` indicates a request for two separate Pods, each requiring 8 GPUs, totaling a requirement of 16 GPUs across two nodes. This effectively triggers the multi-node, "All-or-Nothing" requirement.
4.  **Required Toleration(cloud.google.com/gke-queued)**:. This toleration allows the pod to be scheduled onto nodes that are tainted with the `cloud.google.com/gke-queued:NoSchedule` key/effect, which is automatically applied to nodes in the Queued Provisioning node pool.

Create a file named `multi-node-deploy-qp.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: queued-provisioning-job
  namespace: default
  labels:
    # REQUIRED: Targets the DWS provisioning queue
    kueue.x-k8s.io/queue-name: dws-local-queue 
  annotations:
    # REQUIRED: Maximum duration the job can run after starting
    provreq.kueue.x-k8s.io/maxRunDurationSeconds: "86400s" # 24 hours
spec:
  parallelism: 2 # Requests 2 Pods (2 nodes in this case)
  completions: 2
  suspend: false
  template:
    spec:
      nodeSelector:
        cloud.google.com/gke-accelerator: nvidia-h100-80gb 
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "cloud.google.com/gke-queued" # Targets queued provisioned nodepool (Option B)
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      containers:
      - name: dummy-job
        image: gcr.io/k8s-staging-perf-tests/sleep:v0.0.3
        args: ["3600s"] 
        resources:
          requests:
            nvidia.com/gpu: 8
          limits:
            nvidia.com/gpu: 8
      restartPolicy: Never
```

**Apply the Job:**

```bash
kubectl apply -f multi-node-deploy-qp.yaml
```

When this Job is applied, GKE will first check the availability of all 16 required GPUs (across 2 nodes). If they are not immediately available, the Job will remain in a pending/suspended state until the DWS Flex provisioning mechanism successfully acquires all necessary resources, ensuring your multi-node job starts completely and simultaneously.


## Advanced Considerations for GKE and DWS Flex

### Preventing Autoscaler Scale-Down for Flex-Start Nodes

By default, the GKE Cluster Autoscaler is designed for cost efficiency and will automatically remove (scale down) nodes that are not running any Pods. For DWS Flex nodes, this typically occurs within 1 to 10 minutes of inactivity, depending on your autoscaler configuration.

If your workflow requires Flex-Start nodes to remain available for a longer period (up to their maximum run duration) and you accept the associated cost, you can prevent this automatic scale-down using two standard Kubernetes methods:

1.  **Node Annotation (Granular Control):**
    You can add a specific annotation directly to the node to tell the Cluster Autoscaler to ignore it:
    * `"cluster-autoscaler.kubernetes.io/scale-down-disabled": "true"`

2.  **DaemonSet with Pod Annotation (Easier Implementation):**
    A simpler method for cluster-wide control is to run a small DaemonSet on the nodes and apply the following annotation to the Pod specification:
    * `"cluster-autoscaler.kubernetes.io/safe-to-evict": "false"`

    This signals that a critical Pod is running on the node, preventing the autoscaler from scaling down the underlying node until the Pod is removed.

### Note on Compact Placement

When requesting large GPU resources, users often require compact placement, meaning the nodes are physically located close to one another to minimize network latency (e.g., for multi-node distributed training).

It is important to understand that DWS Flex offers best-effort compact placement but does not provide a hard guarantee.

## Conclusion and Next Steps

We have successfully demonstrated how to integrate DWS Flex with Google Kubernetes Engine, moving your AI workloads from simple VM provisioning to a robust, scalable, and production-ready orchestration environment. By leveraging GKE's cluster autoscaler with DWS Flex, you can achieve powerful scale-to-zero GPU efficiency while maintaining excellent availability for high-demand resources.

### Next Steps

In the next of this series, we will continue exploring sophisticated orchestration methods by diving into the hands-on, step-by-step guide for making use of DWS Flex on Slurm, a powerful workload manager often used in high-performance computing (HPC) environments.

-----



