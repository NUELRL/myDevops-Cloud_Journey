# ⚙️ CI/CD Pipeline with Jenkins | Ansible | Artifactory | SonarQube | PHP

![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-Automation-EE0000?style=for-the-badge&logo=ansible&logoColor=white)
![SonarQube](https://img.shields.io/badge/SonarQube-Code%20Quality-4E9BCD?style=for-the-badge&logo=sonarqube&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-Interpreted-777BB4?style=for-the-badge&logo=php&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Blue Ocean](https://img.shields.io/badge/Blue%20Ocean-Pipeline%20UI-5C4EE5?style=for-the-badge&logo=jenkins&logoColor=white)
---

##  Overview

This project takes my DevOps journey into **Continuous Integration and Continuous Delivery (CI/CD)**. I built an end-to-end pipeline using **Jenkins**, **Ansible**, **SonarQube**, and **Artifactory** for a PHP-based application. Rather than deploying code directly from Git to a server (which I had been doing in earlier projects), I adopted a more production-realistic approach  packaging code with its dependencies and managing releases properly through a pipeline.

PHP is an interpreted language, which means it doesn't need to be compiled into a binary before running. However, that doesn't mean we should just clone raw code onto servers. The smarter approach is to package the code and all its dependencies into a versioned archive (`.tar.gz` or `.zip`) and deploy that artifact — which is exactly what I did here.

---

## Core Concepts I Studied First

Before touching a single server, I made sure I understood the theory deeply enough to explain it clearly. Here's my breakdown:

### What is Continuous Integration?

CI is the practice of merging developer code into a shared repository **multiple times per day**. The idea is simple: the longer branches drift from the mainline, the more painful the eventual merge becomes this is what the industry calls **Merge Hell**. By committing and pushing frequently, I keep the codebase clean, and automated tests catch issues early.

### The CI Workflow I Followed

1. **Run tests locally** — developers test their own code before committing (Test-Driven Development)
2. **Compile / package in CI** — Jenkins picks up the code and builds or packages it server-side
3. **Run further tests in CI** — unit tests, static code analysis, code coverage, security scans
4. **Deploy artifact** — the packaged code is pushed to an artifact repository (Artifactory)

### CI vs CD — What's the Difference?

| Term | Meaning |
|---|---|
| **Continuous Integration (CI)** | Automatically build and test every commit |
| **Continuous Delivery (CD)** | Code is always in a deployable state; release is manually triggered |
| **Continuous Deployment** | Fully automated  every passing build goes straight to production |

### 13 DevOps Metrics I Tracked

These metrics define how healthy a CI/CD pipeline actually is:

1. **Deployment Frequency** — how often are we shipping?
2. **Lead Time** — from work item start to production, how long does it take?
3. **Customer Tickets** — bugs reaching users = pipeline gaps
4. **% of Passed Automated Tests** — higher is better; tracks test reliability
5. **Defect Escape Rate** — how many bugs slip past QA into production?
6. **Availability** — is the app up? Are outages tracked?
7. **SLA Compliance** — are we meeting our commitments to users?
8. **Failed Deployments** — tracked as Mean Time To Failure (MTTF)
9. **Error Rates** — exceptions, DB timeouts, connection failures
10. **Application Traffic** — unusual spikes or drops after deployments signal problems
11. **Application Performance** — monitored via tools like DataDog or New Relic
12. **Mean Time To Detection (MTTD)** — how fast do we spot a problem?
13. **Mean Time To Recovery (MTTR)** — how fast do we fix it?

---

## Target Architecture

```
[GitHub Repo]
      │
      ▼
[Jenkins CI Server]
      │
   ┌──┴──────────────────┐
   ▼                      ▼
[SonarQube]         [Artifactory]
(Code Quality)      (Artifact Store)
      │
      ▼
[Ansible Playbooks]
      │
  ┌───┼───┬────────┬──────────┐
  ▼   ▼   ▼        ▼          ▼
 CI  Dev  SIT     UAT       Prod
```

Nginx sits in front of each environment as a **reverse proxy**, routing traffic to the appropriate application servers.

---

##  Ansible Inventory Structure

I organized my Ansible inventory to mirror each environment in the pipeline:

```
├── ci
├── dev
├── pentest
├── pre-prod
├── prod
├── sit
└── uat
```

### CI Inventory

```yaml
[jenkins]
<Jenkins-Private-IP>

[nginx]
<Nginx-Private-IP>

[sonarqube]
<SonarQube-Private-IP>

[artifact_repository]
<Artifactory-Private-IP>
```

### Dev Inventory

```yaml
[tooling]
<Tooling-Web-Server-Private-IP>

[todo]
<Todo-Web-Server-Private-IP>

[nginx]
<Nginx-Private-IP>

[db:vars]
ansible_user=ec2-user

[db]
<DB-Server-Private-IP>
```

### Pentest Inventory

```yaml
[pentest:children]
pentest-todo
pentest-tooling

[pentest-todo]
<Pentest-Todo-Private-IP>

[pentest-tooling]
<Pentest-Tooling-Private-IP>
```

**Why `pentest:children`?** I used this Ansible concept to group `pentest-todo` and `pentest-tooling` under a single `pentest` group, so I can run playbooks against both simultaneously — or target either one individually. Combined with `group_vars`, I can share variables across both child groups without repetition.

**Why does `db` look different?** The database server runs on **RedHat/CentOS** rather than Ubuntu, so it needs a different SSH user (`ec2-user`) and Python interpreter path. I handled this with `group_vars` so it stays clean and consistent.

---

##  Tools & Their Roles

### Jenkins
My CI server. Every commit triggers Jenkins to pick up the code, run the Jenkinsfile stages, and orchestrate the entire pipeline — from build through deployment.

### SonarQube
I used SonarQube for **static code analysis**. It automatically scans the codebase for bugs, code smells, and security vulnerabilities, giving me a quality gate before code moves further down the pipeline.

### Artifactory
My **artifact repository**. Rather than deploying straight from Git, packaged releases are stored here with version numbers. Ansible then pulls the correct artifact version during deployment using the `uri` module — no raw Git cloning onto servers.

---

## Jenkins Server Setup

I installed Jenkins with all its dependencies following the official documentation, then configured the Java environment:

```bash
sudo -i
nano .bash_profile
```

```bash
export JAVA_HOME=$(dirname $(dirname $(readlink $(readlink $(which java)))))
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/jre/lib:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
```

```bash
source ~/.bash_profile
```

Then I started and enabled Jenkins:

```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

I cloned my Ansible config repo and installed git on the Jenkins server to get started.

---

## Building the Jenkins Pipeline

### Connecting Jenkins to GitHub via Blue Ocean

I installed the **Blue Ocean plugin** in Jenkins and connected it to my GitHub repository. Blue Ocean gives a much cleaner visual view of pipeline stages compared to the classic Jenkins UI.

![](./images/task%2014%20pipe%20crt.jpg)
![](./images/task%2014%20pipe%20crt%202.jpg)


### Creating the Jenkinsfile

Inside my Ansible project repo, I created a `deploy/` directory and started building the `Jenkinsfile` incrementally  beginning with just a Build stage:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                script {
                    sh 'echo "Building Stage"'
                }
            }
        }
    }
}
```
![](./images/task%2014%20build%20config.jpg)

I pointed Jenkins to the Jenkinsfile location at `deploy/Jenkinsfile` inside the pipeline configuration.

![](./images/task%2014%20ci%20config.jpg)

Then i ran the build 
![](./images/task%2014%20build%20sucess.jpg)

### Adding More Stages

I created a feature branch `feature/jenkinspipeline-stages` and expanded the pipeline to simulate a full CI/CD flow:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                script { sh 'echo "Building Stage"' }
            }
        }
        stage('Test') {
            steps {
                script { sh 'echo "Testing Stage"' }
            }
        }
        stage('Package') {
            steps {
                script { sh 'echo "Packaging Stage"' }
            }
        }
        stage('Deploy') {
            steps {
                script { sh 'echo "Deploy Stage"' }
            }
        }
        stage('Clean Up') {
            steps {
                script { sh 'echo "Cleaning Up"' }
            }
        }
    }
}
```
![](./images/task%2014%20test,package.....jpg)

After verifying all stages passed in Blue Ocean, I merged the feature branch back into `main` then ran the it.

![](./images/task%2014%20test....sucess.jpg)

### Multi-Branch Pipeline Behaviour

One thing I found powerful: Jenkins automatically picks up **new branches** when I trigger a repository scan. Each branch gets its own pipeline run, visible separately in Blue Ocean. This means feature branches can be tested in isolation before merging  exactly how CI is meant to work.



## Running Ansible Playbooks from Jenkins

### Step 1 — Install Ansible on the Jenkins Server

```bash
sudo apt update && \
sudo apt install ansible -y && \
sudo apt install python3-pip -y && \
python3 -m pip install --upgrade pip setuptools && \
python3 -m pip install pyMySQL mysql-connector-python psycopg2-binary
```

Had this issue while installing
![](./images/task%2014%20python%20error.jpg)

Here's my fix
![](./images/task%2014%20python%20error%20fix.jpg)
![](./images/task%2014%20anisble%20installed.jpg)

### Step 2 — Install the Ansible Plugin in Jenkins UI

I navigated to **Manage Jenkins → Plugins** and installed the Ansible plugin. This lets Jenkins invoke Ansible playbooks natively as a pipeline step rather than relying on raw shell commands.

![](./images/task%2014%20ansible%20installed%20on%20jenkins.jpg)
### Step 3 — Add SSH Credentials in Jenkins UI

I added my private key as a Jenkins credential so Ansible can authenticate against the target servers via SSH. This keeps secrets out of the Jenkinsfile entirely.


### Step 4 — Configure Ansible Path in Jenkins

I ran `which ansible` on the Jenkins server to get the path to the Ansible executable, then pasted that path into **Manage Jenkins → Tools → Ansible installations**.

![](./images/task%2014%20anible%20path.jpg)

### Step 5 — Generate the Ansible Playbook Syntax

Using Jenkins' built-in **Pipeline Syntax** generator, I generated the correct `ansiblePlaybook` step syntax rather than writing it by hand — this avoids typos and ensures all parameters are correctly formatted.

![](./images/task%2014%20pipesyntax.jpg)
![](./images/task%2014%20pipesyntax%202.jpg)
![](./images/task%2014%20pipe%20syntax%203.jpg)
---

##  Full Jenkinsfile (Built from Scratch)

I deleted the previous stub Jenkinsfile and rewrote it properly to run Ansible end-to-end then ran the build to test the pipeline:

```groovy
pipeline {
  agent any

  environment {
    ANSIBLE_CONFIG="${WORKSPACE}/deploy/ansible.cfg"
  }

  parameters {
    string(name: 'inventory', defaultValue: 'dev', description: 'This is the inventory file for the environment to deploy configuration')
  }

  stages {
    stage("Initial cleanup") {
      steps {
        dir("${WORKSPACE}") {
          deleteDir()
        }
      }
    }

    stage('Checkout SCM') {
      steps {
        git branch: 'main', url: 'https://github.com/IwunzeGE/ansible-config-mgt.git'
      }
    }

    stage('Prepare Ansible For Execution') {
      steps {
        sh 'echo ${WORKSPACE}'
        sh 'sed -i "3 a roles_path=${WORKSPACE}/roles" ${WORKSPACE}/deploy/ansible.cfg'
      }
    }

    stage('Run Ansible playbook') {
      steps {
        ansiblePlaybook become: true, colorized: true, credentialsId: 'private-key', disableHostKeyChecking: true, installation: 'ansible', inventory: 'inventory/${inventory}', playbook: 'playbooks/site.yml'
      }
    }

    stage('Clean Workspace after build') {
      steps {
        cleanWs(cleanWhenAborted: true, cleanWhenFailure: true, cleanWhenNotBuilt: true, cleanWhenUnstable: true, deleteDirs: true)
      }
    }
  }
}
```

![](./images/task%2014%20pipeline%20sucesss.jpg)

### Why each stage matters

**Initial cleanup** — deletes the previous workspace before every run. Without this, stale files from an earlier build can cause Jenkins to behave as if nothing changed, even when you've pushed a fix. I lost time to this once before adding the cleanup step.

**Prepare Ansible For Execution** — uses `sed` to dynamically inject the `roles_path` into `ansible.cfg` at runtime using `${WORKSPACE}`. Since Jenkins checks out the repo to a workspace path that changes per branch, hardcoding the roles path would break multi-branch pipelines.

**Clean Workspace after build** — cleans up on any outcome (failure, abort, unstable) so the next run always starts fresh.

---

##  Ansible Config File

I also created `deploy/ansible.cfg` alongside the Jenkinsfile so everything deployment-related lives in one place:

```ini
[defaults]
timeout = 160
callback_whitelist = profile_tasks
log_path=~/ansible.log
host_key_checking = False
gathering = smart
ansible_python_interpreter=/usr/bin/python3
allow_world_readable_tmpfiles=true

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=30m -o ControlPath=/tmp/ansible-ssh-%h-%p-%r -o ServerAliveInterval=60 -o ServerAliveCountMax=60 -o ForwardAgent=yes
```

---

##  Parameterizing the Jenkinsfile for Multi-Environment Deployment

Hard-coding `inventory/dev` in the pipeline is fine for one environment, but not scalable. Instead of maintaining separate branches or manually editing the Jenkinsfile every time I need to deploy to SIT, UAT, or Pentest, I parameterized the inventory:

```groovy
parameters {
  string(name: 'inventory', defaultValue: 'dev', description: 'This is the inventory file for the environment to deploy configuration')
}
```

Then in the Ansible step, I replaced the hardcoded path with the parameter:

```groovy
inventory: 'inventory/${inventory}'
```

Now when I trigger a build, Jenkins prompts me to specify the environment. The default is `dev` so it still works hands-free for standard runs, but I can target `sit`, `uat`, `pentest`, or `prod` just by changing the input at runtime — no code changes needed.

### Troubleshooting the Parameterized Build

After introducing the parameter, my first build failed because `roles_path` in `ansible.cfg` couldn't be resolved. The fix was ensuring the `sed` command correctly injected the dynamic `${WORKSPACE}/roles` path into the config at execution time — not at checkout time. Once that was sorted, the parameterized build ran cleanly.

---

##  SIT Inventory

I also updated the SIT inventory to prepare for multi-environment deployments:

```yaml
[tooling]
<SIT-Tooling-Web-Server-Private-IP>

