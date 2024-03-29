# Upgrades

## Chat Upgrade

In version 0.9.11 onwards, the default XMPP domain is changed to 'osp.internal'.
New installations that use this version will have 'osp.internal' already set up for them, but existing installations that upgrade to 0.9.11 will need to do additional steps:

### Versions < 0.9.11 to Versions >= 0.9.11

1) Follow the 'Standard Upgrade' instructions to upgrade the OSP instance's code to version 0.9.11.
2) Check in `/opt/osp/conf/config.py` and add to it the following parameter if it's not in the file:
```
### /opt/osp/conf/config.py

# Sets the Ejabberd XMPP Chat Domain
ospXMPPDomain = "osp.internal"
```
3) Update the /path/to/ejabberd/conf/ejabberd.yml to change the 2 domain-specific lines, from your site's domain to "osp.internal"
```
hosts:
  - localhost
  - <your OSP site's domain name>    <-- replace with: osp.internal

host_config:
   "<your OSP site's domain name>":    <-- replace with: "osp.internal"
     auth_method:
       - external
       - anonymous
     allow_multiple_connections: true
     anonymous_protocol: login_anon
```

4) Update /path/to/ejabberd/conf/auth_osp.py to change the "protocol" and "ospAPIServer" variables to the following values:
```
protocol = "http"
ospAPIServer = "127.0.0.1:5010"
```

5) Restart the appropriate services
```
service ejabberd restart
systemctl restart osp.target        # This operation may take much longer compared to the other restarts.
service osp-celery restart
service osp-celery-beat restart
```

## Standard Upgrade
To upgrade any version, make sure that you are on the correct branch (usually master), and follow the steps as listed below:

---

## Versions 0.9.x to x.x.x.x
For all versions of OSP 0.9.0 and greater, the standard upgrade procedure has been changed to the following steps to prevent requiringa separate upgrade script moving forward.

### Single Server Install

1) Open the OSP Directory
```
cd /opt/osp
```
2) Run a Git Pull to pull in the most recent code
```
sudo git pull
```
3) Run the OSP Configuration Tool Script
```
sudo bash osp-config.sh
```
4) Select Option 2: Upgrade...
5) Select Option 1: Upgrade OSP - Single Server

### Split Server Installs
---
Split Server Installs will typically not use the /opt/osp directory as the Git Repository.  In these instances, it would be best to clone the OSP respository

1) Clone the OSP Repo
```
git clone https://gitlab.com/osp-group/flask-nginx-rtmp-manager.git
```
2) Open the Cloned Repo
```
cd flask-nginx-rtmp-manager
```
3) Run the OSP Configuration Tool Script
```
sudo bash osp-config.sh
```
4) Select Option 2: Upgrade...
5) Select the Option Listed for the Component you are updating

To upgrade any version, make sure that you are on the correct branch (usually master), run osp-config.sh and select the compontent to upgrade. If further steps are required for specific versions, they are described below.

---

## Version 0.8.x to Version 0.9.x
Due to changes made with the database upgrade structure, an update script has been made to help reset the state of the database upgrade tracking table. In the process, Nginx will also be updated so it is recommended to backup any customizations made in /usr/local/nginx/conf.  This includes TLS/SSL Cert configurations.

To update to 0.9.x, perform the following:

1) Remove the existing DB Migrations folder
```
sudo rm -rf /opt/osp/migrations
```
2) Perform a git pull of the most recent build
```
sudo git pull
```
3) Run the Upgrade Script
```
sudo bash /opt/osp/setup/upgrade/0.9.0.sh
```
4) After Completion, edit /opt/osp/conf/config.py:
```
sudo nano /opt/osp/conf/config.py
```
5) Add your SMTP Configuration to the config.py file
```
# Email Settings
smtpSendAs="sendAs@email.com"
smtpServerAddress="smtp.email.com"
smtpServerPort=25
smtpUsername=""
smtpPassword=""
# SMTP Encryption Options are ['none', 'tls', 'ssl']
smtpEncryption="none"
```
6) Restart OSP
```
sudo systemctl restart osp.target
```

---

## Versions < 0.8.8 to Versions > 0.8.8 
Due to changes made in v0.8.8 to simplify Nginx Configurations, the upgrade to versions greater to 0.8. will require an extra step to move your server specific changes to the new setup.

> **Note:** Prior to the upgrade, it is recommended you backup the /usr/local/nginx/conf directory to have a copy of your original settings to transpose to the new setup

> **Important Note: When updating, do the following to get the files into alignment:**
{.is-warning}
1) Run the osp-config updater process as normal
2) On completion, exit the osp-config updater
3) Create the the custom directory in the /usr/local/nginx/conf folder
```
sudo mkdir /usr/local/nginx/conf/custom
```
4) Copy the new custom files from the clone repo as listed below to /usr/local/nginx/conf/custom/:
- **OSP-Core, ejabberd, OSP-Edge, Single Servers:** $Repo/installs/nginx-core/osp-custom-*.conf
- **OSP-Edge:** $Repo/installs/osp-edge/setup/nginx/custom/osp-edge-custom-*.conf
- **OSP-RTMP:** $Repo/installs/osp-rtmp/setup/nginx/custom/osp-rtmp-custom-*.conf
5) Reopen the osp-config updater and run the update process a second time.
6) Follow the instructions below to reconfigure the new custom nginx files

