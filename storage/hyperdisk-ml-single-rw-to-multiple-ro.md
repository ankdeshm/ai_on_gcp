# Google Cloud Hyperdisk ML: Single RW to Multi RO Workflow

This guide outlines the steps to create a Hyperdisk ML, populate it with data in Read/Write (RW) mode on a single VM, then convert it to Read-Only Many (RO) mode and attach it to multiple (8) instances in your Managed Instance Group (MIG).

**Important Considerations:**

* **Hyperdisk ML Read-Only Immutability:** Once a Hyperdisk ML volume is set to `READ_ONLY_MANY`, it **cannot be reverted back to a write mode**. Ensure your data population is fully complete and verified before performing the mode update.
* **Permissions:** Ensure your `gcloud` user has the necessary IAM permissions (`compute.disks.*`, `compute.instances.*`, `iam.serviceAccounts.actAs` if applicable, `storage.objectViewer` for `gsutil`).
* **Organization Policies:** Be aware of any `constraints/iam.allowedPolicyMemberDomains` policies that might restrict which service accounts can be granted permissions.

## Prerequisites

Before starting, ensure you have:

* The Google Cloud SDK installed and authenticated.
* Your project ID set.
* The `ZONE` and `PROJECT_ID` variables exported.
* The `VM_NAME_RW` (your initial RW VM) and `INSTANCE_GROUP_NAME` (your MIG name) variables set.
* Your GCS bucket populated with the data you want to copy.

```bash
# Set environment variables
export PROJECT_ID=<PROJECT_ID>
export REGION=<REGION>
export ZONE=<ZONE>

# Name of the Hyperdisk ML you are creating
export HYPERDISK_NAME=>HYPERDISK_NAME>

# Name of the VM you will use to initially populate the disk
# This should be one of the VMs from your MIG
export VM_NAME_RW=<VM_NAME_RW>

# Your Managed Instance Group name
export INSTANCE_GROUP_NAME=<INSTANCE_GROUP_NAME>

# Get all VM names in the MIG (for later attachment)
# This command will list all instances managed by the MIG in your specified zone.
# It uses --flatten and --filter to extract just the instance names.
# Ensure your INSTANCE_GROUP_NAME and ZONE are correct.
ALL_VM_NAMES=$(gcloud compute instance-groups managed list-instances "${INSTANCE_GROUP_NAME}" --zone="${ZONE}" --format="value(instance)" --limit=8)

# You can verify the VM names it found:
echo "VMs in ${INSTANCE_GROUP_NAME}:"
echo "${ALL_VM_NAMES}"
```

---

## Workflow Steps

### Phase 1: Create Hyperdisk ML and Populate Data (Single RW)

1.  **Create the Hyperdisk ML disk in READ_WRITE_SINGLE mode:**

Feel free to change according to your specifications

```bash
    gcloud compute disks create ${HYPERDISK_NAME} \
        --project=${PROJECT_ID} \
        --zone=${ZONE} \
        --size=500GB \
        --type=hyperdisk-ml \
        --provisioned-throughput=1200 \
        --access-mode=READ_WRITE_SINGLE
```

2.  **Attach the Hyperdisk ML to a single VM in Read/Write (RW) mode:**

    ```bash
    gcloud compute instances attach-disk ${VM_NAME_RW} \
        --disk=${HYPERDISK_NAME} \
        --device-name=my-data-disk \
        --mode=rw \
        --zone=${ZONE}
    ```

3.  **SSH into the VM and prepare the disk for data transfer:**

    ```bash
    # SSH into the VM
    gcloud compute ssh ${VM_NAME_RW} --zone=${ZONE}
    ```
    Once inside the VM's shell, execute the following commands:

    ```bash
    # Identify the correct disk device name (e.g., /dev/nvme0n2 from your `lsblk` output)
    lsblk

    # Use the identified device name (e.g., /dev/nvme0n2)
    # IMPORTANT: Confirm this from your lsblk output!
    export DISK_DEVICE_PATH="/dev/nvme0n2" 
    
    # 1. Create the mount point directory
    sudo mkdir -p /mnt/my-data

    # 2. Format the disk with ext4 filesystem
    #    IMPORTANT: This will erase all data on the specified disk.
    sudo mkfs.ext4 -F ${DISK_DEVICE_PATH}

    # 3. Mount the disk to the created directory
    sudo mount -o discard,defaults ${DISK_DEVICE_PATH} /mnt/my-data

    # 4. Change ownership of the mounted directory to your user
    CURRENT_USER=$(whoami)
    sudo chown -R ${CURRENT_USER}:${CURRENT_USER} /mnt/my-data

    # 5. Verify that the disk is mounted and has correct permissions
    df -h /mnt/my-data
    ls -ld /mnt/my-data
    ```

