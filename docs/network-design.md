# Homelab Network Design

## Overview

Hybrid homelab built on dedicated hardware running Proxmox, pfSense, and a managed TP-Link switch with Omada controller. Designed for infrastructure automation, DevOps tooling, and Azure hybrid connectivity labs.

All lab VLANs are isolated from the primary home network. Inter-VLAN routing is handled exclusively by pfSense with explicit firewall rules per segment.

---

## Hardware

| Device | Role |
|--------|------|
| Proxmox Server (Xeon 6-core, 128GB RAM, 4-6TB storage) | Hypervisor — all lab VMs |
| pfSense (dedicated hardware) | Perimeter firewall, routing, VPN |
| TP-Link Managed Switch + Omada Controller | VLAN trunking, AP management |

---

## Network Topology

```
Internet
    │
pfSense (dedicated hardware)
    │   WAN
    │   LAN trunk (802.1q)
    │
TP-Link Managed Switch (Omada)
    │
    ├── Trunk port → Proxmox (vmbr0, VLAN aware)
    ├── Access ports → Physical devices per VLAN
    └── APs → SSID per VLAN
```

---

## VLAN Design

| VLAN ID | Name | Subnet | Gateway | Purpose |
|---------|------|--------|---------|---------|
| Native | Main WiFi | 10.212.46.0/24 | 10.212.46.1 | Primary home network — untouched |
| 200 | Lab Access | 192.168.200.0/24 | 192.168.200.1 | Jump VLAN — access into lab segments |
| 10 | Management | 10.0.10.0/24 | 10.0.10.1 | Proxmox UI, pfSense admin, switch management |
| 20 | Servers | 10.0.20.0/24 | 10.0.20.1 | Lab VMs — DCs, file services, RDS |
| 30 | DevOps | 10.0.30.0/24 | 10.0.30.1 | CI/CD runners, containers, pipeline workloads |
| 50 | IoT | 10.0.50.0/24 | 10.0.50.1 | Isolated IoT and smart devices |
| 99 | Azure VPN | 10.0.99.0/24 | 10.0.99.1 | Reserved — future Azure site-to-site VPN |

---

## Internal DNS

- **Domain suffix:** `mysysadmin.uk`
- DNS resolved internally via pfSense DNS Resolver (Unbound)
- All lab VMs registered under `mysysadmin.uk`

---

## Proxmox Bridge Configuration

Single NIC trunk design — all VLANs tagged over one physical interface.

```
vmbr0 (VLAN aware bridge)
    ├── VM NIC tag 10  → VLAN 10 Management
    ├── VM NIC tag 20  → VLAN 20 Servers
    ├── VM NIC tag 30  → VLAN 30 DevOps
    ├── VM NIC tag 50  → VLAN 50 IoT
    └── VM NIC tag 200 → VLAN 200 Lab Access
```

---

## pfSense Firewall Rules (Design Intent)

| Source VLAN | Destination | Action | Reason |
|-------------|-------------|--------|--------|
| Main WiFi | VLAN 200 | Allow | Home devices access lab jump VLAN |
| VLAN 200 | VLAN 10 | Allow | Lab access reaches management |
| VLAN 200 | VLAN 20 | Allow | Lab access reaches servers |
| VLAN 200 | VLAN 30 | Allow | Lab access reaches DevOps segment |
| VLAN 10 | Any | Allow | Management has full access |
| VLAN 20 | VLAN 30 | Allow | Servers can reach DevOps segment |
| VLAN 50 | Any | Deny | IoT fully isolated |
| Any | Main WiFi | Deny | Lab cannot reach home network |

---

## IP Reservations (Management Segment)

| Host | IP | Role |
|------|----|------|
| pfSense LAN | 10.0.10.1 | Default gateway, DNS |
| Proxmox | 10.0.10.10 | Hypervisor management UI |
| TP-Link Switch | 10.0.10.20 | Switch management |

---

## Design Decisions

- **Separate pfSense hardware** — Proxmox can be wiped without losing network connectivity
- **VLAN aware single trunk** — Scales to any number of VMs without additional physical NICs
- **VLAN 200 as jump VLAN** — Home devices never directly touch management or server segments
- **10.0.x.x lab space** — Azure-safe, no overlap with RFC1918 ranges in common use
- **VLAN 99 reserved** — Azure VPN Gateway site-to-site planned, address space pre-allocated
- **mysysadmin.uk domain** — Internal DNS suffix for all lab resources

---

## Planned Additions

- [ ] Azure site-to-site VPN via VLAN 99
- [ ] Entra ID Connect sync from on-prem AD (VLAN 20) to Azure
- [ ] Monitoring stack — Grafana + Prometheus on VLAN 30
- [ ] Self-hosted GitHub Actions runners on VLAN 30
- [ ] Terraform-managed VM provisioning via Proxmox provider

---

*Last updated: March 2026*
