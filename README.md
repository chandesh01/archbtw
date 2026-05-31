# Arch Linux Installation Guide

A practical Arch Linux installation guide focused on:

* UEFI
* LUKS2 encryption
* Btrfs subvolumes
* Unified Kernel Images (UKI)
* systemd-boot
* Secure Boot
* TPM2 auto-unlock
* DNS hardening
* Recovery snapshots
* Server deployments

## Features

* Full-disk encryption with LUKS2
* Btrfs subvolume layout
* Swapfile with hibernation support
* ZRAM configuration
* Unified Kernel Images (UKI)
* systemd-boot integration
* Secure Boot with `sbctl`
* Secure Boot with Shim + MOK
* TPM2-backed LUKS auto-unlock
* Recovery boot environment
* Desktop and server networking
* SSH configuration
* Firewall configuration
* Secure DNS solutions

## Documentation

### Installation

* [Install Arch Linux](./install/Install.md)

### Post-Install

* [Post-Install Tasks](./install/POST_INSTALL.mdPOST_INSTALL.md)
* [User Setup](./install/post-install/useradd.md)
* [Firewall](./install/post-install/firewall.md)
* [Recovery Environment](./install/post-install/Recovery.md)
* [Secure Boot (sbctl)](./install/post-install/Secure_boot.md)
* [Secure Boot (Shim + MOK)](./install/post-install/Secure-boot-Shim.md)
* [TPM2 Auto-Unlock for LUKS](./install/post-install/tmp_luks.md)
* [Server Setup](./install/post-install/server.md)
* [SSH Setup](./install/post-install/ssh.md)

### DNS

* [DNS Solutions Overview](./install/dns/dns.md)
* [systemd-resolved](./install/dns/systemd-resolved.md)
* [dnsmasq](./install/dns/dnsmasq.md)
* [Unbound](./install/dns/unbound.md)
* [Unbound + dnsproxy](./install/dns/dnsproxy-unbound.md)
* [NextDNS](./install/dns/secure-dns.md)
* [Cloudflare WARP](./install/dns/cloudflare-warp.md)

## Recommended Paths

### Desktop / Laptop

1. Install Arch Linux
2. Create a regular user
3. Configure ZRAM
4. Enable Secure Boot
5. Configure TPM2 auto-unlock
6. Configure NextDNS
7. Configure SSH (optional)

### Server

1. Install Arch Linux
2. Configure static networking
3. Create administrative user
4. Configure SSH
5. Configure firewall
6. Configure Unbound DNS
7. Enable Secure Boot
8. Configure TPM2 auto-unlock

## Security Features

| Feature            | Supported |
| ------------------ | --------- |
| UEFI               | ✓         |
| LUKS2 Encryption   | ✓         |
| Secure Boot        | ✓         |
| TPM2 Auto Unlock   | ✓         |
| DNSSEC             | ✓         |
| DNS-over-TLS       | ✓         |
| Recovery Snapshots | ✓         |
