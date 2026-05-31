# Arch Linux Installation (luks2 + btrfs + uki + systemd-boot)

Boot the live media.

## Verify the boot mode

Confirm target device is using UEFI boot mode:

```sh
cat /sys/firmware/efi/fw_platform_size
```

If the command returns 64, then system is booted in UEFI with 64-bit x64 UEFI.

## Connect to a Wi-Fi Network

Connect using [iwctl](https://wiki.archlinux.org/title/Iwd#iwctl):

```sh
iwctl station wlan0 scan
iwctl station wlan0 connect "SSID"
```

## Check connectivity

```sh
ip addr
ping -c 5 archlinux.org
```

## Setup `ssh` connectivity (Optional)

Install, allow login with password and start the service:

```sh
pacman -S openssh
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
systemctl start sshd
```

Set a password for root:

```sh
passwd
```

Switch to the other computer and ssh into the target device:

```
ssh root@[ip_address]
```

`[ip_address]` is the target device's address obtained with the `# ip addr`

## Update the system clock:

```sh
timedatectl
```

## Prepare Disk for Installation (UEFI)

### List available disks

```sh
lsblk -f
```

### Set Disk Variables

Set disk variables for either a SATA or NVMe disk.

SATA

Example disk: `sda`

```sh
export DISK="/dev/sda"
export ESP_PART="1"
export ROOT_PART="2"

export ESP_DISK="${DISK}${ESP_PART}"
export ROOT_DISK="${DISK}${ROOT_PART}"
```

NVMe

Example disk: `nvme0n1`

```sh
export DISK="/dev/nvme0n1"
export ESP_PART="1"
export ROOT_PART="2"

export ESP_DISK="${DISK}p${ESP_PART}"
export ROOT_DISK="${DISK}p${ROOT_PART}"
```

### Erase DISK (Optional)

Deactivate all active LVM volume groups to avoid:

- `device busy`
- `cannot wipe`
- `cannot re-read partition table`

```sh
vgchange -an
```

Erase existing filesystems and partition tables:

```sh
wipefs -af "$DISK" && sgdisk --zap-all --clear "$DISK"
```

Notify the kernel about partition table changes:

```sh
partprobe "$DISK"
```

### Partition DISK

Create a GPT partition table with:

| Partition                | Purpose               | Mountpoint | Device      |
| ------------------------ | --------------------- | ---------- | ----------- |
| EFI System Partition     | Bootloader / UKI      | `/efi`     | `/dev/sda1` |
| LUKS encrypted partition | Btrfs root filesystem | `/`        | `/dev/sda2` |

### Create Partitions

Using `sgdisk`

```sh
sgdisk -n "${ESP_PART}:0:+1G" -t "${ESP_PART}:EF00" -c "${ESP_PART}:ESP" -n "${ROOT_PART}:0:+64G" -t "${ROOT_PART}:8309" -c "${ROOT_PART}:cryptroot" "$DISK"
```

> Replace `+64G` with `0` to use the remaining disk space.

Display the resulting layout:

```sh
partprobe "$DISK" && sgdisk -p "$DISK"
```

or, Using `gdisk`

```sh
gdisk /dev/sda
```

Steps:

```text
o → y
n → Enter → Enter → +1G → EF00
n → Enter → Enter → +64G → 8309
w → Enter
```

> Replace `+64G` with `0` to use the remaining disk space.

### Format the Partitions

Create FAT32 EFI System Partition

```sh
mkfs.fat -F32 "$ESP_DISK"
```

Encrypt Root Partition

```sh
cryptsetup -y luksFormat "$ROOT_DISK"
```

Open the encrypted container:

```sh
cryptsetup open "$ROOT_DISK" root
```

Export the mapped device:

```sh
export ROOT_DEV="/dev/mapper/root"
```

Create Btrfs Filesystem

```sh
mkfs.btrfs -L root "$ROOT_DEV"
```

### Mount the Filesystem

Mount the root filesystem temporarily:

```sh
mount "$ROOT_DEV" /mnt
```

Create Btrfs Subvolumes

Create subvolumes:

```sh
for subvol in @ @home @pkg @log @var_tmp @srv @snapshots @swap; do
    btrfs subvolume create /mnt/$subvol
done
```

Unmount the temporary mount:

```sh
umount -R /mnt
```

### Remount Subvolumes

Mount Options

```sh
export S_OP="noatime,compress=zstd,discard=async"
export H_OP="nodev,nosuid,noexec"
```

Mount Root

```sh
mount -o ${S_OP},subvol=@ "$ROOT_DEV" /mnt
```

Mount Additional Subvolumes

```sh
mount --mkdir -o ${S_OP},subvol=@home "$ROOT_DEV" /mnt/home

mount --mkdir -o ${S_OP},${H_OP},subvol=@pkg "$ROOT_DEV" /mnt/var/cache/pacman/pkg
mount --mkdir -o ${S_OP},nodev,nosuid,subvol=@log "$ROOT_DEV" /mnt/var/log
mount --mkdir -o ${S_OP},${H_OP},subvol=@var_tmp "$ROOT_DEV" /mnt/var/tmp
mount --mkdir -o ${S_OP},nosuid,subvol=@srv "$ROOT_DEV" /mnt/srv
mount --mkdir -o ${S_OP},${H_OP},subvol=@snapshots "$ROOT_DEV" /mnt/.snapshots
mount --mkdir -o subvol=@swap "$ROOT_DEV" /mnt/swap
```

Disable Copy-on-Write for future swapfiles:

```sh
chattr +C /mnt/swap
```

### Mount EFI Partition

```sh
mount --mkdir "$ESP_DISK" /mnt/efi
```

Verify mounts:

```sh
df -h
```

## Update pacman mirrors

Backup the existing mirrorlist:

```sh
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
```

Synchronize the pacman package databases using the new mirror list:

```sh
pacman -Syyu
```

## Installation

Identify the processor vendor and set the ucode package name:

```sh
export UCODE=$(grep -qm1 GenuineIntel /proc/cpuinfo && echo intel || echo amd)-ucode
```

Choose a text editor

```sh
export EDIT="nano"
```

Install packages using `pacstrap`

```sh
pacstrap -K /mnt base base-devel linux linux-firmware  $UCODE $EDIT man-db btrfs-progs exfatprogs dosfstools e2fsprogs ntfs-3g openssh sudo
```

## Fstab

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

## Chroot

```sh
arch-chroot /mnt
```

## Time

Timezones are located in `usr/share/zoneinfo`.

List timezones:

```sh
timedatectl list-timezones
```

Setup Correct timezone:

```sh
timedatectl  set-timezone Region/City
# ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
```

Sync the system clock:

```sh
hwclock --systohc
```

Enable network time synchronization:

```sh
timedatectl set-ntp true
```

Show current time settings:

```sh
timedatectl
```

## Localization

```sh
echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen && locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "KEYMAP=us" > /etc/vconsole.conf
```

## SWAP and Zram

### Setup Swap with Hibernation Support

Create a persistent swapfile for hibernation (`suspend-to-disk`), memory overflow protection, resume support after reboot

```sh
btrfs filesystem mkswapfile --size 16g --uuid clear /swap/swapfile
swapon /swap/swapfile
echo "/swap/swapfile none swap defaults,pri=0 0 0" >> /etc/fstab
```

> Adjust `16g` according to your RAM size and hibernation requirements.

### Setup Zram

Zram provides compressed in-memory swap, faster swapping, lower SSD wear and improved responsiveness under memory pressure. Using **zram** for fast compressed swap **swapfile** for hibernation is recommended..

Install `zram-generator`

```sh
pacman -S --needed zram-generator
```

Create `zram-generator` Configuration

```sh
cat > /etc/systemd/zram-generator.conf <<EOF
[zram0]
zram-size = ram / 2
swap-priority = 100
compression-algorithm = lzo-rle zstd(level=3)
EOF
```

## Network and Bluetooth

### Desktop and Laptop

Configure NetworkManager, Wi-Fi, DNS resolution, and Bluetooth support.

```sh
echo "archlinux" > /etc/hostname

pacman -S --needed networkmanager iwd bluez bluez-utils

mkdir -p /etc/NetworkManager/conf.d

# Use iwd as the Wi-Fi backend
cat > /etc/NetworkManager/conf.d/wifi_backend.conf << EOF
[device]
wifi.backend=iwd
EOF

# Use systemd-resolved for DNS
cat > /etc/NetworkManager/conf.d/dns.conf << EOF
[main]
dns=systemd-resolved
EOF

# Configure resolv.conf
exit
ln -sf /mnt/run/systemd/resolve/stub-resolv.conf /mnt/etc/resolv.conf
arch-chroot /mnt
# Enable services
systemctl enable NetworkManager.service
systemctl enable bluetooth.service
systemctl enable systemd-resolved.service
```

Start services immediately:

```sh
systemctl start NetworkManager.service
systemctl start bluetooth.service
systemctl start systemd-resolved.service
```

Verify:

```sh
nmcli device status
bluetoothctl show
resolvectl status
```

### Custom DNS (Optional)

[`Setup Custom DNS`](./secure-dns.md)

Configure encrypted DNS, DNS-over-HTTPS (DoH), custom upstream resolvers while integrating with NetworkManager.

### Server

[`Server Network Setup`](./server-network.md)

Configure a static IP address, hostname, DNS resolution, SSH access, and firewall rules for server deployments.

## Modify mkinitcpio preset file

```sh
cp /etc/mkinitcpio.d/linux.preset{,.old}

cat > /etc/mkinitcpio.d/linux.preset <<EOF
ALL_kver="/boot/vmlinuz-linux"

PRESETS=("default")

default_uki="/efi/EFI/Linux/ArchLinux-linux.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

EOF
```

This backs up the original preset file and replaces it with a UKI-based configuration for mkinitcpio. It configures both normal and fallback Unified Kernel Images (UKIs) for Secure Boot with systemd-boot.

Generated file after mkinitcpio -P: `/efi/EFI/Linux/ArchLinux-linux.efi``

The .efi file contains the kernel, initramfs, microcode, and embedded boot parameters in a single bootable EFI image.

## Kernel Parameters

### Kernel Parameters (`encrypt` hook)

Update `HOOKS`

```sh
sed -i 's/^HOOKS=.*/HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt resume filesystems fsck)/' /etc/mkinitcpio.conf
```

Generate `/etc/kernel/cmdline`

```sh
echo "rd.luks.name=$(blkid -s UUID -o value $ROOT_DISK)=root rd.luks.options=$(blkid -s UUID -o value $ROOT_DISK)=tpm2-device=auto root=/dev/mapper/root rootflags=subvol=@ rw noatime resume=/dev/mapper/root resume_offset=$(btrfs inspect-internal map-swapfile -r /swap/swapfile)" | sudo tee /etc/kernel/cmdline
```

### Kernel Parameters (`sd-encrypt` hook)

Update `HOOKS`

```sh
sed -i 's/^HOOKS=.*/HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt  filesystems resume fsck)/' /etc/mkinitcpio.conf
```

Generate `/etc/kernel/cmdline`

```sh
echo "rd.luks.name=$(blkid -s UUID -o value $ROOT_DISK)=root root=/dev/mapper/root rootflags=subvol=/@,noatime resume=/dev/mapper/root resume_offset=$(btrfs inspect-internal map-swapfile -r /swap/swapfile)" > /etc/kernel/cmdline
```

- encrypt + udev uses the traditional mkinitcpio busybox-based initramfs hooks and kernel parameters like cryptdevice=.
- sd-encrypt + systemd uses systemd inside initramfs, supports rd.luks.\* parameters, and integrates better with modern UKI/systemd-boot setups.

## Add user

Create a New User and Set Password

```sh
useradd -m -G wheel newuser
passwd newuser
```

Install sudo (if not installed)

```sh
pacman -S sudo
```

Grant Sudo Access

```sh
EDITOR=nano visudo
```

Uncomment this line:

```sh
%wheel ALL=(ALL:ALL) ALL
```

## Generate UKI and setup boot

```sh
mkdir -p /efi/EFI/Linux
mkinitcpio -P
bootctl install
exit
reboot
```
