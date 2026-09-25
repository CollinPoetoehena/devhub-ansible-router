# Changelog

All notable changes to `devhub-ansible-router-custom` are documented here. See for details about versioning: [Versioning Documentation](https://github.com/CollinPoetoehena/DevHub/blob/main/packages/README.md#versioning).

## [2.0.0] — 2026-09-24

### Added
- Added dual-stack (IPv4 + IPv6) support with SLAAC and Router Advertisements.
- Updated documentation to reflect dual-stack support.
- Completely rewrote the tasks to support dual-stack networking and nftables firewall management; mainly the `networking.yml` and `firewall.yml` tasks.
- Dynamic firewall rules based on VLANs and IP versions, see `firewall.yml` (specifically through the variable `router_vlans`).

### Changed
- Switched from iptables to nftables for firewall management.
- The lab is now dual-stack (IPv4 + IPv6).

## [1.0.0] — 2026-08-20

### Added
- Initial release. Note that this role was first present inside the [DevHub repository/homelab](https://github.com/CollinPoetoehena/DevHub/tree/main/homelab), however it has since been extracted into its own standalone role for easier reuse and maintenance (as explained in [DevHub/homelab/docs/2_Setup/1_Setup_Local_Environment.md](https://github.com/CollinPoetoehena/DevHub/blob/main/homelab/docs/2_Setup/1_Setup_Local_Environment.md#reusable-components)).