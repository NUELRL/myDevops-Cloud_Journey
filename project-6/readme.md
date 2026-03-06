# WEB SOLUTION WITH WORDPRESS

This repository explains the steps involved in preparing storage infrastructure on two Linux servers and implementing a basic web solution using WordPress.

You will gain practical experience that demonstrates three-tier architecture while also making sure that the Linux servers' storage disks are properly partitioned and maintained using tools like gdisk and LVM, respectively.

## The 3-Tier Setup
1. A Laptop or PC to serve as a client
2. An EC2 Linux Server as a web server (This is where you will install WordPress)
3. An EC2 Linux server as a database (DB) server

![alt](https://github.com/IwunzeGE/DevOps-Project/blob/10d8b69bedf06f1feda38efff29df59f9ba92bf4/THREE-TIER%20ARCHITECTURE/images/three-tier.png)

Generally, web, or mobile solutions are implemented based on what is called the Three-tier Architecture.
Three-tier Architecture is a client-server software architecture pattern that comprise of 3 separate layers.

1.	Presentation Layer (PL): This is the user interface such as the client server or browser on your laptop.
2.	Business Layer (BL): This is the backend program that implements business logic. Application or Webserver
3.	Data Access or Management Layer (DAL): This is the layer for computer data storage and data access. Database Server or File System Server such as FTP server, or NFS Server

# STEP 1 - PREPARE A WEB-SERVER
1.	Launch an EC2 instance that will serve as "Web Server". Create 3 volumes in the same AZ as your Web Server EC2, each of 10 GiB.
![alt](./images/three-tier.png)

![A](./images/task%206%20ec2.jpg)

![al](./images/task%206%20creat%20volume.jpg)

2.	Attach all three volumes one by one to your Web Server EC2 instance

![alt](./images/task%206%20attach%20volumes.jpg)

![alt](./images/task%206%20volumes.jpg)

3. Open up the Linux terminal to begin configuration Use `lsblk` command to inspect what block devices are attached to the server. Notice names of your newly created devices. All devices in Linux reside in /dev/ directory. Inspect it with `ls /dev/` and make sure you see all 3 newly created block devices there – their names will likely be xvdf, xvdh, xvdg.

![al](./images/task%206%20lsblk.jpg)

![a](./images/task%206%20ls%20dev.jpg)

4.	Use df -h command to see all mounts and free space on your server

![al](./images/task%206%20df.jpg)

5.	Use fdisk utility to create a single partition on each of the 3 disks
`sudo fdisk /dev/xvdf`

![al](./images/task%206%20fdisk.jpg)

6.	Use `lsblk` utility to view the newly configured partition on each of the 3 disks.

![al](./images/task%206%20lsblk.jpg)

7. Install lvm2 package using `sudo yum install lvm2`. Run sudo lvmdiskscan command to check for available partitions.

*Note: yum is the package installer for RedHat/CentOS*

8. Use `pvcreate` utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM

```sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```
![a](./images/task%206%20pvcreate.jpg)

9.	Verify that your Physical volume has been created successfully by running `sudo pvs`

![a](./images/task%206%20pvcreate.jpg)