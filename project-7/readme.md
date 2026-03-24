# DevOps Tooling Website Solution

![AWS](https://img.shields.io/badge/Amazon%20AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Red Hat](https://img.shields.io/badge/Red%20Hat-EE0000?style=for-the-badge&logo=redhat&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![Apache](https://img.shields.io/badge/Apache-D22128?style=for-the-badge&logo=apache&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-777BB4?style=for-the-badge&logo=php&logoColor=white)
![NFS](https://img.shields.io/badge/NFS-Storage-0078D4?style=for-the-badge&logo=linux&logoColor=white)

---

## Overview

Implemented a forked DevOps tooling platform on AWS to explore distributed system design and shared storage mechanisms. The environment integrates Jenkins, Grafana, Prometheus, and Kibana, accessible through a centralized interface.

My primary was Focused on configuring NFS to enable multiple web servers to share a common file system, ensuring data consistency across instances and simulating a scalable prod-like setup backed by MySQL.


### What is NFS and how does it work?

**NFS (Network File System)** is a way for one computer to share a folder so other computers can use it like it’s on their own machine. Instead of copying files between servers, everyone just connects to one central place and works from there.

In my project, the NFS server stores the website files in `/mnt/apps`, and each web server mounts that to `/var/www/html`. So all the servers are using the exact same files. If I update a file from one server, the others see it immediately because they’re all connected to the same source.

I also used NFS for logs by mounting `/var/log/httpd` from all web servers to one shared location. Instead of checking logs on three different machines, everything shows up in one place, which makes debugging much easier.


 ### Seems great but if the NFS server goes down this means all the Webserver loses all thier files. Hmm, i think in a prod environment i'll consider Amazon EFS just incase .

---

## Architecture

This project uses a **4-component setup** running on AWS:

![mongo list](./images/task%207%20arch.jpg)

All three web servers mount the same NFS export, so they share one `/var/www/html` folder. They also all point to the same MySQL database. From the outside, it looks like one website — but it's actually three servers working together.

---

## Step 1: Prepare the NFS Server

### Set Up LVM Storage

I launched a new RHEL 8 EC2 instance for the NFS server, attached 3 volumes, and set them up using LVM — the same way I did in the previous project. The key difference here is that I formatted the volumes as **xfs** instead of ext4.

Aleardy knew how to create LVMs Thanks to [Project 6](../project-6)

I created 3 logical volumes:

```bash

sudo lvcreate -n lv-apps -L 14G webdata-vg
sudo lvcreate -n lv-logs -L 14G webdata-vg
sudo lvcreate -n lv-opt  -L 14G webdata-vg

# Format as xfs
sudo mkfs -t xfs /dev/webdata-vg/lv-apps
sudo mkfs -t xfs /dev/webdata-vg/lv-logs
sudo mkfs -t xfs /dev/webdata-vg/lv-opt
```

![](./images/task%207%20lvs.jpg)

Then I created the mount points and mounted the volumes:

```bash
sudo mkdir -p /mnt/apps /mnt/logs /mnt/opt

sudo mount /dev/webdata-vg/lv-apps /mnt/apps
sudo mount /dev/webdata-vg/lv-logs /mnt/logs
sudo mount /dev/webdata-vg/lv-opt  /mnt/opt
```

I updated `/etc/fstab` with the UUIDs to make the mounts survive a reboot.

### Install and Start the NFS Server

```bash
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```

### Set Permissions on the Shared Folders

I gave open read/write/execute permissions to the mount points so the web servers could access them freely:

```bash
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```

![](./images/task%207%20nfs%20chown.jpg)

### Export the NFS Mounts

I configured NFS to allow connections from my web servers' subnet CIDR. I found the subnet CIDR from the EC2 networking tab in AWS:

![](./images/task%207%20check%20cidr.jpg)

```bash
sudo vi /etc/exports
```

I added these lines:

```
/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt  <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
```

![](./images/task%207%20perm%20with%20cidr.jpg)

Then I exported the config:

```bash
sudo exportfs -arv
```

### Open NFS Ports in AWS

I checked which ports NFS uses:

```bash
rpcinfo -p | grep nfs
```

![](./images/task%207%20sudo%20exportfs.jpg)

Then I opened the following ports in the NFS server's security group inbound rules:
- **TCP 2049** — NFS
- **TCP 111** and **UDP 111** — RPC port mapper
- **UDP 2049** — NFS over UDP

![](./images/task%207%20config%20of%20nfs.jpg)

---

## Step 2: Configure the Database Server

I launched an UbuntuEC2 instance, installed MySQL, and set up the database for the tooling website:

```sql
sudo mysql

CREATE DATABASE tooling;
CREATE USER 'webaccess'@'<Subnet-CIDR>' IDENTIFIED BY 'password';
GRANT ALL ON tooling.* TO 'webaccess'@'<Subnet-CIDR>';
FLUSH PRIVILEGES;
exit
```

> I restricted the `webaccess` user to connections coming only from the web servers' subnet — not from anywhere on the internet.

![](./images/task%207%20mysql%202.jpg)
![](./images/task%207%20mysql%201.jpg)
---




## Step 3: Prepare the Web Servers

I repeated these steps on all **three** web server instances.

### Install NFS Client

```bash
sudo yum install nfs-utils nfs4-acl-tools -y
```
![](./images/task%207%20nfs-utils.jpg)

### Mount the NFS Share

This is where things clicked for me. I mounted the NFS server's `/mnt/apps` export directly to `/var/www` on the web server:

```bash
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP>:/mnt/apps /var/www
```

Once I did this on all three web servers, they were all pointing to the exact same folder — the one living on the NFS server. Any file I created on one web server instantly appeared on the others.

`My NFS sever`
![](./images/task%207%20test%20nifs%201.jpg)

`My web server`
![](./images/task%207%20test%20nifs%202.jpg)

 That was the moment I really understood how NFS works: the web servers don't actually store the files themselves. They just read and write to the NFS server as if it were a local folder. From the browser's perspective, it doesn't matter which web server handles the request — they all serve from the same `/var/www/html`.

I made the mount permanent by adding it to `/etc/fstab`:

```bash
sudo vi /etc/fstab
```

Added:

```
<NFS-Server-Private-IP>:/mnt/apps /var/www nfs defaults 0 0
```

![](./images/task%207%20sudo%20vi%20etcfstab%20for%20servers.jpg)

### Install Apache and PHP

```bash
sudo yum install httpd -y

sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

sudo dnf module reset php
sudo dnf module enable php:remi-7.4

sudo dnf install php php-opcache php-gd php-curl php-mysqlnd

sudo systemctl start php-fpm
sudo systemctl enable php-fpm

setsebool -P httpd_execmem 1
```
![](./images/task%207%20php%20i.jpg)

### Mount the Log Folder to NFS

I also mounted Apache's log folder to the NFS server so all three web servers share the same logs:

```bash
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP>:/mnt/logs /var/log/httpd
```

And added it to `/etc/fstab` to persist on reboot.

### Deploy the Tooling Application

I forked the tooling source code from and deployed it to the web server:

```bash
# Clone the repo and copy the html folder to /var/www/html
```
![](./images/task%207%20git.jpg)

Since the `/var/www` folder is shared via NFS, I only had to deploy the code once — all three servers picked it up automatically.

### The SELinux Error I Hit

After deploying the code, I opened the browser and got a **403 Forbidden** error. Apache could see the files but couldn't actually serve them. I checked permissions and they looked fine, so I was confused for a while.

The issue turned out to be **SELinux** — a security layer in RHEL that was blocking Apache from reading the files even though the file permissions said it should be allowed. SELinux adds an extra layer of control on top of regular Linux permissions, and by default it's very strict.

To fix it temporarily and confirm that was the problem, I ran:

```bash
sudo setenforce 0
```
![](./images/task%207%20setenforce%200.jpg)

That immediately fixed the 403 error and the site loaded. To make the fix permanent so it doesn't reset after a reboot, I edited the SELinux config file:

```bash
sudo vi /etc/sysconfig/selinux
```

And changed:

```
SELINUX=enforcing
```

to:

```
SELINUX=disabled
```
![](./images/task%207%20dis.jpg)

Then restarted Apache:

```bash
sudo systemctl restart httpd
```

That was a good lesson — on RHEL systems, SELinux can block things that look like they should work fine. When I hit a 403 that permissions alone can't explain, SELinux is now the first place I look.

### Connect to the Database

I updated the website's config file to point to the database server:

```bash
sudo vi /var/www/html/functions.php
```


Then I applied the database schema:

```bash
mysql -h <database-private-ip> -u webaccess -p tooling < tooling-db.sql
```
![](./images/task%207%20sql%20funrtion%20config.jpg)

And created an admin user in MySQL:

```sql
INSERT INTO users (id, username, password, email, user_type, status)
VALUES (1, 'myuser', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1');
```

![](./images/task%207%20final%20sql.jpg)

### Open Port 80

I added an inbound rule in the web server's security group to allow TCP port 80 from anywhere (`0.0.0.0/0`) so the site could be reached from a browser.

### Access the Website

I opened a browser and went to:

```
http://<Web-Server-Public-IP>/index.php
```

I logged in with `myuser` and the site loaded successfully — confirming that the web servers, NFS storage, and MySQL database were all working together.

![](./images/task%207%20finall%20php%20login.jpg)
![](./images/task%207%20php%20inex.jpg)

---

## What I Learned

This was one of the most eye-opening projects so far. A few things really stuck with me:

- **How NFS actually works** — I always knew "shared storage" was a thing, but seeing it in action made it real. The moment I created a file on one web server and saw it instantly appear on the other two, everything clicked. The web servers don't hold the files — they just read from a shared location over the network, and the browser never knows the difference.

- **What "stateless" web servers really means** — Because the files and database both live outside the web servers, I could shut down any one of the three servers and the website would keep running. That's the whole point of this pattern.

- **SELinux is a real thing** — I spent time debugging a 403 error that had nothing to do with file permissions. SELinux was silently blocking Apache in the background. Running `sudo setenforce 0` fixed it instantly and taught me to always check SELinux when something on RHEL refuses to work for no obvious reason.

- **Subnet-scoped database access** — Restricting the MySQL user to only connect from the web servers' subnet (not `0.0.0.0`) is a small but important security habit that I'm now building into every project.