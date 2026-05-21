# Create Hyperdisk ML from a Non-Hyperdisk ML Snapshot

This guide demonstrates how to create a Google Cloud Hyperdisk ML volume using a snapshot taken from a different type of Persistent Disk (e.g., a Balanced Persistent Disk). This is a common pattern for populating Hyperdisk ML with large, pre-existing datasets for read-heavy AI/ML workloads.

**Key takeaway:** When creating Hyperdisk ML from a snapshot, it must be provisioned in `READ_ONLY_MANY` access mode.

## Table of Contents

1.  [Overview](#overview)
2.  [Prerequisites](#prerequisites)
3.  [Experiment Setup](#experiment-setup)
4.  [Step-by-Step Guide](#step-by-step-guide)
    * [Part 1: Create Source Disk and Snapshot](#part-1-create-source-disk-and-snapshot)
    * [Part 2: Create Hyperdisk ML from Snapshot](#part-2-create-hyperdisk-ml-from-snapshot)
    * [Part 3: Verify Hyperdisk ML Contents](#part-3-verify-hyperdisk-ml-contents)
5.  [Cleanup](#cleanup)

## Overview

This experiment validates the process of:
* Creating a standard Google Cloud Persistent Disk (e.g., `pd-balanced`).
* Attaching it to an existing Compute Engine VM.
* Adding sample data to the disk.
* Creating a snapshot of this disk.
* Using that snapshot to provision a new Hyperdisk ML volume.
* Verifying the data integrity on the new Hyperdisk ML volume.

## Prerequisites

* A Google Cloud Project with billing enabled.
* The `gcloud CLI` installed and authenticated on your local machine or Google Cloud Shell.
* An existing Compute Engine VM in your project to serve as a temporary data "hydrator" (e.g., `ankita-a2high4g-mig-940h` in `us-central1-b`).
* Sufficient IAM permissions for your user account (or Cloud Shell service account) to manage Compute Engine instances, disks, and snapshots. Essential roles include `Compute Instance Admin (v1)` and `Compute Storage Admin`.

## Experiment Setup

First, open your **Google Cloud Shell** or your local terminal with `gcloud CLI` configured, and define the necessary environment variables.

```bash
# Configuration Variables
export PROJECT_ID=$(gcloud config get-value project)
export ZONE=<SUPPORTED_ZONE>
export EXISTING_VM_NAME=<VM_NAME>
export SOURCE_DISK_NAME="ankita-source-balanced-disk"
export SNAPSHOT_NAME="ankita-balanced-disk-snapshot"
export MOUNT_POINT="/mnt/exp_data"
export HYPERDISK_ML_NAME="ankita-test-hyperdisk-ml"
```

## Step-by-Step Guide

Follow these steps precisely, paying attention to where commands should be run (Cloud Shell vs. Inside VM).

### Part 1: Create Source Disk and Snapshot

This section involves creating a temporary Balanced Persistent Disk, populating it with data, and then taking a snapshot.

1.  **Create a Balanced Persistent Disk (from Cloud Shell)**
    - This disk will serve as our "non-Hyperdisk ML" source for the snapshot.

    ```bash
    gcloud compute disks create $SOURCE_DISK_NAME \
        --project=$PROJECT_ID \
        --zone=$ZONE \
        --size=10GB \
        --type=pd-balanced
    ```
    *Expected Output:* Confirms disk creation and its `READY` status.

2.  **Attach the Balanced Disk to Your Existing VM (from Cloud Shell)**
    - Attach the newly created disk to your designated VM so you can add data to it.

    ```bash
    gcloud compute instances attach-disk $EXISTING_VM_NAME \
        --disk=$SOURCE_DISK_NAME \
        --zone=$ZONE \
        --device-name=$SOURCE_DISK_NAME
    ```
    *Expected Output:* Confirms disk attachment to the VM.

3.  **SSH into Your VM (from Cloud Shell)**
    - Connect to your VM to perform operations on the attached disk.

    ```bash
    gcloud compute ssh $EXISTING_VM_NAME --zone=$ZONE
    ```
    *Expected Output:* You will be logged into your VM's shell.

4.  **Format, Mount, and Add Data to the Disk (Inside VM)**
    - These commands are run directly within the SSH session on your VM.
    - Identify the device name of the newly attached 10GB disk.
    - Look for a 10G entry without a mountpoint, typically like /dev/nvme0n2 or /dev/sdb.
    - IMPORTANT: Replace `/dev/nvme0n2` with the actual device name identified by `lsblk`.
    - For example: sudo mkfs.ext4 -F /dev/sdb
    - Format the disk with ext4 filesystem.
    - BE CAREFUL: This will erase all data on the target disk.
    - Create a mount point directory for the disk.
    - Mount the formatted disk to the mount point.
    - Change the ownership of the mount point so your user can write to it.
    - Add some sample data files to the disk.
    - Verify that the file(s) are present on the mounted disk.
    - Expected Output:* Messages confirming filesystem creation, successful mount, and `balanced_sample.txt` appearing in `ls -l` output.

    ```bash
    lsblk
    sudo mkfs.ext4 -F /dev/nvme0n2
    sudo mkdir -p $MOUNT_POINT
    sudo mount /dev/nvme0n2 $MOUNT_POINT
    sudo chown -R $(whoami):$(whoami) $MOUNT_POINT
    echo 'This is sample data from the balanced disk.' > $MOUNT_POINT/balanced_sample.txt
    echo 'More data for Hyperdisk ML.' >> $MOUNT_POINT/balanced_sample.txt
    ls -l $MOUNT_POINT
    ```

5.  **Detach the Disk from the VM (from Cloud Shell)**
    - Exit the VM's SSH session (type `exit`) and return to your Cloud Shell. It is crucial to detach the disk *before* taking a snapshot to ensure data consistency.

    ```bash
    gcloud compute instances detach-disk $EXISTING_VM_NAME --disk=$SOURCE_DISK_NAME --zone=$ZONE --quiet
    ```

    *Potential Issue Note:* If you previously tried to run `gcloud` commands from inside the VM and got an "insufficient authentication scopes" error, this step *must* be run from Cloud Shell where your user has sufficient permissions.

    *Expected Output:* Confirmation that the disk has been detached.

6.  **Create a Snapshot of the Source Disk (from Cloud Shell)**
    - Create a snapshot of the balanced persistent disk containing your sample data.

    ```bash
    gcloud compute snapshots create $SNAPSHOT_NAME \
        --project=$PROJECT_ID \
        --source-disk=$SOURCE_DISK_NAME \
        --source-disk-zone=$ZONE
    ```
    *Expected Output:* Confirms snapshot creation.

7.  **Delete the Source Disk (from Cloud Shell)**
    - To avoid unnecessary charges, delete the original Balanced Persistent Disk now that its snapshot is created.

    ```bash
    gcloud compute disks delete $SOURCE_DISK_NAME --zone=$ZONE --quiet
    ```
    *Expected Output:* Confirms disk deletion.

### Part 2: Create Hyperdisk ML from Snapshot

Now, we will use the snapshot created from the balanced disk to provision a new Hyperdisk ML volume.

1.  **Get the Full Path of Your Snapshot (from Cloud Shell)**
    - The `gcloud compute disks create` command requires the full resource path to the snapshot.

    ```bash
    export SNAPSHOT_FULL_PATH=$(gcloud compute snapshots describe $SNAPSHOT_NAME --project=$PROJECT_ID --format="value(selfLink)")
    echo "Snapshot Full Path: $SNAPSHOT_FULL_PATH"
    ```
    *Expected Output:* The full URL path to your snapshot.

2.  **Create the Hyperdisk ML Disk from the Snapshot (from Cloud Shell)**
    - This is the core demonstration. Note the mandatory `--access-mode=READ_ONLY_MANY`.

    ```bash
    gcloud compute disks create $HYPERDISK_ML_NAME \
        --project=$PROJECT_ID \
        --zone=$ZONE \
        --type=hyperdisk-ml \
        --access-mode=READ_ONLY_MANY \
        --source-snapshot=$SNAPSHOT_FULL_PATH \
        --provisioned-throughput=$HYPERDISK_ML_THROUGHPUT
    ```
    *Expected Output:* Confirms Hyperdisk ML creation and its `READY` status.

### Part 3: Verify Hyperdisk ML Contents

Attach the newly created Hyperdisk ML to your VM and verify that the data is accessible.

1.  **Attach the Hyperdisk ML to Your Existing VM in read-only mode (from Cloud Shell)**

    ```bash
    gcloud compute instances attach-disk $EXISTING_VM_NAME \
        --disk=$HYPERDISK_ML_NAME \
        --zone=$ZONE \
        --device-name=$HYPERDISK_ML_NAME \
        --mode=ro # Use 'ro' (lowercase) for read-only mode
    ```
    *Expected Output:* Confirms disk attachment.

2.  **SSH into Your VM and Check Data(from Cloud Shell then Inside VM)**

    ```bash
    gcloud compute ssh $EXISTING_VM_NAME --zone=$ZONE
    ```
    *On the VM's SSH prompt:*
    - Identify the device name for the newly attached Hyperdisk ML (e.g., /dev/sdc or /dev/nvme0n3).
    - Create a new mount point for this specific disk.
    - Mount the Hyperdisk ML.
    - IMPORTANT: Replace `/dev/sdc` with the actual device name identified by `lsblk`.
    - Verify the content. You should see `balanced_sample.txt` and its content.
    - *Expected Output:* Displays the `balanced_sample.txt` file and its contents, confirming successful data transfer and accessibility on the Hyperdisk ML.

    ```bash
    lsblk
    sudo mkdir -p /mnt/hdml_data
    sudo mount /dev/sdc /mnt/hdml_data
    ls -l /mnt/hdml_data
    cat /mnt/hdml_data/balanced_sample.txt
    ```
    
## Cleanup

To avoid ongoing charges, remember to delete all resources created during this experiment. These commands should be run from your **Cloud Shell**.

- Detach Hyperdisk ML from VM
- Delete Hyperdisk ML
- Delete Snapshot
- Optional: If you created a temporary VM just for this demo, delete it here.


```bash
gcloud compute instances detach-disk $EXISTING_VM_NAME --disk=$HYPERDISK_ML_NAME --zone=$ZONE --quiet

gcloud compute disks delete $HYPERDISK_ML_NAME --zone=$ZONE --quiet
gcloud compute snapshots delete $SNAPSHOT_NAME --quiet

# gcloud compute instances delete YOUR_TEMP_VM_NAME --zone=$ZONE --quiet
```

References: 
- [1] https://cloud.google.com/compute/docs/disks/restore-snapshot#create-disk-from-snapshot
- [2] https://gke-ai-labs.dev/docs/tutorials/hyperdisk-ml/
- [3] https://cloud.google.com/compute/docs/disks/hd-types/hyperdisk-ml
