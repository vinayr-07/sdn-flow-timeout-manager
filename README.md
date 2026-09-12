# SDN Flow Timeout Manager

> An SDN controller application for managing OpenFlow flow-rule timeouts using Ryu, OpenFlow 1.3, and Mininet.

This project demonstrates timeout-based flow-rule lifecycle management in a Software Defined Network. It uses a Ryu controller and a Mininet/Open vSwitch topology to observe how forwarding rules are installed, monitored, and removed through idle and hard timeouts.

The project also includes a blocked-host policy, flow-table monitoring, throughput testing, and a regression scenario that verifies forwarding rules can be installed again after timeout expiry.

## Overview

The system is built around a simple SDN topology:

```text
                    ┌─────────────────┐
                    │  Ryu Controller │
                    │  OpenFlow 1.3   │
                    └────────┬────────┘
                             │
                             │ OpenFlow
                             │
                       ┌─────▼─────┐
                       │    s1     │
                       │ OVS Switch│
                       └─┬──┬──┬──┬┘
                         │  │  │  │
                        h1 h2 h3 h4
                       .1 .2 .3 .4

                       h4 = blocked
```

The controller manages flow rules with:

- **Idle timeout:** 10 seconds
- **Hard timeout:** 30 seconds
- **Blocked host:** `10.0.0.4`

## Key Features

- OpenFlow 1.3 controller using Ryu
- Mininet topology with 4 hosts and 1 Open vSwitch
- Idle-timeout based flow expiration
- Hard-timeout based flow expiration
- Flow-table monitoring
- Blocked-host DROP rule
- Flow reinstallation after timeout expiry
- Throughput testing with `iperf`
- Regression testing across repeated runs
- Controller-side logging of flow lifecycle events

## Network Topology

```text
                 [Ryu Controller]
                        |
                    OpenFlow
                        |
                    [s1 OVS]
                 /     |     |     \
               h1      h2    h3      h4
           10.0.0.1  .0.2   .0.3   .0.4
           allowed   allowed allowed BLOCKED
```

### Topology Details

| Component | Configuration |
|---|---|
| Controller | Ryu |
| Protocol | OpenFlow 1.3 |
| Emulator | Mininet |
| Switch | Open vSwitch |
| Hosts | 4 |
| Controller Address | `127.0.0.1:6633` |
| Link Bandwidth | 10 Mbps |
| Idle Timeout | 10 seconds |
| Hard Timeout | 30 seconds |
| Blocked Host | `10.0.0.4` |

## Flow Rule Lifecycle

```text
Packet Arrives
      │
      ▼
Controller Installs Flow
      │
      ├── idle_timeout = 10s
      │
      └── hard_timeout = 30s
      │
      ▼
Switch Forwards Traffic
      │
      ├── No matching traffic for 10s
      │        │
      │        ▼
      │   IDLE_TIMEOUT
      │        │
      │        ▼
      │    Rule Removed
      │
      └── Rule reaches 30s
               │
               ▼
          HARD_TIMEOUT
               │
               ▼
           Rule Removed
```

## Key OpenFlow Concepts

### Idle Timeout

`idle_timeout` removes a flow when no matching packets have been received for the configured period.

In this project:

```text
IDLE_TIMEOUT = 10 seconds
```

This allows inactive forwarding rules to be removed automatically.

### Hard Timeout

`hard_timeout` removes a flow after the configured lifetime regardless of whether traffic is still being received.

In this project:

```text
HARD_TIMEOUT = 30 seconds
```

This provides an upper bound on how long a dynamically installed rule remains active.

### Flow Removal

OpenFlow can notify the controller when a flow is removed. A flow-removal event can include the removal reason, duration, packet count, and byte count, which makes it useful for observing rule lifecycle behavior.

## Project Structure

```text
sdn-flow-timeout-manager/
├── README.md
├── docs/
│   └── Flow_Rule_Timeout_Manager_Report.pdf
├── src/
│   ├── timeout_manager.py
│   └── topology.py
└── screenshots/
    ├── T1-SS1-controller-started.png
    ├── T1-SS2-blocked-flow-installed.png
    ├── T1-SS3-flow-expired-timeout.png
    ├── T2-SS1-scenario1&2.png
    └── T2-SS2-scenario5-regression-test.png
```

## Tech Stack

- **Controller:** Ryu
- **Protocol:** OpenFlow 1.3
- **Network Emulator:** Mininet
- **Switch:** Open vSwitch
- **Language:** Python
- **Environment:** Ubuntu 22.04
- **Virtualization:** VMware
- **Testing:** Mininet CLI, `ping`, `iperf`, flow-table inspection

## Setup

### Prerequisites

Ubuntu 22.04 with Python 3.11, Mininet, and a working Open vSwitch installation.

Install the required packages:

```bash
sudo apt update
sudo apt install -y python3.11 python3.11-venv mininet
```

### Install Ryu

Create a Python virtual environment:

```bash
mkdir ~/ryu
cd ~/ryu

python3.11 -m venv venv
source venv/bin/activate

pip install --upgrade pip
pip install setuptools==67.8.0 wheel
pip install ryu eventlet==0.35.2
```

### Clone the Project

