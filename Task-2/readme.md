# LEMP STACK IMPLEMENTATION (Linux, Nginx, MySQL and PHP)

## Project Overview
In this project, I deployed a complete web application infrastructure using the LEMP stack, an alternative to the traditional LAMP stack I implemented in [Task 1](myDevops-Cloud_Journey/Task-1). This architecture replaces Apache with Nginx to achieve improved performance, scalability, and efficient resource utilization.

**LEMP Stack Components:**
- **L**inux (Ubuntu 22.04 LTS)
- **E**ngine-X (Nginx) - High-performance web server
- **M**ySQL - Relational database management system
- **P**HP - Server-side scripting language

---

## Step 0: Preparing Prerequisites

### AWS EC2 Instance Setup
I created a new EC2 instance with the following specifications:
  - **Instance Type:** t2.nano (Free tier eligible)
  - **Operating System:** Ubuntu Server 22.04 LTS (HVM)
  - **Security Group:** Configured to allow SSH (port 22) and HTTP (port 80)

I then connected to the instance via SSH.

---

## Step 1: Installing the Nginx Web Server

### Why Nginx?
Nginx is a high-performance web server known for:
- Efficient handling of concurrent connections
- Low memory footprint
- Excellent static content delivery
- Robust reverse proxy capabilities

### Installation Process

I updated the server's package index and installed Nginx:

```bash
sudo apt update
sudo apt install nginx
```

![Nginx Installation](./images/task%202%20installed%20nginx%20%201.jpg)

### Verification

I checked if Nginx is running as a service:

```bash
sudo systemctl status nginx
```

![Nginx Status Verification](./images/task%202%20nginx%20verification%202.jpg)

### Configuring Security Group

I opened TCP port 80 in the EC2 security group to allow HTTP traffic from the internet (0.0.0.0/0):

![Security Group Configuration](./images/task%202%20changing%20roles%203.jpg)

### Testing Nginx Locally

I tested the web server locally using curl:

```bash
curl http://localhost:80
```
or
```bash
curl http://127.0.0.1:80
```

![Local Curl Test](./images/task%202%20using%20curl%20to%20test%20port%2080.jpg)

**Note:** The difference between these commands:
- `localhost` uses DNS name resolution
- `127.0.0.1` uses the IP address directly

### Testing Nginx from the Internet

I accessed the server via web browser using the public IP address:

```
http://<Public-IP-Address>:80
```

![Web Browser Test](./images/task%202%20testing%20nginx%20on%20browser.jpg)

**Success!** The Nginx welcome page confirms the web server is correctly installed and accessible through the firewall.

---

## Step 2: Installing MySQL

### Purpose
MySQL serves as the Database Management System (DBMS) to store and manage data for the web application in a relational database format.

### Installation

I installed MySQL server using apt:

```bash
sudo apt install mysql-server
```

Confirmed installation by typing `Y` when prompted.

![Installing in sql](./images/task%202%20installing%20mysql.jpg)

### Initial Configuration

I logged into the MySQL console:

```bash
sudo mysql
```
![Logining in to sql](./images/task%202%20securin%20sql.jpg)

### Securing MySQL Installation

