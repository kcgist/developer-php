# Settings for PHP developers

Custom settings for PHP web development on remote servers via SSH.

[Convert Apache2 vhosts to run multiple versions of PHP using PHP-FPM on Ubuntu 20.04](./multi-php.md)
Convert existing Apache2 vhosts running php7.2 installed from the SURY repository to use PHP-FPM on Ubuntu 20.04 So that we can update to PHP8.1 and run both versions of PHP on the same server.

[Access a remote VM behind a firewall via SSH](./ssh.md)
To allow another user (laptop) to SSH into a Virtual Machine (VM) through a public server (example.com), you can set up SSH port forwarding using the `ssh` command. This is sometimes referred to as SSH tunneling.

[Install+config Xdebug and PHPStorm w/SSH tunnel for remote debugging](./xdebug.md)
This guide will show you how to install and configure Xdebug for remote debugging using a SSH tunnel.

[SSH Reverse Tunnels for Local Web Development with SSL](./reverse_web.md)
How to expose a locally hosted apache server to the internet via a reverse ssh tunnel for local web development with SSL.

## Common settings for SSH

Many of these guides require modifying the ssh server. This is a combo setting for all of these guides.

```bash
sudo nano /etc/ssh/sshd_config
```

```bash
AllowTcpForwarding yes
PermitTunnel yes
GatewayPorts yes
```

Save and close the file. To apply the changes, restart SSH:

```bash
sudo systemctl restart ssh
```