4.  **Copy data from your GCS bucket to the mounted disk on the VM:**

    Still inside the VM's SSH session:

    ```bash
    gsutil -m cp -r gs://<bucket_name>* /mnt/my-data/
    ```
    * This command copies all files and folders from your GCS bucket to the `/mnt/my-data/` directory on your Hyperdisk. 
    
    * if permission errors: Ensure your VM's service account has `storage.objectViewer` (or `storage.objectAdmin`) permissions on the GCS bucket.

---

### Phase 2: Unmount and Detach from RW VM

1.  **Unmount the disk on the VM:**

    Still inside the VM's SSH session:

    ```bash
    sudo umount /mnt/my-data
    ```
    * (Optional verification): `df -h /mnt/my-data` should show an error or no entry.

2.  **Exit the SSH session:**

    ```bash
    exit
    ```
    You should now be back in your Cloud Shell (or local terminal).

3.  **Detach the disk from the VM (from your Cloud Shell/local terminal):**

    ```bash
    gcloud compute instances detach-disk ${VM_NAME_RW} \
        --disk=${HYPERDISK_NAME} \
        --zone=${ZONE}
    ```

---

### Phase 3: Update Disk Access Mode to Read-Only Many

**Important Note for Hyperdisk ML: Once a Hyperdisk ML volume is set to READ_ONLY_MANY, you cannot revert it back to a write mode. Be absolutely sure your data population is complete and correct before proceeding.**

This step changes the fundamental access mode of your Hyperdisk ML.

```bash
gcloud compute disks update ${HYPERDISK_NAME} \
    --zone=${ZONE} \
    --access-mode=READ_ONLY_MANY
```

---

### Phase 4: Attach to All Instances in Read-Only Mode

Now, attach the Hyperdisk ML to all 8 of your VMs in read-only mode. We'll loop through the `ALL_VM_NAMES` variable populated at the beginning.

```bash
echo "${ALL_VM_NAMES}" | while read VM_NAME; do
    echo "Attaching ${HYPERDISK_NAME} to ${VM_NAME}..."
    gcloud compute instances attach-disk "${VM_NAME}" \
        --disk=${HYPERDISK_NAME} \
        --device-name=shared-ml-data \
        --mode=ro \
        --zone=${ZONE}
done
```
* `--device-name=shared-ml-data`: This is the name the OS inside the VM will use to identify the disk (e.g., `/dev/disk/by-id/google-shared-ml-data`).
* `--mode=ro`: Ensures the disk is attached as read-only.

---

### Phase 5: Mount on All Instances (Read-Only)

Finally, SSH into each of your 8 VMs and mount the disk in read-only mode.

```bash
echo "${ALL_VM_NAMES}" | while read VM_NAME; do
    echo "Mounting ${HYPERDISK_NAME} on ${VM_NAME}..."
    gcloud compute ssh "${VM_NAME}" --zone=${ZONE} --command="""
        # Create the mount point if it doesn't exist
        sudo mkdir -p /mnt/shared-data

        # Mount the disk in read-only mode
        # Use the device name from the `attach-disk` command
        sudo mount -o ro /dev/disk/by-id/google-shared-ml-data /mnt/shared-data

        # Verify the mount and data (optional)
        df -h /mnt/shared-data
        ls -l /mnt/shared-data/
    """
done
```

---

You have now successfully created, populated, converted, and distributed your Hyperdisk ML data to all 8 of your VMs in read-only mode!

## Useful Resources
[1] https://cloud.google.com/compute/docs/disks/hyperdisks#when-to-use

[2] https://cloud.google.com/compute/docs/disks/attach-disks#attach_disk

[3] https://cloud.google.com/compute/docs/disks/sharing-disks-between-vms#ro-limitations

