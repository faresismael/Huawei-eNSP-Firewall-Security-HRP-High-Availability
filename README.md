# Huawei eNSP Firewall Security — Dual-Firewall HRP High Availability Cluster

## Overview

This project is an enterprise-grade network security simulation built on the Huawei eNSP (Enterprise Network Simulation Platform). It demonstrates the practical design and implementation of a secure, highly available corporate network infrastructure, integrating routing, switching, and advanced firewall protection.

The network architecture is carefully segmented using VLANs (10, 20, 30, and 40) to logically isolate internal users, corporate servers, and DMZ resources. At the core of the security perimeter is a redundant Huawei USG firewall cluster configured in Active/Standby mode. By utilizing HRP (Huawei Redundancy Protocol) alongside VRRP, the design ensures continuous session synchronization and network reliability during simulated link or device failures.

In addition to internal segmentation, the project implements Easy-IP NAT to securely translate private internal IP addresses for external communication. Strict Security Policies are enforced across Trust, Untrust, and DMZ zones to govern traffic flow.

To validate the design, the simulation is bridged to a local Cloud interface, allowing for live packet capture and analysis using Wireshark. This practical testing verifies NAT source IP translation, analyzes ICMP routing paths (traceroute security), and confirms the seamless operation of the HRP failover mechanism.

## Objectives

- **Implement Security Zones** — segment traffic into Untrust, DMZ, and Trust zones with explicit, least-privilege inter-zone policies.
- **Deploy True Firewall Redundancy** — run two Huawei USG firewalls in Active/Standby HRP mode, unified with VRRP + VGMP for atomic multi-interface failover.
- **Design Routing for High Availability** — anchor every static route and gateway to the VRRP virtual IP rather than a physical firewall address, so failover is actually transparent to the rest of the network.
- **Enable Secure Outbound Access** — implement Easy-IP NAT so internal hosts reach the internet without exposing private addressing.
- **Validate Everything, Not Just Configure It** — confirm zone enforcement, NAT translation, and failover behavior using live traffic capture and controlled failure testing.

## Network Topology

![Network Topology](https://github.com/faresismael/Huawei-eNSP-Firewall-Security-HRP-High-Availability-Cluster/blob/main/Screenshots/topology.png.png?raw=true)

| Zone | Devices | Subnet(s) |
| :--- | :--- | :--- |
| **Untrust Zone** | ISP Router, Internet (via Cloud1) | `203.0.113.0/24` |
| **DMZ Zone** | Web Server | `10.10.30.0/24` |
| **Trust Zone** | PC1–PC3 (VLAN 10), Server1–Server3 (VLAN 20) | `10.10.10.0/24`, `10.10.20.0/24` |
| **Firewall Cluster** | HQ-FW1 (Active) & HQ-FW2 (Standby) | HRP heartbeat: `10.10.40.0/24` |

### Full IP Addressing

![IP Addressing Table](images/ip-addressing-table.png)

## Technical Features & Implementation

- **HRP + VRRP + VGMP** — HQ-FW1/FW2 sync state over a dedicated heartbeat link; VGMP ties three VRRP virtual gateways (WAN, LAN-transit, DMZ) to the HRP Active/Standby state so all three fail over as one atomic event.
- **Security Zones & Policies** — Untrust, DMZ, and Trust zones with explicit inter-zone policies, no implicit permit rules.
- **VIP-Anchored Routing** — every static route and gateway points to the VRRP virtual IP, not a physical firewall address, so failover stays transparent.
- **VLAN Segmentation** — VLAN 10 (users) and VLAN 20 (servers), routed via `Vlanif` interfaces on the Core Switch.
- **Easy-IP NAT** — dynamic source NAT for outbound internet access via an explicit NAT policy.
- **Real-World Bridging (bonus)** — Cloud1 bridges the lab to a real Windows NIC, validated with bidirectional ping.

## Verification & Testing

Live command output and packet captures were used to confirm each part of the design actually behaves as intended — not just "it pings," but the specific mechanism behind each feature.

✅ **Master Firewall — Active State**
`display vrrp brief` on HQ-FW1 shows all three VRRP groups (WAN, LAN-transit, DMZ) in `Master` state, confirming HQ-FW1 is currently the Active firewall handling all traffic.
![HRP Master Active](images/hrp-master-active.png)

✅ **Standby Firewall — Standby State & VGMP Unification**
The same command on HQ-FW2 shows all three groups as `Backup` — and critically, all three report `Type: Vgmp`, confirming the cluster fails over as one atomic unit rather than three independent VRRP groups.
![HRP Standby State](images/hrp-standby.png)

✅ **Routing Resilience — VIP-Anchored Static Routes**
`display ip routing-table` on the Core Switch confirms the default route and the DMZ route both resolve through the VRRP virtual IP (`10.10.100.254`) — not a physical firewall address — with active `RD` flags.
![Routing Table](images/routing-table.png)

✅ **Connectivity — Ping & Traceroute to the Internet**
`ping` and `tracert` from an internal host to `203.0.113.1` confirmed correct routing and hop count through the WAN Switch → ISP Router path.
![Ping and Traceroute](images/ping-tracert.png)

✅ **NAT & Session Validation — Firewall Session Table**
`display firewall session table` on HQ-FW1 shows live NAT sessions in progress — internal host `10.10.10.10` is actively translated through the firewall — alongside the dedicated HRP heartbeat UDP sessions running between the two firewalls.
![Firewall Session Table](images/firewall-session-table.png)

✅ **Packet-Level Validation — Wireshark (NAT Confirmed)**
Two synchronized captures show the same ICMP conversation from both sides of the firewall: internally the source is the private address `10.10.10.10`, but on the WAN link it appears as the firewall's public address `203.0.113.2` — direct proof that Easy-IP NAT is translating traffic, not just routing it.
![Wireshark Capture](images/wireshark-icmp.png)

🌉 **Bonus — Real-World Internet Bridging**
Cloud1's UDP port-binding feature bridges the simulated topology to a real Windows network adapter, confirmed with bidirectional ping between the ISP Router and the host machine.
![Cloud Bridge Test]([images/cloud-bridge-test.png](https://github.com/faresismael/Huawei-eNSP-Firewall-Security-HRP-High-Availability-Cluster/blob/main/Screenshots/cloud-bridge-test.png?raw=true))

> [!NOTE]
> **Not yet captured:** a controlled ping from an Untrust-zone host toward the Trust zone (to confirm the deny policy is enforced), and an FTP/HTTP client test against the DMZ Web Server. Both are natural next tests for this topology.

## Project Structure

```
├── Topology-and-Config/     # Full eNSP project file (.topo) + exported device configs (.txt)
├── images/                  # Topology diagram, IP addressing table, test screenshots
```

## Technologies Used

- Huawei eNSP
- USG Firewalls (Active/Standby)
- HRP (Huawei Redundancy Protocol)
- VRRP + VGMP
- Security Zones & Security Policies
- VLANs & Inter-VLAN Routing (Vlanif)
- Static Routing (VIP-anchored)
- Easy-IP NAT
- Wireshark (traffic analysis & validation)

## Future Enhancements

- **Management VLAN (VLAN 40)** — a dedicated out-of-band VLAN for device management (SSH to the firewalls/switches), separated from user and server data traffic and restricted by ACL to an admin host.

## Author

**Fares Ismael** | HCIA-Security

[LinkedIn](https://www.linkedin.com/in/fares-ismael-058976425/)
