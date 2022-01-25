# Deploy a self-hosted Git service


There are many popular online code hosting platforms that help you store and manage
source code of your projects, [Github](https://github.com) and [Gitlab](https://gitlab.com) are the 
most common solutions among developers. But sometimes you want to host your projects on your local 
server, is that possible ?

Well, yeah! Gitlab offers a self-managed version ready to deploy, setting it up
may be a complex task and requires time and experience. Gitlab is considered as a devops platform, 
if you don't need such complex solution, then **Gitea** is the best choice for you.

Gitea is a cross-platform lightweight code hosting solution written in **Go**. The goal of this 
great product is to offer an easy, fast and painless deployment of a self-hosted Git service.

In this tutorial, we will walk through the steps of Gitea deployment on a Debian server.

## Setup local server virtual machine

For the purpose of this tutorial, we will be using [Vagrant](https://www.vagrantup.com/downloads) to 
create a Debian virtual machine that runs on [Virtualbox](https://www.virtualbox.org/wiki/Downloads)
hypervisor. Make sure that you have Virtualbox and Vagrant installed on your machine.
 
```bash
# Create the Vagrantfile
mkdir -p ~/vagrant/debian-bullseye64
vim ~/vagrant/debian-bullseye64/Vagrantfile
```
 
Here is the Vagrant configuration file to run a Debian 11 VM, with a bridged IP address :

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "debian/bullseye64"
  config.vm.network "public_network", ip: "10.0.0.20"

  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.memory = "512"
  end
end
```

Now you can boot up the VM : 

```bash
cd ~/vagrant/debian-bullseye64
vagrant up
```

Since we've set up a bridged connection, the Debian VM is accessible on the local network. To check 
if the VM responds, you can ping `10.0.0.20` *(the IP address may not be the same in your network, 
check your configuration and update the VM IP address to match your subnet)*.

{{< admonition type=tip title="Accessing the VM" open=true >}}
By default, Vagrant sets up an **SSH server** on the VM, you can access the shell by running `vagrant
ssh`, make sure that you are on the same directory of the Vagrantfile.
{{< /admonition >}}


## Setup database

Gitea requires one of the following relational databases : MySQL, PostgreSQL, MSSQL, SQLite3 or TiDB.
In this tutorial, we are going to use [MariaDB](https://mariadb.com/), the open-source equivalent 
of MySQL. 

To get MariaDB running on Debian VM, you need to install MariaDB server package : 

```bash
# Install server package
sudo apt install -y mariadb-server
# Launch the installation 
mysql_secure_installation
```

After the installation is finished, you need to create a new user and database :
{{< admonition type=tip title="Accessing MySQL prompt" open=true >}}
In order to access MySQL prompt, run `sudo mysql` command. If you are asked for a password, use the
one that you set up during MySQL secure installation.
{{< /admonition >}}


```sql
create database `gitea`;
create user 'gitea'@'localhost' identified by 'superSecReTPASSwD';
grant all privileges on `gitea`.* to 'gitea'@'localhost';
flush privileges;
```

## Install Gitea binaries

At the date of this tutorial, the latest available version of Gitea is `1.15.4`, you can check
the official [Gitea website](https://try.gitea.io/). Run the following commands to download 
Gitea binary : 

```bash
wget -O gitea https://dl.gitea.io/gitea/1.15.4/gitea-1.15.4-linux-amd64
chmod 755 gitea # make the binary executable
sudo mv gitea /usr/local/bin/ # move binary to $PATH
which gitea # check if gitea binary is available in $PATH
```

Now you can execute Gitea binary, a local server will be started on port `3000` and bound to 
`0.0.0.0` IP address, you can access Gitea externally from your browser by visiting 
`http://10.0.0.20:3000`. For now, we need to prepare the Debian VM for Gitea before using it.

## Prepare virtual machine for Gitea

To make Gitea instance work properly on your server, you need to install **Git** :

```bash
sudo apt install -y git
# check installed git version
git --version 
```

Create a new system user for Gitea :

```bash
sudo adduser \
   --system \
   --shell /bin/bash \
   --gecos 'Gitea' \
   --group \
   --disabled-password \
   --home /home/git \
   git
```

Create folder structure for Gitea repositories and configuration files : 

```bash
sudo mkdir -p /var/lib/gitea/{custom,data,log}
sudo mkdir /etc/gitea
```

Update permissions and change ownership of Gitea directories to the previously created Git user :

```bash
sudo chown root:git /etc/gitea
sudo chown -R git:git /var/lib/gitea/
sudo chmod -R 750 /var/lib/gitea/
sudo chmod 770 /etc/gitea
```

{{< admonition type=warning title="Update permissions after installation" open=true >}}
You should update permissions of `/etc/gitea` after installation, it was previously set with write 
permissions so the installer can write the configuration file.

```bash
chmod 750 /etc/gitea
chmod 640 /etc/gitea/app.ini
```
{{< /admonition >}}

Now you can run Gitea server, make sure that you have exported the Gitea working directory variable
*(You can skip this step, we will later run Gitea with a Linux service)* :

```bash
export GITEA_WORK_DIR=/var/lib/gitea/ # working directory
gitea web -c /etc/gitea/app.ini # launch Gitea web server
```

If you get a permission error, don't panic ! The explanation of this behavior is that your current 
user could not read the `app.ini` configuration file, your user should be a member of Git group. 
In my case, the current user is `vagrant` and it is not a member of Git group, adding it to that
latter will fix the permission error.

```bash
sudo usermod -aG git $USER
```

{{< admonition type=info title="Quit Gitea server" open=true >}}
You can kill Gitea server by hitting <kbd>Control</kbd>+<kbd>c</kbd>.
{{< /admonition >}}


## Gitea service

Running Gitea server manually is not practical, the ideal solution is to configure it to run on the 
system startup, to do so, you need to write a custom *Linux service* that starts Gitea every time the
system boots up.

Create a new *systemd service* and open it with your favorite editor, I will be using vim : 

```bash
sudo vim /etc/systemd/system/gitea.service
```

Since Gitea saves data on MariaDB database, our custom service should be run after MariaDB service.

```ini
[Unit]
Description=Gitea
After=syslog.target
After=network.target
Wants=mariadb.service
After=mariadb.service

[Service]
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea
RestartSec=2s
Restart=always

[Install]
WantedBy=multi-user.target
```

Now you can enable and run Gitea service :

```bash
sudo systemctl daemon-reload
sudo systemctl enable gitea --now
```

{{< admonition type=success title="Congratulations!" open=true >}}
Now you have Gitea service running and enabled at system startup, try to reboot the VM and access
Gitea server by visiting [http://10.0.0.20:3000](http://10.0.0.20:3000), you should see the setup page for Gitea
in a web UI.
{{< /admonition >}}

## Firewall configuration

If you are trying to deploy Gitea to a public server, then the usage of a firewall
is recommended as a security measure. We will not dive in details into firewall
configuration, but here are some basic commands to install and enable the firewall, and allow port 
3000 through the firewall *(the one that Gitea uses to serve the web UI)* : 

```bash
sudo apt install -y ufw # install uncomplicated firewall
sudo systemctl enable ufw --now # enable and run firewall service
sudo ufw enable # change firewall status to active
sudo ufw allow 22/tcp # allow ssh
sudo ufw allow 3000/tcp # allow Gitea
```


## Gitea web UI

{{< admonition type=info title="Gitea configuration file" open=true >}}
When you open Gitea web UI for the first time, you will be prompted with a 
configuration page, the web UI saves your configuration to `/etc/gitea/app.ini`,
you can skip the settings page and set your configuration directly by editing
the configuration file.
{{< /admonition >}}

Enter the database information that you previously created :

{{< figure src="/images/posts/002/gitea-initial-configuration.png" title="Gitea initial configuration" alt="Gitea initial configuration">}}

You have also to setup Gitea general parameters : 

{{< figure src="/images/posts/002/gitea-general-settings.png" title="Gitea general settings" alt="Gitea general settings">}}

Set up the administrator account : 

{{< figure src="/images/posts/002/gitea-administrator-account.png" title="Gitea administrator account" alt="Gitea administrator account">}}

{{< admonition type=success title="Congratulations!" open=true >}}
Now you can hit `Install Gitea` to apply configuration, you will be redirected 
to the login page. Use your administrator account to get access to your Gitea
instance.

{{< figure src="/images/posts/002/gitea-login.png" title="" 
	alt="Gitea login page">}}
{{< /admonition >}}


## Custom configuration

There are many configuration options than can be applied to you Gitea instance,
let's assume that we are using Gitea in a team, and only the administrator can
create accounts for new members. 

{{< admonition type=question title="Disable registration" open=true >}}
How to prevent users from creating accounts on our Gitea instance ?
{{< /admonition >}}

The solution is pretty simple, you have to ssh into the Debian VM (or your physical
server) and edit the `app.ini` configuration file. Under the *service* section, 
update `DISABLE_REGISTRATION` to **true**. 

```ini
[service]
REGISTER_EMAIL_CONFIRM  =  false
ENABLE_NOTIFY_EMAIL     =  false
DISABLE_REGISTRATION	=  true      <---
...
```

Restart Gitea service : 

```bash
sudo systemctl daemon-reload
sudo systemctl restart gitea
```

Now the registration feature is disabled :

{{< figure src="/images/posts/002/gitea-registration-disabled.png" 
title="Gitea disabled registration" alt="Gitea disabled registration">}}

{{< admonition type=info title="Advanced Gitea configuration" open=true >}}
All available configuration options are listed in [Gitea configuration cheat sheet](https://docs.gitea.io/en-us/config-cheat-sheet/).
{{< /admonition >}}


## Summary

{{< admonition type=tip title="Gitea on a public server" open=true >}}
If you are trying to deploy Gitea to a public server with a domain name, you can
use **NGINX reverse proxy**. This will let you set up a domain name and also 
**SSL/TLS certficates**.
{{< /admonition >}}

In this tutorial, we set up a Debian VM with a bridged network connection, we
created database configuration and we deployed Gitea. We also configured Gitea 
directly form the configuration file.

