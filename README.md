
Name : Noppanat Sripan
Student ID : A01373963


This repository is a tutorial on setting up a Bash script to generate a static index.html file with system information daily at 05:00 using a systemd service and timer. The generated file is served by an Nginx web server running on an Arch Linux droplet, with UFW configured for firewall protection.

## Features
- Automates daily generation of a static `index.html` with system information.
- Uses `systemd` service and timer for scheduling.
- Serves the HTML file through an Nginx web server.
- Implements UFW to secure the server.
<br>
## Files
- `generate-index.service` : control the script/process `generate_index`
- `generate-index.timer` : control the time to update the script
- `generate_index` : generate html file (contain your system info)
- `nginx.conf` :  config file for package nginx
- `webgenServer.conf` : config for webserver
<br>
## Requirement 
- Packages:
   1. `git` to clone the repository.
   2. `nginx` to host a web server.
   3. `ufw` to control the firewall.

- All files in the repository (use git clone).
- Arch Linux.
- Sudo privileges.




## How to Use
In the following step-by-step tutorial, assuming you are in the repository folder.  
**The steps include**: 
1. Create User.
2. Configure `Systemd`.
3. Set up `nginx`.
4. Set up `ufw`.

<br>

### 1. Create User
this step is to create a System user `webgen` and the home directory (including sub folder) to make `webgen` handle any task that related to generate a static file 

Home directory structure
```text
/var/lib/webgen
|
|_____/bin
|       └-generate_index
|_____/HTML
		└-index
```
- `bin` folder is going to be the place where we put the script ``generate_index`` and execute it from `.service` file (systemd)
- `HTML` is folder a that store HTML file that is created by `generate_index`


1. create system user `webgen`
```bash
useradd -r -d /var/lib/webgen -m webgen -s /usr/bin/nologin
```

2. create `bin`  and `HTML` folder inside `webgen`  home directory
```bash
sudo mkdir /var/lib/webgen/bin 
sudo mkdir /var/lib/webgen/HTML
```

> [!note]
> since we can't switch to user `webgen`, we are need to use `sudo` to create folder inside `webgen` home directory

4. move file `generate_index` from repository to `bin`
```bash 
mv generate_index /var/lib/webgen/bin
```

5. give permission to execute the script
```bash
chmod +x /var/lib/webgen/bin/generate_index
```

since `webgen` user will be the person who control the script and service, we need to make it own the files in home directory 

6. change the permission of the folder (include anything inside) to `webgen`
```bash
sudo chown -R webgen:webgen /var/lib/webgen
```


>[!note]
 the reason to create system user and not regular user is to isolate task and permission, reducing risk of the accidental and malicious changes

>[!Tip]
>By sperate user and give the permission to do the user only what they need, it follows  the principle of principle of least privilege (POLP): Grant only the minimum permissions necessary for a user, process, or application to perform its tasks, and nothing more

<br>


### 2. Config `Systemd`
in this step we will use the `generate-index.service` and `enerate-index.timer`  from the repository to set up `systemd` to run the `generate_index` every day at 5 am.

as the `generate-index.service,generate-index.timer ` file are provided in the repository, you should be able to do the following step, using these files. 

the `generate-index.service` has the following content inside
```bash
[Unit]
Description=HTML-Gen  # Description of the service
Wants=network-online.target # optional dependency
After=network-online.target # set it to run after network-online.target

[Service]
Type=oneshort # run it one time when it was call
User=webgen # user who run
Group=webgen # group who run
ExecStart=/var/lib/webgen/bin/generate_index # the script that it run


[Install]
WantedBy=multi-user.target 

```

the `generate-index.timer` has the following content inside 
```bash
[Unit]
Description=Gen_index_every5am # description 

[Timer]
OnCalendar=*-*-* 05:00:00 # set the time to run every day 5 am

Persistent=true # if server down on 5am, allow it to run as soon as the server start

[Install]
WantedBy=timers.target #ensure timer start automatically

```


1. move `generate-index.service` to `/etc/systemd/system`
```bash
sudo mv generate-index.service /etc/systemd/system
```

2. start the service 
```bash
sudo systemctl start generate-index
```

3. enable the service (run when the server is start)
```bash
sudo system enable generate-index
```

now we are done setting up the `generate-index.service` files, this should create `index.html` in the folder `HTML`

next, we are going to config`generate-index.timer`

