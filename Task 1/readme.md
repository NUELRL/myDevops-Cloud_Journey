# LAMP STACK IMPLEMENTATION (Linux, Apache, MySQL and PHP)

## Step 0 - Preparing Prerequisite

- Create an AWS account.

![alt text](Images/1-create-aws-acct.jpg)

- Launch a new EC2 instance of t3.micro family with Ubuntu Server 24.04 LTS (HVM) on AWS.

![alt text](Images/2-setupEc2.jpg)

![alt text](Images/3-ec2Connect.jpg)

- Connect to the instance on your local machine.

![alt text](Images/4-ec2ToLocal.jpg)

## Step 1 - INSTALLING APACHE AND UPDATING THE FIREWALL

- Install Apache using Ubuntu’s package manager `apt`
- Update a list of packages in package manager
  `sudo apt update`
- Run apache2 package installation
  `sudo apt install apache2`
- To verify that apache2 is running as a Service in our OS, use following command
  `sudo systemctl status apache2`

![alt text](Images/5-aptUpadte.jpg)

![alt text](Images/6-apache-i.jpg)

- Open inbound connection through port 80

![alt text](Images/7-netConfig.jpg)

- To view the Apache default page, Go to http://Public-IP-Address:80 in your browser

![alt text](Images/8-apacheSite.jpg)

## Step 2 - INSTALLING MYSQL

- Again, use `apt` to acquire and install this software:  
  `sudo apt install mysql-server`

- When prompted, confirm installation by typing Y, and then ENTER.

![alt text](<Images/9-sql-i.jpg>)

- When the installation is finished, log in to the MySQL console by typing:  
  `sudo mysql`
- This will connect to the MySQL server as the administrative database user root, which is inferred by the use of sudo when running this command.
- Define the users password using this command:  
  `ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'PassWord'`
- Exit the MySQL shell with:  
  `mysql> exit`

![alt text](<Images/10-sqlSecure.jpg>)

- Start the interactive script by running:  
  `sudo mysql_secure_installation`
- This will ask if you want to configure the VALIDATE PASSWORD PLUGIN.
- Answer Y for yes, or anything else to continue without enabling
- When you’re finished, test if you’re able to log in to the MySQL console by typing:  
  `sudo mysql -p`
- To exit the MySQL console, type:
  `mysql> exit`

![alt text](<Images/11-exitSQL.jpg>)

## Step 3 — INSTALLING PHP

- To install these 3 packages at once, run:  
  `sudo apt install php libapache2-mod-php php-mysql`

![alt text](<Images/12-PHP-i.jpg>)

- Once the installation is finished, you can run the following command to confirm your PHP version:  
  `php -v`

![alt text](<Images/13-php-v.jpg>)

- At this point, your LAMP stack is completely installed and fully operational.

      +Linux (Ubuntu)
      +Apache
      +HTTP Server
      +MySQL
      +PHP

## Step 4 — CREATING A VIRTUAL HOST FOR YOUR WEBSITE USING APACHE

- Create the directory for 'newproject' using mkdir command as follows:  
  `sudo mkdir /var/www/projectlamp`

- Next, assign ownership of the directory with your current system user:  
  `sudo chown -R $USER:$USER /var/www/projectlamp`

- Create and open a new configuration file in Apache’s sites-available directory using:
  ` sudo vi /etc/apache2/sites-available/projectlamp.conf`

![alt text](<Images/14-Host-crt.jpg>)

- Paste in the following bare-bones configuration

  `<VirtualHost *:80>
ServerName projectlamp
ServerAlias www.projectlamp
ServerAdmin webmaster@localhost
DocumentRoot /var/www/projectlamp
ErrorLog ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>`

![alt text](<Images/15-HostConfig.jpg>)

- You can use the `ls` command to show the new file in the sites-available directory:  
  `sudo ls /etc/apache2/sites-available`

- You can now use `a2ensite` command to enable the new virtual host:  
  `sudo a2ensite projectlamp`
- To disable Apache’s default website use `a2dissite` command, type:
  `sudo a2dissite 000-default`
- To make sure your configuration file doesn’t contain syntax errors, run:  
  `sudo apache2ctl configtest`
- Finally, reload Apache so these changes take effect:  
  `sudo systemctl reload apache2`
- Your new website is now active, but the web root /var/www/projectlamp is still empty. Create an index.html file in that location so that we can test that the virtual host works as expected, run:
  `echo "Hello LAMP from hostname $(curl -s http://169.254.169.254/latest/meta-data/public-hostname) with public IP $(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)" | sudo tee /var/www/projectlamp/index.html`



![alt text](<Images/16-echo2server.jpg>)

- Now go to your browser and try to open your website URL using IP address:
  `http://<Public-IP-Address>:80`

If you see the text from ‘echo’ command you wrote to index.html file, then it means your Apache virtual host is working as expected.

![alt text](<Images/17-echoRes.jpg>)

## Step 5 — ENABLE PHP ON THE WEBSITE

- With the default DirectoryIndex settings on Apache, a file named index.html will always take precedence over an index.php file.

- In case you want to change this behavior, you’ll need to edit the /etc/apache2/mods-enabled/dir.conf file and change the order in which the index.php file is listed within the DirectoryIndex directive, run :

`sudo nano /etc/apache2/mods-enabled/dir.conf`

then edit it thus:

`<IfModule mod_dir.c>
        #Change this:
        #DirectoryIndex index.html index.cgi index.pl index.php index.xhtml index.htm
        #To this:
        DirectoryIndex index.php index.html index.cgi index.pl index.xhtml index.htm
</IfModule>`

- After saving and closing the file, you will need to reload Apache so the changes take effect:

`sudo systemctl reload apache2`


![alt text](<Images/18-apacheReload.jpg>)



![alt text](<Images/19hostInsert.jpg>)

- Finally, we will create a PHP script to test that PHP is correctly installed and configured on your server.

- Create a new file named index.php inside your custom web root folder:

`sudo nano /var/www/projectlamp/index.php`

- This will open a blank file. Add the following text, which is valid PHP code, inside the file:

`<?php
phpinfo();
`
![alt text](<Images/21php.jpg>)

- When you are finished, save and close the file, refresh the page and you will see a page similar to this:

![alt text](<Images/20php change.jpg>)
