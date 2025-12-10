# 05\. Advanced Forwarding: Using Eventdev & Pipeline

This guide explains how to switch from the standard **Polling Mode** forwarder (`l2fwd`) to **Event-Driven** forwarding. This demonstrates DPDK's Eventdev library, which uses dynamic scheduling to balance packet load across cores, simulating modern high-scale cloud networking.

## 1\. Overview: Polling vs. Eventdev

| Feature | Standard `l2fwd` | `l2fwd-event` / `pipeline` |
| :--- | :--- | :--- |
| **Model** | **Polling:** Cores are hard-assigned to specific ports. | **Event-Driven:** A Scheduler assigns packets ("events") to available cores. |
| **Load Balancing** | **Static:** If Port 0 is flooded, Core 0 is overwhelmed while Core 1 sits idle. | **Dynamic:** If Port 0 is flooded, the Scheduler can assign *all* cores to help process it. |
| **Architecture** | Simple Loop (Rx -\> Tx) | Pipeline (Rx -\> Scheduler -\> Worker -\> Tx) |
| **Driver Used** | `poll_mode` | `event_sw` (Software Scheduler) |

-----

## 2\. Method 1: The Automated Way (`l2fwd-event`)

This application is a drop-in replacement for `l2fwd` that automatically manages core assignments (Rx, Tx, Worker) behind the scenes.

### A. Code Modification (Hardcoded Snake Topology)

By default, this app swaps MAC addresses, which causes TRex to drop returning packets. We will modify the header file to force the destination MAC to **TRex Port 1**.

1.  **Edit the Header File:**

    ```bash
    nano ~/dpdk-25.11/examples/l2fwd-event/l2fwd_common.h
    ```

2.  **Replace File Content:** Delete the entire file content and replace it with the following. This adds the `extern` declaration and the hardcoded MAC logic.

    ```c
    #ifndef _L2FWD_COMMON_H_
    #define _L2FWD_COMMON_H_

    #include <rte_common.h>
    #include <rte_mbuf.h>
    #include <rte_ether.h>

    static inline void
    l2fwd_mac_updating(struct rte_mbuf *m, unsigned int dest_portid, struct rte_ether_addr *addr)
    {
        struct rte_ether_hdr *eth;
        eth = rte_pktmbuf_mtod(m, struct rte_ether_hdr *);

        /* 1. Source MAC: Use the address passed by the caller */
        rte_ether_addr_copy(addr, &eth->src_addr);

        /* 2. Destination MAC: FORCE TO TREX PORT 1 */
        /* REPLACE with your TRex Port 1 MAC: 52:54:00:f0:6d:aa */
        struct rte_ether_addr target_mac = {{0x52, 0x54, 0x00, 0xf0, 0x6d, 0xaa}};
        rte_ether_addr_copy(&target_mac, &eth->dst_addr);

        (void)dest_portid; /* unused */
    }
    ```

### B. Compilation

```bash
cd ~/dpdk-25.11
rm -rf build/examples/l2fwd-event # Clean build
meson configure build -Dexamples=l2fwd-event
ninja -C build
```

### C. Running the Application

We use the Software Event Driver (`event_sw0`) and dedicate Core 3 (`-s 0x8`) to the Scheduler.

```bash
sudo ./build/examples/dpdk-l2fwd-event \
-l 0-3 -s 0x8 -n 4 --vdev event_sw0 -- \
-p 0x3 --mode=eventdev --eventq-sched=atomic
```

-----

## 3\. Method 2: The Advanced Way (`eventdev_pipeline`)

This application allows manual assignment of specific roles (Rx, Tx, Worker, Scheduler) to cores.

### A. Code Modification (Hardcoded Snake Topology)

By default, this app forwards packets without modifying MACs. We will modify the Worker logic to force a **Port Swap** and **MAC Rewrite**.

1.  **Edit the Worker File:**

    ```bash
    nano ~/dpdk-25.11/examples/eventdev_pipeline/pipeline_worker_generic.c
    ```

2.  **Locate `worker_generic_burst`:** Find the block `if (events[i].queue_id == cdata.qid[0])`.

3.  **Insert Logic:** Paste this code inside that `if` block (after `txq_set`):

    ```c
    /* ... inside if (events[i].queue_id == cdata.qid[0]) ... */

    /* 1. Port Swap: Force 0->1 and 1->0 */
    events[i].mbuf->port = events[i].mbuf->port ^ 1;

    /* 2. MAC Rewrite: Force destination to TRex Port 1 */
    struct rte_ether_hdr *eth = rte_pktmbuf_mtod(events[i].mbuf, struct rte_ether_hdr *);

    /* REPLACE with your TRex Port 1 MAC */
    struct rte_ether_addr target_mac = {{0x52, 0x54, 0x00, 0xf0, 0x6d, 0xaa}};
    rte_ether_addr_copy(&target_mac, &eth->dst_addr);
    ```

### B. Compilation

```bash
cd ~/dpdk-25.11
meson configure build -Dexamples=eventdev_pipeline
ninja -C build
```

### C. Running the Application

Assign roles: Rx(Core 0), Tx(Core 1), Sched(Core 2), Worker(Core 3).

```bash
sudo ./build/examples/dpdk-eventdev_pipeline \
-l 0-3 -n 4 --vdev event_sw0 -- \
-r 0x1 -t 0x2 -e 0x4 -w 0x8 -s 1 -n 0
```

-----

## 4\. Traffic Generator Configuration (TRex)

If you have applied the **Code Modifications** (Step A) above, you do **not** need Promiscuous Mode. TRex will accept the packets naturally because they address Port 1 directly.

**On VM1 (TRex Console):**

```bash
# Start Traffic (Unidirectional Snake Flow)
# -p 0: Forces traffic to start from Port 0 only
start -f stl/bench.py -m 100mbps -p 0
```

**Verification:**
Open the Dashboard (`tui`). You should see:

  * **Tx** on Port 0.
  * **Rx** on Port 1.
  * **Zero Drop Rate** (0.00 bps).
  * **Zero Reflection** (0 Rx on Port 0).

## 5\. Performance Note

Throughput using `event_sw` (Software Scheduler) is typically **lower** than standard polling (`l2fwd`) because moving packets between cores in software adds significant CPU overhead. This architecture is designed for **Hardware Offloading** (SmartNICs), where the scheduling is done by a physical chip.
