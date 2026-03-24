# Nginx Load Balancer with SSL/TLS — Project 10

![Nginx](https://img.shields.io/badge/Nginx-Load%20Balancer-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-20.04%20LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)

---

## Overview

In this project, I configured **Nginx as a Load Balancer** and secured the connection using **SSL/TLS certificates** from Let's Encrypt. This builds on my previous [project 9](../project-9) Apache Load Balancer setup, but with Nginx because a solid DevOps engineer knows multiple tools for the same job, `Plus nginx is faster`.

Beyond load balancing,When data travels between a browser and a web server across the internet, it passes through many network devices. If that data isn't encrypted, it's vulnerable to a **Man-In-The-Middle (MITM) attack** where someone intercepts sensitive info like login credentials or bank details. SSL/TLS prevents this by creating an encrypted session between the browser and the web server, using digital certificates validated by a **Certificate Authority (CA)**.

---

## Architecture

![](./images/task%2010%20arch.jpg)

My Nginx Load Balancer sits in front of two web servers, distributing traffic evenly and terminating SSL so all communication is encrypted end-to-end.

---

### Part 1 — Configured Nginx as a Load Balancer

I spun up a fresh **EC2 instance (Ubuntu LTS)**, opened ports **80 (HTTP)** and **443 (HTTPS)**, then installed and configured Nginx.

![](./images/task%2010%20nginx%20-i.jpg)

I updated `/etc/hosts` so Nginx could resolve my web servers by name:

```bash
sudo vi /etc/hosts
# Added:
# <Web1-Private-IP>  Web1
# <Web2-Private-IP>  Web2
```

![](./images/task%2010%20firstr.jpg)

Then I configured the Nginx upstream block inside `/etc/nginx/nginx.conf`:

```nginx
upstream myproject {
    server Web1 weight=5;
    server Web2 weight=5;
}

server {
    listen 80;
    server_name www.yourdomain.com;
    location / {
        proxy_pass http://myproject;
    }
}
```

![](./images/task%2010%20nginx%20config.jpg)

Finally, I restarted Nginx and confirmed it was running:

```bash
sudo systemctl restart nginx
sudo systemctl status nginx
```

![](./images/task%2010%20test%20nginx.jpg)

---

### Part 2 - Added Elastic IP and DNS mapping

At one point, I ran into a frustrating issue — every time I restarted my EC2 instance, my public IP address changed. That meant my domain would suddenly stop working unless I updated the DNS records again.

I solved this by using an Elastic IP. I allocated a static IP address and attached it to my NGINX load balancer. From that moment on, my domain consistently pointed to the same IP, no matter how many times I restarted the instance.

![](./images/task%2010%20elastic%201.jpg)
![](./images/task%2010%20elastic%202.jpg)
![](./images/task%2010%20elastic%203.jpg)
![](./images/task%2010%20elastic%204.jpg)

To keep things simple, I pointed my domain’s A record directly to my EC2 Elastic IP instead of using Amazon Route 53. This provided a quick and reliable setup for my lab environment.

### Part 3 — SSL/TLS Certificate with Let's Encrypt

I registered a domain name and secured it with a free SSL certificate from Let's Encrypt using **Certbot**.

```bash
# Confirmed snapd was active
sudo systemctl status snapd

# Installed Certbot
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Issued the certificate
sudo certbot --nginx
```

![](./images/task%2010%20certi.jpg)
![](./images/task%2010%20certi%202.jpg)
![](./images/task%2010%20ssl%20complete.jpg)

Certbot automatically detected my domain from `nginx.conf`, issued the certificate, and updated my Nginx config to handle HTTPS. After this, my site was accessible via `https://` with a secure description in the browser.

![](./images/task%2010.png)

---

### Part 3 — Auto-Renewal with Cron

Let's Encrypt certificates expire every **90 days**. I set up a cron job to automatically renew the certificate twice daily:

```bash
crontab -e
# Added:
* */12 * * *   root /usr/bin/certbot renew > /dev/null 2>&1
```

![](./images/task%2010%20finalll.jpg)

---
### My thoughts

In prod, skipping Route 53 (or any equivalent managed DNS) comes with real trade-offs:

The biggest real-world risk of what I did: **if my EC2 instance goes down or its Elastic IP changes (e.g. due to a rebuild), I have to manually update the DNS record at my registrar**, wait for propagation, and hope traffic resumes. With Route 53 and health checks configured, that failover can happen automatically in seconds.

**Bottom line:** Direct IPv4-to-A-record works perfectly fine for learning and lab environments. In prod, Route 53 gives you resilience, speed, and automation that a manual DNS setup simply can't match.

---

## Key Learnings

- Nginx's upstream block makes load balancing simple and readable
- SSL/TLS isn't optional in modern web infrastructure — it's baseline
- Let's Encrypt + Certbot is a powerful, free alternative to paid certificates
- Route 53 adds DNS resilience that's invisible until you need it and then you _really_ need it
- Elastic IPs are essential when associating a domain to an EC2 instance

---
