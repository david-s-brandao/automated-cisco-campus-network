# /ansible — Automation and Infrastructure as Code

This directory contains all Ansible artefacts used to manage the Cisco fleet across the three-site enterprise infrastructure. It includes the inventory definition, host/group variable files, and all playbooks responsible for configuration backup and fleet-wide operational tasks.

---

## Directory Structure

```text
ansible/
├── ansible.cfg
├── backup.yaml
├── hosts.ini
└── README.md              
```

---

## Prerequisites

| Requirement | Minimum Version | Notes |
|---|---|---|
| Ansible | 2.12 | Tested on 2.14 |
| Python | 3.9 | Required by Ansible control node |
| `cisco.ios` collection | 4.x | Install via `ansible-galaxy` (see below) |
| SSH access | — | TCP/22 open from control node to all devices |
| Cisco IOS | 15.x+ | Tested on IOS 15.2 (1900/2900 series) |

### Installation

```bash
# Install the Cisco IOS Ansible collection
ansible-galaxy collection install cisco.ios

# Verify connectivity to all inventory hosts
ansible all -i inventory/hosts.ini -m ping
```

---

## Inventory

### `inventory/hosts.ini`

Defines all managed devices, grouped by role and site. Groups are used to apply targeted variables and scope playbook execution.

```ini
[routers]
hq-zagreb-r1     ansible_host=192.168.1.1
branch-pula-r1   ansible_host=10.10.1.1
branch-split-r1  ansible_host=10.20.1.1

[switches]
hq-zagreb-sw1    ansible_host=192.168.1.10
hq-zagreb-sw2    ansible_host=192.168.1.11
branch-pula-sw1  ansible_host=10.10.1.10
branch-split-sw1 ansible_host=10.20.1.10

[hq:children]
routers
switches

[all:vars]
ansible_connection=network_cli
ansible_network_os=cisco.ios.ios
ansible_user=admin
ansible_password={{ vault_device_password }}
ansible_become=true
ansible_become_method=enable
ansible_become_password={{ vault_enable_password }}
```

> **Security note:** Credentials are never stored in plaintext. All sensitive variables (`vault_device_password`, `vault_enable_password`) must be defined in an Ansible Vault-encrypted file. See the [Vault section](#ansible-vault) below.

---

## Playbooks

### `backup_configs.yml` — Configuration Backup

Retrieves the current running configuration from every device in the inventory and persists it to the local filesystem under `configs/`.

```yaml
- name: Backup Cisco Running Configurations
  hosts: all
  gather_facts: false

  tasks:
    - name: Retrieve running-config from device
      cisco.ios.ios_command:
        commands: show running-config
      register: config_output

    - name: Persist configuration to local filesystem
      copy:
        content: "{{ config_output.stdout[0] }}"
        dest: "configs/{{ inventory_hostname }}_{{ lookup('pipe', 'date +%Y-%m-%d') }}.cfg"
      delegate_to: localhost
```

**Execution:**
```bash
ansible-playbook -i inventory/hosts.ini playbooks/backup_configs.yml
```

**Expected Play Recap (success):**
```
PLAY RECAP *************************************************************
hq-zagreb-r1   : ok=2  changed=1  unreachable=0  failed=0
branch-pula-r1 : ok=2  changed=1  unreachable=0  failed=0
branch-split-r1: ok=2  changed=1  unreachable=0  failed=0
...
```

---

### `verify_ospf.yml` — OSPF Adjacency Validation

Queries the OSPF neighbour table on all routers and asserts that the expected number of adjacencies are in `FULL` state. Intended for use after any routing change or device reboot.

```yaml
- name: Verify OSPF Neighbour Adjacencies
  hosts: routers
  gather_facts: false

  tasks:
    - name: Retrieve OSPF neighbour table
      cisco.ios.ios_command:
        commands: show ip ospf neighbor
      register: ospf_output

    - name: Assert FULL adjacencies present
      assert:
        that:
          - "'FULL' in ospf_output.stdout[0]"
        fail_msg: "WARNING: No FULL OSPF adjacencies found on {{ inventory_hostname }}"
        success_msg: "OK: OSPF adjacencies verified on {{ inventory_hostname }}"
```

**Execution:**
```bash
ansible-playbook -i inventory/hosts.ini playbooks/verify_ospf.yml
```

---

### `verify_vpn_tunnels.yml` — IPSec Tunnel State Validation

Queries the IKE and IPSec SA tables on all routers and verifies that tunnels are in an active, established state. Intended for use after WAN changes or security policy updates.

```yaml
- name: Verify IPSec Tunnel State
  hosts: routers
  gather_facts: false

  tasks:
    - name: Check IKE SA state
      cisco.ios.ios_command:
        commands: show crypto isakmp sa
      register: isakmp_output

    - name: Check IPSec SA state
      cisco.ios.ios_command:
        commands: show crypto ipsec sa
      register: ipsec_output

    - name: Assert IKE SA is QM_IDLE (established)
      assert:
        that:
          - "'QM_IDLE' in isakmp_output.stdout[0]"
        fail_msg: "WARNING: No established IKE SA on {{ inventory_hostname }}"
        success_msg: "OK: IKE SA established on {{ inventory_hostname }}"
```

**Execution:**
```bash
ansible-playbook -i inventory/hosts.ini playbooks/verify_vpn_tunnels.yml
```

---

## Ansible Vault

All sensitive credentials are encrypted using Ansible Vault and must never be committed to version control in plaintext.

```bash
# Create an encrypted variables file
ansible-vault create inventory/group_vars/all/vault.yml

# Edit an existing vault file
ansible-vault edit inventory/group_vars/all/vault.yml

# Run a playbook with vault decryption
ansible-playbook -i inventory/hosts.ini playbooks/backup_configs.yml --ask-vault-pass
```

**Example `vault.yml` contents (before encryption):**
```yaml
vault_device_password: "your_ssh_password"
vault_enable_password: "your_enable_secret"
```

---

## Recommended Workflow

```
1. Make configuration change on device CLI
        ↓
2. Run backup_configs.yml to capture new state
        ↓
3. Commit updated configs/ files to version control (git)
        ↓
4. Run verify_ospf.yml and verify_vpn_tunnels.yml to assert network health
        ↓
5. Review Play Recap — all hosts must show unreachable=0, failed=0
```