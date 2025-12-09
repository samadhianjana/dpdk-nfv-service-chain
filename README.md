# DPDK NFV Service Chain

**A high-performance Network Function Virtualization (NFV) testbed implementing a transparent L2 forwarding service chain using KVM, DPDK, and Cisco TRex.**

## Overview

This project demonstrates the implementation and benchmarking of a virtualized network service chain. By utilizing **Kernel Bypass networking (DPDK)**, this setup achieves high-throughput packet processing between virtual machines, bypassing the standard Linux kernel network stack to minimize latency and packet loss.

The topology implements a **Unidirectional "Snake" Flow**:

  * **Source:** Traffic is generated on **TRex Port 0**.
  * **Forwarding:** The DUT (Device Under Test) receives packets on Port 0, applies a **custom MAC rewrite**, and forwards them to Port 1.
  * **Destination:** Traffic returns to the Generator on **TRex Port 1**.

## Architecture

The setup utilizes two nested Virtual Machines running on a KVM Hypervisor. The network path is fully isolated from the host OS to ensure accurate benchmarking.

![architecture diagram](docs/images/architecture_diagram.png)

## Technology Stack

  * **Hypervisor:** QEMU/KVM with Libvirt (Virtual Machine Manager)
  * **Packet Processing:** DPDK (Data Plane Development Kit) v25.11 **(Customized L2fwd)**
  * **Traffic Generator:** Cisco TRex v3.08
  * **Guest OS:** Ubuntu Linux
  * **Drivers:** `virtio-pci` with `uio_pci_generic`
  * **Network Mode:** Isolated Virtual Networks (No NAT/DHCP interference)

## Documentation

Detailed step-by-step instructions for reproducing this lab are available in the `docs/` directory:

  * [01. Environment Setup & Prerequisites](docs/01_setup_vm.md)
      * Hugepages configuration
      * CPU Pinning and Isolation
  * [02. DPDK Installation & Binding](docs/02_dpdk_binding.md)
      * Compiling DPDK
      * Binding ports to `uio_pci_generic`
  * [03. Configuring the Service Chain](docs/03_service_chain.md)
      * Setting up Isolated Bridges
      * **Implementing Custom MAC Rewrite Logic (C-Code Mod)**
  * [04. Running Benchmarks](docs/04_benchmarking.md)
      * Running L2fwd
      * Executing TRex Scripts with Port Pinning

## Performance Results

*Current benchmark results running on a nested KVM environment:*

| Metric | Result | Notes |
| :--- | :--- | :--- |
| **Throughput (Zero Loss)** | **\~21 Mbps** | Limited by nested virtualization overhead |
| **Packet Loss @ Max Load** | **0%** | Verified via TRex TUI |
| **Topology** | **Unidirectional** | Port 0 -\> Port 1 |

## Quick Start (Summary)

To spin up the service chain once the environment is configured:

**1. Start the Forwarder (VM2):**

```bash
# Note: Do NOT use '--no-mac-updating' because we are using our custom logic.
# Ensure you point to the specific build location:
sudo ./build/examples/l2fwd/dpdk-l2fwd -l 0-1 -n 4 -- -p 0x3 -T 1
```

**2. Start the Generator (VM1):**

```bash
# Start TRex Server (Window 1)
sudo ./t-rex-64 -i

# In a separate console (Window 2), run the benchmark
# We add '-p 0' to force traffic to originate ONLY from Port 0
./trex-console
start -f stl/bench.py -m 10mbps -p 0
```

## License

This project is open-source and available under the MIT License.