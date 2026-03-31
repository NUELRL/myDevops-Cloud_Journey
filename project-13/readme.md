# Ansible Dynamic Assignments, Include and Community Roles

![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)
![AWS](https://img.shields.io/badge/Amazon%20AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![YAML](https://img.shields.io/badge/YAML-CB171E?style=for-the-badge&logo=yaml&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)

---

## Overview

In this project, I took the Ansible setup from the previous project further by introducing **dynamic assignments** using the `include` module, pulling in **community roles** from Ansible Galaxy instead of writing everything from scratch, and making the load balancer switchable between Nginx and Apache using environment variables.

### Why does this matter in real life?

Up until now, every playbook I wrote was completely fixed — the same tasks ran the same way every time regardless of which environment was being targeted. But in real teams, different environments need different settings. Your dev server might use a self-signed SSL cert while production uses a real one. Your UAT environment might run Nginx while production still runs Apache. Your database password in dev is obviously not the same as in production.

Hardcoding those differences into the playbook itself is messy and risky — you end up with commented-out lines, duplicate files, or worse, production credentials sitting in a dev config. The cleaner solution is **environment-specific variable files** that Ansible loads automatically based on which environment it is targeting. The playbook stays the same. Only the variables change.

That is what dynamic assignments solve. And community roles solve a different but equally real problem: why spend two days writing and testing a MySQL installation role when someone already did it, battle-tested it across dozens of Linux flavours, and posted it publicly on Ansible Galaxy? The answer is you should not. You pull it in, configure it with your own variables, and move on.

### Static vs Dynamic — what is actually the difference?

This tripped me up at first so it is worth explaining clearly.

- **Static (`import`)** — Ansible reads and processes everything **before** the playbook runs. Think of it like printing out a full set of instructions before you start cooking. What is on the paper is what gets done. No surprises, easy to debug.

- **Dynamic (`include`)** — Ansible processes things **as it runs**, line by line. Think of it like following a recipe that says "if you have chicken, do step A, if you have fish, do step B" — the decision only gets made in the moment. More flexible, but harder to predict and debug when something goes wrong.

For most playbooks, **static imports are the safer and more reliable choice**. Dynamic includes are best kept for one specific job: loading environment-specific variables at runtime, which is exactly how I used them here.

---

## Project Structure

By the end of this project the repository looked like this:

```
ansible-config-mgt/
├── dynamic-assignments/
│   └── env-vars.yml
├── env-vars/
│   ├── dev.yml
│   ├── stage.yml
│   ├── uat.yml
│   └── prod.yml
├── inventory/
│   ├── dev.yml
│   ├── stage.yml
│   ├── uat.yml
│   └── prod.yml
├── playbooks/
│   └── site.yml
├── roles/
│   ├── webserver/
│   ├── mysql/
│   ├── nginx/
│   └── apache/
└── static-assignments/
    ├── common.yml
    ├── webservers.yml
    └── loadbalancers.yml
```

---

## Step 1: Set Up Dynamic Assignments

I created a new branch to work from:

```bash
git checkout -b dynamic-assignments
```

Then I created a `dynamic-assignments/` folder with a file called `env-vars.yml` inside it, and a separate `env-vars/` folder holding one variable file per environment:

```
env-vars/
├── dev.yml
├── stage.yml
├── uat.yml
└── prod.yml
```

The idea is simple: each environment gets its own YAML file where I can set variables specific to that environment — server names, credentials, feature flags, whatever needs to be different.

### env-vars.yml — Load the Right Variable File Automatically

Inside `dynamic-assignments/env-vars.yml` I wrote:

```yaml
---
- name: collate variables from env specific file, if it exists
  hosts: all
  tasks:
    - name: looping through list of available files
      include_vars: "{{ item }}"
      with_first_found:
        - files:
            - dev.yml
            - stage.yml
            - prod.yml
            - uat.yml
          paths:
            - "{{ playbook_dir }}/../env-vars"
      tags:
        - always
```
![](./images/task%2013%20env-vars,yml.jpg)

What this does:
- `include_vars` loads a YAML file as variables into the current playbook run
- `with_first_found` goes through the list of filenames and loads the **first one it finds** in the `env-vars/` folder
- `{{ playbook_dir }}` is a special Ansible variable that resolves to wherever `site.yml` is sitting — so the path always works regardless of where I run the command from
- `tags: always` means this task runs every single time, no matter what other tags are specified — ensuring variables are always loaded before anything else runs

---

## Step 2: Update site.yml — and the Error I Hit

Once the dynamic assignment folder was ready, I updated `site.yml` to include it. The original documentation showed mixing `import_playbook` and `include` on the same level inside `site.yml`, which looked like this:

```yaml
---
- hosts: all
- name: Include dynamic variables
  tasks:
  import_playbook: ../static-assignments/common.yml
  include: ../dynamic-assignments/env-vars.yml
  tags:
    - always

- hosts: webservers
- name: Webserver assignment
  import_playbook: ../static-assignments/webservers.yml
```
![](./images/task%2013%20site,yml%20change%201.jpg)

When I ran the playbook I got this error:

![](./images/task%2013%20error.jpg)

The problem was two things happening at once. First, mixing `import_playbook` (static) and `include` (dynamic) in the same play block the way the docs showed caused Ansible to get confused about the file paths. Second, `webservers.yml` was being referenced before Ansible could properly resolve where it was.

The fix was to stop mixing the two styles in the same block and instead write `site.yml` as clean, separate `import_playbook` lines — one per file, in the order they should run:

```yaml
---
- import_playbook: ../static-assignments/common.yml
- import_playbook: ../dynamic-assignments/env-vars.yml
- import_playbook: ../static-assignments/uat-webservers.yml
- import_playbook: ../static-assignments/loadbalancers.yml
```
![](./images/task%2013%20fix.jpg)

Clean, readable, and Ansible can resolve every path without any confusion. The error disappeared immediately after making this change. The lesson here is that when Ansible docs show a mixed pattern and it does not work — simplify. Separate imports are easier to read and easier to debug anyway.

---

## Step 3: The Inventory Warning That Broke Everything

After fixing `site.yml`, I ran the playbook and got a wall of warnings:


```
[WARNING]: Invalid characters were found in group names but not replaced,
use -vvvv to see details
[WARNING]: Could not match supplied host pattern, ignoring: webservers
[WARNING]: Could not match supplied host pattern, ignoring: nfs
[WARNING]: Could not match supplied host pattern, ignoring: db
```
![](./images/task%2013%20error%203.jpg)


Ansible was saying it could not find the server groups `webservers`, `nfs`, and `db` — even though I had defined them. The playbook ran but skipped every task for those groups, which meant nothing actually got configured.

The issue was in how the inventory file was written. My `uat.yml` inventory only had the UAT servers listed and was missing all the other groups. Ansible was looking for `[webservers]`, `[nfs]`, and `[db]` groups in the inventory I passed with `-i` and simply could not find them.

The fix was to make sure the inventory file I was running against had **all** the groups defined — not just the ones I was actively targeting in that run:

```ini
[uat-webservers]
172.**.**.*** ansible_ssh_user='ec2-user'
172.**.**.*** ansible_ssh_user='ec2-user'

[nfs]
172.**.*.*** ansible_ssh_user='ec2-user'

[webservers]
172.**.**.*** ansible_ssh_user='ec2-user'
172.**.**.*** ansible_ssh_user='ec2-user'

[db]
172.**.**.*** ansible_ssh_user='ec2-user'

[lb]
172.***.**.*** ansible_ssh_user='ubuntu'
```

![](./images/task%2013%20fix%203.jpg)


Once I updated the inventory to include every group that `site.yml` references — even the ones not being actively changed in this run — the warnings disappeared and Ansible ran cleanly against the right servers.

The takeaway: the inventory file you pass with `-i` needs to know about **every group your playbooks mention**, not just the ones you are targeting today. If `site.yml` references `[webservers]` and your inventory does not have that group defined, Ansible will not throw a hard error — it will just silently skip those tasks and warn you. That silent skip is dangerous because the run appears to succeed but nothing actually happened on those servers.

---

## Step 4: Pull in the MySQL Community Role

Instead of writing a MySQL installation role from scratch, I used one already built and maintained by the community — `geerlingguy.mysql` from Ansible Galaxy. It handles installation, database creation, and user configuration across multiple Linux distributions.

On the Jenkins-Ansible server I made sure git was set up properly:

```bash
git init
git pull https://github.com/<your-name>/ansible-config-mgt.git
git remote add origin https://github.com/<your-name>/ansible-config-mgt.git
git branch roles-feature
git switch roles-feature
```

Then I downloaded the role and renamed it to something cleaner:

```bash
cd roles/
ansible-galaxy install geerlingguy.mysql
mv geerlingguy.mysql/ mysql
```

![](./images/task%2013%20install%20sql.jpg)


I read through the role's `README.md` to understand its variables, then updated `defaults/main.yml` with the correct database name, user, and password for the Tooling website.

Once configured, I committed and pushed:

```bash
git add .
git commit -m "add mysql community role"
git push --set-upstream origin roles-feature
```

Then opened a Pull Request and merged into `main`.

---

## Step 5: Set Up Switchable Load Balancer Roles

The last piece was making the load balancer flexible — instead of hardcoding either Nginx or Apache, I set things up so I could switch between them using an environment variable.

I created two roles — `nginx` and `apache` — and in each role's `defaults/main.yml` I added:

```yaml
# nginx role defaults/main.yml
enable_nginx_lb: false
load_balancer_is_required: false
```
![](./images/task%2013%20rolesdefaultsnginx.jpg)


```yaml
# apache role defaults/main.yml
enable_apache_lb: false
load_balancer_is_required: false
```
![](./images/task%2013%20rolesapache.jpg)


Both are set to `false` by default, meaning neither load balancer runs unless I explicitly turn one on.

### loadbalancers.yml

Inside `static-assignments/loadbalancers.yml`:

```yaml
- hosts: lb
  roles:
    - { role: nginx, when: enable_nginx_lb and load_balancer_is_required }
    - { role: apache, when: enable_apache_lb and load_balancer_is_required }
```
![](./images/task%2013%20loadbalancer,yml.jpg)


The `when` condition means each role only runs if its flag is set to `true`. Both flags being false by default means neither runs unless I turn one on.

### Activate a Load Balancer per Environment

To activate Nginx for the UAT environment, I added this to `env-vars/uat.yml`:

```yaml
enable_nginx_lb: true
load_balancer_is_required: true
```

To switch to Apache instead, I just change it to:

```yaml
enable_nginx_lb: false
enable_apache_lb: true
load_balancer_is_required: true
```
![](./images/task%2013%20nginx.jpg)


No changes to the playbook. No changes to the role. Just one variable file updated and the behaviour of the whole infrastructure changes. That is the power of combining dynamic variables with conditional roles.

Then i ran 
```bash
ansible-playbook -i inventory/uat.yml playbooks/site.yml
```
![](./images/task%2013%20suc.jpg
)
![](./images/task%2013%20cess.jpg)



---

## What I Learned

This project connected a lot of moving parts. Here is what stood out most:

- **The `site.yml` path error taught me to keep imports simple.** When mixing `import_playbook` and `include` in the same block caused file resolution errors, the fix was to stop mixing styles entirely and use clean separate import lines. Simpler is always more debuggable in Ansible.

- **Inventory group warnings are not just warnings — they are silent failures.** Ansible skipping a group without failing the whole run is sneaky. The playbook says `SUCCESS` but nothing happened on those servers. Always make sure every group your playbooks reference exists in the inventory file you are running against.

- **Community roles save real time.** The `geerlingguy.mysql` role does things I would not have got right on the first try — handling different package names across distros, managing service restarts correctly, dealing with edge cases in MySQL user permissions. Using it meant I could focus on wiring things together rather than debugging package installation.

- **Environment variables as feature flags is a clean pattern.** Setting `enable_nginx_lb: false` and `enable_apache_lb: false` by default and only turning one on per environment is a pattern I will keep using. It makes the infrastructure configuration self-documenting — you can open any `env-vars/` file and immediately see what is turned on in that environment.

- **`with_first_found` is more useful than it looks.** The fact that it just skips silently if no file is found means I can have environments that do not need a variable file at all — the playbook keeps running without erroring out. That is the kind of graceful default that makes automation reliable.