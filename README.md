# High Availability Web Server

## What is High Availability?

High Availability (HA) refers to systems that are designed to be operational and accessible for a high percentage of time, minimizing downtime and ensuring continuous service availability. This is often achieved through redundancy, failover mechanisms, and load balancing.

## Key Components in this setup

- **Apache** (Web Server)
  Acts as the backend server that processes HTTP requests and serves web content. In this setup, multiple Apache web servers are used to serve the same content, ensuring that if one server fails, another can continue serving users without disruption.
- **Nginx** (Load Balancer)  
   Sits in front of the web servers and distributes incoming client requests across multiple backend Apache servers. This helps balance the load, prevent any single server from being overwhelmed, and improves the responsiveness of the web service
- **Keepalived** (Failover Management)
  Works together with Nginx to provide fault tolerance. It manages a Virtual IP Address (VIP) that floats between two Nginx instances (Master and Backup). If the master load balancer fails, Keepalived promotes the backup node to master and assigns the VIP to it, ensuring uninterrupted access for users.

## Objectives of This Module?

In this module, we will implement a High Availability Web Server Architecture that ensures reliable and uninterrupted web service delivery through the combination of load balancing, redundancy, and failover mechanisms.

### Network Topology

The following diagram illustrates the high-level architecture of the High Availability Web Server setup
![Topology](./topology.jpg)

### Description

- Client sends request to the Virtual IP (VIP) managed by Keepalived.

- The VIP routes traffic to one of the Load Balancer nodes (Nginx):

  - If Load Balancer Master is available, it handles the traffic.

  - If the Master fails, Keepalived shifts the VIP to the Backup node.

- Nginx forwards the request to one of the two Apache Web Servers, which serve the actual web content.

- Keepalived ensures failover between the two Load Balancer nodes without service disruption.

---

## Spesifikasi Environment

| No  | Virtual Machine      | Spesifikasi      				| NAT  | Host-Only        | Internal		 |
| --- | -------------------- | --------------------------------	| ---- | ---------------- | ---------------- |
| 1   | Load Balancer Master | 2 vCPU, 2 GB RAM, 20 GB Storage 	| DHCP | 192.168.56.50/24 | 10.10.10.10/24   |
| 2   | Load Balancer Slave  | 2 vCPU, 2 GB RAM, 20 GB Storage 	| DHCP | 192.168.56.51/24 | 10.10.10.11/24   |
| 3   | Web Server 1         | 2 vCPU, 2 GB RAM, 32 GB Storage 	| DHCP | 192.168.56.52/24 | 10.10.10.12/24   |
| 4   | Web Server 2         | 2 vCPU, 2 GB RAM, 32 GB Storage 	| DHCP | 192.168.56.53/24 | 10.10.10.13/24   |

### Simple Description

- NAT		: for internet connection, DHCP allowed, no need for static ip
- Host-Only	: for Host & VM connection, need static ip for easier access
- Internal	: for VM interconnection	

---

## Installation and Configuration Steps

### 1. Configure VM Network Interfaces

Each VM must be configured with three interfaces: NAT, Host-Only, and Internal Network.
Edit file:

```bash
sudo vim /etc/netplan/50-cloud-init.yaml
```

Example configuration for Load Balancer Master:

```yaml
# This file is generated from information provided by the datasource. Changes
# to it will not persist across an instance reboot. To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
  ethernets:
    enp0s3: 			# Network Addresses Translation (NAT) - for internet connection - DHCP
      dhcp4: true
    enp0s8: 			# Host-Only Network - for host & VM only connection - Static IP
      addresses:
        - 192.168.56.X/24		# configure based on server - 50 for LB-master, 51 for LB-slave, 52 for web server 1 & 53 for web server 2
      dhcp4: false
    enp0s9: 			# Internal Network - for VM interconnection - Static IP
      addresses:
        - 10.10.10.X/24			# configure based on server - 10 for LB-master, 11 for LB-slave, 12 for web server 1 & 13 for web server 2
      dhcp4: false
  version: 2
```

Apply the configuration

```bash
sudo netplan apply
```

📝 Repeat this step on each VM, adjusting the IP addresses based on the table above.

### 2. Set Hostname for Each VM

Assign a unique hostname to each virtual machine based on its role, to help easily identify them

```bash
# For Load Balancer Master
sudo hostnamectl set-hostname lb-master

# For Load Balancer Slave
sudo hostnamectl set-hostname lb-slave

# For Web Server 1
sudo hostnamectl set-hostname web1

# For Web Server 2
sudo hostnamectl set-hostname web2
```

🔁 Reboot or logout session each VM tty console so the terminal prompt updates to reflect the new hostname

```bash
reboot
# or
logout
```

- Pro tip: you can use CTRL+D instead of typing "logout" to instantly log out of current tty terminal session

### 3. Update /etc/hosts File

To allow all VMs to resolve each other's hostnames, edit the `/etc/hosts` file on each VM:

```bash
10.10.10.50   lb-master
10.10.10.51   lb-slave
10.10.10.52   web1
10.10.10.53   web2
```

Save and exit. This will allow you to ping and communicate with each node using its hostname instead of using ip address.

### 4. Apache Installation on Web Servers

These steps should be executed on both Web Server 1 and Web Server 2

Update System Packages

```bash
sudo apt update -y
```

