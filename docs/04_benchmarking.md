# 04\. Running Benchmarks

This guide details how to execute performance benchmarks using the TRex Console. It covers basic connectivity checks, throughput stress testing, and interpreting the dashboard statistics.

## 1\. Prerequisites

Before running these commands, ensure the "Engine" is running on both VMs:

  * **VM2 (Forwarder):** `l2fwd` must be running and waiting for packets.
  * **VM1 (Generator):** The TRex Server (`./t-rex-64 -i`) must be running in a background terminal.

## 2\. Accessing the TRex Console

All benchmarking is controlled via the TRex Console, which acts as a remote control for the server.

Open a **new terminal** on VM1 and run:

```bash
cd /opt/trex/v3.08
./trex-console
```

*You should see the `trex>` prompt.*

## 3\. Test 1: Connectivity Verification (The "Ping" Test)

Before pushing high speed, we verify the loop is complete by sending a very slow stream (1 packet per second).

**Command:**

```bash
start -f stl/bench.py -m 1pps
```

**Validation:**

1.  Type `tui` to open the Dashboard.
2.  Look at the **Global Statistics** (Top Left).
      * **Total-Tx:** \~64 bps (Transmit)
      * **Total-Rx:** \~64 bps (Receive)
3.  **Result:** If Tx matches Rx, your service chain is connected and functional.

*Press `Esc` to exit the TUI, then type `stop` to end the test.*

## 4\. Test 2: Fixed Bandwidth (Throughput)

This test forces a specific amount of bandwidth through the chain to see if the Forwarder can handle it without drops.

**Command (100 Mbps):**

```bash
start -f stl/bench.py -m 100mbps
```

**Command (Internet Mix / IMIX):**
Instead of sending fixed-size packets, IMIX sends a realistic blend of small (64B), medium, and large (1500B) packets. This is a harder stress test for the CPU.

```bash
start -f stl/imix.py -m 10kpps
```

## 5\. Test 3: Stress Test (Finding the Limit)

To find the Maximum Zero Packet Loss (MZPL) throughput of your virtual testbed, push the generator to 100% and watch for drops.

**Command:**

```bash
start -f stl/bench.py -m 100%
```

**Interpreting the Dashboard (TUI):**

| Metric | Status | Meaning |
| :--- | :--- | :--- |
| **Total-Tx** | Increasing | Generator is pushing packets out. |
| **Total-Rx** | Increasing | Packets are successfully returning. |
| **Drop-Rate** | **RED / \> 0** | **Congestion Detected.** The Forwarder (VM2) cannot handle the speed, or the virtual bridge is saturated. |
| **Queue Full** | Yes | TRex itself is generating faster than the NIC driver can transmit. |

## 6\. Traffic Profiles Explained

  * **`stl/bench.py`**: Sends 64-byte UDP packets. This is the standard "worst-case" test for routers because small packets require the most CPU processing power per second.
  * **`stl/imix.py`**: Simulates real-world traffic (IMIX = Internet Mix). Use this for realistic performance claims.
  * **`stl/udp_1pkt.py`**: A simple single-packet stream, useful for debugging specific flow issues.

## 7\. Graceful Shutdown

To finish your lab session correctly:

1.  Stop traffic in the console: `stop`
2.  Exit the console: `exit`
3.  Stop the TRex Server (in the background window): Press `Ctrl+C`.
4.  Stop the L2fwd App (on VM2): Press `Ctrl+C`.

> **Note:** If you plan to use these VMs for standard networking (SSH/Ping) afterwards, you must unbind the ports from DPDK and return them to Linux:
> `sudo dpdk-devbind.py -b virtio-pci 00:04.0 00:05.0`