# /configs — Device Running Configurations

This directory contains the running configurations extracted from all Cisco routers and switches across the three-site enterprise infrastructure. Each file represents the full operational state of a device at the time of the last Ansible-managed backup.

> **Important:** These files are the authoritative source of truth for the network's configured state. They are generated automatically by the Ansible backup playbook and should not be edited manually. To apply changes, modify the relevant Ansible playbook or script, apply via the device CLI, and trigger a new backup run.

---

## Directory Structure

```text
configs/
├── README.md
├── routers
│   ├── RT1-Pl_config.txt
│   ├── RT1-St_config.txt
│   ├── RT1-Zg_config.txt
│   └── SRV-DHCP_config.txt
└── switches
    ├── MLS1-Zg_config.txt
    ├── MLS2-Zg_config.txt
    ├── SW1-Pl_config.txt
    ├── SW1-St_config.txt
    ├── SW1-Zg_config.txt
    └── SW2-Zg_config.txt
```

---

## Device Index

### Routers

| Hostname | Site | Role | Key Protocols |
|---|---|---|---|
| `hq-zagreb-r1` | Zagreb HQ | Core edge router | OSPF ABR, GRE/IPSec (x2), NAT overload, DHCP server, NTP Stratum 2, SNMPv2c, Syslog |
| `branch-pula-r1` | Pula Branch | Branch edge router | OSPF, GRE/IPSec (x2), NAT overload, Cisco CME, DHCP server, SNMPv2c, Syslog |
| `branch-split-r1` | Split Branch | Branch edge router | OSPF, GRE/IPSec (x2), NAT overload, Cisco CME, DHCP server, SNMPv2c, Syslog |

### Switches

| Hostname | Site | Role | Key Configuration |
|---|---|---|---|
| `hq-zagreb-sw1` | Zagreb HQ | Distribution / L3 | 802.1Q trunking, SVIs (VLANs 10/20/30/40), inter-VLAN routing |
| `hq-zagreb-sw2` | Zagreb HQ | Access layer | Access port assignment per VLAN, PortFast, BPDU Guard |
| `branch-pula-sw1` | Pula Branch | Access / Distribution | 802.1Q trunking, VLAN access port assignment |
| `branch-split-sw1` | Split Branch | Access / Distribution | 802.1Q trunking, VLAN access port assignment |

---

## Configuration File Format

Each `.cfg` file is a plain-text Cisco IOS running configuration, retrieved via `show running-config` and stored verbatim. The general structure of each file follows the standard IOS configuration hierarchy:

```
! ============================================================
! Device   : <hostname>
! Site     : <site name>
! Backed up: <YYYY-MM-DD> by Ansible playbook
! ============================================================

version XX.X
service timestamps log datetime msec
hostname <hostname>
!
! --- AAA / Authentication ---
! --- Interface Configuration ---
! --- Routing (OSPF) ---
! --- Crypto (IKE / IPSec) ---
! --- NAT ---
! --- DHCP ---
! --- NTP ---
! --- SNMP / Logging ---
! --- ACLs ---
! --- Dial-peers (CME routers only) ---
!
end
```

---

## Backup Process

Configuration files in this directory are produced exclusively by the Ansible backup playbook located at `ansible/playbooks/backup_configs.yml`. The playbook:

1. Connects to each device in the inventory over SSH
2. Executes `show running-config` via the `cisco.ios.ios_command` module
3. Writes the output to the appropriate path under `configs/`, using the inventory hostname and execution date as identifiers

Refer to [`ansible/README.md`](../ansible/README.md) for full playbook documentation and execution instructions.

---

## Restoring a Configuration

To restore a device to a previously backed-up state:

```bash
# 1. Copy the config file to the device via SCP
scp configs/routers/hq-zagreb-r1.cfg admin@<device-ip>:flash:/restore.cfg

# 2. On the device CLI, apply the configuration
Router# copy flash:/restore.cfg running-config

# 3. Persist to NVRAM
Router# write memory
```

> **Warning:** Restoring a running configuration will immediately overwrite the active device state. Always verify the target configuration file before applying, and ensure out-of-band (console) access is available in case of connectivity loss.