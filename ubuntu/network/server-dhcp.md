# Configure DHCP with `systemd-networkd`

# Assign Variables:
```sh
export HOSTNAME=linux
export IFACE=eth0

export IPADDR=192.168.1.100
export PREFIXLEN=24
export GATEWAY=192.168.1.1

export DNS1=1.1.1.1
export DNS2=8.8.8.8

MAC=$(cat /sys/class/net/${IFACE}/address)
```

## Netplan (Ubuntu)

Create a DHCP configuration using `systemd-networkd` as the renderer:

```sh
sudo tee /etc/netplan/01-network.yaml >/dev/null <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    ${IFACE}:
      dhcp4: true
EOF
```

Apply:

```sh
sudo netplan apply
```

## Direct `systemd-networkd`

Create a generic Ethernet DHCP profile:

```sh
sudo mkdir -p /etc/systemd/network

sudo tee /etc/systemd/network/20-ethernet.network >/dev/null <<'EOF'
[Match]
Name=en*
Name=eth*

[Link]
RequiredForOnline=routable

[Network]
DHCP=yes
MulticastDNS=yes

[DHCPv4]
RouteMetric=100

[IPv6AcceptRA]
RouteMetric=100
EOF
```

Optional Wi-Fi profile:

```sh
sudo tee /etc/systemd/network/20-wlan.network >/dev/null <<'EOF'
[Match]
Name=wl*

[Link]
RequiredForOnline=routable

[Network]
DHCP=yes
MulticastDNS=yes

[DHCPv4]
RouteMetric=600

[IPv6AcceptRA]
RouteMetric=600
EOF
```

Optional WWAN profile:

```sh
sudo tee /etc/systemd/network/20-wwan.network >/dev/null <<'EOF'
[Match]
Name=ww*

[Link]
RequiredForOnline=routable

[Network]
DHCP=yes

[DHCPv4]
RouteMetric=700

[IPv6AcceptRA]
RouteMetric=700
EOF
```

## Enable Networking

Disable conflicting network managers:

```sh
sudo systemctl disable --now NetworkManager networking 2>/dev/null || true
```

Enable `systemd-networkd` and `systemd-resolved`:

```sh
sudo systemctl enable --now systemd-networkd
sudo systemctl enable --now systemd-resolved
```

Optional:

```sh
sudo systemctl enable systemd-networkd-wait-online
```

## Configure DNS

Create the resolver symlink:

```sh
sudo rm -f /etc/resolv.conf
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

Verify:

```sh
resolvectl status
```

## Verify Connectivity

```sh
networkctl list
networkctl status

ip addr
ip route
```

Test connectivity:

```sh
ping -c 3 1.1.1.1
ping -c 3 google.com
```

## Notes

* Interface priority:

  * Ethernet → `100`
  * Wi-Fi → `600`
  * WWAN → `700`
* `MulticastDNS=yes` enables `.local` hostname resolution.
* The same configuration works in installed systems, live environments, and chroots (provided `systemd-networkd` will run on the target system after boot).
* The variables `IPADDR`, `PREFIXLEN`, `GATEWAY`, `DNS1`, and `DNS2` are not used for DHCP configuration; they are only needed for static addressing.