```bash
git clone https://github.com/vinayr-07/sdn-flow-timeout-manager.git
cd sdn-flow-timeout-manager
```

If the repository is hosted under a different name after renaming, update the URL accordingly.

## Running the Project

> Always clean stale Mininet state before starting a new run.

### 1. Clean Up

```bash
sudo mn -c
```

### 2. Start the Ryu Controller

Open Terminal 1:

```bash
cd ~/ryu
source venv/bin/activate

ryu-manager /path/to/sdn-flow-timeout-manager/src/timeout_manager.py --observe-links
```

Expected startup output should include the configured timeout values and blocked host.

### 3. Start the Mininet Topology

Open Terminal 2:

```bash
source ~/ryu/venv/bin/activate
sudo python3 /path/to/sdn-flow-timeout-manager/src/topology.py
```

## Test Scenarios

| Scenario | Test | Expected Result |
|---|---|---|
| 1 | `h1 ping h2` | `0% packet loss` |
| 2 | `h4 ping h1` | `100% packet loss` |
| 3 | `h1 iperf h3` | Approximately 9–10 Mbits/sec |
| 4 | Wait for idle timeout | Flow count decreases as inactive rules expire |
| 5 | Repeat `h1 ping h2` after timeout | Flow rules are installed again and ping succeeds |

### Manual Tests

Inside the Mininet CLI:

```bash
mininet> h1 ping -c 4 10.0.0.2
mininet> h4 ping -c 4 -W 1 10.0.0.1
mininet> h1 iperf h3
```

After allowing the idle timeout to expire:

```bash
mininet> h1 ping -c 4 10.0.0.2
```

The final test verifies that forwarding rules can be installed again after previous rules have expired.

## Observed Results

### Normal Forwarding

```text
h1 → h2
0% packet loss
```

### Blocked Host

```text
h4 → h1
100% packet loss
```

Traffic from `10.0.0.4` is denied by the controller-installed DROP rule.

### Throughput

The documented topology produced approximately:

```text
9–10 Mbits/sec
```

for the configured `iperf` test.

### Timeout Behavior

The flow table was observed to reduce from approximately:

```text
7 → 3 → 1 active flows
```

as inactive forwarding rules expired.

### Regression

After timeout expiry, repeating the forwarding test caused the required flow rules to be installed again and traffic succeeded:

```text
h1 → h2
0% packet loss
```

This verifies that rule expiration does not permanently disrupt forwarding.

## Controller Behavior

The controller is responsible for:

1. Establishing the OpenFlow connection
2. Installing forwarding rules
3. Applying idle and hard timeout values
4. Installing the DROP rule for the blocked host
5. Monitoring flow-table state
6. Reporting flow lifecycle information through logs

Example controller output:

```text
[FLOW INSTALLED]  dpid=1 priority=1 idle=10s hard=30s
[BLOCKED]         Packet from 10.0.0.4 to 10.0.0.1 — DROP rule installed.
[FLOW TABLE]      Switch dpid=1 - 7 active flow(s)
[FLOW TABLE]      Switch dpid=1 - 1 active flow(s)
[FLOW INSTALLED]  dpid=1 priority=1 idle=10s hard=30s total=13
```

## Design Decisions

### Idle + Hard Timeouts

Both timeout types are configured so the project can demonstrate two distinct expiration policies:

- `idle_timeout` handles inactive flows
- `hard_timeout` limits the maximum lifetime of a rule

### Controller-Managed Blocking

The blocked host is enforced through a DROP rule instead of relying on host-side configuration. This demonstrates how an SDN controller can centrally enforce network policy at the switch.

### Mininet + Open vSwitch

Mininet provides a reproducible virtual topology while Open vSwitch provides the OpenFlow-capable switching layer required by the controller.

## Limitations

This project is a focused SDN experiment rather than a production traffic-management system.

Current limitations include:

- Single-switch topology
- Fixed timeout values
- Static blocked-host configuration
- Limited network scale
- No persistent controller state
- No distributed controller architecture
- No dynamic policy configuration interface
- Limited flow-removal automation beyond the demonstrated lifecycle

## Future Improvements

Possible extensions include:

- Dynamic timeout configuration
- Multiple switches and larger topologies
- Event-driven flow-removal handling
- REST API for policy management
- Dynamic blocked-host rules
- Flow statistics dashboards
- Automated test orchestration
- Controller clustering and high availability
- More detailed packet and byte-level analytics

## Screenshots

### Controller Startup

![Controller startup](screenshots/T1-SS1-controller-started.png)

### Blocked Flow Installation

![Blocked flow](screenshots/T1-SS2-blocked-flow-installed.png)

### Flow Expiration

![Flow timeout](screenshots/T1-SS3-flow-expired-timeout.png)

### Forwarding and Blocking Tests

![Scenario tests](screenshots/T2-SS1-scenario1&2.png)

### Regression Test

![Regression test](screenshots/T2-SS2-scenario5-regression-test.png)

## Documentation

The detailed project report is available here:

[Flow Rule Timeout Manager Report](docs/Flow_Rule_Timeout_Manager_Report.pdf)

## License

MIT License.

## Author

**Vinay R**

Computer Science Engineering Student  
Interested in networking, systems, and software engineering.
