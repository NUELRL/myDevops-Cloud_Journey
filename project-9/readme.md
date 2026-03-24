# Tooling Website Deployment Automation with Jenkins CI

![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)
![AWS](https://img.shields.io/badge/Amazon%20AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Java](https://img.shields.io/badge/Java-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)

---

## Overview

**Continuous Integration (CI)** 

In this project, I modified [Project 8](../project-8/) by connecting Jenkins to my GitHub repository so that every time I push new code, Jenkins automatically picks it up and deploys it to the NFS server  no manual copying, no SSH-ing into servers, no repeating myself. The Tooling Website updates on its own.



### Architecture

The flow I set up looks like this:


![](./images/task%209%20arch.jpg)


---


## Step 1: Install the Jenkins Server

I launched a new Ubuntu EC2 instance and named it `Jenkins`.

Since Jenkins runs on Java, I installed JDK first:

```bash
sudo apt update
sudo apt install default-jdk-headless -y
```

![](./images/task%209%20intall%20jdk.jpg)

Then I added the Jenkins repository and installed it:

```bash
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -

sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
    /etc/apt/sources.list.d/jenkins.list'

sudo apt update
sudo apt-get install jenkins -y
```

![](./images/task%209%20jen%20is.jpg)


I confirmed Jenkins was running:

```bash
sudo systemctl status jenkins
```
![](./images/task%209%20jen%20stats.jpg)


### Open Port 8080

Jenkins runs on **TCP port 8080** by default. I added an inbound rule in the EC2 security group to allow traffic on that port so I could reach the Jenkins dashboard from my browser.

![](./images/task%209%20port%208080.jpg)

### Initial Setup

I opened a browser and went to:

```
http://<Jenkins-Server-Public-IP>:8080
```
![](./images/task%209%20prompet.jpg\)


Jenkins asked for an initial admin password. I grabbed it from the server:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```


![](./images/task%209%20retrieve%20pass.jpg)

I pasted it in, chose **Install Suggested Plugins**, and created an admin user. Jenkins was ready.


![](./images/task%209%20jen%20intials.jpg)

---

## Step 2: Connect Jenkins to GitHub with Webhooks

### Set Up the GitHub Webhook

In my GitHub repository settings, I added a webhook pointing to my Jenkins server:

```
http://<Jenkins-Public-IP>:8080/github-webhook/
```

![](./images/task%209%20enable%20webhook.jpg)


This tells GitHub: "whenever someone pushes code to this repo, send a notification to Jenkins."


### Create a Freestyle Project in Jenkins

In the Jenkins dashboard I clicked **New Item**, named the project `pipeline`, and chose **Freestyle project**.

In the project configuration:

- Under **Source Code Management**, I selected **Git** and pasted in my GitHub repository URL along with my credentials so Jenkins could access it

![](./images/task%209%20added%20creds.jpg)


- Under **Triggers**, I enabled **GitHub hook trigger for GITScm polling** — this is what makes Jenkins listen for the webhook GitHub sends on every push


![](./images/task%209%20jenkin%20git%20config.jpg)


- Under **Post-build Actions**, I added **Archive the artifacts** and set the files pattern to `**` to capture everything


![](./images/task%209%20jenkins%20configg.jpg)


I saved the config and clicked **Build Now** to run it manually the first time. It succeeded. The first build showed up under `#1` in the build history and I could see the console output confirming the code had been pulled from GitHub.


![](./images/task%209%20build%20success.jpg)

![](./images/task%209%20jen%20build%20test.jpg)


### Test the Webhook

I made a small change to the `README.md` file in the GitHub repository and pushed it. Without touching Jenkins at all, a new build triggered automatically. Jenkins had received the webhook from GitHub, pulled the latest code, and archived the files.

The build artifacts are stored locally on the Jenkins server at:

```
/var/lib/jenkins/jobs/tooling_github/builds/<build_number>/archive/
```

![](./images/task%209%20checking%20build%20from%20linux.jpg)

---

## Step 3: Copy Files from Jenkins to the NFS Server via SSH

Having the files on the Jenkins server is not enough — they need to get to the NFS server so all the web servers can serve them. I used the **Publish Over SSH** plugin to make Jenkins push the files there automatically after every build.

### Install the Plugin

In Jenkins: **Manage Jenkins** → **Manage Plugins** → **Available** tab → searched for `Publish Over SSH` → installed it.

![](./images/task%209%20publish%20over%20ssh.jpg)


### Configure the SSH Connection

In Jenkins: **Manage Jenkins** → **Configure System** → scrolled down to the **Publish over SSH** section.

I filled in:

- **Private Key** — the contents of the `.pem` file I use to SSH into the NFS server
- **Name** — a label for the connection (e.g. `NFS-Server`)
- **Hostname** — the private IP address of the NFS server
- **Username** — `ec2-user` (the default user on RHEL EC2 instances)
- **Remote Directory** — `/mnt/apps` (the NFS mount that all web servers read from)


![](./images/task%209%20publish%20over%20ssh.jpg)


I clicked **Test Configuration** and got back `Success`. Port 22 on the NFS server's security group was already open so the connection went through cleanly.

### Add the Post-Build SSH Transfer

Back in the `pipeline` project configuration, I added another **Post-build Action**: **Send build artifacts over SSH**.

I configured it to:
- Use the `NFS-Server` connection I just set up
- Transfer all files using the `**` pattern (everything the build produces)
- Send them to the `/mnt/apps` directory on the NFS server


![](./images/task%209%20postact%20ssh%20config.jpg)


I saved and pushed another change to the GitHub `README.md`.

The build triggered automatically. In the console output I saw:

```
SSH: Transferred 25 file(s)
Finished: SUCCESS
```

### Verify the Files Landed on the NFS Server

I SSH'd into the NFS server and checked:

```bash
cat /mnt/apps/README.md
```

The change **just changed the text from introduction to intro** I had made in GitHub was right there. The full pipeline was working push to GitHub, Jenkins picks it up, files land on the NFS server, all three web servers serve the update. Done automatically, every single time.


![](./images/task%209%20finalee.jpg)


---

## What I Learned

This project changed how I think about deployment. A few things that really stuck with me:

- **Manual deployment does not scale.** When it is two servers it feels manageable. When it is twenty servers it becomes a full-time job. Jenkins removes that problem entirely the same pipeline that handles two servers handles two hundred.

- **Webhooks are what make it instant.** Without the webhook, Jenkins would have to keep checking GitHub every few minutes to see if something changed. The webhook flips it around GitHub tells Jenkins the moment a push happens. No polling, no delay, no wasted cycles.

- **The NFS setup from earlier was more important than I realized.** I only needed Jenkins to copy files to one place — the NFS server. Because all the web servers already mount that same folder, they all get the update at the same time without Jenkins having to connect to each one individually. The architecture built up across these projects is actually paying off.

- **Plugins are what make Jenkins so powerful.** Out of the box Jenkins can pull code and run scripts. But the `Publish Over SSH` plugin is what let me push files to a remote server without writing a single line of custom script. There are plugins for Slack notifications, Docker builds, AWS deployments, test reporting — almost any tool a team uses, Jenkins has a plugin for it.

- **The console output is your best friend.** Every time a build ran, I could open the console and see exactly what Jenkins did, line by line. When something went wrong it was easy to spot. That level of visibility is something you just do not get when you are copying files manually.