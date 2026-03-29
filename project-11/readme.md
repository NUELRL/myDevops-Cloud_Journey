# Ansible Configuration Management – Automating Server Setup

![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![YAML](https://img.shields.io/badge/YAML-CB171E?style=for-the-badge&logo=yaml&logoColor=white)


---

## Overview

It’s kinda crazy how you don’t even need complex Python scripts for automation Ansible just handles automation so cleanly. It’s definitely the smarter way to go.

In this project, I set up **Ansible** to automatically configure all the servers I had been setting up manually across the previous projects the web servers, NFS server, database server, and load balancer. Instead of SSH-ing into each one individually and running commands by hand, I wrote a **playbook** (a file that describes exactly what every server should look like) and Ansible handled everything at once.


### What is a Bastion Host and why i learnt about one?

In a properly secured infrastructure, the web servers and database servers should not be directly reachable from the internet. They live in a private network and only talk to other servers inside that network. But then how does a DevOps engineer get in to manage them?

The answer is a **Bastion Host** (also called a Jump Server) — one server that is publicly reachable, which you SSH into first, and then hop from there to the internal servers. It is the only door into the private network. This limits the number of entry points an attacker has to target.

![](./images/task%2011%20arch%201.jpg)

In this project, my Jenkins-Ansible server acts as that jump box. Ansible runs on it, and from there it reaches out to all the other servers over SSH. The web servers never need to be exposed to the internet directly.



## Architecture

![](./images/task%2011%20arch%202.jpg)

---

## Step 1: Set Up Jenkins-Ansible Server

I renamed my existing Jenkins EC2 instance to `Jenkins-Ansible` since it will now run both Jenkins jobs and Ansible playbooks.

I created a new GitHub repository called `ansible-config-mgt` to hold all my Ansible code.

Then I installed Ansible on the server:

```bash
sudo apt update
sudo apt install ansible -y
ansible --version
```
![](./images/task%2011%20ansible%20-v.jpg)

### Connect Jenkins to the New Repo

I set up a new Freestyle project in Jenkins called `ansible`, pointed it at the `ansible-config-mgt` repository, and configured it exactly the same way as the previous Jenkins project [Project 10](../project-10):

- GitHub webhook to trigger builds on every push to `main`
- Post-build action to archive all files (`**`)

That way, every time I push Ansible code to GitHub, Jenkins saves a copy of the latest build artifacts automatically.

>  Every time the Jenkins-Ansible server restarts, it gets a new public IP which breaks the GitHub webhook. To avoid this, I assigned an **Elastic IP** to the server. Elastic IPs are free as long as they are attached to a running instance.

---

## Step 2: Set Up VS Code for Development

I connected **Visual Studio Code** and connected it to my [ansible-config-mgt repository](https://github.com/NUELRL/-ansible-config-mgt) . Then I cloned the repo directly onto the Jenkins-Ansible server:

![](./images/task%2011%20git%20clone%201.jpg)
![](./images/task%2011%20git%20clone%202.jpg)

Having VS Code talk directly to the server meant I could write and edit playbooks locally And its a look easier than Vim or nano.

---

## Step 3: Set Up the Project Structure

I created a new feature branch to work from:

Inside the repo I created the following folder structure:

- **inventory/** — holds the list of servers for each environment
- **playbooks/** — holds the instructions for what to do on those servers

![](./images/task%2011%20invn%20crt.jpg)
![](./images/task%2011%20playbook%20crt.jpg)
## Step 4: Set Up the Ansible Inventory

The inventory file tells Ansible which servers exist and how to connect to them. I filled in `inventory/dev.yml` with the private IP addresses of all my servers:

```ini
[nfs]
<NFS-Server-Private-IP> ansible_ssh_user='ec2-user'

[webservers]
<Web-Server1-Private-IP> ansible_ssh_user='ec2-user'
<Web-Server2-Private-IP> ansible_ssh_user='ec2-user'

[db]
<Database-Private-IP> ansible_ssh_user='ec2-user'

[lb]
<Load-Balancer-Private-IP> ansible_ssh_user='ubuntu'
```

Notice that the Load Balancer uses `ubuntu` as the SSH user because it runs Ubuntu, while all the RHEL servers use `ec2-user`. Ansible needs to know this upfront.

![](./images/task%2011%20dev,yml.jpg)

### Step 4: Set Up SSH Agent

Ansible connects to servers over SSH, so the Jenkins-Ansible server needs access to my two privates key for all those servers. I used `ssh-agent` to handle this cleanly:

```bash
eval `ssh-agent -s`
ssh-add <path-to-private-key>
```
![](./images/task%2011%20ssh%20a.jpg)

```bash
# Confirm the key was added
ssh-add -l
```

![](./images/task%2011%20ssh-add%20-l.jpg)

Then I SSH'd into the Jenkins-Ansible server with agent forwarding enabled, so it could pass the key along when connecting to other servers:

```bash
ssh -A ubuntu@<jenkins-ansible-public-ip>
```
![](./images/task%2011%20ssh-add%20-l.jpg)


---

## Step 5: Write the Common Playbook

I wrote my first playbook in `playbooks/common.yml`. Its job is to install **Wireshark** (a network analysis tool) on every server in the infrastructure — both RHEL and Ubuntu servers — using the right package manager for each:

```yaml
---
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
    - name: ensure wireshark is at the latest version
      yum:
        name: wireshark
        state: latest

- name: update LB server
  hosts: lb
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
    - name: Update apt repo
      apt:
        update_cache: yes

    - name: ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest
```
![](./images/task%2011%20vscode1.jpg)

The playbook has two plays:

1. The first targets the web servers, NFS server, and database server (all RHEL) — it uses `yum` to install Wireshark
2. The second targets the load balancer (Ubuntu) — it uses `apt` instead

`become: yes` tells Ansible to run the tasks as root, which is needed for installing software. The playbook is clean, readable, and anyone on the team can look at it and immediately understand what it does.

---

Then I opened a Pull Request on GitHub to merge into `main`and merged. The moment the code landed on `main`, Jenkins picked it up via webhook and saved the build artifacts.

![](./images/tasks%2011%20git%20merge.jpg)

---

## Step 6: Run the Ansible Playbook

With the code on the server and the SSH agent running, I ran the playbook:

```bash
cd ansible-config-mgt
ansible-playbook -i inventory/dev.yml playbooks/common.yml
```
![](./images/task%2011%20anisble%20automaion%20suceess.jpg)

Ansible connected to each server in the inventory, ran the tasks, and reported back. I then verified Wireshark was installed on each server:

```bash
wireshark --version
```
![](./images/task%2011%20ansible%20-v.jpg)

It was there on all of them — configured the same way, in one run, without me touching a single server directly.


### Fix: Disable Host Key Checking

The first time Ansible tries to SSH into a server it has never connected to before, it stops and asks:

```
fingerprint .......
Are you sure you want to continue connecting (yes/no)?
```

The fix is to tell Ansible to skip that check entirely by creating an `ansible.cfg` file in the root of the project:


```bash
touch ansible.cfg
```

And adding this inside it:

```ini
[defaults]
host_key_checking = False
```
![](./images/task%2011%20finalle.jpg)

That one line tells Ansible: "trust new servers automatically, do not ask me every time." Once I added this, playbook runs stopped getting interrupted and everything ran cleanly from start to finish. In a prod environment i would manage known hosts more carefully, but for a project environment like this just need something that'll work.

Also, decided to :

```
Create a directory and a file inside it

Change timezone on my servers

Run some shell script
```
![](./images/task%2011%20yml%20for%20new.jpg)

---

## What I Learned

This was a big shift in how I think about managing servers. A few things that really stood out:

- **The `ansible.cfg` file is not optional when automating.** The host key checking prompt seems like a small thing until it blocks an automated run and you realize you are sitting there typing `yes` ten times. Setting `host_key_checking = False` in `[defaults]` is one of the first things I will do on any new Ansible project from now on.

- **The inventory file is what ties everything together.** Ansible does not know what servers exist until you tell it. Keeping separate inventory files for `dev`, `staging`, `uat`, and `prod` means I can run the exact same playbook against different environments just by swapping the inventory file — `ansible-playbook -i inventory/prod.yml playbooks/common.yml`. Same instructions, different targets.

- **`become: yes` is how Ansible gets root access.** Instead of SSHing in as root (which is generally a bad idea), Ansible SSHes in as a regular user and then escalates privileges using `sudo`. It is a cleaner and safer pattern.


- **One playbook, every server.** The moment that clicked — that I could describe the desired state of my entire infrastructure in one file and apply it with one command — was when I really understood why teams invest in configuration management tools. It is not just about saving time. It is about having a single written record of exactly what your infrastructure is supposed to look like.



   Decided to chip in this ''/
![](./images/task%2011%20joke.jpg)
