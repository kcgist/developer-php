# Access a remote VM behind a firewall via SSH

To allow another user (laptop) to SSH into a Virtual Machine (VM) through a public server (example.com), you can set up SSH port forwarding using the `ssh` command. This is sometimes referred to as SSH tunneling.

## Requirements

All ssh access is done with public keys. No passwords are used. Key pairs need to be configured before starting.

- A public server with a public IP address (example.com)
- A registered domain name pointing to the server (example.com)
- SSH server installed and running
- SSH access with a public key

- A VM behind a firewall
- A laptop with SSH access to the public server
- A private key to access the VM
- A user on the VM with sudo privileges

## VM and Server

Configure SSH server to allow remote port forwarding.

- Add the following line to /etc/ssh/sshd_config on the server:
```
GatewayPorts yes
AllowTcpForwarding yes
```

- Restart the ssh server:
```
sudo systemctl restart ssh
```

### Install autossh (VM)

autossh is a program to start a copy of ssh and monitor it, restarting it as necessary should it die or stop passing traffic.

- Install autossh:
```
sudo apt-get install autossh
```

#### Create a script to start the reverse ssh tunnel on the VM.

- Create a new file:
```
nano ~/ssh-reverse-tunnel.sh
```

- Add the following lines to the file:
```bash
#!/bin/bash
autossh -M 0 -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -N -f -R 2223:127.0.0.1:22 user@example.com -i /path/to/private/key
```

- Make the file executable:
```
chmod +x ~/ssh-reverse-tunnel.sh
```

- Test the script:
```
~/ssh-reverse-tunnel.sh
```

- Check that the tunnel is running on the public server:
```
sudo netstat -tulpn | grep 2223
```

- If the tunnel is running, add the script to crontab:
```
crontab -e
```

- Add the following line to the crontab:
```
@reboot /path/to/user/home/ssh-reverse-tunnel.sh
```

- Reboot the computer:
```
sudo reboot
```

- Check that the tunnel is running on the public server:
```
sudo netstat -tulpn | grep 2223
```

## From the Laptop:

- Create a reverse ssh tunnel from the laptop to the public server:

```bash
ssh -N -R 2223:localhost:2223
```
- `-N`: Tells SSH that no command will be sent once the tunnel is up.
- `-R 2223:localhost:2223`: Specifies that the port 2223 on the public server should be forwarded to the port 2223 on the laptop.

You can now long into the VM from the laptop

```bash
ssh -p 2223 dev@localhost
```

- `-p 2223`: Specifies the port (same as the one you used in the local forwarding).
- `dev`: Is the user on the VM.
- `@localhost`: Is your laptop which is forwarded from the VM.

This will create a tunnel from your laptop to the public server, and then to the VM. It effectively allows your laptop to communicate with the VM via the public server.

### Install autossh on the laptop (Recommended)

autossh is a program to start a copy of ssh and monitor it, restarting it as necessary should it die or stop passing traffic. It is recommended to use autossh to start the tunnel on the laptop when you want to mount the VM's filesystem on the laptop.

  On Ubuntu:
```
sudo apt-get install autossh
```

On Mac:
```
brew install autossh
```

```sh
autossh -M 0 -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -N -f -R 2223:localhost:2223 user@example.com -i /path/to/private/key
```

To kill the tunnel:
```sh
sudo killall autossh
```

### Mount the VM's filesystem on the laptop with sshfs (Optional)

- Install sshfs on ubuntu:
```
sudo apt-get install sshfs
```

- Install sshfs on Mac:
```
brew install sshfs
```

- Create a mount point:
```
mkdir ~/vm/html
```

- Mount the VM's filesystem:
```
sshfs -p 2223 dev@localhost:/var/www/html ~/vm/html -o allow_other,auto_cache,reconnect,follow_symlinks,noappledouble,volname=VM
```

- Unmount the VM's filesystem:
```
fusermount -u ~/vm/html
```

### Create some aliases for the tunnel (Optional)

For convience, you can create some aliases for the tunnel and the sshfs mount.

- Create a new file:
```
nano ~/.bash_aliases
```

- Add the following lines to the file:
```
alias ssh-vm='ssh -p 2223 ubuntu@localhost -i ~/.ssh/key -vvvv'
alias mount-vm='sshfs -oIdentityFile=~/.ssh/key  -p 2223 ubuntu@localhost:/var/www/html ~/vm/html  -o allow_other,auto_cache,reconnect,follow_symlinks,noappledouble,volname=VM'
alias unmount-vm='fusermount -u ~/vm'
alias kill-tunnel='sudo killall autossh'
alias start-tunnel='autossh -M 0 -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -N -f -R 2223:localhost:2223 -i ~/.ssh/key

```

### Deploy files from VM to Production

In the VM create a config file in thge `~.ssh/config` with the following contents

```sh
Host prod
    HostName example.com
    Port 22
    User ubuntu
    IdentityFile ~/.ssh/keyfile
```


```sh
rsync -avz -e ssh /var/www/html/project/ ubuntu@prod:/var/www/html/project/
```
