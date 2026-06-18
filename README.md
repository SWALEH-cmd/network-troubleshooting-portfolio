# Case ID: NET-VPC-001 - 3-Tier Network Isolation Layout

## Situation
A financial client deploying a standard 3-tier application requires complete network segregation. Their public-facing web infrastructure must be accessible, but their Database Server containing proprietary data must remain strictly isolated inside a private subnet tier with zero direct exposure to the public internet.

---

## Task
My objective as the Cloud Support Associate was to configure distinct public and private IP subnet spaces within a VMware workstation virtual private environment, manually provision network interfaces, and run diagnostic validation testing to verify complete external isolation of the DB host.

---

## Action & Verification Steps

### 1. Configure the Public Gateway Interface
On the Web Router node, I mapped the internal network interface and assigned the gateway address:

sudo ip addr add 10.0.1.1/24 dev ens33
sudo ip link set ens33 up
ip addr show ens33

VERIFICATION SCREENSHOT #1: PUBLIC IP ALLOCATION
Below is the proof showing that interface ens33 is successfully holding the public gateway IP configuration:

![Public Subnet Configuration](public_subnet_ip.png)

---

### 2. Provision and Isolate the Private Database Host
On the dedicated Database Node, I assigned a private tier IP address and stripped away its default public gateway route to enforce strict network-level isolation:

sudo ip addr add 10.0.2.50/24 dev ens33
sudo ip link set ens33 up
sudo ip route del default
ip route show

VERIFICATION SCREENSHOT #2: CLEAN ISOLATED ROUTING TABLE
Below is the proof showing the DB routing table is stripped clean with no default gateway out to the internet:

![Isolated DB Routing Table](db_isolated_routing.png)

---

### 3. Execute Isolation Validation Testing
From a public network segment, I executed an ICMP echo ping directly targeting the private DB server to test boundary resilience:

ping -c 3 10.0.2.50

VERIFICATION SCREENSHOT #3: ISOLATION PROBE FAILURE
Below is the proof showing 100% packet loss when attempting to access the private database server from outside the network block:

![Isolation Test Probe](isolation_test_fail.png)

---

## Result & Customer Resolution
The network architecture was successfully updated to a highly secure topology. The Web layer is securely positioned inside the 10.0.1.0/24 block, while the Database server is completely isolated inside the 10.0.2.0/24 block. Verification probes confirmed 100% packet dropping from public sources. Direct external attack avenues targeting the data tier have been completely eliminated, resolving the client's security compliance audit.
