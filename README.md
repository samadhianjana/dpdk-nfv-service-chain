# DPDK NFV Service Chain

**A high-performance Network Function Virtualization (NFV) testbed implementing a transparent L2 forwarding service chain using KVM, DPDK, and Cisco TRex.**

  

## Overview

This project demonstrates the implementation and benchmarking of a virtualized network service chain. By utilizing **Kernel Bypass networking (DPDK)**, this setup achieves high-throughput packet processing between virtual machines, bypassing the standard Linux kernel network stack to minimize latency and packet loss.

The topology consists of a **Traffic Generator (TRex)** and a **Traffic Forwarder (L2fwd)** connected via isolated virtual bridges, simulating a real-world VNF (Virtual Network Function) deployment.

## Architecture

The setup utilizes two nested Virtual Machines running on a KVM Hypervisor. The network path is fully isolated from the host OS to ensure accurate benchmarking.

![Architecture Diagram](docs/images/architecture_diagram.png)

## Technology Stack

  * **Hypervisor:** QEMU/KVM with Libvirt (Virtual Machine Manager)
  * **Packet Processing:** DPDK (Data Plane Development Kit) v25.11
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
      * Configuring TRex `trex_cfg.yaml`
  * [04. Running Benchmarks](docs/04_benchmarking.md)
      * Running L2fwd
      * Executing TRex Scripts (Bench/IMIX)

## Performance Results

*Current benchmark results running on a nested KVM environment:*

| Metric | Result | Notes |
| :--- | :--- | :--- |
| **Throughput (Zero Loss)** | *TBD* Mbps | *Insert your max Mbps here* |
| **Latency** | *TBD* µs | *Optional* |
| **Packet Loss @ 100Mbps** | **0%** | Verified via TRex TUI |


## Quick Start (Summary)

To spin up the service chain once the environment is configured:

**1. Start the Forwarder (VM2):**

```bash
# Bind ports and start L2fwd
sudo dpdk-devbind.py -b uio_pci_generic 00:04.0 00:05.0
sudo ./dpdk-l2fwd -l 0-1 -n 4 -- -p 0x3 -T 1 --no-mac-updating
```

**2. Start the Generator (VM1):**

```bash
# Start TRex Server
sudo ./t-rex-64 -i

# In a separate console, run the benchmark
./trex-console
start -f stl/bench.py -m 100mbps
```

## License

This project is open-source and available under the MIT License.