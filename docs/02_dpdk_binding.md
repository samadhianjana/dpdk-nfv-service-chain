# 02\. DPDK Installation & Binding

This guide details the process of installing the Data Plane Development Kit (DPDK) and binding the network interfaces to the userspace drivers. This step is critical to bypassing the Linux kernel for high-speed packet processing.

## 1\. Install & Build DPDK (Both VMs)

We will install DPDK v25.11 (LTS) or later on both the Traffic Generator (`dpdk-1`) and the Forwarder (`dpdk-2`).

### A. Download Source Code

```bash
cd ~
wget http://fast.dpdk.org/rel/dpdk-25.11.tar.xz  # Or your specific version
tar xf dpdk-25.11.tar.xz
cd dpdk-25.11
```

### B. Configure and Build

We use `meson` and `ninja` for the build process. We specifically enable all examples to ensure `helloworld`, `skeleton`, and `l2fwd` are available.

```bash
# Configure the build
meson build -Dexamples=all

# Compile (This may take a few minutes)
ninja -C build

# Install libraries system-wide
sudo ninja install
sudo ldconfig
```

## 2\. Identify Network Ports

Before binding, you must identify which PCI addresses correspond to your isolated network links (`link-0` and `link-1`).

Run the following command on both VMs:

```bash
dpdk-devbind.py -s
```

**Output Interpretation:**

  * **Network devices using kernel driver:** You should see your standard `virtio-pci` devices here.
  * **0X:00.0** (e.g., `04:00.0`): Note the PCI address of the interfaces that are **NOT** your management IP (the ones *without* the active SSH connection).

> **Warning:** Do not bind the interface used for SSH (usually `enp1s0`). If you do, you will lose connection to the VM immediately.

## 3\. Prepare Interfaces (Link Down)

Before binding a device to DPDK, you **must** bring the interface down in the Linux kernel. If you skip this step, the kernel may refuse to release the device, resulting in "Device or resource busy" errors.

Using the interface names identified in the previous step (e.g., `enp4s0`, `enp5s0`):

```bash
# Switch to root
sudo -i

# Bring down the interfaces (Replace with your actual names)
ip link set dev enp4s0 down
ip link set dev enp5s0 down
```

*Verification: Run `ip a`. These interfaces should no longer show the `UP` flag.*

## 4\. Load Userspace Drivers

DPDK requires a specific kernel module to interact with the hardware. For virtual environments, `uio_pci_generic` is recommended.

```bash
# Load the module
modprobe uio_pci_generic
```

## 5\. Bind Interfaces to DPDK

We now disconnect the network cards from the Linux Kernel and hand control over to DPDK.

Execute the binding command using the PCI addresses identified in Step 2:

```bash
# Syntax: dpdk-devbind.py -b <driver> <pci_address_1> <pci_address_2>

# Example (Replace with your actual PCI IDs):
dpdk-devbind.py -b uio_pci_generic 04:00.0 05:00.0
```

### Verification

Run the status command again to confirm success:

```bash
dpdk-devbind.py -s
```

  * You should now see the two ports listed under **"Network devices using DPDK-compatible driver"**.
  * They should no longer be visible if you run `ip a` or `ifconfig`.

## 6\. Testing the Setup (Optional)

To ensure DPDK can access the hardware and memory, run the `helloworld` example:

```bash
cd build/examples
sudo ./dpdk-helloworld -l 0-1 -n 2
```

  * **Success:** You will see "hello from core X" messages.
  * **Failure:** If you see "EAL: Cannot get hugepage information," verify your Hugepages setup from Document 01.