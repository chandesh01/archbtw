# Configure Static IP with `systemd-networkd`

Assumes the following variables are already defined:

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

Create a static network configuration using `systemd-networkd` as the renderer:

```sh
sudo tee /etc/netplan/01-network.yaml >/dev/null <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    ${IFACE}:
      dhcp4: false
      addresses:
        - ${IPADDR}/${PREFIXLEN}
      routes:
        - to: default
          via: ${GATEWAY}
      nameservers:
        addresses:
          - ${DNS1}
          - ${DNS2}
EOF
```

Apply:

```sh
sudo netplan apply
```

## Direct `systemd-networkd`

Create the network configuration:

```sh
sudo mkdir -p /etc/systemd/network

sudo tee /etc/systemd/network/10-static.network >/dev/null <<EOF
[Match]
Name=${IFACE}
MACAddress=${MAC}

[Link]
RequiredForOnline=routable

[Network]
Address=${IPADDR}/${PREFIXLEN}
Gateway=${GATEWAY}
DNS=${DNS1}
DNS=${DNS2}
MulticastDNS=yes
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
networkctl status "${IFACE}"

ip addr show "${IFACE}"
ip route
```

## Test Connectivity

```sh
# Test IP connectivity
ping -c 3 1.1.1.1

# Test DNS resolution
ping -c 3 google.com
```

## Notes

* Matching both `Name=` and `MACAddress=` ensures the configuration is applied to the correct interface.
* `MulticastDNS=yes` enables `.local` hostname resolution on local networks.
* The same configuration can be created from a live environment, chroot, or installed system.
* `systemd-resolved` manages DNS and should be used together with `systemd-networkd`.
