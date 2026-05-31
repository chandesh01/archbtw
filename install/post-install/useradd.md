# User Setup

> **Note:** Replace `newuser` with your actual username before running the commands.

## Create a New User and Set Password
```sh
useradd -m -G wheel newuser
passwd newuser
```
- *Remove password `sudo passwd -d user`*
- *Lock User `sudo passwd -l user`*
- *Unlock User `sudo passwd -u username`* 
## Sudo User Setup
### Install sudo (if not installed)
```sh
pacman -S sudo
```

### Grant Sudo Access

#### Using `visudo` (Recommended)
```sh
EDITOR=nano visudo
```
Uncomment this line:
```sh
%wheel ALL=(ALL:ALL) ALL
```
Save and exit (`Ctrl + X`, then `Y` and `Enter`).

#### Using a `sudoers.d` File
```sh
echo "newuser ALL=(ALL) ALL" | tee /etc/sudoers.d/newuser
chmod 0440 /etc/sudoers.d/newuser
```

### Add an Existing User to the `wheel` Group
```sh
usermod -aG wheel your_username
```

### Verify Sudo Access
Switch to the new user:
```sh
su - newuser
```
Test sudo:
```sh
sudo whoami
```
It should return:
```
root
```

## Doas User Setup

### Install Doas (if not installed)
```sh
pacman -S opendoas
```

### Grant Doas Access
```sh
echo "permit persist newuser as root" | tee /etc/doas.conf
```
Set the correct permissions:
```sh
chmod 0400 /etc/doas.conf
```

### Verify Doas Access
Switch to the new user:
```sh
su - newuser
```
Test doas:
```sh
doas whoami
```
It should return:
```sh
root
```
