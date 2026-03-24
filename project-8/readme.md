# Load Balancer Solution with Apache

![Apache](https://img.shields.io/badge/Apache-D22128?style=for-the-badge&logo=apache&logoColor=white)
![AWS](https://img.shields.io/badge/Amazon%20AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Red Hat](https://img.shields.io/badge/Red%20Hat-EE0000?style=for-the-badge&logo=redhat&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

---

## Overview

In [Project 7](../project-7) I had three web servers all serving the same content via NFS. The problem was each one had its own IP address, meaning a user would need three different URLs to access them. That makes no sense in the real world. This project fixes that by putting a single load balancer in front of all of them.

In this project, I added an **Apache Load Balancer** on top of the Tooling Website I built in the previous project. Instead of users having to know and pick between three different server addresses, the load balancer sits in front of everything and handles traffic for them one address, multiple servers working behind the scenes.


### Why does this matter in real prod environment?.

This is the difference between **vertical scaling** (buying a bigger, more powerful server) and **horizontal scaling** (adding more servers and spreading the load). Vertical scaling has a ceiling at some point you simply cannot add more RAM or CPU to a single machine. Horizontal scaling has almost no ceiling. That is how companies like Google serve billions of requests a day not with one massive computer, but with thousands of ordinary ones behind a load balancer.








---

## Architecture

![](./images/task%208%20arch.jpg)

From the user's point of view, there is only one website at one address. Everything below the load balancer is invisible to them.


## Step 1: Launch the Load Balancer Server

I created a new Ubuntu EC2 instance and named it `Project-8-apache-lb`.

Then I opened **TCP port 80** in its security group inbound rules so it could receive web traffic from the internet.

---

## Step 2: Install Apache and Enable Load Balancing Modules

I SSH'd into the load balancer server and installed Apache:

```bash
sudo apt update
sudo apt install apache2 -y
sudo apt-get install libxml2-dev -y
```

![](./images/task%208%20installs.jpg)

Then I enabled the Apache modules needed for load balancing:

```bash
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic
```


Each module plays a role:
- `proxy` and `proxy_http` — allow Apache to forward requests to other servers
- `proxy_balancer` — adds the actual load balancing logic
- `lbmethod_bytraffic` — enables the traffic-based balancing method I used
- `headers` — lets Apache manage request/response headers during proxying

I restarted Apache and confirmed it was running:

```bash
sudo systemctl restart apache2
sudo systemctl status apache2
```

![](./images/task%208%20a2emod.jpg)

---

## Step 3: Configure the Load Balancer

I opened the default Apache virtual host config:

```bash
sudo vi /etc/apache2/sites-available/000-default.conf
```

Inside the `<VirtualHost *:80>` block, I added the load balancer configuration:

```apache
<Proxy "balancer://mycluster">
    BalancerMember http://<WebServer1-Private-IP>:80 loadfactor=5 timeout=1
    BalancerMember http://<WebServer2-Private-IP>:80 loadfactor=5 timeout=1
    ProxySet lbmethod=bytraffic
</Proxy>

ProxyPreserveHost On
ProxyPass / balancer://mycluster/
ProxyPassReverse / balancer://mycluster/
```

What this does:
- `balancer://mycluster` — defines a group of servers called "mycluster"
- `BalancerMember` — adds each web server to that group
- `loadfactor=5` — both servers get equal traffic since the value is the same
- `lbmethod=bytraffic` — traffic is routed based on current load
- `ProxyPass` and `ProxyPassReverse` — tells Apache to forward all incoming traffic to the cluster and correctly rewrite response headers on the way back

Then I restarted Apache to apply the changes:

```bash
sudo systemctl restart apache2
```

![](./images/task%208%20config.jpg)
---

## Step 4: Test the Load Balancer

I opened a browser and went to:

```
http://<Load-Balancer-Public-IP>/index.php
```

The Tooling Website loaded — served through the load balancer.


![](./images/task%208%20test%20load.jpg)

### Verify Traffic Is Reaching Both Servers

To confirm both web servers were actually receiving requests, I opened two SSH sessions (one per web server) and watched the Apache access log live:

```bash
sudo tail -f /var/log/httpd/access_log
```

Every time I refreshed the browser, new entries appeared in the logs on both servers — alternating between them. Since both had `loadfactor=5`, the traffic was split roughly 50/50.



> **Note:** when i mounted `/var/log/httpd` to the NFS server in the previous project,i unmounted now first before checking logs. Each web server needs its own separate log directory.

```bash
sudo umount /var/log/httpd
```

---

## Step 5: Configure Local DNS Names (Optional but Useful)

Typing out IP addresses in config files gets old fast, especially when you have many servers to manage. I set up local name resolution on the load balancer so I could refer to each web server by a simple name instead.

I edited the `/etc/hosts` file on the load balancer:

```bash
sudo vi /etc/hosts
```

And added these two lines:

```
<WebServer1-Private-IP>  Web1
<WebServer2-Private-IP>  Web2
```


![](./images/task%208%20more%20config.jpg)


Then I updated the load balancer config to use the names instead of IPs:

```apache
BalancerMember http://Web1:80 loadfactor=5 timeout=1
BalancerMember http://Web2:80 loadfactor=5 timeout=1
```

![](./images/task%208%20update.jpg)


I tested it by curling each server directly from the load balancer:

```bash
curl http://Web1
curl http://Web2
```

Both returned the Tooling Website HTML — confirming the name resolution was working.


![](./images/task%202%20using%20curl%20to%20test%20port%2080.jpg)


> **Important:** These names only work on the load balancer itself. They are not registered anywhere on the internet or on any other server in the network. If you SSH into a web server and try to `curl http://Web1`, it will not resolve. It is purely a local shortcut for managing the LB config.

---

## What I Learnt

This project connected a lot of dots for me. Here is what stood out:

When a request comes in to the load balancer, it uses a **balancing method** to decide which web server gets it. Apache supports several:

- **bytraffic** — sends traffic based on how much data each server is currently handling. Busier servers get fewer new requests.
- **byrequests** — takes turns sending each new request to the next server in line (round robin).
- **bybusyness** — routes to whichever server has the fewest active connections at that moment.

I used `bytraffic` in this project. The idea is that if one server is already dealing with a large file upload, the load balancer will not pile more work on top of it it will send the next request somewhere with more breathing room.

You can also control the **loadfactor** for each server. A server with `loadfactor=10` will receive twice as much traffic as one with `loadfactor=5`. This is useful if your servers have different specs — you can send more traffic to the stronger ones.

- **Load balancing is not magic — it is just smart forwarding.** Apache receives a request, looks at its list of backend servers, picks one based on the balancing method, and forwards the request there. The response comes back through Apache to the user. The user has no idea a second (or third) server was involved.

- **Horizontal scaling makes much more sense than vertical scaling** once you think about it. Adding a second server and putting a load balancer in front is cheaper, more flexible, and more reliable than endlessly upgrading one machine. If one server goes down, the load balancer just stops sending traffic to it. The site stays up.

- **The loadfactor setting is more useful than it sounds.** In this project both servers had equal factors, but the moment you have servers with different specs — say one has 4 CPUs and another has 8 — you would want the stronger one to handle more traffic. Adjusting `loadfactor` is how you do that without changing anything else.

- **Watching the access logs in real time made it real.** Seeing requests land on web server 1, then web server 2, then back to 1 — watching it happen live in the terminal made the concept click in a way that reading about it never would have.

- **`/etc/hosts` is a handy trick.** It is not something you would use in a large production setup (you would use proper DNS for that), but for a small cluster it is a clean way to make your config files readable without setting up a full DNS server.