# SSH Reverse Tunnels for Local Web Development with SSL

How to expose a locally hosted apache server to the internet via a reverse ssh tunnel for local web development with SSL.

## Requirements

Server side:
- A server with a public IP address
- A registered domain name pointing to the server
- Apache2 installed and running
- SSH server installed and running
- SSH access with a public key
- A user with sudo privileges

Client side:
- A computer with Apache2 installed and running
- SSH client installed and running
- A private key for the server
- Software we will be installing:
  - autossh
  - certbot
  - python3-certbot-apache

### Server Side

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

- Create a new vhost file for the domain name:
```
sudo nano /etc/apache2/sites-available/example.com.conf
```

- Add the following lines to the vhost file:
```
<VirtualHost *:80>
    ServerName example.com
    ProxyPreserveHost On
    ProxyRequests Off
#   LogLevel debug proxy:trace5 # useful for debugging
    ProxyTimeout 1200
    ProxyPass / http://127.0.0.1:7000/
    ProxyPassReverse / http://127.0.0.1:7000/
</VirtualHost>
```

- Enable the vhost:
```
sudo a2ensite example.com.conf
```

- Create a vhost for ssl:
```
sudo nano /etc/apache2/sites-available/example.com-ssl.conf
```

- Add the following lines to the vhost file:
```
<VirtualHost *:443>
    ServerName example.com
    ProxyPreserveHost On
    ProxyRequests Off
    LogLevel debug proxy:trace5
    ProxyTimeout 1200
    SSLProxyEngine on
    ProxyPass / https://127.0.0.1:8000/
    ProxyPassReverse / https://127.0.0.1:8000/
</VirtualHost>
```

- Enable the vhost:
```
sudo a2ensite example.com-ssl.conf
```

- Restart Apache2:
```
sudo systemctl restart apache2
```

### Client Side (VM)

- Copy the private key from the server to the client:
-
```sh
scp -P 2222
```

Setup Apache2 vhost for the domain.

- Create a new vhost file for the domain name:

```sh
sudo nano /etc/apache2/sites-available/example.com.conf
```

- Add the following lines to the vhost file:

```conf
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/html/example.com
    <Directory /var/www/html/example.com>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
    </Directory>
</VirtualHost>
```

- Enable the vhost:

```sh
sudo a2ensite example.com.conf
```

- Restart Apache2:

```sh
sudo systemctl restart apache2
```

### Install autossh and certbot. (Client Side)

- Install autossh:

```sh
sudo apt-get install autossh certbot python3-certbot-apache
```

### Get a free SSL certificate from Let's Encrypt. (Client Side)

Before we can get the certificate we need http access to the server. We will use a reverse ssh tunnel on port `2222`, with `-vvvv` verbose mode for troubleshooting, to get the certificate.

```sh
ssh -NR 7000:127.0.0.1:80 user@public.server.com -p 2222 -i /path/to/private/key -vvvv
```

Visit http://example.com in a browser to verify that the tunnel is working before proceeding.


- Get the certificate:

```sh
sudo certbot --apache -d yourdomain.com,www.yourdomain.com
```

- Follow the prompts to get the certificate.
- Restart Apache2:

```sh
sudo systemctl restart apache2
```

#### Create a script to start the reverse ssh tunnel.

- Create a new file:

```sh
nano ~/ssh-reverse-tunnel.sh
```

- Add the following lines to the file:

```bash
#!/bin/bash
autossh -M 0 -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -N -f -R 8000:127.0.0.1:443 user@public.server.com -p 2222 -i /path/to/private/key
```

- Make the file executable:
```
chmod +x ~/ssh-reverse-tunnel.sh
```

- Test the script:
```
~/ssh-reverse-tunnel.sh
```

- Check that the tunnel is running:
```
sudo netstat -tulpn | grep 8000
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

- Check that the tunnel is running:
```
sudo netstat -tulpn | grep 8000
```

- If the tunnel is running, you should be able to access the server from the internet at https://example.com

### Configure HTTP.

Client side:
`a2dissite example.com.conf`

Server side:
- Edit the vhost file:
```
sudo nano /etc/apache2/sites-available/example.com.conf
```

- Add the following lines to the vhost file:
```
<VirtualHost *:80>
    ServerName example.com
     RewriteEngine On
     RewriteCond %{SERVER_NAME} =example.com
     RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
```

## References

- https://www.digitalocean.com/community/tutorials/how-to-configure-apache-as-a-reverse-proxy-for-apache-tomcat
- https://www.digitalocean.com/community/tutorials/how-to-secure-apache-with-let-s-encrypt-on-ubuntu-18-04
- https://www.digitalocean.com/community/tutorials/how-to-install-and-use-autossh-on-ubuntu-14-04
- https://www.golinuxcloud.com/setup-ssh-port-forwarding/
- https://np.courtesycs.com/index.php?ID=465
- https://www.ssh.com/academy/ssh/tunneling-example
-











