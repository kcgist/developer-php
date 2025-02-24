# Convert Apache2 vhosts to run multiple versions of PHP using PHP-FPM on Ubuntu 20.04

Convert existing Apache2 vhosts running php7.2 installed from the SURY repository to use PHP-FPM on Ubuntu 20.04 So that we can update to PHP8.1 and run both versions of PHP on the same server.

## Install PHP

* Install PHP8.1

```bash
sudo apt-get install php8.1-common php8.1-cli php8.1-fpm -y
# After it's done installing you should see this message:
    NOTICE: Not enabling PHP 8.1 FPM by default.
    NOTICE: To enable PHP 8.1 FPM in Apache2 do:
    NOTICE: a2enmod proxy_fcgi setenvif
    NOTICE: a2enconf php8.1-fpm
    NOTICE: You are seeing this message because you have apache2 package installed.
```

* Install PHP8.1 modules

```bash
sudo apt-get install php8.1-{bcmath,curl,dom,gd,imagick,intl,mbstring,mysqli,mysql,pdo-mysql,readline,simplexml,xml,xmlreader,xmlwriter,xsl,zip,opcache} -y
# Use this instead if you want to install xdebug too
sudo apt-get install php8.1-{bcmath,curl,dom,gd,imagick,intl,mbstring,mysqli,mysql,pdo-mysql,readline,simplexml,xml,xmlreader,xmlwriter,xsl,zip,opcache,xdebug} -y
# After it's done installing you should see this message:
    NOTICE: Not enabling PHP 8.1 FPM by default.
    NOTICE: To enable PHP 8.1 FPM in Apache2 do:
    NOTICE: a2enmod proxy_fcgi setenvif
    NOTICE: a2enconf php8.1-fpm
    NOTICE: You are seeing this message because you have apache2 package installed.
```

* Install PHP7.2 FPM

```bash
sudo apt-get install php7.2-fpm -y
# After it's done installing you should see this message:
    php7.2-fpm is already the newest version (7.2.34-43+ubuntu20.04.1+deb.sury.org+1).
```

## Configure PHP and Apache2

* Enable PHP7.2 FPM

```bash
sudo a2enconf php7.2-fpm
# After it's done installing you should see this message:
    Enabling conf php7.2-fpm.
    To activate the new configuration, you need to run:
    systemctl reload apache2

sudo systemctl start php7.2-fpm
```

* Enable php8.1-fpm

```bash
sudo a2enconf php8.1-fpm
# After it's done installing you should see this message:
    Enabling conf php8.1-fpm.
    To activate the new configuration, you need to run:
    systemctl reload apache2

sudo systemctl start php8.1-fpm
```

* Restart Apache2

```bash
sudo apachectl configtest
sudo systemctl restart apache2
```

* Check PHP7.2-FPM is running

```bash
sudo systemctl status php7.2-fpm
```

* I used my IDE diff tool to compare my current ini file and created a `.user.ini` file this is similar to `.htaccess` in Apache. I then copied the `.user.ini` file to the `/var/www/html` folder which is my `apache servers document root` so it would be picked up by PHP-FPM, in all sub-folders, regardless of which php version was being used. This will standardize the PHP settings across all versions. This will also simplify changes to the php.ini file as we will only need to change the settings in the `.user.ini` file which can be edited by the `ubuntu` user and doesn't require PHP to be restarted.

```sh
sudo nano /var/www/html/.user.ini
```

```ini
html_errors = On
pdo_mysql.cache_size = 2000
mysqli.cache_size = 2000

max_execution_time = 300
memory_limit = 1024M
post_max_size = 15M
upload_max_filesize = 20M
```

* Prevent Apache from serving .user.ini files by adding the following to the `/etc/apache2/apache2.conf` file or the `.htaccess` file in the `/var/www/html` folder.

```sh
sudo nano /var/www/html/.htaccess
```


```conf
<Files ".user.ini">
    Require all denied
</Files>
```

* Copy the current xdebug.ini to the 8.1 folder

```sh
sudo cp /etc/php/7.2/mods-available/xdebug.ini /etc/php/8.1/mods-available/xdebug.ini
```

* Modify the vhost file to use PHP-FPM

```bash
sudo nano /etc/apache2/sites-available/site.conf
sudo nano /etc/apache2/sites-available/archive.conf


```

Here we are serving the site from /var/www/html using PHP8.1 and the archive folder from /var/www/html/archive using PHP7.2

```conf
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
    <Directory "/var/www/html">
        Options -Indexes
        AllowOverride All
    </Directory>
    #php-fpm 8.1 Server default
    <FilesMatch \.php$>
        # From the Apache version 2.4.10 and above, use the SetHandler to run PHP as a fastCGI process server
        # no trailing slash
        SetHandler "proxy:unix:/run/php/php8.1-fpm.sock|fcgi://localhost"
    </FilesMatch>

    #php-fpm 7.2 for archive folder
    <Directory "/var/www/html/archive">
        <FilesMatch \.php$>
            SetHandler "proxy:unix:/run/php/php7.2-fpm.sock|fcgi://localhost"
        </FilesMatch>
    </Directory>
</VirtualHost>
```

* Check in .htaccess files for php_value and php_flag and move them to the .user.ini file or disable them if they are not needed. I commented out the following lines in my .htaccess file.

```ini
# php_flag display_startup_errors on
# php_flag display_errors on
```

## Enable PHP FPM

* To use php-fpm, you need to switch to `mpm_event` and disable `mpm_prefork` and `php7.2`
* Also enable `proxy` `proxy_fcgi`, required for php-fpm

```sh
sudo a2dismod php7.2 mpm_prefork
sudo a2enmod mpm_event proxy proxy_fcgi
```

* Check PHP8.1-FPM is running

```bash
sudo systemctl status php8.1-fpm
```

## Set PHP8.1 as default

The PHP 8.1 CLI will be installed at /usr/bin/php8.1 location by default. Similarly, other PHP binary files will be located in the same directory (/usr/bin/php8.0, /usr/bin/php7.4, etc). The default php name will be symlinked to the latest PHP version by default, but it is possible to change where the default php command links to.

The update-alternatives command provides an easy way to switch between PHP versions for PHP CLI.

```bash
sudo update-alternatives --config php
```

This brings up a prompt to interactively select the alternative PHP binary path that php points to.

There are 2 choices for the alternative php (providing /usr/bin/php).

```bash
  Selection    Path             Priority   Status
------------------------------------------------------------
* 0            /usr/bin/php8.1   81        auto mode
  1            /usr/bin/php8.0   80        manual mode
  2            /usr/bin/php8.1   81        manual mode
```

Press 0 to select php8.1 as the default PHP version.

To set the path without the interactive prompt:

```bash
sudo update-alternatives --set php /usr/bin/php8.1
sudo update-alternatives --set phar /usr/bin/phar8.1
sudo update-alternatives --set phar.phar /usr/bin/phar.phar8.1
sudo update-alternatives --set phpize /usr/bin/phpize8.1
sudo update-alternatives --set php-config /usr/bin/php-config8.1
```


* Restart Apache2

```bash
sudo apachectl configtest
sudo systemctl restart apache2
```
