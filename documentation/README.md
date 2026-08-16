# Network Implementation Documentation: Hierarchical Enterprise Design

## 1. Executive Summary

This document outlines the architecture, configuration, and verification of a hierarchical enterprise network implemented in Cisco Packet Tracer.

The design follows the Cisco three-tier hierarchical model:

- Core Layer
- Distribution Layer
- Access Layer

The environment implements VLAN segmentation, Rapid PVST+, OSPF dynamic routing, centralized DHCP, DHCP relay, inter-VLAN routing, and PAT for external connectivity.


## 2. IP Addressing & VLAN Schema

The enterprise network uses private IPv4 addressing for internal networks and a simulated public network for external connectivity.

### VLAN Configuration

| VLAN ID | Name | Department | Subnet | Primary Gateway |
|--------:|------|------------|--------|-----------------|
| 10 | Sales | Sales/Staff | 10.10.0.0/24 | 10.10.0.1 |
| 20 | IT | IT Department | 10.20.0.0/24 | 10.20.0.1 |
| 30 | Admin | Administration | 10.30.0.0/24 | 10.30.0.1 |
| 90 | Servers | Server Network | 10.90.0.0/24 | 10.90.0.1 |
| 99 | Management | Network Management | 10.99.0.0/24 | 10.99.0.1 |

CORE-SW2 maintains the corresponding `.2` SVI address on each VLAN.

### Infrastructure Links

- **CORE-SW1 → CORE-R1:** `10.255.255.0/30`
  - CORE-R1: `10.255.255.1`
  - CORE-SW1: `10.255.255.2`

- **CORE-SW2 → CORE-R1:** `10.255.255.4/30`
  - CORE-R1: `10.255.255.5`
  - CORE-SW2: `10.255.255.6`

- **CORE-R1 → INTERNET-R1:** `203.0.113.0/30`
  - CORE-R1: `203.0.113.1`
  - INTERNET-R1: `203.0.113.2`

- **Simulated Internet Address:** `100.100.100.100/32`


## 3. Layer 2: Redundancy & Optimization

### Rapid PVST+ Implementation

Rapid Per-VLAN Spanning Tree Plus is enabled across the switching infrastructure to prevent Layer 2 loops while maintaining redundant paths.

### STP Load Balancing

- **CORE-SW1**
  - Primary root for VLANs `10`, `30`, and `90`
  - STP priority: `4096`

- **CORE-SW2**
  - Primary root for VLANs `20` and `99`
  - STP priority: `4096`

Each core switch acts as the secondary root for the VLANs primarily handled by the opposite core.

This allows VLAN traffic to be distributed across both core switches instead of forcing every VLAN to prefer the same Layer 2 path.

### Trunking

Inter-switch connections use IEEE 802.1Q trunking.

Allowed VLANs:

`10,20,30,90,99`

Access switches connect upward to redundant distribution paths, while distribution switches connect upward toward the core layer.


## 4. Layer 3: Routing Strategy

### Inter-VLAN Routing

Inter-VLAN routing is performed by Switched Virtual Interfaces (SVIs) on the multilayer core switches.

Examples:

- VLAN 10 → `10.10.0.1/24`
- VLAN 20 → `10.20.0.1/24`
- VLAN 30 → `10.30.0.1/24`
- VLAN 90 → `10.90.0.1/24`
- VLAN 99 → `10.99.0.1/24`

`ip routing` is enabled on both core switches.

### Dynamic Routing — OSPF

OSPF Process 1 provides dynamic Layer 3 routing between CORE-R1 and the two multilayer core switches.

All routing operates in **OSPF Area 0**.

#### Router IDs

- CORE-R1 → `1.1.1.1`
- CORE-SW1 → `2.2.2.2`
- CORE-SW2 → `3.3.3.3`

CORE-R1 advertises the default route into OSPF using:

`default-information originate always`

This allows the core switches to learn a route toward external networks through CORE-R1.


## 5. Network Services & Security

### DHCP

CORE-R1 operates as the centralized DHCP server.

DHCP pools are configured for:

- Sales — `10.10.0.0/24`
- IT — `10.20.0.0/24`
- Admin — `10.30.0.0/24`

The first five addresses in each DHCP subnet are excluded from dynamic allocation.

Example:

`10.10.0.1 - 10.10.0.5`

DNS supplied to DHCP clients:

`8.8.8.8`

### DHCP Relay

Because the DHCP server is located on CORE-R1 rather than directly inside the client VLANs, the core SVIs use `ip helper-address`.

CORE-SW1 forwards DHCP requests toward:

`10.255.255.1`

CORE-SW2 forwards DHCP requests toward:

`10.255.255.5`

This converts the client DHCP broadcast into routed traffic capable of reaching CORE-R1.

### NAT & PAT

CORE-R1 provides Port Address Translation (PAT/NAT Overload) for internal networks.

Internal interfaces:

- G0/0 → `ip nat inside`
- G0/1 → `ip nat inside`

External interface:

- G0/2 → `203.0.113.1`
- `ip nat outside`

The NAT ACL permits:

`10.0.0.0/8`

PAT translates internal private addresses to the CORE-R1 outside interface address:

`203.0.113.1`

This allows multiple internal hosts to share a single external IPv4 address.


## 6. External Connectivity

INTERNET-R1 simulates an upstream Internet provider.

The router uses:

- WAN address: `203.0.113.2/30`
- Simulated Internet loopback: `100.100.100.100/32`

Internal hosts use `100.100.100.100` as the external destination when validating routing and PAT functionality.


## 7. Verification Results

The network implementation was validated successfully.

### DHCP Verification

Sales, IT, and Admin client devices successfully received:

- IPv4 addresses
- Subnet masks
- Default gateways
- DNS information

from CORE-R1.

### Inter-VLAN Connectivity

Connectivity between routed VLAN networks was successfully verified through the multilayer core switches.

### OSPF Verification

OSPF neighbor relationships were verified using:

`show ip ospf neighbor`

Routing information was verified using:

`show ip route`

The routing tables contained connected networks, OSPF-learned routes, and the default route toward CORE-R1.

### External Connectivity

Internal client devices successfully reached the simulated Internet address:

`100.100.100.100`

Verification command:

`ping 100.100.100.100`

### PAT Verification

Active translations were confirmed on CORE-R1 using:

`show ip nat translations`

The output demonstrated private internal addresses being translated through the public-facing address:

`203.0.113.1`

### Spanning Tree Verification

Rapid PVST+ operation and VLAN root-bridge selection were verified using:

`show spanning-tree`

The results confirmed the intended per-VLAN STP load-balancing design.


## 8. Implementation Summary

The completed enterprise network demonstrates:

- Three-tier hierarchical network architecture
- VLAN-based departmental segmentation
- IEEE 802.1Q trunking
- Rapid PVST+
- Per-VLAN STP load balancing
- Multilayer switching
- Inter-VLAN routing using SVIs
- OSPF Area 0 dynamic routing
- Centralized DHCP services
- DHCP relay
- NAT/PAT overload
- Simulated external Internet connectivity
- LLDP device discovery
- End-to-end connectivity verification
