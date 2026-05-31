# Post-Install Tasks

## Add Users

[`Add Users`](./post-install/useradd.md)

Create regular user accounts and configure sudo access. Daily system usage should be performed from a non-root account.

## Swap and ZRAM

Configure compressed RAM-based swap (ZRAM) to improve memory usage and reduce disk swapping on systems with limited RAM.

Start ZRAM:

```sh
systemctl daemon-reload
systemctl enable systemd-zram-setup@zram0.service
```

Verify swap devices:

```sh
swapon --show
zramctl
```

## Setup Firewall

[`Setup Firewall`](./post-install/firewall.md)

Enable and configure a firewall to restrict unwanted network access while allowing required services.

## Setup Recovery

[`Setup Recovery`](./post-install/recovery.md)

Create a Btrfs recovery snapshot and add a boot entry for it. This provides a fallback environment if the main system becomes unbootable.

## Setup Secure Boot

[`Setup Secure Boot`](./post-install/secure_boot.md)

[`Setup Secure Boot using Shim`](./post-install/secure-boot-shim.md)

Enable UEFI Secure Boot by signing boot components and enrolling custom keys. This helps protect the boot process from unauthorized modifications.

## Setup TPM for LUKS

[`Setup TPM for LUKS`](./post-install/tpm_luks.md)

Bind the LUKS disk encryption key to the TPM, allowing automatic unlocking on trusted hardware while keeping data encrypted at rest.

## Setup Network for Server

[`Server Network Setup`](./post-install/server.md)

Configure a static IP address, hostname, DNS resolution, SSH access, and firewall rules for server deployments.

## Start SSH

[`SSH`](./post-install/ssh.md)

Enable and start the SSH service for remote administration and file transfers.

## Setup Secure DNS

[`DNS Solutions`](./dns/dns.md)

Configure encrypted DNS, DNSSEC validation, DNS caching, ad blocking, or self-hosted DNS infrastructure.

Recommended:

- **Desktop/Laptop:** NextDNS
- **Server/Homelab:** Unbound
- **Advanced Users:** Unbound + dnsproxy