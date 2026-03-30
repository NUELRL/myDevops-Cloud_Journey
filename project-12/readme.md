# Ansible Refactoring, Static Assignments, Imports and Roles

![Ansible](https://img.shields.io/badge/Ansible-000?style=for-the-badge&logo=ansible&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![AWS](https://img.shields.io/badge/Amazon%20AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![YAML](https://img.shields.io/badge/YAML-CB171E?style=for-the-badge&logo=yaml&logoColor=white)


---

## Overview

In this project, I cleaned up and reorganised the Ansible code from the previous project. Instead of having everything jammed into one playbook file, I split things into smaller, focused files and introduced **roles** — a proper way to package up all the tasks, templates, and config for a specific type of server. 


I also improved the Jenkins pipeline so artifacts get stored more cleanly, and used imports to wire everything together through a single entry-point playbook.

---

## Architecture

![](./images/task%2012%20arch.jpg)


## Step 1: Improve the Jenkins Pipeline


### Create the Artifact Directory

On the Jenkins-Ansible server I created a dedicated folder to store the latest artifacts:

```bash
sudo mkdir /home/ubuntu/ansible-config-artifact
```

Then I needed to give Jenkins permission to write files into that directory. This is where I hit a wall.

### The chmod Permissions Issue

Jenkins kept failing when it tried to copy artifacts into `/home/ubuntu/ansible-config-artifact`. The build logs showed permission denied errors — Jenkins could not write to the folder even though I had created it.

![](./images/task%2012%20error.jpg)

I tried the standard fix first:

```bash
chmod -R 0777 /home/ubuntu/ansible-config-artifact
```

![](./images/task%2012%201.jpg)

That did not fully sort it. The Jenkins process still could not access the directory because of how the parent directory permissions were set up. I ended up also running:

```bash
chmod o+x /home/ubuntu/
```

That finally let Jenkins through and the artifact copying started working.

I want to be honest about this: `chmod o+x` on a home directory and `0777` on any folder are the kind of permissions you would **never** touch in production. Opening everything up that wide on a real server means any process running on the machine — or any user — can read, write, and execute files in that location. In a production environment the correct fix is to add the Jenkins user to the `ubuntu` group and set group permissions on the folder (`chmod -R 775` and `chown -R ubuntu:jenkins`), keeping access locked down to only the processes that actually need it. For this learning environment, the broad permissions got things moving — but I filed it away as a shortcut I would not repeat on a real system.

![](./images/task%2012%20dix.jpg)

### Install the Copy Artifact Plugin

In Jenkins I installed the **Copy Artifact** plugin:
**Manage Jenkins** → **Manage Plugins** → **Available** → search `Copy Artifact` → install without restart.

![](./images/task%2012%20copy%20arit.jpg)

### Create the save_artifacts Job

I created a new Freestyle project in Jenkins called `save_artifacts` and configured it to:

- Trigger automatically when the `ansible-config-mgt` job completes
- Copy all artifacts from the `ansible` project into `/home/ubuntu/ansible-config-artifact`

I also configured both jobs to only keep the last 5 builds to stop disk space from filling up.

![](./images/task%2012%20jen%201.jpg)
![](./images/task%2012%20jen%202.jpg)
![](./images/task%2012%20jen%203.jpg)

Now every time I push code to GitHub, the flow is:

1. `ansible` job pulls the latest code and archives it
2. `save_artifacts` job copies everything into the stable artifact folder

From then on I always run playbooks from `/home/ubuntu/ansible-config-artifact` — one place, always up to date.

![](./images/task%2011%20arch%202.jpg)

---

## Step 2: Refactor the Playbooks Using Imports

### Why One Big Playbook Gets Messy

In the previous project everything lived in `common.yml`. That is fine to start, but the moment you need different tasks for different server types or environments, a single file becomes hard to manage. The fix is to break things up and use `import_playbook` to pull them all together.

### New Folder Structure

I created a new branch to work from:

```bash
git checkout -b refactor
```

Then I reorganised the repo into this structure:
![](./images/task%2012%20crt%20switch%20branch.jpg)

```
ansible-config-mgt/
├── playbooks/
│   └── site.yml          ← entry point for everything
├── static-assignments/
│   ├── common.yml        ← moved here from playbooks/
│   └── common-del.yml    ← new: removes wireshark
├── inventory/
│   ├── dev.yml
│   ├── staging.yml
│   ├── uat.yml
│   └── prod.yml
└── roles/
    └── webserver/        ← new role for UAT web servers
```

![](./images/task%2012%20tree.jpg)

### site.yml — The Master Entry Point

I created `playbooks/site.yml` as the single file that imports everything else:

```yaml
---
- hosts: all
- import_playbook: ../static-assignments/common.yml

```


The idea is simple: instead of running five different playbooks separately, I run `site.yml` and it handles everything in the right order. Adding a new server type or environment means adding one more `import_playbook` line — nothing else needs to change.

### Test with common-del.yml

To test that imports were working properly, I created `static-assignments/common-del.yml` to remove Wireshark from all servers (the opposite of what `common.yml` installed):

```yaml
---
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
    - name: delete wireshark
      yum:
        name: wireshark
        state: removed

- name: update LB server
  hosts: lb
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
    - name: delete wireshark
      apt:
        name: wireshark-qt
        state: absent
        autoremove: yes
        purge: yes
        autoclean: yes
```
![](./images/task%2012%20wire%20shark.jpg)

I updated `site.yml` to import `common-del.yml` instead of `common.yml` and ran it:

![](./images/task%2012%20play%20update.jpg)

```bash
cd /home/ubuntu/ansible-config-mgt/
ansible-playbook -i inventory/dev.yml playbooks/site.yml
```
![](./images/task%2012%20deleted%20wire%20shark.jpg)

Wireshark was gone from every server. I confirmed with:

```bash
ansible all -i inventory/dev.yml -m shell -a "wireshark --version || echo not installed"
```
![](./images/task%2012%20wire%20shark%20not%20installed.jpg)

---

## Step 3: Create the Webserver Role for UAT Servers

### Launch UAT Web Servers

I launched two fresh RHEL EC2 instances — `Web1-UAT` and `Web2-UAT` — to use as a staging environment for testing the role before it ever touches production.

### Create the Role Structure

Inside the repo I created the role using `ansible-galaxy`:

```bash
mkdir roles
cd roles
ansible-galaxy init webserver
```
![](./images/task%2012%20ansible%20crt%20roles.jpg)

After removing the folders I did not need (`tests`, `vars`, `files`), the structure looked like this:

```
roles/
└── webserver/
    ├── README.md
    ├── defaults/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── meta/
    │   └── main.yml
    ├── tasks/
    │   └── main.yml
    └── templates/
```
![](./images/task%2012%20rm%20test.jpg)

Each folder has a specific purpose:
- **tasks/** — the actual steps Ansible runs
- **handlers/** — tasks that only run when triggered (e.g. restart Apache only if config changed)
- **defaults/** — default variable values for the role
- **templates/** — Jinja2 template files the role can deploy
- **meta/** — role metadata like author and dependencies

### Write the Role Tasks

Inside `roles/webserver/tasks/main.yml` I wrote the tasks to configure each UAT web server from scratch:

```yaml
---
- name: install apache
  become: true
  ansible.builtin.yum:
    name: "httpd"
    state: present

- name: install git
  become: true
  ansible.builtin.yum:
    name: "git"
    state: present

- name: clone a repo
  become: true
  ansible.builtin.git:
    repo: https://github.com/NUELRL/tooling.git
    dest: /var/www/html
    force: yes

- name: copy html content to one level up
  become: true
  command: cp -r /var/www/html/html/ /var/www/

- name: Start service httpd, if not started
  become: true
  ansible.builtin.service:
    name: httpd
    state: started

- name: recursively remove /var/www/html/html/ directory
  become: true
  ansible.builtin.file:
    path: /var/www/html/html
    state: absent
```

![](./images/task%2012%20ansible%20apache.jpg)

This role installs Apache, installs git, clones the Tooling website from GitHub directly into `/var/www/html`, cleans up the nested folder structure the repo creates, and makes sure the web server is running.

### Update the UAT Inventory

I filled in `inventory/uat.yml` with the private IPs of the two new servers:

```ini
[uat-webservers]
<Web1-UAT-Private-IP> ansible_ssh_user='ec2-user'
<Web2-UAT-Private-IP> ansible_ssh_user='ec2-user'
```
![](./images/task%2012%20uat%20webserver.jpg)

### Tell Ansible Where to Find Roles

I updated `/etc/ansible/ansible.cfg` to point at my roles directory:

```ini
roles_path = /home/ubuntu/ansible-config-mgt/roles
```

![](./images/task%2012%20roles%20config.jpg)

Without this, Ansible would not know where to look when a playbook references the `webserver` role by name.



### Create the UAT Assignment

Inside `static-assignments/` I created `uat-webservers.yml`:

```yaml
---
- hosts: uat-webservers
  roles:
    - webserver
```

![](./images/task%2012%20uat%20webserver%20config.jpg)

And updated `site.yml` to include it:

```yaml
---
- hosts: uat-webservers
- import_playbook: ../static-assignments/uat-webservers.yml
```
![](./images/task%2012%20chnage%20site,yml.jpg)
---

## Step 4: Commit, Merge and Run

I committed everything, pushed to the `refactor` branch, opened a Pull Request, reviewed it, and merged into `main`. Jenkins picked it up automatically.

Then I ran the full playbook against the UAT inventory:

```bash
sudo ansible-playbook -i inventory/uat.yml playbooks/site.yml
```
![](./images/task%2012%20finalle.jpg)

Both UAT web servers were configured automatically Apache installed, repo cloned, service started. I confirmed by opening a browser:

```
http://<Web1-UAT-Public-IP>/index.php
http://<Web2-UAT-Public-IP>/index.php
```

The Tooling Website loaded on both.

![](./images/task%2012%20ultmate.jpg)

---

## What I Learned

This project changed how I think about structuring Ansible code. A few things that really stuck:

- **The chmod story is a real lesson.** Permissions issues are one of those things that can waste an hour if you do not know what to look for. `chmod -R 0777` and `chmod o+x` on a home directory got things working, but ... The right fix  adding Jenkins to the right group and using `775` keeps the principle of least privilege intact. I now know what the shortcuts are and exactly why I should not use them outside of a learning environment.

- **Roles are the right way to think about servers.** A role is not just a list of tasks — it is a complete description of what a server type should look like. Once the `webserver` role exists, I can apply it to any number of servers just by listing them under `[uat-webservers]` in the inventory. The role does not care how many servers there are.

- **`site.yml` as the entry point is a game changer.** Having one file that imports everything means there is always one place to look and one command to run. As the infrastructure grows, I just add more imports. The overall structure stays clean regardless of how many playbooks are underneath.

- **`import_playbook` vs including files directly** — importing is static, meaning Ansible resolves everything before the run starts. This makes it easier to predict what will happen and catch errors early. There is a dynamic version (`include_playbook`) that resolves things at runtime, but static imports are the right default until you have a specific reason to go dynamic.

- **The UAT environment pattern makes total sense now.** Test on `uat`, then promote to `prod`. Running `ansible-playbook -i inventory/uat.yml site.yml` versus `ansible-playbook -i inventory/prod.yml site.yml` is the cleanest possible way to manage that difference. Same code, same role, different target. Nothing gets rebuilt or rewritten between environments.