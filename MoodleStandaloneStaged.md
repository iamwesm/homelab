# Getting Started 
In this demo, we'll be setting up a small standalone Moodle server using LAMP setup to provide a self-hosted in-house course platform. Moodle LMS is an open-source learning management system (LMS).
[Learn More Here](https://moodle.com/solutions/lms/)

## Goals
- Create a database
- Setup reverse proxy
- Configure Moodle

# Table of Contents
- Server's Resources
	- Deployment Recommendation
- Server's Configuration
	- MariaDB Setup
		- Database Creation
	- Apache2 Setup
		- HTTPS Configuration
- Moodle Setup
	- Moodle Data Directory
	- Moodle Config 
- Extras
	- Securing Server
		- Firewall (UFW)
		- Intrusion Prevention (Fail2Ban)
		- User Authentication (SSH)
	- Moodle Plugins
		- Plugins Update
	- Moodle Update
	- Moodle Cron Backup 

## Others
- Docker Deployment, maybe
- Moodle Cluster, maybe


---


## Server's Resources 
Let's get our server set up first. For this staged demo, we'll setup a virtual server under a Proxmox hypervisor. We'll configure the virtual machine with a single core vCPU of 64-bit architecture with 4GB of memory. In terms of storage, we'll start with 16GB of storage for now. Since this is a small server, 10mpbs uplink should be more than enough to handle the network traffic. For operating system, we'll be using an Ubuntu image.

#### Deployment Recommendation: 
- Upgrade from Single Core to Dual Core vCPU and from 4GB to 8GB Memory
	- As the old saying goes, "it's better to be over-prepared than under-prepared". 
- Local Storage to Network Storage (NAS)
	- For future scalability and disaster recovery, off-load Moodle's data directory separately from the host. Moodle's data directory consists of entities that are created by users such as courses, topics, quizzes, and etc.
- Standard Release to Upstream LTS OS Release (Rocky/Debian)
	- An upstream distribution like Debian 10 LTS would be less prone to any unstable packages and releases that might cause some issue.
- File Cache and Memory Cache Tandem (Redis)
	- If the Moodle's data directory has been off-loaded to a network storage, users might experience some performance issue depending the network and storage setup. As such, setting up a memory cache like Redis will keep frequently requested data  in-memory to serve them quicker.
- Database Security
	- It is best practice to avoid using root user as much as possible; and creating a specified user with specified or limited access it needs to a database. 
- HTTPS and Valid Certificate
	- It goes without saying that self-signed certificate is only meant for development and staging cycles. HTTP traffic requests should also be redirected to HTTPS.
- Server Security
	- It's should be a standard practice to harden your server even if it is already under a firewall and other security system.


## Server's Configuration
Once we got the server up and running, let's run an update to make sure that our server and its packages are up to date. Then, we'll install our required packages. For this demo, we'll be installing Apache2, PHP, and MariaDB.
```bash
sudo apt update && apt upgrade -y && \
sudo apt install apache2 php mariadb-server -y \
```

We'll also have to installing the following PHP extensions that required and needed for Moodle.
```bash
sudo apt install php-mysql php-xml php-mbstring php-curl php-zip php-gd php-intl php-xmlrpc php-soap -y
```


## MariaDB Setup
Let's setup our database service. First, we need to create database; then, we'll create our user. Lastly, we'll grant our designated user access to our new database.

We'll initiate a secure installation. 
```bash
sudo mysql_secure_installation && \
sudo mysqyl -u root -p \
```

Then, we can now create our database and user.
```mysql
create DATABASE 'mdatabase' default character set utf8mb4;
create USER 'mdbuser'@'localhost' IDENTIFIED by 'superSECRETphrase101';
grant ALL PRIVILEGES on mdatabase.* to "muser"@"localhost";
```


## Apache2 Setup
Next, we'll setup our reverse proxy service. This can setup can be a little lengthy. We'll need to create our own configuration file and update with our own information. Then, we need to enable it and test to make sure that it's running. Lastly, we want to make sure that we are using secured proxy, HTTPS(443); and that our HTTP(80) traffic is being redirected to HTTPS(443).

First, we need to create our configuration file under the sites-available directory. We'll use "mstaged.demo.local" for our domain name. I've already created a template for us to use for this config file. 
```bash
sudo touch /etc/apache2/sites-available/moodle-proxy.conf && \
sudo vim /etc/apache2/sites-available/moodle-proxy.conf 
```

```moodle-proxy.conf
<VirtualHost *:80> 
  ServerName mstaged.demo.local
  ServerAlias mstaged.demo.local
</VirtualHost>
```

Then, we need to enable our configuration and restart our service for update to take in effect. Lastly, we'll update our host file to add our new DNS for the local address.
```bash
sudo a2ensite /etc/apache2/sites-available/moodle-proxy.conf && \
systemctl restart apache2-service && \
sudo vim /etc/hosts
```

We can now test our proxy service that our newly added specified DNS resolves to our hosts.
```bash
curl http://mstaged.demo.local/
```

### HTTPS Configuration
For this demo, we'll generate our own self-signed certificate. We'll our domain name from dot-local to dot-com; "mstaged.demo.com".
```bash
sudo mkdir /etc/ssl/self-signed/ && \
cd /etc/ssl/self-signed/ 
sudo openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout ./certificate.key -out ./certificate.crt -subj "/C=US/ST=California/L=Los Angeles/O=Homelab/OU=Moodle/CN=Moodle"	
```

Now that we have a certificate, let's update our Apache config file. I've created a template for this as well that we can use. We'll edit our config file and add the this template underneath the closing line for port 80.
```moodle-proxy.conf
<VirtualHost *:443>
	ServerName mstaged.demo.com

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html/moodle
        	<Directory /var/www/html/moodle>
                	Options FollowSymLinks MultiViews
                	AllowOverride None
        	</Directory>

	SSLEngine on
	SSLCertificateFile      /etc/ssl/self-signed/certificate.crt
	SSLCertificateKeyFile /etc/ssl/self-signed/certificate.key

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

Finally, we need to enable SSL mod for Apache and restart our service.
```bash
sudo a2enmod ssl && \
sudo systemctl restart apache2-server \
```


## Moodle's Setup and Configuration
Now that we have just about all the pre-requisites for Moodle, we can proceed with downloading Moodle source to our host and updating the Moodle configuration file with our previously configured resources.

In this demo, we'll be using Git to clone a stable branch of Moodle for Github. We'll have to update the folder permissions, move it to /opt/ directory, and link it to /html/ directory.
```bash
sudo apt install git -y && \
git clone -b MOODLE_311_STABLE git://git.moodle.org/moodle.git moodle && \
mv ./moodle /opt/moodle-active
ln -s /opt/moodle-active /var/www/html/moodle
find /opt/moodle -type f -exec chmod 0644 {} \; && \
find /opt/moodle -type d -exec chmod 0755 {} \;
```

Next, we'll create a directory for our Moodle's data. We'll add it under /srv/ directory. This directory should not be under Moodle's directory or in publicly accessible directory.
```bash 
mkdir /srv/mdatafiles -P && \
chown 0700 /srv/mdatafiles
```

Finally, we can create our Moodle configuration. I've already created a template for this that we can use based from the sample.
```config.php
<?php 
// Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype = 'mysqli';
$CFG->dblibrary = 'native';
$CFG->dbhost = 'localhost';
$CFG->dbname = 'mdatabase';
$CFG->dbuser = 'mdbuser';
$CFG->dbpass = 'superSECRETphrase101';
$CFG->prefix = 'mdbdemo_';
$CFG->dboptions = array (
	'dbpersist' => 0,
	'dbport' => '',
	'dbsocket' => '',
	'dbcollation' => 'utf8mb4_unicode_ci',
);
  
$CFG->wwwroot = 'http://mstaged.demo.com/';
// $CFG->urlrewriteclass = '\local_cleanurls\url_rewriter';
$CFG->dataroot = '/srv/mdatafiles';
$CFG->directorypermissions = 02777;
$CFG->admin = 'admin';
$CFG->upgradekey = 'moodleDemoUPGRADEkee';

$CFG->passwordsaltalt1 = 'SOMEpasswordsaltalt1';
$CFG->passwordsaltalt2 = 'SOMEpasswordsaltalt1';
$CFG->passwordsaltalt3 = 'SOMEpasswordsaltalt1';
$CFG->passwordsaltalt4 = 'SOMEpasswordsaltalt1';
$CFG->passwordsaltalt5 = 'SOMEpasswordsaltalt1';


require_once(__DIR__ . '/lib/setup.php');
```


# Congratulations!
You have now successfully setup a standalone Moodle server.



---
# Extras
Here are some extra things to consider.

## Moodle Plugins
Moodle have plenty of plugins created by different communities to provide more tools and utilities that Moodle don't have by default. This can be a bit repetitive as well. We need to download the plug-ins, move them to Moodle's directory, and update their properties. As such, we'll create a script to automate and lighten up the process.

Each plug-ins have their own purpose and use, which is required to be identified to make sure that we are uploading them to the appropriate Moodle sub-directory. For this demo, we'll install three types of plug-ins; a theme, an activity, and an admin tool.
```moodle-plugins.sh
echo Downloading plugins to temporal directory...
mkdir /tmp/ && cd /tmp/
apt update && apt install wget -y
wget https://moodle.org/plugins/download.php/27913/tool_courserating_moodle40_2022111300.zip
wget https://moodle.org/plugins/download.php/28458/mod_jitsi_moodle41_2023012900.zip
wget https://moodle.org/plugins/download.php/24598/theme_moove_moodle311_2021052100.zip

rm /tmp/*
```

### Moodle Plugins Update
The script above can be use as template to update plugins by changing the download links for the plugins. 



## Upgrading Moodle
Updating any software or service is inevitable, Moodle is no exception. For such repetitive tasks, we'll create a script of our own to ease up the process. I've already created a template for this that we can use.
```moodle-update.sh
!#/bin/bash
echo Initiating MySQLDump...
mysqldump -u mdbuser -p mdatabase >> /opt/moodle/mdatabase.sql && \
echo Updating server's reposity and upgrading packages... && \
sudo apt update && sudo apt upgrade apache2 mariadb-server git php* -y && \ 
echo Setting up temporary directory for the new Moodle branch... && \
mkdir /opt/tmp && cd /opt/tmp/ && \
git clone  moodle && cd /opt/ && \
echo Copying Moodle's config to the new Moodle branch... && \
cp /opt/moodle/config.php /tmp/moodle/ && \
echo Archiving the current Moodle directory and replacing with newer version... && \
tar -czf moodle.tar.gz moodle && rm /opt/moodle/ -rf && \
echo Running clean up... && \
mv /opt/tmp/moodle /opt/moodle -r &&
rm /opt/tmp/ -rf \
echo Restarting services... && \
sudo systemctl restart apache2-servicer mariadb-service 
echo Testing... && \
curl https://mstaged.demo.com/
```

**Warning**: This script does not include Moodle plugins.
Note: Moodle branch release will need to updated before running the script. Variables can be use to make this script more interactive.

Finally, simply run the script.
```bash
sudo sh ./moodle-update.sh
```

## Securing/Hardening Server
This is somewhat optional, its always a good practice to protect our server.

### UFW
First, we'll install our firewall, this should installed by default, and set up to allow or reject the ports we want. Since this is a standalone web server, we'll just allow SSH(22) and HTTPS(443) ports.
```bash
sudo apt update && apt install ufw -y && \
ufw enable && \
ufw allow 22 && \
ufw allow 443 && \
ufw reload 
```

### Fail2Ban
Then, we'll install a solution for preventing intrusion using Fail2Ban. 
```bash
sudo apt update && apt install fail2ban -y && \
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local 

```


## Setting a Backup and Restore Solution
A simple back up will always save the day. To avoid such repetitive task, we'll set up a cron task to run a script that automate the process for us. We need to backup our database and Moodle data. This two are critical for restoration.
```bash
sudo mkdir /opt/moodle.bak/ && \
sudo vim /opt/moodle.bak/moodle-backup.sh && \
sudo chmod +x moodle-backup.sh
```

Template:
```moodle-backup.sh
cd /opt/moodle.bak && \
sudo mysqldump -u mdbuser -p superSECRETphrase101 --mdatabase >> mdatabase.sql
cd /srv/ && tar -czf mdatafiles.tar.gz ./mdatafiles/ && \
mv ./mdatafiles.tar.gz /opt/moodle.bak 
```

Note: Of course having a proper backup solution is more than advisable than this simple cron script backup.