I set a password for the root user using mysql_native_password authentication:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'PassWord.1';
```

Exited the MySQL console:

```sql
exit
```

I then ran the security script to remove insecure defaults:

```bash
sudo mysql_secure_installation
```

Configuration steps:
1. **VALIDATE PASSWORD PLUGIN:** I enabled password validation
2. **Password Strength Level:** I selected level 1 (MEDIUM)
3. **Root Password:** I set a strong password
4. **Remove anonymous users:** Yes
5. **Disallow root login remotely:** Yes
6. **Remove test database:** Yes
7. **Reload privilege tables:** Yes

![](./images/task%202%20sql%20validation.jpg)
![](./images/task%202%20sql%20validation-2.jpg)
![](./images/task%202%20sql%20validation-3.jpg)

### Testing MySQL Access

I logged in using the new password:

```bash
sudo mysql -p
```

The `-p` flag prompts for the password that was set earlier.

Exited the console:

```sql
exit
```
![sql-validation-test](./images/task%202%20sql%20finallle.jpg)
---

## Step 3: Installing PHP

### PHP Components for LEMP

Unlike Apache which embeds PHP in each request, Nginx requires external PHP processing. I installed the following packages:

- **php-fpm:** PHP FastCGI Process Manager (handles PHP processing)
- **php-mysql:** PHP module for MySQL database communication

### Installation Command

```bash
sudo apt install php-fpm php-mysql
```

Confirmed installation by typing `Y` when prompted.

**Result:** PHP components are now installed and ready to be configured with Nginx.

![php-fpm php-mysql-i](./images/task%202%20php%20fpm%20i.jpg)
---

## Step 4: Configuring Nginx to Use PHP Processor

### Server Blocks Concept
Nginx uses server blocks (similar to Apache's virtual hosts) to host multiple domains on a single server.

### Creating Project Directory Structure

I created the root web directory for the domain:

```bash
sudo mkdir /var/www/projectLEMP
```

Assigned ownership to the current system user:

```bash
sudo chown -R $USER:$USER /var/www/projectLEMP
```

### Nginx Configuration File

I created a new configuration file:

```bash
sudo nano /etc/nginx/sites-available/projectLEMP
```

![nginx-cofigs](./images/task%202%20pfm%20preconfig.jpg)

I added the following configuration based on my php fpm version:

![php-fpm-v](./images/task%202%20php%20fpm-v.jpg)


```nginx
#/etc/nginx/sites-available/projectLEMP

