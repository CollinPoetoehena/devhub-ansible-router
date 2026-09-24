# devhub-ansible-router

> Part of [DevHub/Ansible](https://github.com/CollinPoetoehena/DevHub/blob/main/packages/Ansible.md) — see that file for conventions, structure guidelines, and the full role index.

Configures a Debian-based system (e.g. Raspberry Pi) as a dedicated lab router providing network isolation, DHCP, DNS, NAT and an iptables firewall (e.g. for a homelab). The lab is **IPv4-only**. 

Primarily used for my personal [homelab in DevHub](https://github.com/CollinPoetoehena/DevHub/blob/main/homelab/README.md) but also suitable for other small-scale lab environments.

## What This Role Does

| Task File | Purpose |
|-----------|---------|
| `packages.yml` | Install utility packages (dnsutils, tcpdump, curl) for debugging |
| `networking.yml` | Physical LAN interface (no IP), VLAN sub-interfaces (`eth1.10`, `eth1.20`, `eth1.30`) via NetworkManager, enable IPv4 forwarding |
| `dhcp_dns.yml` | Install and configure `dnsmasq` for per-VLAN DHCP + DNS on the lab network |
| `firewall.yml` | NAT (masquerading), forwarding rules, lab→home blocking, inter-VLAN routing, INPUT protection |
| `verify.yml` | Post-configuration checks: networking, DHCP/DNS, firewall, connectivity |

## Network Topology

```
ISP Modem (192.168.2.0/24)
    │
    └── eth0 (WAN) ─── Lab Router ─── eth1 (LAN, no IP) → Lab Switch
                                                   │ VLANs, such as:
                                                   ├── eth1.10 (VLAN 10 — Management, 10.42.10.1/24)
                                                   ├── eth1.20 (VLAN 20 — Services,   10.42.20.1/24)
                                                   └── eth1.30 (VLAN 30 — IoT,        10.42.30.1/24)
```

## Requirements

- Raspberry Pi 4 (or later) with Raspberry Pi OS Lite (64-bit)
- Two Ethernet interfaces: built-in (`eth0`) + USB adapter (`eth1`)
- SSH enabled and `ansibleremote` user created (via the users bootstrap play)
- `eth0` connected to ISP modem, `eth1` connected to lab switch

## Key Variables

### Required (no default — set in `group_vars/router/main.yml`)

| Variable | Description |
|----------|-------------|
| `router_lan_subnet` | Full CIDR for the lab supernet (e.g. `10.42.0.0/20`) |
| `router_vlans` | List of VLAN dicts with `id`, `name`, `interface`, `gateway_ip`, `cidr`, `netmask`, `dhcp_range_start`, `dhcp_range_end` |
| `router_home_subnet` | Home network to block from lab (e.g. `192.168.2.0/24`) |
| `router_home_gateway` | ISP modem/gateway IP (e.g. `192.168.2.254`) |
| `router_dhcp_domain` | DNS search domain (e.g. `lab.local`) |
| `router_static_leases` | Static MAC→IP reservations (list of `{mac, ip, hostname}`) |

### Optional (have sensible defaults)

| Variable | Default | Description |
|----------|---------|-------------|
| `router_wan_interface` | `eth0` | WAN interface (ISP modem side) |
| `router_lan_interface` | `eth1` | LAN interface (lab switch side) |
| `router_lan_connection_name` | `lab-lan` | NetworkManager connection profile name |
| `router_dhcp_lease_time` | `24h` | DHCP lease duration |
| `router_dns_upstream_servers` | `[8.8.8.8, 8.8.4.4, 1.1.1.1]` | Upstream DNS forwarders |
| `router_dns_cache_size` | `1000` | DNS cache entries |
| `router_enable_nat` | `true` | Enable NAT on WAN interface |
| `router_block_lab_to_home` | `true` | Firewall: block lab→home traffic |
| `router_iptables_persistent` | `true` | Save iptables rules across reboots |

## Usage

```bash
# Run only the router role:
ansible-playbook site.yml --tags router

# Run a specific part:
ansible-playbook site.yml --tags firewall
ansible-playbook site.yml --tags dhcp
ansible-playbook site.yml --tags networking

# Run only verification:
ansible-playbook site.yml --tags verify
```

## Tags

- `router` — all router tasks
- `packages` — utility package installation
- `networking` — interfaces, addressing, IP forwarding
- `dhcp`, `dns`, `dnsmasq` — DHCP/DNS/RA configuration
- `firewall`, `nftables`, `nat` — nftables rules
- `verify` — post-configuration verification checks