[todo]
<SIT-Todo-Web-Server-Private-IP>

[nginx]
<SIT-Nginx-Private-IP>

[db:vars]
ansible_user=ec2-user

[db]
<SIT-DB-Server-Private-IP>
```

---

##  CI/CD Pipeline for the PHP Todo Application

### Phase 1 — Prepare Jenkins

I forked the PHP Todo repository into my GitHub account, then installed PHP and all required dependencies on the Jenkins server:

```bash
sudo apt update
sudo apt install -y php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysql php-fpm
sudo systemctl start php8.3-fpm
sudo systemctl enable php8.3-fpm
```
![](./images/task%2014%20php%20install.jpg)

### Install Composer

```bash
sudo apt install -y curl php-cli php-mbstring unzip
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```
![](./images/task%2014%20php%20install%202.jpg)

### Install PHPUnit and phploc

```bash
sudo apt install -y wget php-cli php-mbstring php-xml php-zip unzip

# PHPUnit
wget -O phpunit https://phar.phpunit.de/phpunit-9.phar
chmod +x phpunit
sudo mv phpunit /usr/local/bin/phpunit

# phploc
wget -O phploc.phar https://phar.phpunit.de/phploc.phar
chmod +x phploc.phar
sudo mv phploc.phar /usr/local/bin/phploc

