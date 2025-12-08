# 03\. Configuring the Service Chain

This guide details the configuration of the two critical components of the testbed: the Traffic Generator (Cisco TRex) and the Traffic Forwarder (DPDK L2fwd).

## 1\. Configure the Traffic Forwarder (VM2: `dpdk-2`)

The forwarder acts as the "Device Under Test" (DUT). It receives packets on one port and re-transmits them on the other, effectively looping the traffic back to the generator.

We use the `l2fwd` example application included with DPDK. It provides better performance and MAC address handling than the basic skeleton example.

### Running the Application

No configuration file is needed for `l2fwd`. It is configured via command-line arguments at runtime.

**Command Syntax:**

```bash
sudo ./dpdk-l2fwd -l <core_list> -n <mem_channels> -- -p <port_mask> -T <stats_interval> --no-mac-updating
```

**Operational Command:**
Run this on **VM2**:

```bash
cd ~/dpdk-21.11/build/examples
sudo ./dpdk-l2fwd -l 0-1 -n 4 -- -p 0x3 -T 1 --no-mac-updating
```

**Argument Explanation:**

  * `-l 0-1`: Uses Logical Cores 0 and 1 (1 Master, 1 Worker).
  * `-n 4`: Optimizes for 4 memory channels (adjust based on host hardware).
  * `-p 0x3`: Port Mask. `0x3` (Binary `11`) enables Port 0 and Port 1.
  * `--no-mac-updating`: **Crucial.** Prevents the forwarder from altering the destination MAC address. This ensures the packet acts as a "reflection" so TRex recognizes it upon return.

-----

## 2\. Configure the Traffic Generator (VM1: `dpdk-1`)

Cisco TRex requires a specific YAML configuration file (`/etc/trex_cfg.yaml`) to map the DPDK ports and define the destination MAC addresses of the DUT (VM2).

### A. Install TRex

If not already installed:

```bash
mkdir -p /opt/trex
cd /opt/trex
sudo wget --no-cache --no-check-certificate https://trex-tgn.cisco.com/trex/release/latest
sudo tar -xzvf latest
cd v3.08  # (Version number may vary)
```

### B. Configure Ports & MAC Addresses

TRex provides an interactive script to generate the configuration file.

1.  **Run the Setup Script:**

    ```bash
    cd /opt/trex/v3.08
    sudo ./dpdk_setup_ports.py -i
    ```

2.  **Follow the Prompts:**

      * **Type of interface:** Select `2` (DPDK).
      * **Enter PCI addresses:** Press `Enter` (it will auto-detect bound ports).
      * **Destination MAC:** When prompted, you must enter the **MAC addresses of VM2**.
          * *Interface 1 Dest:* Enter MAC of VM2 Port 0.
          * *Interface 2 Dest:* Enter MAC of VM2 Port 1.

    > **Note:** To find VM2's MAC addresses, run `ip link` on VM2 (before binding to DPDK) or check the Virtual Machine Manager hardware details.

3.  **Verify Configuration:**
    The generated file `/etc/trex_cfg.yaml` should look similar to this:

    ```yaml
    - port_limit: 2
      version: 2
      interfaces: ['04:00.0', '05:00.0'] # VM1 PCI Addresses
      port_info:
          - dest_mac: 52:54:00:9b:eb:54  # VM2 Port 0 MAC
            src_mac:  52:54:00:6b:e8:ad
          - dest_mac: 52:54:00:c7:b5:0f  # VM2 Port 1 MAC
            src_mac:  52:54:00:f0:6d:aa
    ```

### C. Start the TRex Server

The TRex server acts as the engine. It must be running in a separate terminal window or `tmux` session.

```bash
cd /opt/trex/v3.08
sudo ./t-rex-64 -i
```

*Wait for the message: **"Server is active"**.*