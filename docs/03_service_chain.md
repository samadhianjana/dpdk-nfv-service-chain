# 03\. Configuring the Service Chain

This guide details the configuration of the Traffic Generator (Cisco TRex) and the customization of the Traffic Forwarder (DPDK L2fwd) to support a specific "Snake" topology (Port 0 -\> Port 1).

## 1\. Configure the Traffic Forwarder (VM2: `dpdk-2`)

The forwarder acts as the "Device Under Test" (DUT). We will modify the standard `l2fwd` example to act as a **Smart Forwarder**, explicitly rewriting destination addresses to ensure the return traffic is accepted by the Generator.

### A. Modifying the Source Code

We need to hardcode the Destination MAC address of the Generator's receiving port (TRex Port 1) into the forwarding logic.

1.  **Open the source file:**

    ```bash
    nano ~/dpdk-25.11/examples/l2fwd/main.c
    ```

2.  **Locate the Mac Updating Logic:**
    Search for the function `l2fwd_mac_updating` (Press `Ctrl+W` -\> type `l2fwd_mac_updating`).

3.  **Replace the function:**
    Delete the existing function body and paste the following code.
    *(Note: The MAC address `52:54:00:f0:6d:aa` below corresponds to TRex Port 1. Update this if your specific VM MAC differs).*

    ```c
    static void
    l2fwd_mac_updating(struct rte_mbuf *m, unsigned dest_portid)
    {
        struct rte_ether_hdr *eth;
        eth = rte_pktmbuf_mtod(m, struct rte_ether_hdr *);

        /* Update Source Addr: Set it to VM2's Own MAC */
        /* This tells TRex that the packet came from VM2 */
        rte_ether_addr_copy(&l2fwd_ports_eth_addr[dest_portid], &eth->src_addr);

        /* Update Dest Addr: HARDCODE to TRex Port 1 */
        /* This ensures TRex Port 1 accepts the packet naturally */
        struct rte_ether_addr target_mac = {{0x52, 0x54, 0x00, 0xf0, 0x6d, 0xaa}};
        rte_ether_addr_copy(&target_mac, &eth->dst_addr);
    }
    ```

4.  **Recompile the Application:**
    Since we changed the code, we must rebuild the binary.

    ```bash
    cd ~/dpdk-25.11
    meson configure build -Dexamples=l2fwd
    ninja -C build
    ```

### B. Running the Application

Run the modified application. **Do NOT** use the `--no-mac-updating` flag, as we want our custom function to execute.

```bash
# Point to the newly built binary
sudo ./build/examples/l2fwd/dpdk-l2fwd -l 0-1 -n 4 -- -p 0x3 -T 1
```

  * **`-p 0x3`**: Enables Port 0 and Port 1.
  * **Logic:** The app will receive packets on Port 0 and automatically forward them to Port 1 (and vice versa), applying our new MAC address fix to every packet.

-----

## 2\. Configure the Traffic Generator (VM1: `dpdk-1`)

TRex requires a configuration file to map the DPDK ports.

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