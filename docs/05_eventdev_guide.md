# 05\. Advanced Forwarding: Using Eventdev & Pipeline

This guide explains how to switch from the standard **Polling Mode** forwarder (`l2fwd`) to **Event-Driven** forwarding. This demonstrates DPDK's Eventdev library, which uses dynamic scheduling to balance packet load across cores, simulating modern high-scale cloud networking.

## 1\. Overview: Polling vs. Eventdev

| Feature | Standard `l2fwd` | `l2fwd-event` / `pipeline` |
| :--- | :--- | :--- |
| **Model** | **Polling:** Cores are hard-assigned to specific ports. | **Event-Driven:** A Scheduler assigns packets ("events") to available cores. |
| **Load Balancing** | **Static:** If Port 0 is flooded, Core 0 is overwhelmed while Core 1 sits idle. | **Dynamic:** If Port 0 is flooded, the Scheduler can assign *all* cores to help process it. |
| **Architecture** | Simple Loop (Rx -\> Tx) | Pipeline (Rx -\> Scheduler -\> Worker -\> Tx) |
| **Driver Used** | `poll_mode` | `event_sw` (Software Scheduler) |

## 2\. Method 1: The Automated Way (`l2fwd-event`)

This application is a drop-in replacement for `l2fwd` that automatically manages core assignments (Rx, Tx, Worker) behind the scenes. It is the easiest way to start with Eventdev.

### A. Compilation

You must explicitly configure the build target for this example.

**Run on VM2:**

```bash
cd ~/dpdk-25.11
meson configure build -Dexamples=l2fwd-event
ninja -C build
```

### B. Running the Application

We must use the **Software Event Driver** (`event_sw0`) and dedicate one core to be the "Scheduler" service.

**Run on VM2:**

```bash
sudo ./build/examples/dpdk-l2fwd-event \
-l 0-3 -s 0x8 -n 4 --vdev event_sw0 -- \
-p 0x3 --mode=eventdev --eventq-sched=atomic
```

  * **`-s 0x8`**: Dedicates Core 3 (Binary `1000`) as the Service Core. This core *only* schedules packets; it does not process them.
  * **`-l 0-3`**: Uses Cores 0, 1, and 2 as Workers.

-----

## 3\. Method 2: The Advanced Way (`eventdev_pipeline`)

This application allows you to manually assign specific roles (Rx, Tx, Worker, Scheduler) to each CPU core via command-line flags. It offers granular control over the pipeline stages.

### A. Compilation

**Run on VM2:**

```bash
cd ~/dpdk-25.11
meson configure build -Dexamples=eventdev_pipeline
ninja -C build
```

### B. Running the Application

Assign roles using bitmasks. We use a **Software Scheduler** (`event_sw0`) again.

**Run on VM2:**

```bash
sudo ./build/examples/dpdk-eventdev_pipeline \
-l 0-3 -n 4 --vdev event_sw0 -- \
-r 0x1 -t 0x2 -e 0x4 -w 0x8 -s 1 -n 0
```

  * **`-r 0x1`**: Core 0 is Rx Adapter (Receive).
  * **`-t 0x2`**: Core 1 is Tx Adapter (Transmit).
  * **`-e 0x4`**: Core 2 is Scheduler.
  * **`-w 0x8`**: Core 3 is Worker.

-----

## 4\. Traffic Generator Configuration (TRex) - **Crucial Step**

Because these standard examples do not rewrite MAC addresses (unlike our custom `l2fwd`), they will forward packets with their *original* destination MACs. TRex receives these packets on Port 1, sees that the destination MAC does not match its own Port 1 address, and drops them.

**The Fix: Enable Promiscuous Mode**
This forces the TRex NICs to accept *all* incoming traffic, regardless of the MAC address.

**On VM1 (TRex Console):**

```bash
# 1. Enable Promiscuous Mode on All Ports
portattr -a --prom on

# 2. Start Traffic (Unidirectional Snake Flow)
start -f stl/bench.py -m 100mbps -p 0
```

**Verification:**
Open the Dashboard (`tui`). You should see:

  * **Tx** on Port 0.
  * **Rx** on Port 1.
  * **Zero Drop Rate** (or very close to zero).

## 5\. Performance Note

Throughput using `event_sw` (Software Scheduler) is typically **lower** than standard polling (`l2fwd`) because moving packets between cores in software adds significant CPU overhead. This architecture is designed for **Hardware Offloading** (SmartNICs), where the scheduling is done by a physical chip.