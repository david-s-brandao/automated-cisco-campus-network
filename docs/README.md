# Project Documentation

This directory contains all reference documentation for the multi-site enterprise network infrastructure, including IP addressing tables, logical and physical topology diagrams, and hardware validation assets.

---

## Directory Structure

```text
docs/
├── addressing_table.md
├── assets
│   ├── ansible_poc_photo.jpg
│   ├── hardware_setup.jpg
│   ├── topology_diagram.jpg
│   └── voip_call_poc.GIF
└── README.md
```

---

## Addressing Documentation

### `addressing/ip_addressing_table.md`

The master addressing table consolidates every assigned subnet across all three sites. Each entry includes the device hostname, interface designation, IP address, subnet mask, and functional role. This document serves as the authoritative reference for any configuration or troubleshooting task.

**Scope:** HQ Zagreb, Branch Pula, Branch Split — all routed interfaces, loopbacks, and management addresses.

### `addressing/vlan_definitions.md`

Documents the VLAN taxonomy deployed across the switching infrastructure:

| VLAN ID | Name | Purpose |
|---|---|---|
| 10 | STAFF | End-user workstations and corporate devices |
| 20 | GUEST | Untrusted external visitors — Internet-only access |
| 30 | MANAGEMENT | Network device OOB management plane (SSH, SNMP) |
| 40 | VOICE | VoIP endpoints and Cisco CME signalling traffic |
| 99 | NATIVE | 802.1Q native VLAN (untagged trunk traffic) |

### `addressing/tunnel_addressing.md`

Documents the point-to-point `/30` subnets assigned to each GRE/IPSec tunnel interface in the full-mesh WAN topology, along with the corresponding crypto map and transform-set references.

---

## Topology Diagrams

### Logical Topology

The logical topology diagram depicts:
- Layer 3 routing domains and OSPF Area 0 adjacencies
- GRE/IPSec tunnel interfaces and their IP assignments
- Inter-VLAN routing boundaries (SVIs) at each site
- NAT translation boundaries (inside/outside interfaces)
- NTP hierarchy (Stratum 2 HQ master → branch clients)

### Physical Topology

The physical topology diagram depicts:
- Rack elevation and device placement per site
- Physical cabling between routers and switches (trunk/access port designation)
- WAN uplink interfaces and their ISP-facing IP assignments
- Console and management cabling for out-of-band access

> **Tooling note:** Diagrams are maintained in [Draw.io](https://app.diagrams.net/) (`.drawio` XML format) to allow version-controlled, diff-friendly edits. PDF exports are generated for distribution and printed reference.

---

## Validation Assets

The `assets/` directory contains photographic and animated evidence of key proof-of-concept milestones:

| File | Description | Referenced In |
|---|---|---|
| `topology_diagram.jpg` | High-level logical overview of the three-site topology | `README.md` — Figure 1 |
| `voip_call_poc.GIF` | Screen capture of a live inter-branch VoIP call | `README.md` — Figure 2 |
| `ansible_poc_photo.jpg` | Terminal output of Ansible playbook execution | `README.md` — Figure 3 |
| `hardware_setup.jpg` | Physical lab staging of Cisco hardware | `README.md` — Figure 4 |