# Step-by-Step UFW Setup on Arch Linux

1. Install UFW
```sh
sudo pacman -S ufw
```
2. Enable UFW at Boot
```sh
sudo systemctl enable ufw
```
3. Set Default Policies
```sh
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
- This blocks all incoming connections unless explicitly allowed.
- Outgoing connections are allowed by default (you can change this later if needed).

4. Allow Essential Services
- Allow SSH (so you don’t lock yourself out if you're using remote access):
```
sudo ufw allow ssh
sudo ufw allow 22/tcp
sudo ufw allow 2222/tcp

# HTTP
sudo ufw allow 80
# HTTPS
sudo ufw allow 443
```
5. Enable UFW
```sh
sudo ufw enable
```

6. Check Status
```sh
sudo ufw status verbose
```
### Optional: GUI for UFW
```sh
sudo pacman -S gufw
gufw
```
### Common UFW Commands
- Enable UFW:	`sudo ufw enable`
- Disable UFW:	`sudo ufw disable`
- Reset Rules:	`sudo ufw reset`
- Allow Port:	`sudo ufw allow 8080`
- Deny Port:	`sudo ufw deny 23`
- Delete Rule:	`sudo ufw delete allow 22`
- Check Status:	`sudo ufw status verbose`
