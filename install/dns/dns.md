# DNS Solutions

Arch Linux supports multiple DNS configurations ranging from simple encrypted DNS providers to fully self-hosted recursive resolvers.

This guide helps you choose the right solution for your use case.

## Quick Recommendations

| Use Case | Recommended Solution |
|-----------|-----------|
| Most desktop users | [NextDNS](./secure-dns.md) |
| Cloudflare ecosystem users | [Cloudflare WARP](./cloudflare-warp.md) |
| Local DNS server / Homelab | [Unbound](./unbound.md) |
| Self-hosted recursive DNS with encrypted upstreams | [Unbound + dnsproxy](./dnsproxy-unbound.md) |
| Local DNS caching only | [dnsmasq](./dnsmasq.md) |
| Standard Linux resolver | [systemd-resolved](./systemd-resolved.md) |

---

# systemd-resolved

[`Setup Guide`](./systemd-resolved.md)

The default resolver used by modern Linux distributions.

### Advantages

- Integrated with systemd
- Local DNS caching
- DNSSEC support
- DNS-over-TLS support
- Split DNS support
- Works seamlessly with NetworkManager

### Disadvantages

- Not a recursive resolver
- Limited advanced DNS features
- No ad blocking

### Recommended For

- Desktop systems
- Laptops
- General-purpose installations

---

# NextDNS

[`Setup Guide`](./secure-dns.md)

A cloud-hosted DNS service providing encrypted DNS and filtering.

### Advantages

- Very easy setup
- Ad and tracker blocking
- Malware protection
- Analytics dashboard
- DNS-over-HTTPS and DNS-over-TLS
- Family filtering

### Disadvantages

- Relies on a third-party provider
- Requires a NextDNS account for advanced features
- Not self-hosted

### Recommended For

- Most desktop users
- Families
- Users who want DNS filtering
- Users who want encrypted DNS with minimal maintenance

---

# Cloudflare WARP

[`Setup Guide`](./cloudflare-warp.md)

Cloudflare's encrypted DNS and optional VPN service.

### Advantages

- Easy setup
- Encrypted DNS
- Optional VPN tunnel
- Malware filtering options
- Good performance

### Disadvantages

- Uses Cloudflare infrastructure
- Limited customization
- No local DNS server functionality

### Recommended For

- Mobile devices
- Laptops
- Users already using Cloudflare services

---

# dnsmasq

[`Setup Guide`](./dnsmasq.md)

A lightweight DNS forwarder and cache.

### Advantages

- Very lightweight
- Local DNS cache
- Local DNS records
- Simple configuration
- Popular for routers and small networks

### Disadvantages

- Not a recursive resolver
- Limited DNSSEC features compared to Unbound
- Requires upstream DNS servers

### Recommended For

- Small networks
- Local DNS overrides
- DNS caching
- Router deployments

---

# Unbound

[`Setup Guide`](./unbound.md)

A validating, recursive DNS resolver.

Instead of relying on Cloudflare, Google, or NextDNS, Unbound can perform DNS recursion directly.

### Advantages

- Recursive DNS
- DNSSEC validation
- Local DNS cache
- Self-hosted
- Local DNS zones
- Split-horizon DNS
- Excellent privacy

### Disadvantages

- More complex than public DNS services
- No built-in ad blocking
- Requires maintenance

### Recommended For

- Homelabs
- NAS users
- Self-hosted services
- Local DNS infrastructure
- Privacy-focused users

---

# Unbound + dnsproxy

[`Setup Guide`](./dnsproxy-unbound.md)

Combines Unbound with AdGuard DNS Proxy.

### Advantages

- Recursive DNS
- DNSSEC validation
- DNS-over-HTTPS (DoH)
- DNS-over-TLS (DoT)
- DNS-over-QUIC (DoQ)
- Self-hosted
- Local DNS records

### Disadvantages

- Most complex setup
- Higher maintenance requirements
- More components to troubleshoot

### Recommended For

- Advanced users
- Homelabs
- Self-hosted infrastructure
- Privacy-focused deployments

---

# Comparison

| Feature | systemd-resolved | NextDNS | WARP | dnsmasq | Unbound | Unbound + dnsproxy |
|----------|----------|----------|----------|----------|----------|----------|
| Easy Setup | Yes | Excellent | Excellent | Yes | Moderate | Advanced |
| DNSSEC | Yes | Yes | Yes | Partial | Yes | Yes |
| DoH/DoT | Yes | Yes | Yes | No | Optional | Yes |
| Ad Blocking | No | Yes | Optional | No | No | Optional |
| Recursive DNS | No | No | No | No | Yes | Yes |
| Local DNS Records | No | No | No | Yes | Yes | Yes |
| Split DNS | Yes | Limited | No | Limited | Yes | Yes |
| Self Hosted | Yes | No | No | Yes | Yes | Yes |
| Privacy | Good | Good | Good | Good | Excellent | Excellent |
| Maintenance | Low | Low | Low | Low | Medium | High |

## Recommendation

### Most Users

Use:

```text
NextDNS
```

### Desktop and Laptop Systems

Use:

```text
systemd-resolved + NextDNS
```

### Homelabs and Self-Hosted Services

Use:

```text
Unbound
```

### Maximum Control and Privacy

Use:

```text
Unbound + dnsproxy
```

### Simple Local DNS Cache

Use:

```text
dnsmasq
```