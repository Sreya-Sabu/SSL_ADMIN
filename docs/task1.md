# Task 1 — Initial Setup

## 1.1 Azure VM Creation

Created an Ubuntu 22.04 VM on Microsoft Azure using NITC student benefits. Azure is a cloud platform — the VM runs on Microsoft's physical servers via a **hypervisor**, which allows multiple isolated VMs to share the same hardware.

Updated the VM's public IP in the shared document and was assigned a domain pointing to it.

> **Concept:** A domain is a human-readable name that DNS resolves to your VM's IP. When someone visits your domain, DNS translates it to the IP and connects to your VM.

## 1.2 System Updates and Security

Updated all packages and enabled unattended upgrades for automatic daily security patches.

```bash
sudo apt update                                  # Refreshes the package list
sudo apt upgrade                                 # Installs latest package versions
sudo dpkg-reconfigure unattended-upgrades        # Enables automatic security updates
```

Verified the config:

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
```

> **Note:** `apt update` only refreshes the index — it does not install anything. `apt upgrade` does the actual installation. Config files live in `/etc/`, the standard Linux directory for system-wide settings.