phpunit --version
phploc --version
```
![](./images/task%2014%20php%20install%203.jpg)

### Jenkins Plugins Installed

- **Plot plugin** — displays test reports and code coverage graphs directly in the Jenkins UI
- **Artifactory plugin** — handles uploading build artifacts to the Artifactory server automatically

---

### Phase 2 — Integrate Artifactory with Jenkins

I ran the CI inventory build first to let Ansible provision and configure the Artifactory server. After it came up, I navigated to `<public-ip>:8081`, logged in with the default credentials, changed the password, and created a **generic local repository** to store PHP build artifacts.

> **Note:** Ports **8081** and **8082** must be open in the EC2 security group inbound rules for Artifactory to work.

I then connected Jenkins to Artifactory via **Manage Jenkins → Configure System → Artifactory**, providing the server URL and credentials.

---

### Phase 3 — MySQL Role Configuration for the Todo App

In the MySQL Ansible role (`roles/mysql/defaults/main.yml`), I created the `homestead` database and a user scoped to the Jenkins server's IP:

```sql
CREATE DATABASE homestead;
CREATE USER 'homestead'@'<Jenkins-IP>' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'homestead'@'%';
```

The user is scoped to the Jenkins server IP because Jenkins needs remote access to run migrations against the database.

---

### Phase 4 — Todo App Jenkinsfile

I created a multibranch pipeline in Blue Ocean for the PHP Todo repo, then wrote the Jenkinsfile:

```groovy
pipeline {
  agent any

  stages {
    stage("Initial cleanup") {
      steps {
        dir("${WORKSPACE}") {
          deleteDir()
        }
      }
    }

    stage('Checkout SCM') {
      steps {
        git branch: 'main', url: 'https://github.com/IwunzeGE/php-todo.git'
      }
    }

    stage('Prepare Dependencies') {
      steps {
        sh 'mv .env.sample .env'
        sh 'composer install'
        sh 'php artisan migrate'
        sh 'php artisan db:seed'
        sh 'php artisan key:generate'
      }
    }

    stage('Execute Unit Tests') {
      steps {
        sh './vendor/bin/phpunit'
      }
    }
  }
}
```

![](./images/task%2014%20pipeline%20working.jpg)

### What the Prepare Dependencies stage does

- **`.env.sample` → `.env`** — PHP requires a `.env` file at runtime; the sample file is renamed to activate it
- **`composer install`** — downloads and installs all PHP library dependencies declared in `composer.json`
- **`php artisan migrate`** — runs database migrations, creating all the required tables in the `homestead` database
- **`php artisan db:seed`** — populates the tables with initial seed data
- **`php artisan key:generate`** — generates the application encryption key


### Troubleshooting DB Connectivity

When the pipeline first ran, I hit a database connection error. The fix required two things: installing `mysql-client` on the Jenkins server, and updating the `bind-address` in the DB server's MySQL config to allow remote connections from the Jenkins server's IP.

![](./images/task%2014%20pipefail.jpg)

#  Code Quality Analysis, SonarQube Quality Gates & Jenkins Agents


##  Phase 3 — Code Quality Analysis

### Code Analysis with phploc

I added a Code Analysis stage to the Jenkinsfile that runs `phploc` against the application source and outputs the results to a CSV file:

```groovy
stage('Code Analysis') {
  steps {
    sh 'phploc app/ --log-csv build/logs/phploc.csv'
  }
}
```

`phploc` scans the PHP codebase and produces metrics like lines of code, cyclomatic complexity, number of classes, methods, functions, and test coverage stats. The output is stored in `build/logs/phploc.csv` which subsequent stages consume.

### Plotting the Metrics

I used the **Plot Jenkins plugin** to visualise the `phploc` CSV data across builds. Each plot tracks a different dimension of code quality over time:

```groovy
stage('Plot Code Coverage Report') {
  steps {
    plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv',
         csvSeries: [[file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING',
         exclusionValues: 'Lines of Code (LOC),Comment Lines of Code (CLOC),Non-Comment Lines of Code (NCLOC),Logical Lines of Code (LLOC)']],
         group: 'phploc', numBuilds: '100', style: 'line', title: 'A - Lines of code', yaxis: 'Lines of Code'
}
```

After this stage runs, a **Plot menu item** appears in the Jenkins job sidebar showing line charts for each metric across all builds. This gives developers and team leads a visual history of how code quality trends over time — not just a snapshot.

![](./images/task%2014%20plots.jpg)

---

##  Packaging and Publishing the Artifact

### Package the Application

After quality checks pass, I zip the entire workspace into a deployable artifact:

```groovy
stage('Package Artifact') {
  steps {
    sh 'zip -qr php-todo.zip ${WORKSPACE}/*'
  }
}
```

### Upload to Artifactory

The zipped artifact is then pushed to the Artifactory repository:

```groovy
stage('Upload Artifact to Artifactory') {
  steps {
    script {
      def server = Artifactory.server 'artifactory-server'
      def uploadSpec = """{
        "files": [
          {
            "pattern": "php-todo.zip",
            "target": "<name-of-artifact-repository>/php-todo",
            "props": "type=zip;status=ready"
          }
        ]
      }"""
      server.upload spec: uploadSpec
    }
  }
}
```

Every build now has a versioned artifact in Artifactory. Deployments pull from here rather than from Git directly.

---

## Deploying to the Dev Environment

### Ansible Deployment Playbook

I created `static-assignments/deployment.yml` to handle deploying the PHP application to the Todo server. The playbook installs all required packages (httpd, PHP 7.4 via Remi repo, php-fpm), downloads the artifact from Artifactory, unpacks it, copies the code to the web root, and restarts httpd:

```yaml
---
- name: Deploying the PHP Application to Dev Environment
  become: true
  hosts: todo
  tasks:
    - name: install remi and rhel repo
      ansible.builtin.yum:
        name:
          - https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
          - dnf-utils
          - https://rpms.remirepo.net/enterprise/remi-release-8.rpm
        disable_gpg_check: yes

    - name: install httpd on the webserver
      ansible.builtin.yum:
        name: httpd
        state: present

    - name: ensure httpd is started and enabled
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: yes

    - name: install PHP
      ansible.builtin.yum:
        name:
          - php
          - php-mysqlnd
          - php-gd
          - php-curl
          - unzip
          - php-common
          - php-mbstring
          - php-opcache
          - php-intl
          - php-xml
          - php-fpm
          - php-json
        enablerepo: php:remi-7.4
        state: present

    - name: ensure php-fpm is started and enabled
      ansible.builtin.service:
        name: php-fpm
        state: started
        enabled: yes

    - name: Download the artifact
      get_url:
        url: http://<artifactory-server-ip>:8082/artifactory/<repo>/php-todo
        dest: /home/ec2-user/
        url_username: admin
        url_password: "{{ artifactory_password }}"

    - name: unzip the artifacts
      ansible.builtin.unarchive:
        src: /home/ec2-user/php-todo
        dest: /home/ec2-user/
        remote_src: yes

    - name: deploy the code
      ansible.builtin.copy:
        src: /home/ec2-user/var/lib/jenkins/workspace/php-todo_main/
        dest: /var/www/html/
        force: yes
        remote_src: yes

    - name: remove nginx default page
      ansible.builtin.file:
        path: /etc/httpd/conf.d/welcome.conf
        state: absent

    - name: restart httpd
      ansible.builtin.service:
        name: httpd
        state: restarted
```

### Triggering Ansible from the Todo Pipeline

Inside the php-todo Jenkinsfile, the Deploy stage triggers the Ansible pipeline as a downstream job:

```groovy
stage('Deploy to Dev Environment') {
  steps {
    build job: 'ansible-config-mgt/main',
          parameters: [[$class: 'StringParameterValue', name: 'env', value: 'dev']],
          propagate: false,
          wait: true
  }
}
```

When the php-todo pipeline reaches this stage, it triggers `ansible-config-mgt` to run with `dev` as the target environment. The two pipelines are now connected — php-todo builds and packages the app, Ansible deploys it.

---

## SonarQube Installation

### Why SonarQube?

SonarQube enforces **quality gates** — predefined acceptance criteria that code must pass before it can progress through the pipeline. Without it, bugs and security vulnerabilities can slip through to production undetected. I used the predefined **Sonar Way** quality gate profile for this project.

### Tune Linux Kernel First

SonarQube is resource-intensive. Before installing, I adjusted kernel parameters for optimal performance:

```bash
sudo sysctl -w vm.max_map_count=262144
sudo sysctl -w fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
```

To make these permanent, I appended to `/etc/security/limits.conf`:

```
sonarqube   -   nofile   65536
sonarqube   -   nproc    4096
```

### Install Dependencies

```bash
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install wget unzip -y
sudo apt-get install openjdk-11-jdk -y
sudo apt-get install openjdk-11-jre -y
sudo update-alternatives --config java  # select OpenJDK 11
```

### Set Up PostgreSQL as the Backend Database

SonarQube dropped MySQL support, so I used PostgreSQL:

```bash
# Add PostgreSQL repo
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -

sudo apt-get -y install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

Then I created the SonarQube database and user:

```bash
sudo passwd postgres
su - postgres
createuser sonar
psql
```

```sql
ALTER USER sonar WITH ENCRYPTED password 'sonar';
CREATE DATABASE sonarqube OWNER sonar;
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;
\q
```

### Install SonarQube

```bash
cd /tmp && sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-7.9.3.zip
sudo unzip sonarqube-7.9.3.zip -d /opt
sudo mv /opt/sonarqube-7.9.3 /opt/sonarqube
```

### Configure SonarQube

SonarQube cannot run as root. I created a dedicated user and group:

```bash
sudo groupadd sonar
sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar
sudo chown sonar:sonar /opt/sonarqube -R
```

I then edited `/opt/sonarqube/conf/sonar.properties` to uncomment and set the PostgreSQL credentials, and set `RUN_AS_USER=sonar` in the sonar.sh script.

### Run SonarQube as a systemd Service

```bash
sudo nano /etc/systemd/system/sonar.service
```

```ini
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonar
Group=sonar
Restart=always
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl start sonar
sudo systemctl enable sonar
sudo systemctl status sonar
```

Access SonarQube at `http://<server-ip>:9000` and log in with `admin / admin`.

![](./images/task%2014%20sonar%20sucesss.jpg)

---

##  Configuring SonarQube and Jenkins Together

### In Jenkins

I installed the **SonarQube Scanner plugin**, then navigated to **Manage Jenkins → Configure System** and added the SonarQube server URL and authentication token.

I also configured the **SonarQube Scanner** under **Global Tool Configuration** so Jenkins knows which scanner binary to invoke.

### In SonarQube

I configured a **Jenkins webhook** so SonarQube can report quality gate results back to Jenkins:

```
http://<JENKINS_HOST>/sonarqube-webhook/
```
![](./images/task%2014%20sonarwebhook.jpg)


Without this webhook, the `waitForQualityGate` step in the Jenkinsfile would hang indefinitely because Jenkins would have no way of knowing whether the quality gate passed or failed.

### Add the Quality Gate Stage to Jenkinsfile

```groovy
stage('SonarQube Quality Gate') {
  environment {
    scannerHome = tool 'SonarQubeScanner'
  }
  steps {
    withSonarQubeEnv('sonarqube') {
      sh "${scannerHome}/bin/sonar-scanner"
    }
  }
}
```

### Configure sonar-scanner.properties

Jenkins installs the scanner tool under its tools directory. I navigated there and configured the properties file:

```bash
cd /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/conf/
sudo vi sonar-scanner.properties
```

```properties
sonar.host.url=http://<SonarQube-Server-IP>:9000
sonar.projectKey=php-todo
sonar.sourceEncoding=UTF-8
sonar.php.exclusions=**/vendor/**
sonar.php.coverage.reportPaths=build/logs/clover.xml
sonar.php.tests.reportPath=build/logs/junit.xml
sonar.sources=/var/lib/jenkins/workspace/php-todo_main
```

> **Note:** I had to explicitly add `sonar.sources` after the scanner returned an error saying the project source wasn't specified. This points SonarQube directly to the Jenkins workspace where the code was checked out.

---

##  What SonarQube Found

After the first scan, I navigated to the php-todo project in the SonarQube UI. The results were sobering:

- **8 bugs** detected
- **0.0% code coverage** — no unit tests covering the functions
- **6 hours of technical debt**
- Multiple **code smells** and **security vulnerabilities**

In a development environment, this is expected — developers are still iterating. But the pipeline as it stood was letting this code through without enforcing any quality gate. That had to change.

![](./images/task%2014%20code%20smell.jpg)


---

##  Enforcing the Quality Gate

I updated the Jenkinsfile so the quality gate actually blocks the pipeline when code fails:

```groovy
stage('SonarQube Quality Gate') {
  when { branch pattern: "^develop*|^hotfix*|^release*|^main*", comparator: "REGEXP" }
  environment {
    scannerHome = tool 'SonarQubeScanner'
  }
  steps {
    withSonarQubeEnv('sonarqube') {
      sh "${scannerHome}/bin/sonar-scanner -Dproject.settings=sonar-project.properties"
    }
    timeout(time: 1, unit: 'MINUTES') {
      waitForQualityGate abortPipeline: true
    }
  }
}
```

Two important additions here:

**`when { branch pattern: ... }`** — the quality gate only runs on specific protected branches (`develop`, `hotfix`, `release`, `main`). Feature branches are excluded, giving developers freedom to push work-in-progress without every commit being blocked. This aligns with the **GitFlow** branching strategy.

**`waitForQualityGate abortPipeline: true`** — Jenkins waits for SonarQube's webhook callback. If the quality gate fails, the pipeline is aborted immediately. Nothing proceeds to packaging or deployment until the code meets the quality bar.

---

##  Jenkins Agents / Slave Nodes

### Why Slave Nodes?

A single Jenkins master handling all pipeline jobs becomes a bottleneck as the number of pipelines grows. By adding **agent/slave nodes**, Jenkins can distribute workloads — running multiple pipelines in parallel across different machines.

### Install Java on Each Slave Node

```bash
sudo yum install java-11-openjdk-devel -y
sudo -i
nano .bash_profile
```

```bash
export JAVA_HOME=$(dirname $(dirname $(readlink $(readlink $(which java)))))
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/jre/lib:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
```

```bash
source ~/.bash_profile
```

### Register Slave Nodes in Jenkins UI

I navigated to **Manage Jenkins → Manage Nodes → New Node**, configured each slave with its SSH connection details, and set the number of executors. Once registered, Jenkins can now schedule pipeline jobs on any available node — including the two new slaves — distributing load automatically.

---
![](./images/task%2014%20agent%201.jpg)
![](./images/task%2014%20agent%202.jpg)
![](./images/task%2014%20agent%203.jpg)



## GitHub Webhook for Automatic Triggers

I configured a **GitHub webhook** pointing to the Jenkins server so that every `git push` to the repository automatically triggers the relevant pipeline. This removes the need to manually click "Build Now"  the pipeline fires the moment code lands in GitHub, completing the CI loop.

![](./images/task%2014%20multi%20line%20pipe.jpg)
![](./images/task%2014%20multi%20line%20pipe%202.jpg)


# Troubleshooting Log — CI/CD Pipeline Project

---


This documents every real error I hit during this project, what caused each one, and exactly how I fixed it. This isn't theory — these are the actual problems I debugged in sequence, and working through them taught me more about Linux, Jenkins, and production systems than the happy-path setup ever could.

---

##  Error 1 — Disk Completely Full

### What I Saw

```bash
df -h
# /dev/root  8.7G  8.7G  0  100% /
```

```
fallocate: fallocate failed: No space left on device
```

### Root Cause

I ran `sudo du -h --max-depth=1 / | sort -hr` and traced the culprit:

```
4.0G  /opt
```
![](./images/task%2014%20df%20-h.jpg)



Drilling deeper, the entire 4GB was coming from `/opt/jfrog/artifactory`. Artifactory's logs, data, and backups had grown to fill the disk. On top of that, 40+ old Jenkins builds, journal logs, and Jenkins plugins (~326MB) were piling on.

Linux needs free space to write logs, create temp files, and run services. With 0 bytes free, everything started misbehaving — Artifactory failed, Jenkins couldn't extract its WAR file to `/tmp`, and services started crashing in cascade.

### What I Tried First (Partial Fix)

```bash
# Deleted old journal logs
sudo journalctl --vacuum-size=50M      # freed 79.7MB

# Cleared other logs
sudo truncate -s 0 /var/log/syslog
sudo rm -f /var/log/dmesg*
sudo rm -rf /tmp/*
```

This helped a little but the disk was still at ~99% with only 73MB free. Not enough. Jenkins couldn't start — it still couldn't extract its WAR to `/tmp`.

### The Real Fix — New EBS Volume

I attached a fresh **10GB EBS volume** (`nvme1n1`) in AWS and mounted it:

```bash
lsblk                          # confirmed nvme1n1 available
sudo mkfs.ext4 /dev/nvme1n1   # formatted the volume
sudo mount /dev/nvme1n1 /mnt  # mounted at /mnt
```
![](./images/task%2014%20created%20a%20new%20volume.jpg)


Now the root disk (`/`) was full, but `/mnt` was a fresh 10GB. I moved Artifactory there then linked it to the original folder so the system thinks the files are in the same folder:

```bash
sudo systemctl stop artifactory
sudo mv /opt/jfrog/artifactory /mnt/
```



This freed 4GB from the root disk immediately. After the move:

```bash
df -h
# / = 56% 
```

---

##  Error 2 — Artifactory Failed to Start After Being Moved

### What I Saw

```
connection refused
cluster join failed
HTTP 404 on port 8081/8082
```

### Root Cause

Artifactory was hardcoded to look for its files at `/opt/jfrog/artifactory`. I had moved everything to `/mnt/artifactory`, so when the service tried to start, it couldn't find its own files. The process started, failed silently, and the browser got a 404.


```bash
ln -s /mnt/artifactory /opt/jfrog/artifactory
```

This tells the system: whenever something looks for `/opt/jfrog/artifactory`, actually go to `/mnt/artifactory`. Artifactory had no idea it was moved — it found its files exactly where it expected them.

Without this symlink, Artifactory would fail to start every single time regardless of what else was fixed.

![](./images/task%2014%20art%20moved.jpg)
---

##  Error 3 — Artifactory Still Failing After Symlink (Permission Denied)

### What I Saw

```
permission denied
artifactory cannot read/write files
service keeps restarting
```

### Root Cause

When I moved the files with `sudo mv`, ownership stayed as root. The `artifactory` service runs as the `artifactory` system user, not root. So even though the files existed at the right path, Artifactory couldn't actually read or write them.

### Fix

```bash
sudo chown -R artifactory:artifactory /mnt/artifactory
sudo chmod -R 755 /mnt/artifactory
```

Permissions and ownership must be corrected after any file migration. This is one of those mistakes that's obvious in hindsight but easy to forget in the middle of troubleshooting.

---

##  Error 4 — Artifactory Showing 404 Even After Permissions Fixed

### What I Saw

```
Artifactory UI still returning 404
Logs showing: failed to connect to 127.0.0.1:5432
```

### Root Cause

Port `5432` is PostgreSQL. Artifactory uses PostgreSQL as its backend database. The database was down — it hadn't been started after the disk operations. Artifactory was up and running, but it had no database to connect to, so it kept retrying and returning errors to the browser.

```bash
pg_lsclusters
# Status: down
```

### Fix

```bash
sudo systemctl start postgresql
pg_lsclusters
# Status: online 
```

Once PostgreSQL came back up, Artifactory connected to its database and everything loaded:

- Port 8081 → running 
- Port 8082 → UI accessible 
- Browser → loads correctly 

---

##  Error 5 — SonarQube Java Compatibility Error

### What I Saw

```
java.lang.ExceptionInInitializerError
module java.base does not "opens java.lang" to unnamed module
```

### Root Cause

SonarQube 7.9.3 was being scanned using Java 17, which enforces strict module encapsulation rules introduced in Java 9+. The SonarQube scanner was trying to access internal Java modules that Java 17 locks down by default, causing the JVM to throw an `ExceptionInInitializerError` before the scan even started.

### Fix

I added `SONAR_SCANNER_OPTS` to the SonarQube stage in the Jenkinsfile to explicitly unlock the affected module:

```groovy
stage('SonarQube Quality Gate') {
  environment {
    scannerHome = tool 'SonarQubeScanner'
    SONAR_SCANNER_OPTS = "--add-opens java.base/java.lang=ALL-UNNAMED"
  }
  steps {
    withSonarQubeEnv('sonarqube') {
      sh "${scannerHome}/bin/sonar-scanner"
    }
  }
}
```

This tells the JVM to allow the scanner access to the restricted module without downgrading Java.

---

##  Error 6 — Jenkins Node Went Offline

### What I Saw

```
Disk space is below threshold of 1.00 GiB
Node going offline immediately after being brought back online
```

### Root Cause

Jenkins continuously monitors its node's disk health. When free disk space dropped below 1GiB (and later, free temp space dropped to 67MB), Jenkins automatically marked the built-in node as offline and refused to run any jobs on it. Even after I brought it back online manually, it went offline again instantly because the temp directory was still pointing at the full root disk.

### Fix

I created a fresh temp directory on the new 10GB volume and told Jenkins to use it:

```bash
mkdir -p /mnt/jenkins-data/tmp
```

Then updated the Jenkins service file:

```
JENKINS_HOME=/mnt/jenkins-data/jenkins
JAVA_OPTS=-Djava.io.tmpdir=/mnt/jenkins-data/tmp
```

Moved the entire Jenkins home to the new volume:

```bash
cp -rp /var/lib/jenkins /mnt/jenkins-data/jenkins
ln -s /mnt/jenkins-data/jenkins /var/lib/jenkins
```

After restarting Jenkins with the new config:

| Resource | Before | After |
|---|---|---|
| Free Disk Space | 73MB (99% full) | 8.78GB |
| Free Swap Space | 0B | 1.03GB |
| Free Temp Space | 67MB | 8.8GB |
| Jenkins Node | Offline | Online  |

---

##  Error 7 — Jenkins Failed to Start After Switching Java Versions

### What I Saw

```
Job for jenkins.service failed
```

### Root Cause

In trying to fix the SonarQube Java error, I switched the system to Java 11, thinking it would be more compatible. But Jenkins itself requires Java 17 to run. Switching to Java 11 broke Jenkins entirely — it couldn't start at all.

### Fix

Switched the system back to Java 17 for Jenkins:

```bash
sudo update-alternatives --config java  # selected Java 17
sudo systemctl start jenkins
```

Then handled the SonarQube compatibility separately using `SONAR_SCANNER_OPTS` (see Error 5) instead of downgrading Java. The right solution was to tell the scanner to relax its module restrictions — not to downgrade the runtime.

---

## Error 8 — Pipeline Waiting Forever

### What I Saw

```
Still waiting to schedule task - Waiting for next available executor
```

### Root Cause

Old stuck builds were holding the only available executor on the node. Combined with the node being offline due to disk space issues, new pipeline jobs had nowhere to run and just queued indefinitely.

### Fix

I aborted all stuck builds from the Jenkins UI, cleared disk space (as covered in Errors 1–4), restarted Jenkins, and brought the node back online. Once the node had a free executor and sufficient disk space, the pipeline scheduled and ran normally.

---

##  What This All Taught Me

Working through these errors in sequence gave me a much deeper understanding of how these systems actually interact than any documentation could:

**Disk space is a hidden dependency for everything.** Services don't just fail — they fail in confusing, indirect ways. Artifactory returning a 404 had nothing to do with Artifactory's config. It was a database that wasn't running because the disk was too full to operate normally.

**Moving files without relinking and fixing permissions is incomplete.** Three separate steps are always required after migrating data to a new volume: move the data, create a symlink back to the expected path, and fix ownership. Miss any one of them and the service fails in a different way each time.

**Java version mismatches are subtle.** The error message (`does not opens java.lang to unnamed module`) doesn't obviously say "wrong Java version" — it takes knowing that Java 9+ introduced module encapsulation to connect those dots. And the fix (`--add-opens`) is in the JVM flags, not in the application config.

**Jenkins node monitoring is strict by design.** The automatic offline threshold exists to protect pipelines from failing due to resource exhaustion. Understanding it means knowing to check temp space, not just disk space, because Jenkins monitors them separately.

**Never downgrade a runtime to fix a tool's compatibility issue.** Switching to Java 11 to fix SonarQube broke Jenkins. The right fix was always at the tool level (`SONAR_SCANNER_OPTS`), not the system level.