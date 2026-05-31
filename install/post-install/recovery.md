# Recovery using btrfs Snapshot

## Set Disk Variables

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

## Recovery Model

```text
Normal boot: subvol=@ read-write

Recovery boot: subvol=@snapshots/recovery read-only
```

## Install systemd-boot

```sh
sudo bootctl install
```

Verify:

```sh
bootctl status
```

Configure Loader

Create:

```text
/efi/loader/loader.conf
```

Contents:

```ini
default @saved
timeout 5
console-mode max
editor no
auto-firmware yes
```

## Configure Kernel Cmdline

Main Kernel

```sh
tee /etc/kernel/cmdline >/dev/null <<EOF
rd.luks.name=$(blkid -s UUID -o value $ROOT_DISK)=root root=/dev/mapper/root rootflags=subvol=@ rw noatime resume=/dev/mapper/root resume_offset=$(btrfs inspect-internal map-swapfile -r /swap/swapfile)
EOF
```

Recovery Kernel

```sh
sudo tee /etc/kernel/cmdline_recovery >/dev/null <<EOF
rd.luks.name=$(blkid -s UUID -o value $ROOT_DISK)=root root=/dev/mapper/root rootflags=subvol=@snapshots/recovery ro noatime resume=/dev/mapper/root resume_offset=$(btrfs inspect-internal map-swapfile -r /swap/swapfile)
EOF
```

Recovery boots readonly.

## Configure mkinitcpio For Dual UKIs

```sh
# /etc/mkinitcpio.d/linux.preset
cp /etc/mkinitcpio.d/linux.preset{,.old}

cat > /etc/mkinitcpio.d/linux.preset <<EOF
ALL_kver="/boot/vmlinuz-linux"

PRESETS=("default" "recovery")

# Main UKI
default_uki="/efi/EFI/Linux/ArchLinux-linux.efi"
default_options="--cmdline /etc/kernel/cmdline --splash /usr/share/systemd/bootctl/splash-arch.bmp"

# Recovery UKI
recovery_uki="/efi/EFI/Linux/ArchLinux-recovery.efi"
recovery_options="--cmdline /etc/kernel/cmdline_recovery --splash /usr/share/systemd/bootctl/splash-arch.bmp"
EOF
```

## Generate UKIs

```sh
sudo mkinitcpio -P
```

Verify:

```sh
ls -lh /efi/EFI/Linux/
```

Sign and EFI binaries(If Secure Boot is Configured):

```sh
sudo sbctl sign -s /efi/EFI/Linux/ArchLinux-recovery.efi
sbctl verify
```

## Create Initial Recovery Snapshot

Mount Btrfs top-level subvolume:

```sh
sudo mount -o subvolid=5 /dev/mapper/root /mnt
```

Create readonly recovery snapshot:

```sh
sudo btrfs subvolume snapshot -r /mnt/@ /mnt/@snapshots/recovery
```

Verify:

```sh
sudo btrfs subvolume list /mnt
```

Expected:

```text
path @snapshots/recovery
```

Unmount:

```sh
sudo umount /mnt
```

## Automatic Recovery Snapshot Updates

Create:

```text
/usr/local/bin/update-recovery-snapshot
```

Contents:

```bash
#!/usr/bin/env bash

set -Eeuo pipefail

ROOT_DEV="/dev/mapper/root"

mkdir -p /mnt

# Mount Btrfs top-level
mount -o subvolid=5 "${ROOT_DEV}" /mnt

# Remove previous recovery snapshot
if [[ -d /mnt/@snapshots/recovery ]]; then
    btrfs subvolume delete /mnt/@snapshots/recovery
fi

# Create new readonly recovery snapshot
btrfs subvolume snapshot -r /mnt/@ /mnt/@snapshots/recovery

# Cleanup mount
umount /mnt


echo "Recovery snapshot updated successfully"
```

Make executable:

```sh
sudo chmod +x /usr/local/bin/update-recovery-snapshot
```

## Create Pacman Hook

Create:

```text
/etc/pacman.d/hooks/50-recovery-snapshot.hook
```

Contents:

```ini
[Trigger]
Operation = Install
Operation = Upgrade
Operation = Remove
Type = Package
Target = *

[Action]
Description = Updating recovery snapshot...
When = PreTransaction
Exec = /usr/local/bin/update-recovery-snapshot
```

## Optional EFI Backup

Create backup directory:

```sh
sudo mkdir -p /root/efi-backup
```

Create:

```text
/etc/pacman.d/hooks/95-efi-backup.hook
```

Contents:

```ini
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = linux
Target = systemd

[Action]
Description = Backing up EFI partition...
When = PostTransaction
Exec = /usr/bin/rsync -aHAX --delete /efi/ /root/efi-backup/
```

## Verify Boot Entries

```sh
bootctl list
```

Expected:

```text
Arch Linux
Arch Linux Recovery
```
## Recovery Workflow

If update breaks system:

1. Reboot
2. Select:

```text
Arch Linux Recovery
```

3. Verify snapshot boot:

```sh
findmnt -no OPTIONS /
```

Expected:

```text
subvol=/@snapshots/recovery
```

4. Mount main system manually if needed:

```sh
sudo mount -o subvol=@,rw /dev/mapper/root /mnt
```

5. Repair system.

## Verify Current Root

```sh
findmnt -no OPTIONS /
```

Normal boot:

```text
subvol=/@
```

Recovery boot:

```text
subvol=/@snapshots/recovery
```
## Auto Recovery

If the system becomes unbootable after an update:

1. Reboot the system.
2. Select:

```text
Arch Linux Recovery
```

from the systemd-boot menu.

The recovery entry boots the readonly snapshot:

```text
@snapshots/recovery
```

Verify:

```sh
findmnt -no OPTIONS /
```

Expected:

```text
subvol=/@snapshots/recovery
```

### Restore Recovery Snapshot

Run:

```sh
sudo rollback-to-recovery
```

Script:

```bash
#!/usr/bin/env bash

set -Eeuo pipefail

ROOT_DEV="/dev/mapper/root"

mkdir -p /mnt

mount -o subvolid=5 "$ROOT_DEV" /mnt

mv /mnt/@ /mnt/@broken

btrfs subvolume snapshot \
    /mnt/@snapshots/recovery \
    /mnt/@

umount /mnt

echo "Rollback complete. Reboot into Arch Linux."
```

The script:

1. Mounts the Btrfs top-level subvolume.
2. Renames the current root subvolume to `@broken`.
3. Creates a new writable `@` from the recovery snapshot.
4. Unmounts the filesystem.

### Reboot

```sh
sudo reboot
```

Select:

```text
Arch Linux
```

### Verify Recovery

```sh
findmnt -no OPTIONS /
```

Expected:

```text
subvol=/@
```

The system is now running from the restored root filesystem.

### Cleanup

After confirming everything works:

```sh
sudo mount -o subvolid=5 /dev/mapper/root /mnt

sudo btrfs subvolume delete /mnt/@broken

sudo umount /mnt
```

This permanently removes the previously broken root subvolume.
