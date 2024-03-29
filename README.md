# Code-Server-Hub
Simple hub page for [code-server](https://github.com/cdr/code-server) . Each user has one work-space, authenticate with Linux PAM module.

Each user has one workspace, login with your **linux account** and password.

If you want add user, type ```sudo adduser``` in command line. Make sure you are a sudoer.

## What is this
This is an nginx reverse proxy config which will try to authenticate user:password with linux pam module ,and try to execute command to spawn a code-server workspace by that user, and then proxy_pass to it.

## All in one Script for Ubuntu 20.04
**Warning:** This script will reinstall your nginx.

Run this in terminal

```sudo sh -c "$(wget -O- https://raw.githubusercontent.com/HuJK/Code-Server-Hub/master/install.sh)"```

And goto url : https://\[your_server_ip\]:8443

Demo:
[https://nlvm.whojk.com:8443/](https://nlvm.whojk.com:8443/)

user|passwd
------|---------
demo01|demo)!


## Manual install 

### Preinstall (for Ubuntu. If you use different distro, please install equivalent package ): 
```bash
set -e
echo "###update phase###"
apt-get update
set +e
# In my distro(debian 10), It seems nginx and nginx-full are not compatible. I have to remove nginx than I can install nginx-full.
apt-get remove -y nginx
# The install script will detect npm exist or not on the system. If exist, it will not use itself's npm
# But in Ubuntu 19.04, npm from apt are not compatible with it. So I have to remove first, and install back later.
apt-get autoremove -y
set -e
echo "###install dependanse phase###"
apt-get install -y nginx-full
apt-get install -y lua5.2 lua5.2-doc liblua5.2-dev luajit
apt-get install -y libnginx-mod-http-auth-pam libnginx-mod-http-lua
apt-get install -y tmux gdb git python python3 python3-pip wget libncurses-dev nodejs sudo
apt-get install -y zsh fish tree ncdu aria2 p7zip-full python3-dev perl wget curl vim htop

set +e # folling command only have one will success
#cockpit for user management
apt-get install -y -t bionic-backports cockpit cockpit-pcp #for ubuntu 18.04
apt-get install -y cockpit cockpit-pcp                     #for ubuntu 20.04
set -e
```

### Install

```bash
echo "###doenload files###"
cd /etc
git clone https://github.com/HuJK/Code-Server-Hub.git code-server-hub


cd /etc/code-server-hub
mkdir -p .cshub
mv code /etc/nginx/sites-available/
ln -s ../sites-available/code /etc/nginx/sites-enabled/

echo "###doenload latest code-server###"
curl -s https://api.github.com/repos/cdr/code-server/releases/latest \
| grep "browser_download_url.*linux-x86_64.tar.gz" \
| cut -d : -f 2,3 \
| tr -d \" \
| wget -i - -O code-server.tar.gz

echo "###unzip code-server.tar.gz###"

tar xzvf code-server.tar.gz -C .cshub
mv .cshub/*/* .cshub/
set +e
echo "###add nginx to shadow to make pam_module work###"
usermod -aG shadow nginx
usermod -aG shadow www-data
set -e
echo "###set permission###"
chmod -R 755 /etc/code-server-hub/.cshub
chmod -R 775 /etc/code-server-hub/util
chmod -R 773 /etc/code-server-hub/sock
chgrp shadow /etc/code-server-hub/util/anime_pic

sudo sh -c "$(wget -O- https://raw.githubusercontent.com/HuJK/Code-Server-Hub/master/install2.sh)"
```

### Postinstall.

Edit ```/etc/nginx/sites-enabled/code``` with vim, nano, or any other text editior with root. And follow following instructions.

#### 1. Configure ssl certificates

use self-signed certificates:
```bash
echo "generate self signed cert"
apt-get install -y install openssl
mkdir /etc/code-server-hub/cert
chmod 600 /etc/code-server-hub/cert
cd /etc/code-server-hub/cert
openssl genrsa -out ssl.key 2048
openssl req -new -x509 -key ssl.key -out ssl.pem -days 3650 -subj /CN=localhost
cat ssl.pem ssl.key > /etc/cockpit/ws-certs.d/0-self-signed.cert
```

#### 2. Use valid ssl certificates

1. Buy or get a free domain
2. Get a valid certificate from letsencrypt
3. put your ssl cert and key at following path
```
    ssl_certificate     /etc/code-server-hub/cert/ssl.pem;
    ssl_certificate_key /etc/code-server-hub/cert/ssl.key;
```
4. configure ssl key for cockpit
```
cd /etc/code-server-hub/cert
cat ssl.pem ssl.key > /etc/cockpit/ws-certs.d/0-self-signed.cert
```

#### 3. Change port number(if you want)
Edit line 8~9
```
    listen 8443 ssl;
    listen [::]:8443 ssl;
``` 
from 8443 to other ports that you prefer.

Now, reload services with 
```bash
echo "restart nginx and cockpit"
systemctl enable nginx
systemctl enable cockpit.socket
service nginx stop
service nginx start
service cockpit stop
service cockpit start
exit 0
```
