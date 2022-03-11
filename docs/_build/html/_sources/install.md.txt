# Installation
## Requirements
OSP has been verified to work with the following requirements

- Ubuntu 18.04 or later, Debian 10 or later
- Python 3.7 or later
- MySQL 5.7.7 or later, or MariaDB > 10.1, if not using SQLite
- SMTP Mail Server for Email Address Validation and Subscriptions
- FFMPEG 4 or greater
- Dual Core Processor at 2.4 Ghz
- 4 GB RAM
- 120 GB HDD Storage
- Upstream Bandwidth > 35Mbps for 720p/30fps  Streams to 10 people @  3500kbps bit rate

## Install Open Streaming Platform
### Script Install - Single Server (OSP-Core, OSP-RTMP, Ejabberd, Redis, MySQL)
1) Clone the git repository
```bash
git clone https://gitlab.com/Deamos/flask-nginx-rtmp-manager.git
```
2) Install the Config Tool Prerequisites (if not already installed)
```bash
sudo apt-get install dialog
```
3) Run the OSP Configuration Tool
```bash
cd flask-nginx-rtmp-manager
sudo bash osp-config.sh
```
4) Select Option 1 - "Install..."
5) Select Option 1 - "Install OSP - Single Server"
6) During the install process, the Config Tool will ask for an Ejabberd Full Qualified Domain Name (FQDN). This should be the same as the public domain name which will be used to access OSP. This should be a valid DNS entry as it is used to configure Ejabberd's Chat Domain and by default is used by the chat client to connect users to the XMPP chat system. IP addresses may not function properly.
7) On completion, exit the OSP Config Tool.
8) Review the values in the OSP /opt/osp/conf/config.py.
> **_NOTE:_** secretKey and passwordSalt should be changed from their default values.
```bash
sudo nano /opt/osp/conf/config.py
```
9) Restart the OSP Core Workers
```bash
sudo systemctl restart osp.target
```
To test streaming on the server, see the "Testing OSP server" section of the [Streaming](/Usage/Streaming) page.
### Script Install - Split Server Install - OSP Components on Different Servers
Starting with OSP version 0.8.0, OSP components can be split over multiple servers. This helps with spreading the load required for a busy OSP install with many viewers. In addition, splitting the components can be useful to set up load balancing by having multiple copies of the component and using a load balancer, such as HAproxy.
To perform a Split Server Setup, please review the following requirements:
* **Componentaization** - Multiple components can be installed on a single server to reduce cost. Doing so can also prevent needing some of the considerations in this list. For Example, if you consolidate OSP-Core and OSP-RTMP and do not require OSP-Edge Servers, you will not need Centralized storage as they
* **Centralized Storage** - OSP requires some form of mounted centralized storage for Videos, Clips, & Stream/Video Thumbnails. This can be accomplished easily by using an S3-based storage bucket and using s3fs to mount the bucket to the servers file systems. Another method would be a NFS mount in the require location. Below is the required drive mounts and locations
* **Mounts**
* /var/www/videos - OSP-Core, OSP-RTMP
* /var/www/stream-thumb - OSP-Core, OSP-RTMP
* /var/www/images - OSP-Core
* **SSL/TLS** - If OSP Core systems use HTTPS with SSL/TLS certificates, certificates will also be needed for the Ejabberd, Edge, Proxy, or OSP-RTMP (Only if using Proxy) Servers to prevent issues with HTTP(s) mixed content.
* **MySQL & Redis** - the OSP Config Tool does not have an option for MySQL and Redis installs. It is recommended to be familar with their install and configuration prior to a Split Server Install
In some instances, some services can be co-located on the same server. See rules below:
* OSP-Core, OSP-RTMP, Redis, ejabberd, and Database can exist on the same server
* OSP-Edge can not exist on the same server as OSP-RTMP
* OSP-Proxy can not exist on the same server as OSP-Edge, OSP-Core, OSP-RTMP, or ejabberd
#### Recommended Install Order
Below is the recommended order of setting up split servers. This is due to some of the dependancies requires for some servers to function properly.
1) Centralized Storage - Have a server ready to connect to
2) Ejabberd
3) Redis
4) Database
5) OSP-Core
6) OSP-RTMP
7) OSP-Edge
8) OSP-Proxy
#### **Centralized Storage Mounting (Use with OSP-Core and OSP-RTMP)**
These instructions are intended to be used **after the OSP-Core and OSP-RTMP install processes**. After installing the OSP components, it is recommended to determine and establish a central storage server setup on completion of component deployment.
Due to the many possible configurations of using central, shared storage, it is impossible to cover a "best" method for doing so. For the purposes of this documentation, it is assumed that an S3-compatable bucket will be used and mounting is covered below using s3fs.
**DO NOT USE s3fs 1.86-1.**
**A critical bug in caching can lead to mount failure of the s3 bucket, leading to loss of recordings. Currently the latest Ubuntu LTS (20.04) only has this version of s3fs available.**
1) Install s3fs
```bash
sudo apt-get update && sudo apt-get install s3fs
```
2) Create a password file. This will contain the S3 key and secret token:
```bash
echo S3KEY:S3TOKEN > ~/.passwd-s3fs
```
3) Set the permissions to secure the file
```bash
chmod 600 ~/.passwd-s3fs
```
4) Edit the Fuse Configurations to allow access by non-root users to files
```bash
sudo nano /etc/fuse.conf
```
5) Comment out the following line ```user_allow_other```
6) Identify and write down the uid and gid of the www-data user. Example: ```uid=33(www-data) gid=33(www-data) groups=33(www-data)```
```bash
sudo -u www-data id
```
7) Create the required stub locations:
```bash
sudo mkdir -p /var/www/videos
sudo mkdir -p /var/www/images
sudo mkdir -p /var/www/stream-thumb
```
8) Mount the directories to the S3-compatible bucket using your S3 Credentials and the identified UID & GID frm Step 6
```bash
s3fs <space_name> /var/www/videos -o url=<s3 endpoint> -o use_cache=/tmp -o allow_other -o use_path_request_style -o uid=<UID> -o gid=<GID>
s3fs <space_name> /var/www/images -o url=<s3 endpoint> -o use_cache=/tmp -o allow_other -o use_path_request_style -o uid=<UID> -o gid=<GID>
s3fs <space_name> /var/www/stream-thumb -o url=<s3 endpoint> -o use_cache=/tmp -o allow_other -o use_path_request_style -o uid=<UID> -o gid=<GID>
```
9) Verify the Mount was successful. The buckets should show as a s3fs mount at the bottom of the output
```bash
mount
```
> Note: To mount persistently, you can add the following to your fstab file
```
s3fs#<bucket_name> /var/www/images fuse url=<endpoint_url>,use_cache=/tmp,allow_other,use_path_request_style,_netdev,uid=33,gid=33 0 0
```
#### Ejabberd
1) Clone the git repository
```bash
git clone https://gitlab.com/Deamos/flask-nginx-rtmp-manager.git
```
2) Install the Config Tool Prerequisites (if not already installed)
```bash
sudo apt-get install dialog
```
3) Run the OSP Configuration Tool
```bash
cd flask-nginx-rtmp-manager
sudo bash osp-config.sh
```
4) Select Option 1 - "Install..."
5) Select Option 6 - "Install Ejabberd"
6) During the install process, the Config Tool will ask for an Ejabberd Full Qualified Domain Name (FQDN). This should be the same as the public domain name which will be used to access OSP. This should be a valid DNS entry. Use of an IP address may not function properly.
7) On completion, exit the OSP Config Tool.
8) Setup a new Ejabberd admin account.
```bash
sudo /usr/local/ejabberd/bin/ejabberdctl register admin localhost <password>
```
9) Edit the ejabberd.yml
```bash
sudo nano /usr/local/ejabberd/conf/ejabberd.yml
```
10) Change the following lines to match your expected configuration:
* Line 43-44
```yaml
port: 5443
ip: "::"
```
* Line 55-56
```yaml
port: 5280
ip: "::"
```
* Line 77-78
```yaml
port: 4560
ip: "::"
```
* Line 91-93 (Add one line per OSP core or use CIDR Notation to allow a block of IPs)
```yaml
ip:
- 127.0.0.0/8
- ::1/128
- <ip address of OSP Core>
```
11) Save the ejabberd.yml file
12) Edit the auth_osp.py Authentication Handler
```
sudo nano /usr/local/ejabberd/conf/auth_osp.py
```
13) Edit the protocol and ospAPIServer variables to match your OSP Core Instance.
```python
protocol = "https"
ospAPIServer = "osp.example.com"
```
14) Save the auth_osp.py file
15) Restart Ejabberd
```bash
sudo systemctl restart ejabberd
```
#### Redis
1) Install Redis
```bash
sudo apt update
sudo apt install redis-server
```
2) Edit the redis.conf file
```bash
sudo nano /etc/redis/redis.conf
```
3) Find & Edit the bind location to listen on all interfaces
```conf
bind 0.0.0.0
```
4) Find and Set a Redis Password
```conf
requirepass <Password>
```
5) Save the redis.conf file
6) Restart Redis
```bash
sudo systemctl restart redis.service
```
#### Database
1) Install MariaDB
```bash
sudo apt-get update && sudo apt-get install mariadb-server
```
2) Download the OSP Modifications for MariaDB
```bash
sudo wget "https://gitlab.com/Deamos/flask-nginx-rtmp-manager/-/raw/master/setup/mysql/mysqld.cnf" -O /etc/mysql/my.cnf
```
3) Edit the my.cnf file
```bash
sudo nano /etc/mysql/my.cnf
```
4) Edit the bind-bind address to listen on all interfaces
```cnf
bind-address = 0.0.0.0
```
5) Restart MariaDB
```bash
sudo systemctl restart mysql
```
6) Log into MariaDB
```bash
sudo mysql
```
7) Create the Database & User. Be aware the remote server ips should be the IP address(es) of the OSP-Core Systems (See https://mariadb.com/kb/en/configuring-mariadb-for-remote-client-access/#granting-user-connections-from-remote-hosts)
```sql
CREATE DATABASE osp;
CREATE USER '<username>'@'<remote_server_ip>' IDENTIFIED BY '<password>';
GRANT ALL PRIVILEGES ON osp.* TO '<username>'@'<remote_server_ip>';
```
8) Quit the MariaDB Console
```sql
quit;
```
9) The Database will be initialized on the successful run of an OSP-Core Instance
#### OSP-Core
1) Clone the git repository
```bash
git clone https://gitlab.com/Deamos/flask-nginx-rtmp-manager.git
```
2) Install the Config Tool Prerequisites (if not already installed)
```bash
sudo apt-get install dialog
```
3) Run the OSP Configuration Tool
```bash
cd flask-nginx-rtmp-manager
sudo bash osp-config.sh
```
4) Select Option 1 - "Install..."
5) Select Option 2 - "Install OSP-Core"
6) On completion, exit the OSP Config Tool.
7) Copy the OSP config.py.dist file to config.py
```bash
sudo cp /opt/osp/conf/config.py.dist /opt/osp/conf/config.py
```
8) Edit the config.py file
```bash
sudo nano /opt/osp/conf/config.py
```
9) Change the dbLocation variable to match your database credentials and IP/DNS
```python
dbLocation = 'mysql+pymysql://<user>:<password>@<db_host>/<db_name>?charset=utf8mb4'
```
10) Change the Redis variables to match the IP/DNS and password set for it
```python
redisHost="redis.example.com"
redisPort=6379
redisPassword="redis_password"
```
11) Change the Ejabberd variables to match your configuration
```python
ejabberdAdmin = "admin" <--Leave this as admin
ejabberdPass = "ejabberd_admin_password"
ejabberdHost = "localhost" <--Leave this as localhost
ejabberdServer ="ejabberd.example.com"
```
12) Review the values in the OSP /opt/osp/conf/config.py.
> **_NOTE:_** secretKey and passwordSalt should be changed from their default values.
```bash
sudo nano /opt/osp/conf/config.py
```
13) Restart the OSP Core Workers
```bash
sudo systemctl restart osp.target
```
14) Save the config.py file
15) Initialize the Database by running the command line upgrader
```bash
sudo bash osp-config.sh upgrade db
```
16) Open a web browser and browse to:
```http://<OSPCore IP Address or DNS>```
17) Setup OSP using the Initial Configuration Wizard
#### OSP-RTMP
1) Clone the git repository
```bash
git clone https://gitlab.com/Deamos/flask-nginx-rtmp-manager.git
```
2) Install the Config Tool Prerequisites (if not already installed)
```bash
sudo apt-get install dialog
```
3) Run the OSP Configuration Tool
```bash
cd flask-nginx-rtmp-manager
sudo bash osp-config.sh
```
4) Select Option 1 - "Install..."
5) Select Option 3 - "Install OSP-RTMP"
6) On completion, exit the OSP Config Tool.
7) Copy and Edit the OSP-RTMP config.py file
```bash
sudo cp /opt/osp-rtmp/conf/config.py.dist /opt/osp-rtmp/conf/config.py
sudo nano /opt/osp-rtmp/conf/config.py
```
8) Change the ospCoreAPI Variable to point at your OSP-Core instance
```python
ospCoreAPI = "http://ospcore.example.com"
```
9) Start the OSP-RTMP Instance
```bash
sudo systemctl start osp-rtmp
```
10) Open a web browser and go to your OSP-Core Instance
```
http://ospcore.example.com
```
11) Log on as an Admin and Open the Admin Settings
12) Select RTMP Servers
13) Click the Plus Sign Button
14) Type in the IP or Fully Qualified Domain Name for your OSP-RTMP Server and click Add
#### OSP-Edge
1) Clone the git repository
```bash
git clone https://gitlab.com/Deamos/flask-nginx-rtmp-manager.git
```
2) Install the Config Tool Prerequisites (if not already installed)
```bash
sudo apt-get install dialog
```
3) Run the OSP Configuration Tool
```bash
cd flask-nginx-rtmp-manager
sudo bash osp-config.sh
```
4) Select Option 1 - "Install..."
5) Select Option 4 - "Install OSP-Edge"
6) When prompted, input the IP address of your OSP-RTMP Instance
7) On completion, exit the OSP Config Tool.
8) If you need to add additional authorized OSP-RTMP Instances, edit the osp-edge-rtmp.conf file for Nginx
```bash
sudo nano /usr/local/nginx/conf/services/osp-edge-rtmp.conf
```
9) Add any additional authorize publishing IPs to stream-data and stream-data-adapt:
```conf
allow publish <IP Address>;
```
10) Restart Nginx
```bash
sudo systemctl restart nginx-osp
```
12) Open a web browser and go to your OSP-Core Instance
```
http://ospcore.example.com
```
13) Log on as an Admin and Open the Admin Settings
14) Select Edge Streamers
15) Add the Fully Qualified Domain Name or IP Address of the Edge Server
16) Add the Load Percentage that the Edge Server will use.
> **_NOTE:_** The sum of all Edge Servers must equal 100%.
17) Restart Nginx on all OSP-Core Servers
```bash
sudo systemctl restart nginx-osp
```
#### OSP-Proxy
1) Clone the git repository
```bash
git clone https://gitlab.com/Deamos/flask-nginx-rtmp-manager.git
```
2) Install the Config Tool Prerequisites (if not already installed)
```bash
sudo apt-get install dialog
```
3) Run the OSP Configuration Tool
```bash
cd flask-nginx-rtmp-manager
sudo bash osp-config.sh
```
4) Select Option 1 - "Install..."
5) Select Option 5 - "Install OSP-Proxy"
6) When prompted, input the Protocol and Fully Qualified Domain Name of your OSP-Core Instance (ex: https://osp.example.com)
7) On completion, exit the OSP Config Tool.
8) If you are using TLS/SSL on your Core Site, Acquire a TLS Certificate
9) Edit /usr/local/nginx/conf/custom/osp-proxy-custom-servers.conf
```bash
sudo nano /usr/local/nginx/conf/custom/osp-proxy-custom-servers.conf
```
10) Change Line 8 to match your OSP Core FQDN
```
valid_referers server_names osp.example.com ~.;
```
11) If you are using TLS, Comment the following Line:
```listen 80 default_server;```
12) Uncomment the following lines and add the TLS configuration:
```
# listen 443 ssl http2 default_server;
# ssl_certificate /etc/letsencrypt/live/osp.example.com/fullchain.pem;
# ssl_certificate_key /etc/letsencrypt/live/osp.example.com/privkey.pem;
# ssl_protocols TLSv1.2 TLSv1.3;
```
13) Edit the OSP-Proxy Configuration File
```bash
sudo nano /opt/osp-proxy/conf/config.py
```
14) Change the Flask Secret Key to a Random Value
```
# Flask Secret Key
secretKey="CHANGEME"
```
15) If you are dedicating the proxy to a specific source (An Edge or Another Proxy), uncomment the ForceDestination Lines and set to match your source
```
# Force Destination - Use to point to Specified Edge Server or Tiered Proxy. Uncomment to override API's RTMP List and use the destination you list
forceDestination = "edge.example.com"
forceDestinationType = "edge" # Choices are "edge", "proxy"
```
15) Restart Nginx-OSP and OSP-Proxy
```bash
sudo systemctl restart nginx-osp
sudo systemctl restart osp-proxy
```
16) Ensure all Edge or RTMP Servers are listed in the OSP-Core Admin Panel. OSP-Proxy will query the OSP API and generate configurations to handle the proxy connection to retrieve the HLS fragments.
17) Perform a first run of the Configuration File Generator
```bash
cd /opt/osp
sudo bash updateUpstream.sh
```
18) Connect to each OSP-RTMP/Single Server and perform the following on each:
- If using TLS on the Core, Generate a TLS certificate
- Edit the /usr/local/nginx/conf/custom/osp-rtmp-custom-authorizeproxy.conf
```bash
sudo nano /usr/local/nginx/conf/custom/osp-rtmp-custom-authorizeproxy.conf
```
- Add the IP Address of each OSP-Proxy that will be accessing the source for /stream-thumb, /live-adapt, and /live:
```
# allow <ip of proxy>;
allow 201.13.12.50;
allow 193.10.3.9;
```
- If using TLS, Edit the /usr/local/nginx/conf/custom/osp-rtmp-custom-server
```
sudo nano /usr/local/nginx/conf/custom/osp-rtmp-custom-server
```
- Comment out the first line, uncomment the TLS information and add your certificate info (Note: Port 5999 will stay the same):
```
#listen 5999 default_server;
### Comment Above and Uncomment/Edit Below for OSP-Proxy TLS ###
listen 5999 ssl http2 default_server;
ssl_certificate /etc/letsencrypt/live/osp.example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/osp.example.com/privkey.pem;
ssl_protocols TLSv1.2 TLSv1.3;
```
- Restart the Nginx-OSP instance
```bash
sudo systemctl restart nginx-osp
```
18) If you forced an Edge Server on Step 14, Connect to the Edge Server and edit /usr/local/nginx/conf/locations/osp-edge-redirects.conf
```bash
sudo nano /usr/local/nginx/conf/custom/osp-edge-redirects.conf
```
- Comment the add_headers line for /edge and /edge-adapt
```
#add_header 'Access-Control-Allow-Origin' "*" always;
```
- Edit the custom referes file at /usr/local/nginx/conf/custom/osp-edge-custom-refer.conf
```
sudo nano /usr/local/nginx/conf/custom/osp-edge-custom-refer.conf
```
- Add the OSP Core Server to the Valid Referers list for /edge and /edge-adapt
```
valid_referers server_names osp.example.com ~.;
```
- Restart Nginx-OSP
```
sudo systemctl restart nginx-osp
```
19) Add the OSP-Proxy Domain to the OSP-Core's Admin Panel under Settings
20) Test a Stream and verify that the video is displaying
