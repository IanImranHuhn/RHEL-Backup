# Azure Virtual Machine Backup & Restore Guide

## Overview

This guide covers how to back up virtual machines on the Azure platform, collect system state information, restore VMs from backups, and cleanly remove all related resources. Two VMs are used throughout: a **master machine** (record-keeping) and a **client machine** (the machine being backed up and restored).

---

## Prerequisites

- An active Azure account with CLI access
- Sufficient permissions to create resource groups, VMs, storage accounts, and Recovery Services Vaults

---

## 1. Setting Up the Environment

### 1.1 Create a Resource Group

```bash
az group create --name rg-rhel --location eastus2
```

### 1.2 Create Two Virtual Machines

Use the Azure CLI to create both virtual machines in a single command (using the `--count` flag, which is currently in preview and may be unstable):

```bash
# Create two VMs (master and client) via CLI
# Note: The --count flag is in preview and may fail intermittently.
# If it fails, delete the VMs and discs, then retry or use the Azure Portal.
```

> **Note:** If the CLI command fails due to the `--count` flag being in preview (beta), remove the VMs and attached disks, then re-execute or create the VMs manually through the Azure Dashboard.

---

## 2. Configuring the Virtual Machines

### 2.1 Set Hostnames

SSH into each VM and rename them for clarity.

**Master Machine:**

```bash
ssh azureuser@<master-public-ip>
sudo su -
hostnamectl set-hostname <VM-name>-master
```

**Client Machine:**

```bash
ssh azureuser@<client-public-ip>
sudo su -
hostnamectl set-hostname <VM-name>-client
```

### 2.2 Create Working Directories

**On the master machine** — create a directory to store collected client data:

```bash
mkdir /client-vm
```

**On the client machine** — create a `before` directory to store pre-backup system state:

```bash
mkdir /before
```

### 2.3 Restart VMs to Apply Hostname Changes

**Best practice:** Make sure to reboot either on the GUI dashboard or in Azure CLI with the command below. DO NOT reboot within the VM

```bash
az vm restart --resource-group <ResourceGroupName> --name <VMName>
```

Reconnect after the reboot and verify the new hostnames appear in the shell prompt.

---

## 3. Collecting System State Information (Client Machine)

SSH into the client machine as root and run the following three commands to capture the current system state into the `before` directory.

### 3.1 Installed Packages List

```bash
dnf list installed > /before/installed_list.txt
```

### 3.2 Packages Needing Updates

```bash
dnf check-update > /before/check_update.txt
```

> This command may take a while as it checks all packages against the repository.

### 3.3 RPM Package Database

```bash
rpm -qa > /before/package_list.txt
```

---

## 4. Transferring Data to the Master Machine

### 4.1 Enable Root SSH Login on the Master Machine

By default, root SSH login is disabled. To allow SCP transfers as root:

```bash
# On the master machine
vi /etc/ssh/sshd_config
# Add the following line:
PermitRootLogin yes

# Restart SSH daemon
systemctl restart sshd
systemctl status sshd

# Set a password for the root account
passwd root
```

### 4.2 Secure Copy the `before` Directory

From the client machine, copy the `before` directory to the master machine:

```bash
scp -r /before <master-private-ip>:/client-vm/
```

Verify the transfer on the master machine:

```bash
ls /root/clientVM/before
cat /root/clientVM/before/check_update.txt
cat /root/clientVM/before/installed_list.txt
cat /root/clientVM/before/package_list.txt
```

---

## 5. Creating a Backup of the Client Machine

### 5.1 Set Up a Recovery Services Vault

1. In the Azure Portal, navigate to the **Client VM**.
2. In the left panel, under **Operations**, click **Backup**.
3. Configure the Recovery Services Vault:
   - **Name:** `rhel-backups`
   - **Resource Group:** Use the same resource group as the VMs
   - **Region:** East US 2
4. Edit the backup policy — change from **Hourly** to **Weekly** (or your preferred schedule).
5. Click **OK**, then scroll down and click **Enable Backup**.