server {
    listen 80;
    server_name projectLEMP www.projectLEMP;
    root /var/www/projectLEMP;

    index index.html index.htm index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

#### Configuration Directives Explained:

- **listen 80:** Nginx listens on port 80 (default HTTP port)
- **root:** Document root directory for website files
- **index:** Priority order for index files
- **server_name:** Domain names/IP addresses for this server block
- **location /:** Handles URI requests, returns 404 if file not found
- **location ~ \.php$:** Handles PHP processing via FastCGI
- **location ~ /\.ht:** Denies access to .htaccess files

I saved and closed the file (`CTRL+X`, then `Y`, then `ENTER`).

### Activating Configuration

I created symbolic link to enable the configuration:

```bash
sudo ln -s /etc/nginx/sites-available/projectLEMP /etc/nginx/sites-enabled/
```

I tested configuration for syntax errors:

```bash
sudo nginx -t
```

Expected output:
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

I disabled the default Nginx host:

```bash
sudo unlink /etc/nginx/sites-enabled/default
```

I reloaded Nginx to apply changes:

```bash
sudo systemctl reload nginx
```

### Testing the New Server Block

I created a test index.html file:

```bash
sudo echo 'Hello LEMP from hostname' $(curl -s http://169.254.169.254/latest/meta-data/public-hostname) 'with public IP' $(curl -s http://169.254.169.254/latest/meta-data/public-ipv4) > /var/www/projectLEMP/index.html
```

![php-fpm php-mysql-i](./images/task%202%20pfpm%20test.jpg)

![php-fpm php-mysql-i](./images/task%202%20php%20fpm-troublshoot.jpg)


I accessed the website via browser:

```
http://<Public-IP-Address>:80
```
![php-fpm php-mysql-i](./images/task%202%20pfp%20%20%20webtest%202.jpg)

**Success!** The page displayed the echo command output, confirming the Nginx server block works correctly.

---

## Step 5: Testing PHP with Nginx

### Creating PHP Test File

I created a test PHP file to validate PHP processing:

```bash
sudo nano /var/www/projectLEMP/info.php
```

I added the following PHP code:

```php
<?php
phpinfo();
```
![php-fpm php-mysql-i](./images/task%202%20pfpm-php-testConfig.jpg)

Saved and closed the file.

### Accessing PHP Info Page

I opened the page in a web browser:

```
http://<server_domain_or_IP>/info.php
```
![php-fpm php-mysql-i](./images/task%202%20pfm%20is-a-go.jpg)

**Result:** A detailed PHP information page appeared, displaying:
- PHP version
- Configuration settings
- Loaded modules
- Server information

### Security Cleanup

I removed the info.php file as it contains sensitive information:

```bash
sudo rm /var/www/projectLEMP/info.php
```

**LEMP Stack Status:** Fully configured and operational!

---

## Step 6: Retrieving Data from MySQL Database with PHP

### Project Goal
Create a test database with a "To-Do List" and configure Nginx to query and display data from the database.

### Creating Database and User

I connected to MySQL console:

```bash
sudo mysql
```

I created a new database:

```sql
CREATE DATABASE `example_database`;
```

I created a new user with mysql_native_password authentication:

```sql
CREATE USER 'example_user'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
```

I granted full privileges on the database:

```sql
GRANT ALL ON example_database.* TO 'example_user'@'%';
```

Exited MySQL:

```sql
exit
```

![php-fpm php-mysql-i](./images/task%202%20sql-db.jpg)


### Testing User Permissions

I logged in with the new user credentials:

```bash
mysql -u example_user -p
```

I verified database access:

```sql
SHOW DATABASES;
```

![php-fpm php-mysql-i](./images/task%202%20sql-db2.jpg)


### Creating Test Table

I created the todo_list table:

```sql
CREATE TABLE example_database.todo_list (
    item_id INT AUTO_INCREMENT,
    content VARCHAR(255),
    PRIMARY KEY(item_id)
);
```

I inserted sample data:

```sql
INSERT INTO example_database.todo_list (content) VALUES ("My first important item");
INSERT INTO example_database.todo_list (content) VALUES ("Wash plstes");
INSERT INTO example_database.todo_list (content) VALUES ("Finish MAT 102");
INSERT INTO example_database.todo_list (content) VALUES ("Learn DATA structures and ALGORITHIMS");
```

I verified the data:

```sql
SELECT * FROM example_database.todo_list;
```

![php-fpm php-mysql-i](./images/task%202%20sql-db3.jpg)


Exited MySQL:

```sql
exit
```

### Creating PHP Script

I created a PHP file to connect to MySQL and display the to-do list:

```bash
nano /var/www/projectLEMP/todo_list.php
```

I added the following PHP script:

```php
<?php
$user = "example_user";
$password = "121.S3l@h";
$database = "example_database";
$table = "todo_list";

try {
    $db = new PDO("mysql:host=localhost;dbname=$database", $user, $password);
    echo "<h2>TODO</h2><ol>";
    foreach($db->query("SELECT content FROM $table") as $row) {
        echo "<li>" . $row['content'] . "</li>";
    }
    echo "</ol>";
} catch (PDOException $e) {
    print "Error!: " . $e->getMessage() . "<br/>";
    die();
}
```
![php-fpm php-mysql-i](./images/task%202%20sql-php%20config.jpg)


Saved and closed the file.

### Testing PHP-MySQL Integration

I accessed the page via web browser:

```
http://<Public_domain_or_IP>/todo_list.php
```

![phpfpm sql success](./images/task%202%20finalleee.jpg)


**Success!** The page displayed the to-do list items from the MySQL database, confirming:
- PHP is properly configured
- PHP can connect to MySQL
- PHP can query and display database content

---

## Project Completion

### What Was Accomplished

**Nginx Web Server:** Installed, configured, and serving web content  
**MySQL Database:** Installed, secured, and managing data  
**PHP Processing:** Installed and integrated with Nginx via FastCGI  
**Server Blocks:** Configured for hosting multiple domains  
**Database Integration:** PHP successfully queries and displays MySQL data  

### LEMP Stack Architecture

```
┌─────────────────┐
│   Web Browser   │
└────────┬────────┘
         │ HTTP Request (Port 80)
         ▼
┌─────────────────┐
│  Nginx Server   │ ◄── Static content served directly
└────────┬────────┘
         │ PHP Request
         ▼
┌─────────────────┐
│   PHP-FPM       │ ◄── Processes PHP code
└────────┬────────┘
         │ Database Query
         ▼
┌─────────────────┐
│  MySQL Server   │ ◄── Stores and retrieves data
└─────────────────┘
```

### WHAT I LEARNT

1. **Nginx vs Apache:** Nginx uses an event-driven architecture for better performance with concurrent connections
2. **PHP-FPM:** Nginx requires external PHP processing, unlike Apache's embedded approach
3. **Server Blocks:** Nginx's method for hosting multiple sites on one server
4. **Database Security:** Importance of secure passwords and proper user permissions
5. **MySQL Authentication:** Using mysql_native_password for PHP compatibility


## Technologies Used

- **Cloud Platform:** AWS EC2
- **Operating System:** Ubuntu Server 22.04 LTS
- **Web Server:** Nginx
- **Database:** MySQL 8.0
- **Programming Language:** PHP 8.3
- **Package Manager:** APT
