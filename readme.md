# homelab-network

Ansible playbooks to provision and configure the homelab network stack: OPNsense as router/firewall and Cudy TR3000 running OpenWrt as access point.

Part of the full IaC stack described in [homelab-infra](https://github.com/luisbrancher/homelab-infra).

---

## Hardware

| Device | Role |
|--------|------|
| N150 (4 NICs) | OPNsense bare metal — router, firewall, VLANs |
| Cudy TR3000 | OpenWrt — bench test AP |
| SKS3200-8E2X | Managed switch — trunk and per-port VLAN segmentation |

---

## VLANs

| VLAN    | ID | Subnet          | Access |
|---------|----|-----------------|--------|
| trusted | 10 | 10.10.10.0/24   | unrestricted |
| iot     | 20 | 10.10.20.0/24   | internet only |
| guest   | 30 | 10.10.30.0/24   | internet only |
| lab     | 40 | 10.10.40.0/24   | internet + trusted |

DHCP range `.100`–`.200` per VLAN. Range `.1`–`.99` reserved for static IPs (switch, APs, workstation).

---

## Structure

```
homelab-network/
├── ansible/
│   ├── group_vars/
│   │   └── all/
│   │       └── vault.yml       → encrypted secrets (Ansible Vault)
│   ├── hosts.ini               → inventory (opnsense + cudy_ap)
│   ├── playbooks/
│   │   ├── setup_ap.yml        → configures OpenWrt as dumb AP via SSH (raw)
│   │   ├── setup_opnsense.yml  → provisions OPNsense via SSH + REST API
│   │   └── upgrade_opnsense.yml → firmware update + reboot (run on demand, not in site.yml)
│   └── site.yml                → main entrypoint, imports setup_opnsense.yml (Cudy playbook disabled)
└── readme.md
```

---

## Prerequisites

**Manual steps before running Ansible:**

OPNsense (via console):
- Install OPNsense (UFS, disk `nda0`)
- Assign interfaces via auto-detect (option 1 in console)
- LAN: `10.10.10.1/24`, DHCP `.100`–`.200` (option 2, answer N to "Restore web GUI access defaults")
- WAN: DHCP (option 2)
- Create `root` or admin user with **shell access enabled** (System → Access → Users → check "Login shell")
- Generate API Key + Secret for that user

OpenWrt (via LuCI):
- Set LAN IP to static `10.10.10.5`
- SSH is enabled by default

---

## Running

```bash
# Install dependency
ansible-galaxy collection install oxlorg.opnsense

# Confirm connectivity to both hosts
ansible -i ansible/hosts.ini n150 -m ping --ask-vault-pass
ansible -i ansible/hosts.ini cudy_ap -m raw -a "/bin/true" --ask-vault-pass

# Provision OPNsense (system + VLANs + Kea DHCP + firewall rules via API)
ansible-playbook -i ansible/hosts.ini ansible/playbooks/setup_opnsense.yml --ask-vault-pass

# Update OPNsense firmware (reboots the router if anything is updated — the whole house loses network)
ansible-playbook -i ansible/hosts.ini ansible/playbooks/upgrade_opnsense.yml --ask-vault-pass

# Configure OpenWrt as dumb AP
ansible-playbook -i ansible/hosts.ini ansible/playbooks/setup_ap.yml --ask-vault-pass
```

---

## Secrets (Ansible Vault)

```yaml
# ansible/group_vars/all/vault.yml
vault_opnsense_api_key: ""
vault_opnsense_api_secret: ""
vault_ansible_password: ""
vault_wifi_trusted_pass: ""
vault_wifi_iot_pass: ""
vault_wifi_guest_pass: ""
vault_wifi_lab_pass: ""
```

```bash
ansible-vault create ansible/group_vars/all/vault.yml
ansible-vault edit ansible/group_vars/all/vault.yml
```

---

## What each playbook does

**`setup_opnsense.yml`**

Play 1 (SSH):
- Disables VLAN hardware offload on `igc0` and `igc3`
- Enables PowerD in hiadaptive mode

**`upgrade_opnsense.yml`**

Kept separate on purpose: updating reboots the router. Never imported by `site.yml`.
- Refreshes the package catalog (`pkg update -f`) and lists pending upgrades
- Runs `opnsense-update` only if something is pending, then reboots and waits for the host to come back

Play 2 (REST API via `oxlorg.opnsense`):
- Creates VLANs 20, 30 and 40 on `igc3`
- Enables Kea DHCP and binds it to all interfaces (`lan`, `opt1`, `opt2`, `opt3`)
- Configures subnets, DHCP ranges, gateway and DNS per VLAN
- Creates static IP reservations for switch, AP and workstation
- Creates firewall rules: inter-VLAN isolation (sequence 10) + internet passthrough (sequence 100)
- Disables Dnsmasq (Kea handles DHCP)

**`setup_ap.yml`**

Uses `raw` throughout — OpenWrt has no Python interpreter.

- Static IP `10.10.10.5` on LAN (idempotent check before applying)
- DHCP server disabled (OPNsense handles it)
- WAN interfaces removed
- eth0 (2.5GbE) added to LAN bridge
- Bridge VLAN filtering enabled: VLAN 1 untagged + VLANs 20/30/40 tagged on eth0 and eth1
- Logical interfaces: `lan` → `br-lan.1`, `iot` → `br-lan.20`, `guest` → `br-lan.30`, `lab` → `br-lan.40`
- SSIDs created with explicit named sections (idempotent): `wifinet_trusted`, `wifinet_iot`, `wifinet_lab`, `wifinet_guest` on radio1 (5GHz); `wifinet_trusted_2g`, `wifinet_iot_2g` on radio0 (2.4GHz)

---

## Dependencies

```bash
ansible-galaxy collection install oxlorg.opnsense
```

---

## Idempotency

`uci set` tasks overwrite without duplicating. `uci add` tasks check before creating:

```bash
if ! uci show network | grep -q "bridge-vlan.*vlan='20'"; then
  uci add network bridge-vlan
  ...
fi
```

SSIDs use named sections instead of anonymous entries, so re-running won't create duplicates:

```bash
uci set wireless.wifinet_trusted=wifi-iface
```

Playbooks can be run multiple times without side effects.

---

## References

- [OPNsense docs](https://docs.opnsense.org)
- [oxlorg.opnsense collection](https://github.com/O-X-L/ansible-opnsense)
- [OpenWrt UCI reference](https://openwrt.org/docs/guide-user/base-system/uci)