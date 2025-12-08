# 01\. Environment Setup & Prerequisites

This guide outlines the steps to prepare the Host Machine (Hypervisor) and configure the two Virtual Machines (VMs) required for the Service Chain.

## 1\. Host Machine Requirements

The physical machine hosting the lab must have **Hardware Virtualization** enabled in the BIOS (VT-x for Intel or AMD-V for AMD).

**Required Software (Ubuntu Host):**

```bash
sudo apt update
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
```

*Ensure your user is added to the libvirt group to manage VMs without sudo:*

```bash
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER
# (Log out and log back in for changes to take effect)
```

## 2\. Virtual Network Configuration

To simulate a physical cable between the Traffic Generator and the Forwarder, we must create **Isolated Networks**. These networks function like a "dumb switch" or a patch cable—they have no DHCP, no DNS, and no connection to the outside world/internet.

### Create Network 1 (Link-0)

1.  Open **Virtual Machine Manager**.
2.  Go to **Edit -\> Connection Details -\> Virtual Networks**.
3.  Click **+ (Add Network)**.
4.  **Name:** `link-0`
5.  **Mode:** Isolated (Private). *Ensure IPv4/IPv6 configuration is disabled/none.*
6.  **Finish.**

### Create Network 2 (Link-1)

1.  Repeat the steps above.
2.  **Name:** `link-1`
3.  **Mode:** Isolated (Private).

*Why Isolated? using NAT or Bridged mode introduces background traffic (ARP, DHCP, Multicast) that interferes with DPDK packet counting. Isolated mode ensures the wire is perfectly silent.*

## 3\. Virtual Machine Creation

Create two Virtual Machines using Ubuntu 22.04 or 24.04.

### VM 1: Traffic Generator (`dpdk-1`)

  * **vCPUs:** 2 or 4
  * **RAM:** 4 GB (Recommended for TRex)
  * **Storage:** 30 GB
  * **Network Interfaces:**
    1.  **NIC 1:** `default` (NAT) -\> *For SSH and Internet access.*
    2.  **NIC 2:** `link-0` (Isolated) -\> *Device Model: e1000*
    3.  **NIC 3:** `link-1` (Isolated) -\> *Device Model: e1000*

### VM 2: Traffic Forwarder (`dpdk-2`)

  * **vCPUs:** 4
  * **RAM:** 4 GB
  * **Storage:** 30 GB
  * **Network Interfaces:**
    1.  **NIC 1:** `default` (NAT) -\> *For SSH and Internet access.*
    2.  **NIC 2:** `link-0` (Isolated) -\> *Device Model: e1000*
    3.  **NIC 3:** `link-1` (Isolated) -\> *Device Model: e1000*

## 4\. Guest OS Preparation (Inside VMs)

Perform these steps on **BOTH** VMs to prepare the Operating System for DPDK.

### A. Install Dependencies

```bash
sudo apt update
sudo apt install build-essential meson python3-pyelftools pkgconf libnuma-dev 
```

### B. Enable Hugepages

DPDK requires Hugepages to reserve large chunks of memory for packet processing.

1.  Edit the Grub configuration:

    ```bash
    sudo nano /etc/default/grub
    ```

2.  Find the line `GRUB_CMDLINE_LINUX_DEFAULT` and add the hugepages parameter to the end (inside the quotes):

    ```text
    GRUB_CMDLINE_LINUX_DEFAULT="quiet splash default_hugepages=1G hugepagesz=1G hugepages=1"
    ```

    *(Note: This allocates 1GB of Hugepage memory).*

3.  Update Grub and Reboot:

    ```bash
    sudo update-grub
    sudo reboot
    ```

### C. Verify Setup

After rebooting, check that Hugepages are allocated:

```bash
grep Huge /proc/meminfo
```

*You should see `HugePages_Total: 1` (or similar).* 