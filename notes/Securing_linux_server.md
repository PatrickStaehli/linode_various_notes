# Secure a Linux server
After deploying  a Linux distribution (as for example Ubuntu 20.04 LTS, as in this example), the unauthorized access needs to be prevented.

Sources:

- [Linode](https://www.linode.com/docs/guides/securing-your-server/)
- [NetworkChuck](https://www.youtube.com/watch?v=ZhMw53Ud2tY)
- [Ubuntu Help](https://help.ubuntu.com/community/AutomaticSecurityUpdates)

## #1: Automatic Security Updates
There is a discussion going on, whether automatic updates are useful or not [Pro and Cons about automatic updates](https://fedoraproject.org/wiki/AutoUpdates#Why_use_Automatic_updates.3F). 
Anyway, in my particular case, the pros seem to outweigh the cons.

#### Configuration
```
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

## #2: Add a limited User Account
Root has unlimited privileges and can execute any command. Therefore, it is recommended to create a limited user account and add it to the sudo group.

```
adduser <example_user>
adduser <example_user> sudo
```

Disconnect  ```exit``` and log back in as the new user: ```ssh <example_user>@<your_ip>```

## #3: Harden SSH Access
Password authentication to connect via SSH is used by default. This password, however, might be hacked via brute-force. A cryptographic key-pair is much more difficult to brute-force and is therefore more secure. If you want to know more about Public Key Cryptograpy these videos will give you an excellent overview 
- [Secret Key Exchange (Diffie-Hellman) - Computerphile](https://www.youtube.com/watch?v=NmM9HA2MQGI)
- [Diffie Hellman - the Mathematics bit- Computerphile](https://www.youtube.com/watch?v=Yjrfm_oRO0w&t=321s)
- [Public Key Cryptography - Computerphile](https://www.youtube.com/watch?v=GSIDS_lvRv4) 

In the following, a authentication key-pair is generated under Windows 10. If you are using a Linux system, the steps are exactely the same.

**Linode**
If there exits no ~/.ssh directory yet, we have to create it. In this directory (more exactely in a subdirectory, see later), all public-keys are located.

```
mkdir -p ~/.ssh && sudo chmod -R 700 ~/.ssh/
```


**Local Machine**
The key-pair is created on the local machine. The name and a passphrase can be defined when generating the key-pair. 
This key-pair is finally stored in `C:\Users\<user_name>\.ssh`. 
The next step is to upload the public key to the linode.
```
ssh-keygen -b 4096

scp C:\Users\<user_name>\.ssh/id_rsa.pub <example_user>@<your_ip>:~/.ssh/authorized_keys
```

## #4: Lockdown logins

#### Disallow root logins, disable SSH password authentication and listen only on one internet protocol
Root login is not needed anymore, since our limited user can access administrative privileges by using `sudo`. 
Further, we just created a cryptographic key-pair, so we do not need to be able to login via password anymore, unless we want to connect from a different local machine.
Finally, usually only one internet protocol is needed, e.g. IPv4. Thus, we disable the other option (IPv6)

All this is done by modifying the following file `/etc/ssh/sshd_config`

```
...
AddressFamily inet
...
PermitRootLogin no
...
PasswordAuthentication no
...
```

## #5: Fail2Ban
A server that gets spammed with unsuccessful logins indicates attempted malicious access. Fail2Ban bands IP addresses after too many failed login attempts

#### Installation
```
sudo apt update && apt upgrade -y
sudo apt install fail2ban
```
Allow SSH access through UFW and then enable the firewall:
```
sudo ufw allow ssh
sudo enable
```

#### Configuration
The file that contains the fail2ban configuration is located in `/etc/fail2ban/fail2ban.conf`. It is best practice to make a copy of it to `fail2ban.local`. Fail2ban will not overwrite this file, e.g when fail2ban is updated.


The IP address from which one is usually connecting can be ignored (will not be banned after too many login attempts) by editing `jail.local`. 
The bantime and findtime (certain failed login attempts within this time limit) can also be defined in this file. 
```
...
ignoreip = 127.0.0.1/8 ::1 <your_IP>
...
bantime = 10m
...
findtime = 10m
...
maxretry = 5
```

One can see which jails are active via:
```
sudo fail2ban-client status
``` 