4. move `generate-index.service` to `/etc/systemd/system`
```bash
sudo mv generate-index.service /etc/systemd/system
```

5. start the timer 
```bash
sudo systemctl start generate-index.timer
```

6. enable the service (run when the server is start)
```bash
sudo systemctl enable generate-index
```

7. finally, to check if the `.timer` is running
```bash
sudo systemctl list timer
```
this will shows the list of timer that currently running on your system


> [!Tip]
> if the error shows on the status, use `journal -ex -u <service>` to check the log of service or timer file or use `sudo systemctl status <process/timer_name>.timer`

### 3. Set up Nginx
in this step, you will set up the nginx using the file `nginx.conf` in this repository (make sure `nginx` is downloaded)

the `niginx.conf` have the following content 
(the following setting are from https://wiki.archlinux.org/title/Nginx)
```text
user webgen;
worker_processes auto;
worker_cpu_affinity auto;

events {
    multi_accept on;
    worker_connections 1024;
}

http {
    charset utf-8;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    server_tokens off;
    log_not_found off;
    types_hash_max_size 4096;
    client_max_body_size 16M;

    # MIME
    include mime.types;
    default_type application/octet-stream;

    # logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;

    # load configs
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

1. make `nginx` back-up file
```bash
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx_backup.conf
```

>[!note]
>the reason that we are making nginx backup is to avoid the error that can happen from the new nginx.conf, since we made the backup then we can go back and use default setting anytime

2. move the `nginx.conf` from repository to `/etc/nginx/`
```bash
sudo mv nginx.conf /etc/nginx/
```

3. make folder contain server configuration
```bash
sudo mkdir /etc/nginx/sites-available
sudo mkdir /etc/nginx/sites-enabled
```
- `sites-available` folder is for storing _all_ of your vhost configurations, whether or not they're currently enabled.
- `sites-enabled` folder contains symlinks to files in the sites-available folder. This allows you to selectively disable vhosts by removing the symlink.

4. move the `webgenServer.conf` sites-available 
```bash
sudo mv webgenServer.conf /etc/nginx/sites-available
```

5. create symbolic link from `webgenServer.conf` in sites-enable pointing to sites-available
```bash
sudo ln -s /etc/nginx/sites-available/webgenServer.conf /etc/nginx/sites-enabled/webgenServer.conf
```

6.  start the `nginx` and make it start when server boost
```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

After you done everything, it will create a website from your droplet (arch linux) ip address showing information of your system (port 80)

**Why separate server block file?**
Using separate server block files for virtual hosts is important because each website has its own file. This allows for easy enabling or disabling by moving files in and out of `sites-enabled`. It also simplifies configuration management by keeping files organized and easier to maintain.


https://serverfault.com/questions/527630/difference-in-sites-available-vs-sites-enabled-vs-conf-d-directories-nginx

https://wiki.archlinux.org/title/Nginx

>[!troubleshoot]
>`sudo nginx -t` to check the syntax and test the file
> `sudo systemctl status nginx` to check the status of the process
> `journal -ex -u nginx` to see the log file of the process

### 4. Set up `ufw`
in this step, we will config the  `ufw` fire wall to:
-  allow ssh and http from anywhere (22)
- enable ssh rate limiting
- allow http connections

this allows http connection, since we want other people to be able to access your  website and allows port 22 for you (also other people since we didnt limit to your IP address) to be able to connect to your droplet, additional we limit the time of the fail ssh to prevent other people to brute force to the droplet. 

1. enable and start the service
```bash
sudo systemctl enable --now ufw.service
```
2. set the table rule to allow ssh from anywhere
```bash
sudo ufw allow SSH
```
3. limit the incoming ssh
```bash
sudo ufw limit ssh
```
4. allow http
```bash
sudo ufw allow http
```
5. check that ssh is set in the table rules
```bash
sudo ufw app list
```
6. after make sure that ssh is in the app list, turn on firewall
```bash
sudo ufw enable
```

**To check the status of the firewall**
```bash
sudo ufw status verbose
```
this should show the following configuration:

`[pichere]`

explanation 
- **allow** (ipv4 and ipv6) port 22 (SSH) but **limit** the ssh times, any IP address can ssh
- **allow** (ipv4 and ipv6)port 80 (http) from any ip 


## DONE
you are done setting up everything, now try to access the website from your droplet IP address. This should show your information about your system. 

[pichere]





## 