# Client-Server Architecture Using MySQL DBMS

![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![AWS](https://img.shields.io/badge/Amazon%20AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

---

## Overview

In this project, I set up a Client-Server architecture using two Linux servers in AWS, with MySQL as the database management system. One server acts as the **MySQL Server** (it holds and manages the data) and the other acts as the **MySQL Client** (it connects to the server and makes requests).

### Why does this matter in real life?

This kind of setup is everywhere in the real world. When you use a banking app, the app on your phone is the **client** — it sends requests like "show me my balance" or "transfer money." A **server** somewhere receives those requests, talks to a **database**, fetches the data, and sends it back to your phone. None of that data lives on your device; it lives on a secure, dedicated server.

The same pattern applies to e-commerce websites, hospital record systems, social media platforms, and pretty much any app that stores user data. By setting this up myself, I got to see firsthand how a web server and a database server communicate over a network — and why it matters to keep them separate and secure.

---

## What Is Client-Server Architecture?

Client-Server is a setup where two or more computers are connected over a network and talk to each other by sending and receiving requests. The machine sending the request is the **Client**, and the machine responding is the **Server**.

![Client-Server Illustration](./images/illustration%201.png)

When a database is added into the mix, the web server becomes a client itself — it connects to the database server to read and write data:

![Client-Server with DB](./images/illustration%202.png)

A quick way to see this in action is by running this command in a terminal:

```bash
curl -Iv www.bing.com
```

> If `curl` isn't installed, I can add it with: `sudo apt install curl`

My terminal becomes the client and `www.bing.com` is the server. The response shows that the request is being handled by a computer at IP `23.12.147.174` on port 80.

![Bing curl test](./images/task%205%20bing%20test.jpg)

---

## Step 1: Create Two EC2 Instances in AWS

I created two Linux-based virtual servers in AWS:

- `mysql-server` — this is where the database lives
- `mysql-client` — this is what connects to the database remotely

![EC2 Instances](./images/task%205%20ec2s.jpg)

---

## Step 2: Set Up MySQL Server

I SSH'd into the `mysql-server` instance and ran the following:

```bash
sudo apt update
sudo apt install mysql-server
```

Then I made sure the MySQL service was running:

```bash
sudo systemctl start mysql.service
sudo systemctl status mysql.service
```

![MySQL Server Running](./images/task%205%20mysql%20server%20i.jpg)

### Secure the Root Account

I logged into MySQL and set a password for the root user:

```bash
sudo mysql
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'PassWord.1';
```

![SQL Auth](./images/task%205%20sql%20auth.jpg)

Then I ran the secure installation script to lock things down:

```bash
sudo mysql_secure_installation
```

![SQL Secure Install](./images/task%205%20sql%20auth%202.jpg)

### Create a Database and User

I logged back in as root:

```bash
sudo mysql -p
```

Then I created a new database:

```sql
CREATE DATABASE example_database;
```

Next, I created a new user and gave them full access to that database:

```sql
CREATE USER 'example_user'@'%' IDENTIFIED WITH mysql_native_password BY 'PassWord';
GRANT ALL ON example_database.* TO 'example_user'@'%';
```

> The `'%'` means this user can connect from any IP address — which is what allows the client server to reach the database remotely.

![SQL Privileges](./images/task%205%20sql-p.jpg)

I exited the MySQL shell and tested that the new user worked:

```bash
mysql -u example_user -p
```

Once logged in, I confirmed the database was visible:

```sql
SHOW DATABASES;
```

![Show Databases](./images/task%205%20db.jpg)

### Allow Remote Connections

By default, MySQL only accepts connections from the same machine (`127.0.0.1`). I changed that so the client server could connect remotely:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

I changed the `bind-address` value from `127.0.0.1` to `0.0.0.0`.

![MySQL Config](./images/task%205%20sql%20nano.jpg)

Then I restarted MySQL to apply the change:

```bash
sudo systemctl restart mysql
sudo systemctl status mysql.service
```

![MySQL Restart](./images/task%205%20sql%20R.jpg)

### Open Port 3306 in AWS

MySQL uses TCP port 3306 by default. I added an inbound rule in the `mysql-server` security group to allow traffic on that port — but only from the **private IP of the mysql-client** instance, not from the whole internet. This keeps things more secure.

![Inbound Rules](./images/task%205%20ports.jpg)

---

## Step 3: Set Up MySQL Client

I SSH'd into the `mysql-client` instance and installed the MySQL client package. I only needed the client tool here — not a full MySQL server installation:

```bash
sudo apt update && sudo apt upgrade
sudo apt install mysql-client -y
```

![Install MySQL Client](https://github.com/IwunzeGE/DevOps-Project/blob/f5ebc19e0f6692cf2cf39858b8de061a25381f92/CLIENT-SERVER%20ARCHITECTURE/images/install%20mysql-client.png)

---

## Step 4: Connect Remotely to the MySQL Server

From the `mysql-client` instance, I connected to the `mysql-server` database using its private IP address:

```bash
sudo mysql -u example_user -h <mysql-server-private-ip> -p
```

Once connected, I ran:

```sql
SHOW DATABASES;
```

The database I created on the server showed up — meaning the two machines were successfully talking to each other over the network.

![Final Result](./images/task%205%20finalle.jpg)

---

## What I Learned

This project helped me understand how real-world applications split their responsibilities across multiple servers. Instead of putting everything on one machine, I kept the database on its own server and accessed it remotely from a separate client — just like production systems do. I also learned how to lock things down by restricting access to a specific IP and using proper user permissions in MySQL.