Install Apache Web Server

```bash
sudo apt install apache2 -y
```

Enable Apache to Start Apache service

```bash
sudo systemctl enable --now apache2
```

check Apache Service Status

```bash
sudo systemctl status apache2
```

### 5. Install other package (Optional, in this case i'll use php)

To test the Apache server with PHP, install the following packages

```bash
sudo apt install php libapache2-mod-php -y
```

### 6. Prepare Sample Web Content

Create a new directory for the website (usually this directory already exists by default after installing apache)

```bash
sudo mkdir -p /var/www/website
```

Create an `index.php` file with sample content (index.html if you not installing php)

```bash
sudo vim /var/www/website/index.php
```

Example content for Web Server 1

```php
<h1> MAIN PAGE WEB 1</h1>
<h2> Other content here</h2>
<?php phpinfo(); ?>
```

In Web Server 2, change the heading to "WEB 2" to help identify which server is responding during load balancing tests.

### 7. Configure Apache Virtual Host

Create a new virtual host file

```bash
sudo vim /etc/apache2/sites-available/website.conf
```

Add following Configuration

```apacheconf
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/website
    ServerName dinus.local

    <Directory /var/www/website>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Disable the default configuration

```bash
sudo a2dissite 000-default.conf
```

Enable the custom virtual host

```bash
sudo a2ensite website.conf
```

Reload Apache to apply changes

```bash
sudo systemctl reload apache2
```

Test the web server content by opening browser and accessing web server by http://{web server ip} or by executing command below:

```bash
curl http://{web server ip}
```

### 8. Install and Configure Nginx & Keepalived on Load Balancer nodes

To ensure load balancing and automatic failover, this tutorial will install Nginx and Keepalived on both Load Balancer Master and Load Balancer Slave nodes.

#### Install Required Packages

Update the package list

```bash
sudo apt update -y
```

Install Nginx

```bash
sudo apt install nginx -y
```

Install Keepalived

```bash
sudo apt install keepalived -y
```

Enable and start both services

```bash
sudo systemctl enable --now nginx && sudo systemctl enable --now keepalived
```

### 9. Configure Nginx as Load Balancer

Create a new configuration file on both Load Balancer (Load Balancer Master and Load Balancer Slave) nodes

```bash
sudo vim /etc/nginx/conf.d/loadbalancer.conf
```

Add the following configuration to set up Nginx as a load balancer

```nginx
upstream web_backend {
    server 10.10.10.12; # Web Server 1
    server 10.10.10.13; # Web Server 2
}

server {
    listen 80;
    server_name dinus.local;

    location / {
        proxy_pass http://web_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
}
```

Check Nginx syntax for any errors

```bash
sudo nginx -t
```

Reload Nginx to apply the new configuration

```bash
sudo systemctl reload nginx
```

### 10. Configure Keepalived for Failover

Now we will configure Keepalived on both Load Balancer nodes to ensure automatic failover

#### On the Load Balancer Master (lb-master)

edit the Keepalived configuration file

```bash
sudo vim /etc/keepalived/keepalived.conf
```

Paste the following configuration

```conf
vrrp_instance VI_1 {
 interface enp0s8
 state MASTER
 priority 500
 advert_int 1
 unicast_src_ip 192.168.56.50
 unicast_peer {
 192.168.56.51
 }
 virtual_router_id 33
 virtual_ipaddress {
 192.168.56.100/24
 }
 authentication {
 auth_type PASS
 auth_pass udinus
 }
}
```

#### On the Load Balancer Slave (lb-slave)

edit the Keepalived configuration file

```conf
vrrp_instance VI_1 {
 interface enp0s8
 state BACKUP
 priority 100
 advert_int 1
 unicast_src_ip 192.168.56.51
 unicast_peer {
 192.168.56.50
 }
 virtual_router_id 33
 virtual_ipaddress {
 192.168.56.100/24
 }
 authentication {
 auth_type PASS
 auth_pass udinus
 }
}
```

After configuring both nodes,restart the Keepalived service on both Load Balancer nodes

```bash
sudo systemctl restart keepalived
```

Verify that the Virtual IP (VIP) is assigned to the Master node

```bash
ip a
```

### 11. Testing the High Availability Setup

Now that everything is configured, we can test the High Availability setup

#### Access the Web Application

- open a web browser and navigate to the Virtual IP address (192.168.56.100)
  ![web](./test.png)

- Refresh the page several times. You should see alternating content like WEB 1 and WEB 2, which confirms that Nginx is distributing traffic between both backend web servers.
  ![web](./test2.png)

- try power off the Load Balancer Master node (lb-master) and refresh the page again. The VIP should automatically switch to the Load Balancer Slave node (lb-slave), and you should still be able to access the web application without interruption.
  ![Power Off](./vm.png)
- Or you can also test by stopping the Nginx service on the Master node and checking if the Slave node takes over.

```bash
sudo systemctl stop nginx
```

- After stopping the Nginx service on the Master node, refresh the web page. The VIP should automatically switch to the Slave node, and you should still be able to access the web application without interruption.
  ![final](./final.png)

### Conclusion

In this module, we successfully implemented a High Availability Web Server Architecture using Apache, Nginx, and Keepalived. This setup ensures that web services remain available even in the event of server failures, providing a robust solution for maintaining continuous service delivery.
