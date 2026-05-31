## Host Machine Setup
```sh
pacman -S openssh
sed -i 's/^#Port 22/Port 2222/' /etc/ssh/sshd_config
systemctl enable --now sshd
passwd
```

## (Optional) Allow Root Login (Not Recommended)
```sh
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
systemctl restart sshd
```

## Configure Firewall (If Enabled)
```sh
iptables -A INPUT -p tcp --dport 2222 -j ACCEPT
ufw allow 2222/tcp
```

## Verify SSH on Host Machine
```sh
ssh -p 2222 username@localhost
```

## Get Host Machine IP Address
```sh
ip a | grep "inet"
```

## Generate SSH Key on Guest Machine
```sh
mkdir -p ~/.ssh && chmod 700 ~/.ssh
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

## Transfer SSH Key to Host Machine
```sh
ssh-copy-id -p 2222 username@<host-ip>
```
*OR manually copy the public key:*
```sh
scp -P 2222 ~/.ssh/id_rsa.pub username@<host-ip>:~/
ssh -p 2222 username@<host-ip> "mkdir -p ~/.ssh && cat ~/id_rsa.pub >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && rm ~/id_rsa.pub"
```

## Disable Password Authentication on Host Machine
```sh
sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd
```

## Connect Using Key Authentication
```sh
ssh -p 2222 username@<host-ip>
```

## (Optional) Additional Security Enhancements

### Change Default SSH Banner
```sh
echo "Welcome to Secure SSH" | sudo tee /etc/issue.net
sed -i 's|^#Banner none|Banner /etc/issue.net|' /etc/ssh/sshd_config
systemctl restart sshd
```

### Restrict SSH to Specific Users
```sh
echo "AllowUsers user1 user2 user3" | sudo tee -a /etc/ssh/sshd_config
systemctl restart sshd
```

### Limit SSH Login Attempts (Prevent Brute Force)
```sh
echo "MaxAuthTries 3" | sudo tee -a /etc/ssh/sshd_config
systemctl restart sshd
```
