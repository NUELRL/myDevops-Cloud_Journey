# Web Solution with WordPress

![WordPress](https://img.shields.io/badge/WordPress-21759B?style=for-the-badge&logo=wordpress&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![Apache](https://img.shields.io/badge/Apache-D22128?style=for-the-badge&logo=apache&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-777BB4?style=for-the-badge&logo=php&logoColor=white)
![AWS](https://img.shields.io/badge/Amazon%20AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

---


In this project, I set up a full web solution using WordPress on two Linux servers in AWS. One server runs the WordPress website and another runs the MySQL database  and I configured the storage on both servers from scratch using disk partitioning and LVM (Logical Volume Manager).

`For large enterprise environments, I would choose Red Hat Enterprise Linux because it provides strong security policies, long-term stability, and dedicated enterprise-level support but in this project i chose good old UBUNTU`


### Why does this matter in real life?

Your browser just sent a request. Somewhere, a server caught it, handed it to a database, and fired back a response all before you finished blinking. That split-second chain of events is what I spent my time building from scratch. Two servers, one purpose: don't break.

---

## Three-Tier Architecture

This project follows the **Three-Tier Architecture** pattern, which splits a web application into three layers:

1. **Presentation Layer** — The browser or client that the user interacts with
2. **Business Layer** — The web server running WordPress and Apache
3. **Data Layer** — The database server running MySQL

![Three-Tier Architecture](./images/three-tier.png)

My setup used:
- A laptop as the client
- An EC2 Linux server as the **Web Server** (WordPress lives here)
- An EC2 Linux server as the **Database Server** (MySQL lives here)

---

## Step 1: Prepare the Web Server

### Launch the EC2 Instance and Attach Volumes

I launched a EC2 instance to serve as my web server, then created **3 volumes of 10 GiB each** in the same availability zone and attached them to the instance.

![EC2 Instance](./images/task%206%20ec2.jpg)

![Create Volumes](./images/task%206%20creat%20volume.jpg)

![Attach Volumes](./images/task%206%20attach%20volumes.jpg)

![Volumes Attached](./images/task%206%20volumes.jpg)

### Inspect the Attached Disks

I used `lsblk` to confirm the three new block devices were visible (they showed up as `xvdf`, `xvdg`, and `xvdh`):

```bash
lsblk
ls /dev/
```

![lsblk output](./images/task%206%20lsblk.jpg)

![ls /dev output](./images/task%206%20ls%20dev.jpg)

I also checked the current disk usage and available space:

```bash
df -h
```

![df -h output](./images/task%206%20df.jpg)

### Partition the Disks

I created a single partition on each of the 3 disks using `fdisk`:

```bash
sudo fdisk /dev/xvdf
sudo fdisk /dev/xvdg
sudo fdisk /dev/xvdh
```

![fdisk output](./images/task%206%20fdisk.jpg)

I ran `lsblk` again to confirm the partitions were created.

### Set Up LVM

Then I marked each partition as a **Physical Volume (PV)** so LVM could use them:

```bash
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```

![pvcreate output](./images/task%206%20pvcreate.jpg)

I grouped all 3 physical volumes into a single **Volume Group (VG)** called `webdata-vg`:

```bash
sudo vgcreate webdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1
sudo vgs
```

![vgs output](./images/task%206%20vgs.jpg)

I then split the volume group into two **Logical Volumes (LVs)**:
- `apps-lv` — to store website files
- `logs-lv` — to store log data

```bash
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
sudo lvs
```

![lvs output](./images/task%206%20lvs.jpg)

I verified the full setup looked correct:

```bash
sudo vgdisplay -v
sudo lsblk
```

![vgdisplay output](./images/task%206%20sudo%20vgdisplay.jpg)

![lsblk output](./images/task%206%20lsblk2.jpg)

### Format and Mount the Logical Volumes

I formatted both logical volumes with the `ext4` filesystem:

```bash
sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
sudo mkfs -t ext4 /dev/webdata-vg/logs-lv
```

![mkfs output](./images/task%206%20sudo%20mkfs.jpg)

Then I created the directories and mounted everything:

```bash

sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs


sudo mount /dev/webdata-vg/apps-lv /var/www/html/


sudo rsync -av /var/log/. /home/recovery/logs/
```

![rsync backup](./images/task%206%20sudorsuync.jpg)

```bash
# Mount the logs volume
sudo mount /dev/webdata-vg/logs-lv /var/log

# Restore the log files
sudo rsync -av /home/recovery/logs/. /var/log
```

![rsync restore](./images/task%206%20rsync2.jpg)

### Make the Mounts Permanent

Without this step, the mounts would disappear after a server restart. I used the device UUIDs to update `/etc/fstab`:

```bash
sudo blkid
```

![blkid output](./images/task%206%20sudo%20blkid.jpg)

```bash
sudo nano /etc/fstab
```

![fstab edit](./images/task%206%20nano1.jpg)

Then I tested the config and reloaded the system daemon:

```bash
sudo mount -a
sudo systemctl daemon-reload
```

![mount test](./images/task%206%20sudomount%20-a.jpg)

I ran `df -h` one more time to confirm everything was mounted correctly:

![df -h final](./images/task%206%20test%20mounts.jpg)

---

## Step 2: Prepare the Database Server

I launched a second EC2 instance for the database server and repeated all the same disk setup steps from Step 1. The only differences were:

- I created `db-lv` instead of `apps-lv`
- I mounted it to `/db` instead of `/var/www/html`

---

## Step 3: Install WordPress on the Web Server

I updated the server and installed Apache, PHP, and all the tools WordPress needs:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wget apache2 php php-mysqlnd php-fpm php-json
```

![Apache install](./images/task%206%20-i%20apache%20and%20the%20rest.jpg)

I started and enabled Apache so it runs automatically on reboot:

```bash
sudo systemctl start apache2
sudo systemctl enable apache2
```

![Apache test](./images/task%206%20apache%20test.jpg)

Then I installed the PHP extensions WordPress requires:

```bash
sudo apt install php8.3 php8.3-fpm php8.3-opcache php8.3-gd php8.3-curl php8.3-mysql
```

![PHP install](./images/task%206%20php%20i.jpg)

![PHP start](./images/task%206%20php%20staert.jpg)

I downloaded WordPress, extracted it, and copied it into the web directory:

```bash
mkdir wordpress && cd wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar xzvf latest.tar.gz
sudo rm -rf latest.tar.gz
sudo cp wordpress/wp-config-sample.php wordpress/wp-config.php
sudo cp -R wordpress /var/www/html/
```

![WordPress download](./images/task%206%20wordpress%20i.jpg)

Then I set the correct file ownership and permissions so Apache could read and serve the WordPress files:

```bash
sudo chown -R www-data:www-data /var/www/html/wordpress
sudo chmod -R 755 /var/www/html/wordpress
sudo systemctl restart apache2
```

![WordPress permissions](./images/task%206%20wp%20perm.jpg)

---

## Step 4: Install MySQL on the Database Server

On the DB server, I installed MySQL:

```bash
sudo apt update
sudo apt install mysql-server
sudo systemctl restart mysqld
sudo systemctl enable mysqld
```

![MySQL running](./images/task%206%20sql.jpg)

---

## Step 5: Configure the Database for WordPress

I created a dedicated database and user for WordPress, and only allowed connections from the web server's private IP:

```sql
sudo mysql

CREATE DATABASE wordpress;
CREATE USER 'myuser'@'<Web-Server-Private-IP-Address>' IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit
```

![MySQL DB setup](./images/task%206%20sql2.jpg)

![MySQL DB confirm](./images/task%206%20sql3.jpg)

---

## Step 6: Connect WordPress to the Database

### Open MySQL Port on the DB Server

I added an inbound rule on the DB server's security group to allow TCP port 3306 — but only from the web server's IP address (`/32`) for security.

![Security Group Rules](./images/task%206%20roles.jpg)

### Update the MySQL Config

On the DB server, I edited the MySQL config file to allow the web server to connect:

```bash
sudo nano /etc/my.cnf
sudo systemctl restart mysqld
```

![MySQL config](./images/task%206%20nano%20mysql.jpg)

### Update the WordPress Config

On the web server, I edited the WordPress config file with the database details:

```bash
cd /var/www/html/wordpress
sudo nano wp-config.php
```

![wp-config.php](./images/task%206%20nanowp.jpg)

### Remove the Default Apache Page

I moved the default Apache welcome page out of the way so WordPress could be seen instead:

```bash
sudo mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf_backup
sudo systemctl restart apache2
```

![Remove default page](./images/task%206%20rm%20def%20page.jpg)

### Test the Database Connection

I tested the remote connection from the web server to the database server:

```bash
sudo mysql -u admin -p -h <DB-Server-Private-IP-address>
```

![DB connection test](./images/task%206%20testdb.jpg)

### Access WordPress in the Browser

I opened a browser and went to:

```
http://<Web-Server-Public-IP-Address>/wordpress/
```

![WordPress setup page](./images/task%206%20wp%20page.jpg)

I filled in my credentials to finish setting up the site. Seeing this screen confirmed that WordPress had successfully connected to the remote MySQL database.

![WordPress login](./images/task%206%20wplogin.jpg)

![WordPress dashboard](./images/task%206%20wp%20dashboard.jpg)

---

## What I Learned

This project taught me a lot more than just "how to install WordPress." I learned how to:

- Set up and partition raw disk storage on a Linux server from scratch
- Use LVM to manage storage flexibly grouping physical disks into logical volumes that are easier to resize and manage
- Separate a web application across multiple servers the way real production systems are built
- Secure database access by restricting connections to specific IP addresses
- Configure Apache and PHP to serve a WordPress site on Linux
