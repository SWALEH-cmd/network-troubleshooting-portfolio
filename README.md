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



---

# Case ID: NET-SEC-003 - Multi-Tier Firewall Protection (NACLs vs Security Groups)

## Situation
An enterprise application deployment features a public-facing network tier and a sensitive backend database tier. Without network isolation, the Database Server (`192.168.10.128`) was completely exposed within the virtualized architecture, creating an insecure environment vulnerable to unauthorized internal lateral movement and external network scans.

---

## Task
My objective as the Cloud Support Associate was to apply a Defense-in-Depth security posture to completely isolate that database server by deploying a stateless perimeter Network ACL on the central routing gateway and a stateful Security Group directly on the database compute host.

---

## Action & Verification Steps

### 1. Configure the Stateless Subnet Policies (NACL)
On the Jump Box Gateway Router node, I deployed stateless `iptables` forwarding rules to drop global external traffic targeting the private environment while explicitly routing allowed inter-subnet traffic:

sudo iptables -F
sudo iptables -A FORWARD -s 0.0.0.0/0 -d 192.168.10.128 -j DROP
sudo iptables -A FORWARD -s 192.168.10.129 -d 192.168.10.128 -p tcp --dport 3306 -j ACCEPT
sudo iptables -A FORWARD -s 192.168.10.128 -d 192.168.10.129 -p tcp --sport 3306 -j ACCEPT
sudo iptables -L -n -v

VERIFICATION SCREENSHOT #1: STATELESS FORWARDING TABLE
Below is the proof showing the active stateless forwarding rules on the Jump Box Gateway dropping external traffic while permitting the Web subnet:

![NACL Rules](nacl-iptables-rules.png)

---

### 2. Configure Stateful Host Protection (Security Group)
On the dedicated Database Node, I initialized the Uncomplicated Firewall (`ufw`) to create an instance-level firewall that drops all general ingress while opening a stateful trust window for the production Web Server IP on the MySQL port:

sudo ufw default deny inbound
sudo ufw default allow outbound
sudo ufw allow from 192.168.10.129 to any port 3306 proto tcp
sudo ufw enable
sudo ufw status verbose

VERIFICATION SCREENSHOT #2: STATEFUL SECURITY GROUP STATUS
Below is the proof showing the UFW status on the Database Server explicitly restricting ingress to the Web Server IP on port 3306:

![Security Group Status](security-group-status.png)

---

### 3. Execute Authorized Path Verification
From the authorized Web Server environment (`192.168.10.129`), I ran a network socket probe against the database target to verify that legitimate production traffic successfully clears both firewall layers:

nc -zv 192.168.10.128 3306

VERIFICATION SCREENSHOT #3: PRODUCTION ROUTE VALIDATION
Below is the proof showing a successful network connection from the Web Server to the Database Server on port 3306:

![Web to DB Success](web-to-db-success.png)

---

### 4. Execute Unauthorized Perimeter Verification
From the upstream Jump Box segment outside the allowed application layer, I executed the same network socket probe targeting the database instance to test boundary resistance:

nc -zv 192.168.10.128 3306

VERIFICATION SCREENSHOT #4: PERIMETER DROPPED PROBE
Below is the proof showing the connection timing out and being dropped by the perimeter firewall when trying to connect from the Jump Box:

![External Blocked](external-to-db-blocked.png)

---

## Result & Customer Resolution
The network security architecture was successfully upgraded to an enterprise-standard multi-layer defense topology. By combining stateless subnet constraints on the gateway with stateful instance rules on the database host, lateral discovery paths were eliminated. Verification audits confirmed that authorized application queries pass seamlessly, while unauthorized perimeter probes are fully dropped before reaching the database stack.

---

