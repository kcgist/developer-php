# Install+config Xdebug and PHPStorm w/SSH tunnel for remote debugging

This guide will show you how to install and configure Xdebug for remote debugging using a SSH tunnel.

## Prerequisites

1) A server running Ubuntu 18.04
1) SSH access to the server
1) Sudo privileges

The ssh server needs to have the following settings enabled on the server:

```bash
sudo nano /etc/ssh/sshd_config
```

```bash
AllowTcpForwarding yes
PermitTunnel yes
```

Save and close the file. To apply the changes, restart SSH:

```bash
sudo systemctl restart ssh
```

## Step 1: Install Xdebug

Xdebug is available in the default Ubuntu 18.04 repositories. To install it, run the following command:

```bash
sudo apt-get install php7.2-xdebug
```

## Step 2: Configure Xdebug

Xdebug is installed, but it is not enabled by default. To enable it, open the Xdebug configuration file:

```bash
sudo nano /etc/php/7.2/mods-available/xdebug.ini
```

Add the following lines to the file:

```bash
zend_extension=xdebug.so
xdebug.mode = develop,debug,trace,profile
xdebug.client_port = 9003
xdebug.start_with_request = trigger
xdebug.trigger_value = "donottell"
xdebug.client_host = 127.0.0.1
xdebug.profiler_output_name = "cachegrind.out.%t-%s"
xdebug.trace_output_name = "trace.%t-%s"
xdebug.output_dir = /var/www/html/xdebug
xdebug.collect_return = on
xdebug.collect_assignments = on
```

Save and close the file. To apply the changes, restart Apache:

```bash
sudo systemctl restart apache2
```

## Step 3 - Install autossh on the developers machine

To create a SSH tunnel, we will use autossh to keep our tunnel connected. To install it, run the following command:

```bash
sudo apt-get install autossh
```

## Step 4 - Create the SSH tunnel

To create a SSH tunnel, run the following command:

```bash
autossh -M 0 -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -i /path/to/private-key.pem -N -f -R 9003:127.0.0.1:9003 ubuntu@hostsite.com -p 2222
```

## Step 5 - Configuring PHPStorm

Open PHPStorm and go to `Settings` > `Languages & Frameworks` > `PHP` > `Debug`. Click on the `...` button next to `Debug port` and select `Use non-local server`. Enter `9003` as the port number and click `OK`.

## Step 5a - Configure VSCODE

Install this extension https://marketplace.visualstudio.com/items?itemName=xdebug.php-debug

Create or edit your workspace settings like this. The path in the settings is a fuse mount point see [Mount the VM's filesystem on the laptop with sshfs](./ssh.md#mount-the-vms-filesystem-on-the-laptop-with-sshfs-optional)

```json
{
	"folders": [
		{
			"path": "."
		},
		{
			"path": "../../../vm/html",
			"name": "html"
		}
	],
	"launch": {
		"version": "0.2.0",
		"configurations": [
			{
				"name": "Xdebug",
				"type": "php",
				"request": "launch",
				"port": 9003,
				"pathMappings": {
					"/var/www/html": "${workspaceFolder:html}"
				}
			},
			{
				"name": "HTML",
				"request": "launch",
				"type": "msedge",
				"configurations": [
					"Open Edge DevTools"
				],
				"url": "https://hostsite.com/",
				"webRoot": "${workspaceFolder:html}"
			}
		],
	}
}
```

## Step 6 - Debugging

To test if everything is working, create a new PHP file on the remote server with the following content:

```php
<?php
 phpinfo();
```

Open the file in your browser and click on the `Start Listening for PHP Debug Connections` button in PHPStorm and click on the browser extension icon and select `Debug`. Reload the page in your browser and you should see the debugger in PHPStorm. To stop debugging, click on the browser extension icon and select `Stop Debugging`.

## Step 7 - Browser extension

Install the Xdebug browser extension to make it easier to start and stop debugging, profiling, and tracing.

Add the secret key to the browser extension "becourteous" (without the quotes) by right clicking on the browser extension icon and selecting `Options`. Enter the secret key in the `IDE key` field, the `Trace Trigger Value` field, the `Profile Trigger Value` field and click on the `Save` button.


## Step 8 - Using the browser extension

To start debugging, click on the browser extension icon and select `Debug`. To stop debugging, click on the browser extension icon and select `Stop Debugging`.

To start profiling, click on the browser extension icon and select `Profile`. To stop profiling, click on the browser extension icon and select `Stop Profiling`.

To start tracing, click on the browser extension icon and select `Trace`. To stop tracing, click on the browser extension icon and select `Stop Tracing`.

To disable the Xdebug, click on the browser extension icon and select `Disable`.

Remember to `Disable` Xdebug when you are done debugging, profiling or tracing. Otherwise it will slow down your application and fill up the disk space.

## Step 9 - View the profile file

To view the profile file in a web browser, install Webgrind on the remote server.

Webgrind is installed in the `/var/www/html/webgrind` directory. To view the profile file, open the following URL in a web browser: http://hostsite.com/webgrind


You can also view the profile file in KCachegrind  on a linux machine. For other options see Xdebug Profiler.

## Step 10 - View the trace file

To view the trace file, open the trace file in your IDE.

All trace and profile output files are stored on the server at

```bash
/var/www/html/xdebug
```

## Conclusion

You have successfully installed and configured Xdebug for remote debugging using a SSH tunnel. You can now debug your PHP applications on a remote server using PHPStorm.

## Links

 - [Xdebug](https://xdebug.org/)
 - [Remote debugging via SSH tunnel | PhpStorm](https://www.jetbrains.com/help/phpstorm/remote-debugging-via-ssh-tunnel.html)
 - [Browser Extension](https://xdebug.org/docs/step_debug#browser-extensions)
 - [Videos by Xdebug](https://xdebug.org/docs/all_related_content)
