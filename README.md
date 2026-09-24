# devhub-ansible-router

> Part of [DevHub/Ansible](https://github.com/CollinPoetoehena/DevHub/blob/main/packages/Ansible.md) — see that file for conventions, structure guidelines, and the full role index.

Configures a Debian-based system (e.g. Raspberry Pi) as a dedicated lab router providing network isolation, DHCP, DNS, SLAAC/Router Advertisements, NAT and an nftables firewall (e.g. for a homelab). The lab is **dual-stack (IPv4 + IPv6)**. 

Primarily used for my personal [homelab in DevHub](https://github.com/CollinPoetoehena/DevHub/blob/main/homelab/README.md) but also suitable for other small-scale lab environments.

## What This Role Does

| Task file | Purpose |
| --- | --- |
| `packages.yml` | Install utility packages (dnsutils, tcpdump, ndisc6, conntrack, …) for debugging |
| `networking.yml` | Physical LAN interface (no IP), dual-stack VLAN sub-interfaces (eth1.10/20/30) via NetworkManager, IPv4 **and** IPv6 forwarding, RA handling |
| `dhcp_dns.yml` | Install and configure dnsmasq for per-VLAN DHCPv4 + SLAAC/RA + DNS |
| `firewall.yml` | nftables: install, legacy cleanup, deploy all rule fragments, validate and persist |
| `verify/` | Post-configuration checks: networking, DHCP/DNS, firewall, connectivity, summary |

## Firewall

`firewall.yml` is a single task file that contains **no firewall rules of its own**. It is deployment logic: it installs nftables, removes the legacy stacks, and renders the rule templates into numbered fragments. The rules themselves live in `templates/nftables/`.

### Sections in `firewall.yml`

Each section is marked with a `####` comment separator, in the order the fragments load:

| Section | Deploys | Purpose |
| --- | --- | --- |
| Install nftables | — | Install the package (kept here so `--tags firewall` is self-contained) |
| Remove legacy stacks | — | Disable/purge `ufw`, `iptables-persistent`, `netfilter-persistent`; flush leftover kernel rules |
| Main config | `/etc/nftables.conf` | `flush ruleset` + `include /etc/nftables.d/*.nft` |
| Fragment 10 | `10-base.nft` | Defines, `inet filter` + `inet nat` tables, chains, **default policy drop** |
| Fragment 20 | `20-nat.nft` | IPv4 masquerade and optional IPv6 NAT66 |
| Fragment 30 | `30-default.nft` | conntrack, loopback, ICMP/ICMPv6, lab→home block, WAN→lab block, SSH from home |
| Fragment 40 | `40-dynamic.nft` | **Generated** per-VLAN rules from `router_vlans[].firewall` (+ policy validation) |
| Stale cleanup | — | Remove any `.nft` in the include dir not managed by this role |
| Validate & persist | — | `nft --check` dry run, enable `nftables.service`, load if not active |

### Why the rules live in the templates

1. **Rule order is policy.** nftables is first-match-wins, so "block lab → home" *must* be evaluated before any per-VLAN allow. As a sequence of Ansible tasks that ordering is invisible in the resulting firewall; in a template it is literally line order, and the numeric prefixes make the order between concerns explicit (10 → 20 → 30 → 40).
2. **Atomicity.** `nft -f` applies a complete ruleset in one transaction — all or nothing. Adding rules one task at a time (the old `ansible.builtin.iptables` approach) leaves a half-open firewall if the play fails midway.
3. **Reviewability.** The rendered fragments in git and `nft list ruleset` on the Pi read the same. No mental translation between "what Ansible did" and "what the kernel enforces".
4. **Data-driven policy.** `40-dynamic.nft.j2` generates every per-VLAN rule from `router_vlans[].firewall`, so adding a VLAN or opening a port is a change in `group_vars/router/main.yml` only.

### Template → target mapping

| Template | Deployed to |
| --- | --- |
| `templates/nftables/nftables.conf.j2` | `/etc/nftables.conf` |
| `templates/nftables/10-base.nft.j2` | `/etc/nftables.d/10-base.nft` |
| `templates/nftables/20-nat.nft.j2` | `/etc/nftables.d/20-nat.nft` |
| `templates/nftables/30-default.nft.j2` | `/etc/nftables.d/30-default.nft` |
| `templates/nftables/40-dynamic.nft.j2` | `/etc/nftables.d/40-dynamic.nft` |

Only `/etc/nftables.conf` is loaded by `nftables.service`; it pulls in the fragments via `include`. The directory is `router_nftables_include_dir`. Do not drop files there by hand — the stale-cleanup task deletes anything that is not one of the four managed fragments.

## Verification layout (`tasks/verify/`)

`networking.yml` → `dhcp_dns.yml` → `firewall.yml` → `connectivity.yml` → `summary.yml`

This stays a directory (unlike the firewall) because each file registers variables that `summary.yml` reads, so the split is functional, not just organisational. Summary must run last.

## Why nftables (not iptables, not ufw)

All three drive the same kernel subsystem (netfilter). See for detailed explanation and setup [tasks/firewall.yml](tasks/firewall.yml).

- **iptables** — legacy interface; separate rule sets for IPv4 (`iptables`) and IPv6 (`ip6tables`), no atomic apply, needs `iptables-persistent` to survive reboot, and on Debian today it is usually just a wrapper around nftables anyway.
- **ufw** — a *frontend*, not a firewall. Great for a single-homed host ("which inbound ports are open"), but a router's NAT, per-interface forwarding and inter-VLAN policy end up as raw iptables in `before.rules` — you get two syntaxes and ufw's own chains in between.
- **nftables** — native and modern; **one `inet` ruleset covers IPv4 + IPv6**, **atomic** loading (`nft -f` applies everything or nothing), declarative files under version control, native sets/maps/counters, and built-in persistence.

Rule of thumb used in this homelab: **router → nftables**, **simple single-host VMs → ufw is fine**, **raw iptables → only for reading old docs**. Never run two stacks at once.

## Why IPv6 is enabled

- **Practice** — SLAAC, Router Advertisements, Neighbour Discovery, ULA vs GUA and IPv6 firewalling can only be learned by running them.
- **More possibilities** — dual-stack services, AAAA records, IPv6-only clients, dual-stack Kubernetes later.
- **Better firewalling discipline** — no NAT to hide behind, so every inbound flow is explicit (default policy `drop`).
- **Addressing** — stable ULA (`fd42::/48`, one `/64` per VLAN) instead of a delegated prefix that changes on every modem reboot. Outbound IPv6 is masqueraded (`router_enable_ipv6_nat`) until real prefix delegation is available.

## Network Topology

```
ISP Modem (e.g. 192.168.2.0/24)
    │
    └── eth0 (WAN) ─── Lab Router ─── eth1 (LAN, no IP) → Lab Switch
                                                   │ VLANs, such as:
                                                   ├── eth1.10 (VLAN 10 — Management, 10.42.10.1/24, fd42:10::1/64)
                                                   ├── eth1.20 (VLAN 20 — Services,   10.42.20.1/24, fd42:20::1/64)
                                                   └── eth1.30 (VLAN 30 — IoT,        10.42.30.1/24, fd42:30::1/64)
```


## Requirements

- Raspberry Pi 4 (or later) with Raspberry Pi OS Lite (64-bit)
- Two Ethernet interfaces: built-in (eth0) + USB adapter (eth1)
- SSH enabled and an automation user (e.g. `ansibleremote`) user created (via the [users role: `devhub-ansible-users`](https://github.com/CollinPoetoehena/devhub-ansible-users) )
- eth0 connected to ISP modem, eth1 connected to a switch

## Variables

### Required (no default — set in `group_vars/router/main.yml`)

| Variable | Description |
| --- | --- |
| `router_lan_subnet` | Full CIDR for the lab supernet (e.g. `10.42.0.0/20`) |
| `router_lan_subnet_v6` | IPv6 ULA supernet (e.g. `fd42::/48`) |
| `router_vlans` | List of VLAN dicts: `id, name, interface, gateway_ip, cidr, netmask, subnet, gateway_ip6, cidr6, subnet6, dhcp_range_start, dhcp_range_end, firewall` |
| `router_home_subnet` | Home network to block from lab (e.g. `192.168.2.0/24`) |
| `router_home_gateway` | ISP modem/gateway IP (e.g. `192.168.2.254`) |
| `router_dhcp_domain` | DNS search domain (e.g. `lab.local`) |
| `router_static_leases` | Static MAC→IP reservations |

### Optional (sensible defaults)

| Variable | Default | Description |
| --- | --- | --- |
| `router_wan_interface` | `eth0` | WAN interface (ISP modem side) |
| `router_lan_interface` | `eth1` | LAN interface (lab switch side) |
| `router_lan_connection_name` | `lab-lan` | NetworkManager connection profile name |
| `router_dhcp_lease_time` | `24h` | DHCPv4 lease duration |
| `router_dhcp_lease_time_v6` | `4h` | SLAAC/RA lifetime |
| `router_dns_upstream_servers` | dual-stack list | Upstream DNS forwarders |
| `router_dns_cache_size` | `1000` | DNS cache entries |
| `router_enable_ipv6` | `true` | Enable dual-stack |
| `router_ipv6_mode` | `ula` | ULA addressing instead of delegated prefix |
| `router_enable_ipv6_nat` | `true` | NAT66 for ULA egress |
| `router_enable_nat` | `true` | IPv4 NAT on WAN |
| `router_block_lab_to_home` | `true` | Block lab→home traffic |
| `router_inter_vlan_default_policy` | `drop` | Policy for unmatched VLAN→VLAN traffic |
| `router_nftables_include_dir` | `/etc/nftables.d` | Where rule fragments live |
| `router_nftables_main_config` | `/etc/nftables.conf` | File loaded by `nftables.service` |
| `router_nftables_log_drops` | `true` | Rate-limited logging of dropped packets |
| `router_nftables_log_rate` | `5/minute` | Rate limit for that logging |
| `router_nftables_persistent` | `true` | Enable `nftables.service` at boot |
| `router_remove_legacy_iptables` | `true` | Purge iptables-persistent/ufw and flush legacy rules |

### Per-VLAN firewall policy

```yaml
firewall:
  allow_to_internet: true
  router_services: [ssh, dns, dhcp, icmp, ntp]   # services ON the router this VLAN may use
  inter_vlan:
    - { action: drop,  to: 10, comment: "IoT -> Management denied" }
    - { action: allow, to: 20, proto: tcp, ports: [1883, 8123] }
  custom:
    - { action: allow, chain: input, proto: tcp, ports: [9100], saddr: "10.42.10.0/24" }
```

Explicit denies are emitted before allows; anything unmatched falls through to `router_inter_vlan_default_policy` and then to the chain's `drop` policy. Invalid policy (non-dict `firewall`, or an `inter_vlan.to` that is not an existing VLAN id) fails early with a readable message instead of an `nft -f` syntax error.

## Usage

Requirements file example (same directory as ansible.cfg, create a file called requirements.yml):
```yaml
---
roles:
  - name: devhub.router
    src: https://github.com/CollinPoetoehena/devhub-ansible-router.git
    scm: git
    version: 1.0.0
``` 

Then install with: 
```sh
# NOTE: Example of roles path for -p is "roles/" (you can also specify this in ansible.cfg)
ansible-galaxy install -r requirements.yml -p <path/to/roles>
```

Example playbook using this role (e.g. site.yml):
```yaml
- hosts: all
  roles:
    - role: devhub.router
```

Running the playbook with tags:

```bash
# Run only the router role:
ansible-playbook site.yml --tags router

# Run a specific part:
ansible-playbook site.yml --tags firewall
ansible-playbook site.yml --tags nftables
ansible-playbook site.yml --tags dhcp
ansible-playbook site.yml --tags networking

# Run only verification:
ansible-playbook site.yml --tags verify
```

### Handy manual checks

```bash
sudo nft list ruleset                  # full firewall, both address families
sudo nft --check -f /etc/nftables.conf # validate without applying
sudo nft list table inet filter        # filter chains only
ls -l /etc/nftables.d/                 # the four deployed fragments
sudo journalctl -k | grep 'nft '       # dropped packets (logging rules)
ip -6 addr show eth1.10                # VLAN ULA gateway address
rdisc6 eth0                            # inspect the ISP modem's Router Advertisement
```

## Tags

- `router` — all router tasks
- `packages` — utility package installation
- `networking` — interfaces, addressing, IP forwarding
- `dhcp`, `dns`, `dnsmasq` — DHCP/DNS/RA configuration
- `firewall`, `nftables`, `nat` — nftables rules
- `verify` — post-configuration verification checks