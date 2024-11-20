
Name : Noppanat Sripan
Student ID : A01373963

this is read me for assignment 3



## 0. Prerequisites
this is the starting of the tutorial, you will need this following resource/package in order to create a website that auto generate static html

package needed :
1. `git` to clone to repository 
2. `nginx` to host a webserver
3. `ufw` to control the firewall ( control port communication )

to install the package: 
```bash
sudo pacman -S <packagename>
```

since in this tutorial, you will need to use the resource from this repository, you will need to do the following command
```bash
git clone https://github.com/GetMeoutt/linux_assignment3.git
```

this will clone the folder `linux_assignment3` to your directory, it includes the files the you will need in order to complete the following step.  

files in the repository 
- `generate-index.service` : control the script/process `generate_index`
- `generate-index.timer` : control the time to run  `generate-index.service
- `generate_index` : bash script which will generate html file when run 
- `nginx.conf` :  config file for package nginx
- `webgenServer.conf` : config for webserver
<br>

## 1. Create User 

this step is to create a System user and the home directory to handle any task that related to generate a static file and host it on server IP address (include nginx)

create system user `webgen`
```bash
useradd -r -d /var/lib/webgen -m webgen -s /usr/bin/nologin
```
`useradd` : to create user <br>
	`-r` to create a system user <br>
	`-d` to sets the path of home directory <br>
	`-m` to create home directory <br>
	`-s` to sets the path to the user's login shell <br>


 the reason the create system user and not regular user is to isolate task and permission, reducing risk of the accidental and malicious changes

>[!Tip]
>By sperate user and give the permission to do the user only what they need, it follows  the principle of principle of least privilege (POLP): Grant only the minimum permissions necessary for a user, process, or application to perform its tasks, and nothing more



### 1.1 Create Sub-folder inside `webgen` home directory
this step is to create `bin` and `HTML` folders inside `webgen` home directory

> [!note]
> since we can't switch to user `webgen`, we are going to use `sudo` to create folder inside `webgen` home directory

to create `bin`  inside `webgen`  home directory
```bash
sudo mkdir /var/lib/webgen/bin 
```

to create `HTML` folder inside `webgen` home directory
```bash
sudo mkdir /var/lib/webgen/HTML
```
`sudo` to elevated privileges
`mkdir` to create folder 



### 1.2 Move the bash script to `bin` directory 

in the directory that you clone the repository
```bash 
mv generate_index /var/lib/webgen/bin
```
`mv` to move file

give permission to execute the script
```bash
chmod +x /var/lib/webgen/bin/generate_index
```
`chmod +x` to add execute permission

> [!note] 
> this script will run by .service file and generate static HTML in the folder `/var/lib/webgen/HTML`


### 1.3 Change Permission to `webgen` user
since `webgen` is a user that will control service and script inside, you need to change the permission for `webgen` user

```bash
sudo chown -R webgen:webgen /var/lib/webgen
```
`chown` to change the file/ folder ownership
	 `-R` to make it recursive 

<br>
<br>

## 2. Create service for `generate_index` script
in this step we will use the `generate-index.service` from the repository to set up `systemctl` to run the `generate_index`


### 2.1 get `generate-index.service`
as the `generate-index.service` file is provided in the repository, you should be able to do the following step, using this file. 

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


### 2.2 Move `generate-index.service` to `systemctl`
in this step, we will move `generate-index.service` to the `system` where `systemctl` will read from, and `systemctl` will execute from `.service` files

to move `generate-index.service` to `/etc/systemd/system`
```bash
sudo mv generate-index.service /etc/systemd/system
```


start the service 
```bash
sudo systemctl start generate-index
```
after start `generate-index.service` it will execute the script `generate_index` in the `bin` folder in the `webgen` home directory


enable the service (run when the server is start)
```bash
sudo system enable generate-index
```


### 2.3 get  `generate-index.timer`
as the `generate-index.timer` file is provided in the repository, you should be able to do the following step, using this file. 

the `generate-index.timer` has the following content inside
```bash
[Unit]
Description=Gen_index_every5am # description 

[Timer]
OnCalendar=*-*-* 05:00:00 # set the time to run every day 5 am

Persistent=true

[Install]
WantedBy=timers.target

```

### 2.4 Move `generate-index.timer` to `systemctl`
in this step, we will move `generate-index.timer` to the `system` directory where `systemctl` will read from, and `systemctl` will execute from `.timer` file

to move `generate-index.service` to `/etc/systemd/system`
```bash
sudo mv generate-index.service /etc/systemd/system
```

start the timer 
```bash
sudo systemctl start generate-index.timer
```
after start `generate-index.timer` it will execute the script `generate_index` in the `bin` folder in the `webgen` every day on 5 am


enable the service (run when the server is start)
```bash
sudo systemctl enable generate-index
```

### 2.5 Check the status

to check if `.service` and `.timer` is run successfully
```bash
sudo systemctl status generate-index
```

and to check status of timer 
```bash
sudo systemctl status generate-index.timer
```

finally, to check if the `.timer` is running
```bash
sudo systemctl list timer
```
this will shows the list of timer that currently running on your system


> [!tip]
> if the error shows on the status, use `journal` to check the log of service or timer file