The following files will need to be updated for each OSP service:
1) **OSP-Core, OSP-Edge, ejabberd, Single Servers:** SSL/TLS Certificates will need to be transposed to the new location "/usr/local/nginx/conf/custom/osp-custom-servers.conf".  The original information is located in the original /usr/local/nginx/conf/nginx.conf file
- Edit the /usr/local/nginx/conf/custom/osp-custom-servers.conf
```
sudo nano /usr/local/nginx/conf/custom/osp-custom-servers.conf
```
- Comment the following Line:
```listen 80 default_server;```
- Uncomment the following lines and input the information for your TLS/SSL Certificate
```
# listen 443 ssl http2 default_server;
# ssl_certificate /etc/letsencrypt/live/osp.example.com/fullchain.pem;
# ssl_certificate_key /etc/letsencrypt/live/osp.example.com/privkey.pem;
# ssl_protocols TLSv1.2 TLSv1.3;
```
2) **OSP-RTMP:**  This step is only needed when configuring OSP-RTMP to work with OSP-Proxy.
- Edit the /usr/local/nginx/conf/custom/osp-rtmp-custom-authorizeproxy.conf file
- Replace the following line with a list of allow statements for each IP address of OSP-Proxy that will be accessing the RTMP Server's HLS files
```
# allow <ip of proxy>;
```
to (example)
```
allow 23.10.1.34;
allow 23.10.1.35;
```
- Edit the /usr/local/nginx/conf/custom/osp-rtmp-custom-server.conf if using SSL/TLS on your OSP-Core service
```
sudo nano /usr/local/nginx/conf/custom/osp-rtmp-custom-server.conf
```
- Comment out the following line:
```listen 5999 default_server;```
- Uncomment the following lines and change the information for your TLS/SSL Certificate
> Note: The port will remain as 5999
```
# listen 5999 ssl http2 default_server;
# ssl_certificate /etc/letsencrypt/live/osp.example.com/fullchain.pem;
# ssl_certificate_key /etc/letsencrypt/live/osp.example.com/privkey.pem;
# ssl_protocols TLSv1.2 TLSv1.3;
```
3) **OSP-Edge:** This step is only needed if you are using an external OSP-Edge Service or plan on continuing to use one
- Edit the /usr/local/nginx/conf/custom/osp-edge-custom-allowedpub.conf file
```
sudo nano /usr/local/nginx/conf/custom/osp-edge-custom-allowedpub.conf
```
- Replace the following line and add the IP Addresses of all OSP-RTMP Instances
```#allow publish CHANGEME;```
to (example)
```
allow publish 23.10.2.2;
allow publish 23.10.2.3;
```
- Edit /usr/local/nginx/conf/custom/osp-edge-custom-nginxstat.conf
```
sudo nano /usr/local/nginx/conf/custom/osp-edge-custom-nginxstat.conf
```
- Replace the following line and add the IP addresses of all of your OSP-Core Instances
```#allow CHANGEME;```
to (example)
```
allow 23.10.3.2;
allow 23.10.3.3;
```
4) Restart the nginx-osp service on each server
```
sudo systemctl restart nginx-osp
```

---

## 0.8.x to 0.8.x

1) Run the OSP config tool
```bash
sudo bash osp-config.sh
```
2) Select -> 2 Upgrade...
3) Select the component to upgrade (-> 1 if single server installation)

---

## 0.7.x to 0.8.x
> **Note:** SQLite is no longer supported, starting with OSP Version 0.8.0. Migrate to MySQL before upgrading or export a copy of the SQLite Database and reimport into a new MySQL DB while upgrading. (https://dbconvert.com/sqlite/mysql/)  
1) Perform Git Pull on your OSP directory
```bash
cd /opt/osp
sudo git pull
```
2) Run the 0.8.0.sh upgrade prep script
```bash
sudo bash /opt/osp/setup/upgrade/0.8.0.sh
```
3) Run the OSP config tool
```bash
sudo bash osp-config.sh
```
4) Select Install -> 1: Install OSP - Single Server
5) If prompted, reenter your OSP Ejabberd Chat Domain
6) Exit the OSP Config Tool after completed
7) Review the OSP config.py and copy your dbLocation Information from the config.py.old
```bash
cat /opt/osp/conf/config.py.old
sudo nano /opt/osp/conf/config.py
```
8) Review any SSL/TLS Certificate Reinstalls needed for Nginx
```
sudo nano /usr/local/nginx/conf/nginx.conf
```
9) Restart Ejabberd
```
sudo systemctl restart ejabberd
```
10) Restart OSP
```
sudo systemctl restart osp.target
```

---

## Resetting DB Migrations
In some instances, when upgrading you may receive an error about duplicate columns existing.  If this occurs, you can reset the Flask-Migrate settings to allow the system to perform a reset upgrade.

1: Delete the migrations directory
```
sudo rm -rf /opt/osp/migrations
```
2: Open the DB using a Database tool (mysql for mysql-server or sqlite3 for sqllite)
```
sudo mysql
```
3: Select the OSP Database
```
use osp;
```
4: Drop the Alembic Versions Table
```
drop table alembic_version;
```
5: Exit and Rerun a Database Upgrade
```
exit;
sudo bash /opt/osp/osp-config.sh upgrade_db
```