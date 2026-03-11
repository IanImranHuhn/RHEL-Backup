# RHEL-Backup

# Patch Management on Red Hat Enterprise Linux (RHEL)

> **Platform:** Microsoft Azure | **OS:** Red Hat Enterprise Linux 9.7  
> **Privilege Required:** Root

---

## Table of Contents

1. [What is Patch Management?](#what-is-patch-management)
2. [Prerequisites](#prerequisites)
3. [Connecting to the Virtual Machine](#connecting-to-the-virtual-machine)
4. [Switching to Root User](#switching-to-root-user)
5. [Inspecting Installed Packages](#inspecting-installed-packages)
6. [Identifying the OS Version](#identifying-the-os-version)
7. [Checking Repository Configuration](#checking-repository-configuration)
8. [Checking for Available Updates](#checking-for-available-updates)
9. [Performing the Update](#performing-the-update)
10. [Verifying the Update](#verifying-the-update)
11. [Rebooting for Kernel Updates](#rebooting-for-kernel-updates)
12. [Best Practices & Warnings](#best-practices--warnings)

---

## What is Patch Management?

Patch management is the process of updating a system with packages that fix known vulnerabilities identified by vendors (such as Red Hat). These vulnerabilities can represent security flaws that threat actors may exploit.

**Why it matters:**
- Closes known security vulnerabilities
- Minimizes the risk of unauthorized system access
- Keeps software dependencies up to date

---

## Prerequisites

- Access to an Azure virtual machine running RHEL
- SSH client (e.g., terminal with SSH support)
- Root or sudo privileges on the target machine

---

## Connecting to the Virtual Machine

Open an **elevated terminal** and SSH into the virtual machine using its public IP address:

```bash
ssh <azure_user>@<public_ip_address>
```

> ✅ Confirm you're on the correct machine by verifying the hostname displayed matches the expected VM hostname.

---

## Switching to Root User

Patch management requires **root-level privileges**. Switch to the root user using:

```bash
sudo su -
```

Enter your password when prompted. A successful switch is indicated by the shell prompt changing from `$` (standard user) to `#` (root user).

---

## Inspecting Installed Packages

### List All Available Packages

To view packages available for installation (not yet installed):

```bash
rpm -qa
```

### List Currently Installed Packages

To view packages that are currently installed on the system:

```bash
dnf list installed
```

---

## Identifying the OS Version

Before updating, it is important to confirm the exact OS version, as different versions may require different repositories and may not be cross-compatible.

```bash
cat /etc/redhat-release
```

**Example output:**
```
Red Hat Enterprise Linux release 9.7
```

> ⚠️ Always note both the **major** (e.g., `9`) and **minor** (e.g., `7`) version numbers before proceeding.

---

## Checking Repository Configuration

Repositories are collections of packages maintained by vendors. Your system pulls updates from these configured sources.

List the repository configuration directory:

```bash
ls /etc/yum.repos.d/
```

View the default repository configuration file:

```bash
cat /etc/yum.repos.d/rh-cloud.repo
```

> This configuration file defines where the system downloads packages from. The default points to Red Hat's trusted repositories.

---

## Checking for Available Updates

Run the following command to scan the system and display all packages that have updates available. **No changes are made at this step.**

```bash
dnf check-update
```

### Understanding the Output

The output is organized in three columns:

| Column | Description |
|--------|-------------|
| **Package Name** | The name of the package with a pending update |
| **New Version** | The version number the package will be updated to |
| **Repository** | The source repository providing the update |

> ℹ️ This step only **checks** — it does not install or modify anything.

---

## Performing the Update

Once you have reviewed the available updates, run the following command to apply them:

```bash
dnf update
```

When prompted, confirm the operation by entering `y` (yes).

### Transaction Summary

Before installation begins, DNF will display a **transaction summary**, which includes:

- Number of **new packages** to be installed
- Number of **existing packages** to be upgraded
- **Total download size**
- **Disk space** to be used

### What Happens During the Update

1. **Download** — All packages are downloaded from the configured repositories
2. **Verify** — Each package is verified (e.g., via cryptographic hash) to ensure integrity and prevent tampering
3. **Install/Upgrade** — New packages are installed and existing ones are upgraded
4. **Cleanup** — Old package versions are removed to free up disk space

> ⏳ This process can take time. **Do not interrupt it.** Cancelling mid-update can corrupt packages and destabilize the system.

---

## Verifying the Update

After the update completes, confirm that all packages are now current:

```bash
dnf check-update
```

If no packages are listed in the output, the system is fully up to date.

### Checking the Running Kernel Version

```bash
uname -r
```

> If a kernel update was applied, the currently **running** kernel may differ from the **installed** kernel until a reboot is performed.

---

## Rebooting for Kernel Updates

A kernel update requires a **system reboot** to load the new kernel into memory.

```bash
reboot
```

After the system restarts, SSH back in and verify the active kernel:

```bash
uname -r
```

Confirm the version matches the latest installed kernel from the update.

---

## Best Practices & Warnings

| ⚠️ Warning | Recommendation |
|-----------|----------------|
| Do not update production systems without prior testing | Test all updates in a **staging/test environment** first |
| Package updates may disrupt running services | Review the package list and assess impact before applying |
| Kernel updates require a reboot | Schedule reboots during **maintenance windows** |
| Interrupted updates can corrupt packages | Allow updates to **complete fully** before taking any action |
| Always make backups before patching | Create a **VM snapshot** or backup prior to updating |
| Monitor after patching | Keep an eye on system health and services **post-update** |

---

## Summary

```
1. SSH into the VM
2. Switch to root: sudo su -
3. Check OS version: cat /etc/redhat-release
4. Review available updates: dnf check-update
5. Apply updates: dnf update
6. Verify updates: dnf check-update
7. Reboot if kernel was updated: reboot
8. Confirm new kernel: uname -r
```

---

*Documentation based on a live demonstration using Azure-hosted RHEL 9.7.*