> **Soft Delete Notice:** Azure enables soft delete by default on Recovery Services Vaults. This prevents immediate deletion of backups — you must wait the configured number of days (e.g., 14 days or 1 month) before backups are fully removed. A workaround is shown in [Section 8](#8-cleaning-up-resources).

### 5.2 Trigger the Backup

1. Navigate to the Client VM → **Backup**.
2. Click **Backup Now**.
3. To monitor progress: VM → **Backup** → **View Jobs**.

> The backup process may take approximately 30 minutes depending on VM size.

---

## 6. Restore Scenario 1 — Reverting a Failed Patch Update

This scenario simulates a failed patching operation where you need to revert the VM to its pre-update state.

### 6.1 Update All Packages on the Client Machine

```bash
dnf update -y
```

### 6.2 Collect Post-Update System State

```bash
mkdir /after

dnf list installed > /after/installed_list.txt
dnf check-update > /after/check_update.txt
rpm -qa > /after/package_list.txt
```

Reboot to apply kernel updates, then append updated check results:

```bash
reboot
# After reconnecting:
dnf check-update >> /after/check_update.txt
```

### 6.3 Transfer the `after` Directory to Master Machine

```bash
scp -r /after <master-private-ip>:/client-vm/
```

### 6.4 Simulate Pre-Restore Conditions

Delete the `before` directory from the client machine to verify it is restored later:

```bash
rm -rf /before
```

### 6.5 Deallocate the Client VM

> **Important:** The VM **must be stopped and deallocated** before restoring from a backup.

In the Azure Portal: VM → **Stop**. Wait for the status to show **Deallocated**.

### 6.6 Create a Staging Storage Account

A staging storage account (blob) is required for the restore process:

1. In the Azure Portal, create a new **Storage Account**:
   - **Name:** `rhelblob311` (lowercase letters and numbers only)
   - **Region:** East US 2 (must match VM region)
   - **Redundancy:** Locally Redundant Storage (LRS)
   - **Access Tier:** Hot
2. Click **Review + Create**, then **Create**.

### 6.7 Restore the VM

1. Navigate to the Client VM → **Backup** → **Restore VM**.
2. **Select a restore point** — choose the most recent snapshot, click **OK**.
3. Choose **Replace Existing** (to overwrite the current VM rather than creating a new one).
4. Select the staging storage account (`rhelblob311`).
5. Click **Restore**.

Monitor progress: VM → **Backup** → **View Jobs**.

### 6.8 Verify the Restore

Start the VM and SSH in:

```bash
ssh azureuser@<client-public-ip>
sudo su -
ls /root/
# Expected output: before (restored), no 'after' directory
```

Verify the package versions were reverted:

```bash
dnf check-update
rpm -q xfsdump
# Should show original version without the updated suffix (e.g., no '_3')
```

---

## 7. Restore Scenario 2 — Recovering an Inoperable VM

This scenario simulates a critically damaged VM (e.g., deleted system files) that cannot boot.

### 7.1 Corrupt the VM (Simulation)

```bash
# Move the GRUB configuration file
mv /boot/grub2/grub.cfg /boot/grub2/grub.cfg.bak

# Remove all files in root directory (DESTRUCTIVE)
rm -rf --no-preserve-root /
```

The VM will become unresponsive after this. Attempts to SSH or connect via Bastion will time out.

### 7.2 Confirm the VM is Inoperable

- The Azure Portal may show the VM as **Running**, but SSH connections will time out.
- Attempting to connect via **Bastion** will also result in a connection error.
- This confirms the machine is not operable.

### 7.3 Deallocate and Restore

1. In the Azure Portal, **Stop** the VM (deallocate it).
2. Navigate to VM → **Backup** → **Restore VM**.
3. Select the restore point → **Replace Existing** → select staging blob → **Restore**.
4. Monitor the job in **View Jobs**.

### 7.4 Verify the Restore

Start the VM and SSH in:

```bash
ssh azureuser@<client-public-ip>
sudo su -
ls /
# Expected: 'before' directory is present

dnf check-update
rpm -q xfsdump
# Original package versions should be restored
```

---

## 8. Cleaning Up Resources

### 8.1 Delete the Resource Group (Initial Attempt)

```bash
az group delete --name rg-rhel --yes
```

> This will fail for the Recovery Services Vault due to **soft delete** protection. The VMs will be deleted, but the vault will remain.

### 8.2 Remove the Recovery Services Vault

#### Step 1 — Stop Scheduled Backups

1. In the Azure Portal, go to the **Recovery Services Vault** (`rhel-backups`).
2. Navigate to **Protected Items** → **Backup Items** → **Azure Virtual Machine**.
3. Click the **ellipsis (⋯)** next to the backup item.
4. Select **Stop Backup**.
5. Choose **Delete Backup Data**, provide a reason (e.g., "Others"), and confirm.

#### Step 2 — Delete the Backup Data

If you selected **Disable Backup** instead of **Delete Backup Data** in Step 1:

1. Return to **Backup Items** → click the **ellipsis (⋯)**.
2. Select **Delete Backup**.
3. Copy the backup item name shown on screen and paste it into the confirmation field.
4. Click **Delete**.

### 8.3 Delete the Resource Group (Final)

Once the vault is cleared, delete the resource group:

```bash
az group delete --name rg-rhel --yes
```

Verify in the Azure Portal that all resources have been removed.

---

## Summary

| Step | Action |
|------|--------|
| 1 | Create resource group and two VMs (master + client) |
| 2 | Configure hostnames and working directories |
| 3 | Collect system state on client (`before` directory) |
| 4 | Transfer state data to master machine via SCP |
| 5 | Create Recovery Services Vault and trigger backup |
| 6 | Simulate patch failure, collect `after` state, restore VM |
| 7 | Simulate VM corruption, restore from backup |
| 8 | Stop backups, delete vault, remove all resources |

---

## Key Notes

- The VM must be **stopped and deallocated** before restoring from a backup.
- The VM can be in **any state** (running or deallocated) when a backup is initiated.
- A **staging storage account** in the same region as the VM is required for restore operations.
- **Soft delete** on Recovery Services Vaults prevents accidental deletion — follow the cleanup steps above to bypass it intentionally.
- The `--count` flag in Azure CLI VM creation is currently in **preview** and may be unstable.
