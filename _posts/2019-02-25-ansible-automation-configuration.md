---
layout: post
title: "Ansible 自動化配置管理"
date: 2019-02-25 10:00:00 +0800
categories: [DevOps, Automation]
tags: [Ansible, Configuration Management, Automation, Infrastructure]
---

上週學了 Terraform 管理基礎設施（參考 [Terraform Infrastructure as Code 基礎](/posts/2019/02/18/terraform-infrastructure-as-code/)），這週來解決另一個問題：機器建好後，如何安裝和設定軟體？

目前的流程：
1. Terraform 建立 EC2 instances
2. SSH 進去每台機器
3. 手動執行一堆指令安裝軟體
4. 複製設定檔
5. 啟動服務
6. 重複 N 次（有幾台機器就做幾次）

痛點：
- **重複勞動**：有 10 台機器就要重複 10 次
- **容易出錯**：第 8 台可能漏了某個步驟
- **環境不一致**：每台機器微妙不同
- **難以追蹤**：不知道誰改了什麼設定

這週研究 Ansible，讓配置管理自動化。

> 使用版本：Ansible 2.7.7（2019 年初的穩定版）

## Ansible 是什麼

Ansible 是 Red Hat 開發的自動化工具，用於：
- **配置管理**：安裝軟體、設定系統
- **應用部署**：部署應用程式
- **編排**：協調多台機器的操作

特色：
- **無 Agent**：不用在目標機器安裝任何軟體，只需要 SSH
- **宣告式**：描述想要的狀態，Ansible 會想辦法達成
- **冪等性**：重複執行結果相同，不會重複安裝
- **易學**：使用 YAML 語法，簡單直觀

## 安裝 Ansible

```bash
# macOS
brew install ansible@2.7

# Linux (Ubuntu/Debian)
sudo apt-add-repository ppa:ansible/ansible
sudo apt update
sudo apt install ansible=2.7.7+dfsg-1ppa~bionic

# 驗證
ansible --version
# ansible 2.7.7
```

## 基本概念

### Inventory

定義目標機器：

`inventory.ini`：
```ini
[webservers]
web1 ansible_host=192.168.1.10 ansible_user=ubuntu
web2 ansible_host=192.168.1.11 ansible_user=ubuntu

[databases]
db1 ansible_host=192.168.1.20 ansible_user=ubuntu

[production:children]
webservers
databases
```

測試連線：
```bash
ansible all -i inventory.ini -m ping

# web1 | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
# web2 | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
# db1 | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
```

### Ad-hoc 指令

快速執行單一指令：

```bash
# 檢查磁碟空間
ansible webservers -i inventory.ini -m shell -a "df -h"

# 安裝套件
ansible webservers -i inventory.ini -m apt -a "name=nginx state=present" --become

# 重啟服務
ansible webservers -i inventory.ini -m service -a "name=nginx state=restarted" --become

# 複製檔案
ansible webservers -i inventory.ini -m copy -a "src=/local/file dest=/remote/file"
```

`--become` 表示使用 sudo 權限。

### Playbook

Playbook 是 Ansible 的核心，用 YAML 描述要做的事。

`webserver.yml`：
```yaml
---
- name: Setup web servers
  hosts: webservers
  become: yes
  
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
    
    - name: Install Nginx
      apt:
        name: nginx
        state: present
    
    - name: Start Nginx service
      service:
        name: nginx
        state: started
        enabled: yes
    
    - name: Copy index.html
      copy:
        src: files/index.html
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'
```

執行：
```bash
ansible-playbook -i inventory.ini webserver.yml

# PLAY [Setup web servers] ***********************************************
# 
# TASK [Update apt cache] ************************************************
# ok: [web1]
# ok: [web2]
# 
# TASK [Install Nginx] ***************************************************
# changed: [web1]
# changed: [web2]
# 
# TASK [Start Nginx service] *********************************************
# ok: [web1]
# ok: [web2]
# 
# TASK [Copy index.html] *************************************************
# changed: [web1]
# changed: [web2]
# 
# PLAY RECAP *************************************************************
# web1                       : ok=4    changed=2    unreachable=0    failed=0
# web2                       : ok=4    changed=2    unreachable=0    failed=0
```

## 實際案例：部署 Spring Boot 應用

### 目標環境

- 2 台 web server
- 1 台 database server
- 部署 Spring Boot 應用

### Inventory

`inventory/production.ini`：
```ini
[webservers]
web1 ansible_host=10.0.1.10 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/mykey.pem
web2 ansible_host=10.0.1.11 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/mykey.pem

[databases]
db1 ansible_host=10.0.1.20 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/mykey.pem

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

### 變數

`group_vars/all.yml`：
```yaml
# 應用設定
app_name: myshop
app_version: 1.0.0
app_jar: "{{ app_name }}-{{ app_version }}.jar"
app_home: "/opt/{{ app_name }}"
app_user: myshop
app_group: myshop

# Java 設定
java_version: 8
java_home: /usr/lib/jvm/java-8-openjdk-amd64
```

`group_vars/webservers.yml`：
```yaml
app_port: 8080
jvm_opts: "-Xms512m -Xmx1024m"
```

`group_vars/databases.yml`：
```yaml
mysql_root_password: "{{ vault_mysql_root_password }}"
mysql_database: myshop
mysql_user: myshop
mysql_password: "{{ vault_mysql_password }}"
```

### 加密敏感資訊

使用 Ansible Vault 加密密碼：

```bash
# 建立加密檔案
ansible-vault create group_vars/vault.yml

# 編輯內容
vault_mysql_root_password: SuperSecret123!
vault_mysql_password: MyShopPass456!
```

執行時需要密碼：
```bash
ansible-playbook -i inventory/production.ini site.yml --ask-vault-pass
```

或使用密碼檔案：
```bash
echo "my-vault-password" > .vault_pass
chmod 600 .vault_pass
ansible-playbook -i inventory/production.ini site.yml --vault-password-file .vault_pass
```

### Playbook - 安裝 Java

`playbooks/java.yml`：
```yaml
---
- name: Install Java
  hosts: webservers
  become: yes
  
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600
    
    - name: Install OpenJDK 8
      apt:
        name: openjdk-8-jdk
        state: present
    
    - name: Set JAVA_HOME
      lineinfile:
        path: /etc/environment
        regexp: '^JAVA_HOME='
        line: 'JAVA_HOME={{ java_home }}'
    
    - name: Verify Java installation
      command: java -version
      register: java_version_output
      changed_when: false
    
    - name: Display Java version
      debug:
        msg: "{{ java_version_output.stderr }}"
```

### Playbook - 設定資料庫

`playbooks/database.yml`：
```yaml
---
- name: Setup MySQL database
  hosts: databases
  become: yes
  
  tasks:
    - name: Install MySQL server
      apt:
        name:
          - mysql-server
          - python3-mysqldb
        state: present
    
    - name: Start MySQL service
      service:
        name: mysql
        state: started
        enabled: yes
    
    - name: Set MySQL root password
      mysql_user:
        name: root
        password: "{{ mysql_root_password }}"
        login_unix_socket: /var/run/mysqld/mysqld.sock
        state: present
    
    - name: Create application database
      mysql_db:
        name: "{{ mysql_database }}"
        state: present
        login_user: root
        login_password: "{{ mysql_root_password }}"
    
    - name: Create application user
      mysql_user:
        name: "{{ mysql_user }}"
        password: "{{ mysql_password }}"
        priv: "{{ mysql_database }}.*:ALL"
        state: present
        login_user: root
        login_password: "{{ mysql_root_password }}"
    
    - name: Configure MySQL to listen on all interfaces
      lineinfile:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: '^bind-address'
        line: 'bind-address = 0.0.0.0'
      notify: Restart MySQL
  
  handlers:
    - name: Restart MySQL
      service:
        name: mysql
        state: restarted
```

### Playbook - 部署應用

`playbooks/deploy.yml`：
```yaml
---
- name: Deploy Spring Boot application
  hosts: webservers
  become: yes
  
  tasks:
    - name: Create application group
      group:
        name: "{{ app_group }}"
        state: present
    
    - name: Create application user
      user:
        name: "{{ app_user }}"
        group: "{{ app_group }}"
        system: yes
        shell: /bin/false
        home: "{{ app_home }}"
        createhome: no
    
    - name: Create application directory
      file:
        path: "{{ app_home }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: '0755'
    
    - name: Copy application JAR
      copy:
        src: "../../target/{{ app_jar }}"
        dest: "{{ app_home }}/{{ app_jar }}"
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: '0644'
      notify: Restart application
    
    - name: Copy application configuration
      template:
        src: templates/application.properties.j2
        dest: "{{ app_home }}/application.properties"
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: '0600'
      notify: Restart application
    
    - name: Create systemd service file
      template:
        src: templates/app.service.j2
        dest: "/etc/systemd/system/{{ app_name }}.service"
        mode: '0644'
      notify:
        - Reload systemd
        - Restart application
    
    - name: Enable and start application service
      service:
        name: "{{ app_name }}"
        state: started
        enabled: yes
  
  handlers:
    - name: Reload systemd
      systemd:
        daemon_reload: yes
    
    - name: Restart application
      service:
        name: "{{ app_name }}"
        state: restarted
```

### Template - application.properties

`templates/application.properties.j2`：
```properties
# Server
server.port={{ app_port }}

# Database
spring.datasource.url=jdbc:mysql://{{ groups['databases'][0] }}:3306/{{ mysql_database }}
spring.datasource.username={{ mysql_user }}
spring.datasource.password={{ mysql_password }}
spring.datasource.driver-class-name=com.mysql.jdbc.Driver

# JPA
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect

# Logging
logging.level.root=INFO
logging.level.com.mycompany={{ 'DEBUG' if env == 'dev' else 'INFO' }}
```

### Template - Systemd Service

`templates/app.service.j2`：
```ini
[Unit]
Description={{ app_name }} Spring Boot Application
After=syslog.target network.target

[Service]
User={{ app_user }}
Group={{ app_group }}
WorkingDirectory={{ app_home }}

Environment="JAVA_HOME={{ java_home }}"
Environment="JAVA_OPTS={{ jvm_opts }}"

ExecStart={{ java_home }}/bin/java $JAVA_OPTS -jar {{ app_home }}/{{ app_jar }} --spring.config.location={{ app_home }}/application.properties

StandardOutput=journal
StandardError=journal
SyslogIdentifier={{ app_name }}

Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### 主 Playbook

`site.yml`：
```yaml
---
- import_playbook: playbooks/java.yml
- import_playbook: playbooks/database.yml
- import_playbook: playbooks/deploy.yml
```

執行完整部署：
```bash
ansible-playbook -i inventory/production.ini site.yml --vault-password-file .vault_pass
```

## Ansible Roles

將 Playbook 模組化成 Role。

### 建立 Role 結構

```bash
ansible-galaxy init roles/springboot
```

產生結構：
```
roles/springboot/
├── defaults/
│   └── main.yml          # 預設變數
├── files/                # 靜態檔案
├── handlers/
│   └── main.yml          # Handler
├── meta/
│   └── main.yml          # Role 相依性
├── tasks/
│   └── main.yml          # 主要任務
├── templates/            # Jinja2 模板
├── tests/
│   ├── inventory
│   └── test.yml
└── vars/
    └── main.yml          # Role 變數
```

### 範例 Role

`roles/springboot/tasks/main.yml`：
```yaml
---
- name: Create application user
  user:
    name: "{{ app_user }}"
    group: "{{ app_group }}"
    system: yes
    shell: /bin/false

- name: Create application directory
  file:
    path: "{{ app_home }}"
    state: directory
    owner: "{{ app_user }}"
    group: "{{ app_group }}"

- name: Deploy application
  include_tasks: deploy.yml

- name: Setup systemd service
  include_tasks: service.yml
```

使用 Role：
```yaml
---
- name: Deploy application
  hosts: webservers
  become: yes
  
  roles:
    - java
    - springboot
```

## 整合到 GitLab CI

`.gitlab-ci.yml`：
```yaml
stages:
  - build
  - deploy

variables:
  ANSIBLE_HOST_KEY_CHECKING: "False"

build:
  stage: build
  image: maven:3.5.4-jdk-8
  script:
    - mvn clean package -DskipTests
  artifacts:
    paths:
      - target/*.jar
    expire_in: 1 week

deploy_staging:
  stage: deploy
  image: williamyeh/ansible:2.7.7-alpine
  before_script:
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - echo "$VAULT_PASSWORD" > .vault_pass
  script:
    - ansible-playbook -i inventory/staging.ini site.yml --vault-password-file .vault_pass
  environment:
    name: staging
  only:
    - develop

deploy_production:
  stage: deploy
  image: williamyeh/ansible:2.7.7-alpine
  before_script:
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - echo "$VAULT_PASSWORD" > .vault_pass
  script:
    - ansible-playbook -i inventory/production.ini site.yml --vault-password-file .vault_pass
  environment:
    name: production
  when: manual
  only:
    - master
```

## 常用模組

### 檔案操作

```yaml
# 複製檔案
- name: Copy file
  copy:
    src: source.txt
    dest: /tmp/dest.txt

# 使用模板
- name: Deploy config
  template:
    src: config.j2
    dest: /etc/app/config.conf

# 建立目錄
- name: Create directory
  file:
    path: /opt/app
    state: directory
    owner: appuser
    mode: '0755'

# 建立軟連結
- name: Create symlink
  file:
    src: /opt/app/current
    dest: /opt/app/releases/v1.0
    state: link
```

### 套件管理

```yaml
# APT
- name: Install packages
  apt:
    name:
      - nginx
      - redis
      - postgresql
    state: present
    update_cache: yes

# YUM
- name: Install package
  yum:
    name: httpd
    state: latest
```

### 服務管理

```yaml
- name: Start service
  service:
    name: nginx
    state: started
    enabled: yes

- name: Restart service
  service:
    name: myapp
    state: restarted

- name: Reload systemd
  systemd:
    daemon_reload: yes
```

### 命令執行

```yaml
# Shell 指令
- name: Run shell command
  shell: |
    cd /opt/app
    ./script.sh
  args:
    creates: /opt/app/initialized

# Command（更安全）
- name: Run command
  command: ls -la /tmp
  register: result
  changed_when: false

- name: Display result
  debug:
    var: result.stdout_lines
```

## 遇到的問題

### 問題一：SSH 連線失敗

執行 playbook 時無法連線到目標機器。

原因：
1. SSH key 權限不對
2. 目標機器防火牆封鎖
3. `known_hosts` 驗證失敗

解決方法：
```bash
# 檢查 SSH 連線
ansible all -i inventory.ini -m ping -vvv

# 停用 host key checking（僅測試用）
export ANSIBLE_HOST_KEY_CHECKING=False

# 或在 ansible.cfg
[defaults]
host_key_checking = False
```

### 問題二：權限不足

執行需要 sudo 的操作失敗。

解決方法：
```yaml
- name: Install package
  apt:
    name: nginx
  become: yes      # 使用 sudo

# 或在 play 層級
- name: Setup server
  hosts: webservers
  become: yes      # 整個 play 都用 sudo
```

### 問題三：Idempotence 問題

某些任務重複執行會改變系統狀態。

例如：
```yaml
# 不好的寫法
- name: Add line to file
  shell: echo "something" >> /etc/config
```

每次執行都會新增一行。

改用冪等的模組：
```yaml
# 好的寫法
- name: Add line to file
  lineinfile:
    path: /etc/config
    line: "something"
    state: present
```

### 問題四：執行太慢

50 台機器跑一個 playbook 要很久。

解決方法：
1. 增加並行數
```ini
# ansible.cfg
[defaults]
forks = 20     # 預設 5
```

2. 使用 `strategy`
```yaml
- name: Fast deployment
  hosts: all
  strategy: free   # 不等其他機器，各自跑完就結束
  tasks:
    - ...
```

3. 使用 `async`
```yaml
- name: Long running task
  shell: /opt/long_script.sh
  async: 300        # 最多 5 分鐘
  poll: 0           # Fire and forget

- name: Check result later
  async_status:
    jid: "{{ async_result.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 30
```

## 心得

Ansible 真的大幅提升了我們的效率。以前部署一次要花 2 小時，現在只要 10 分鐘，而且絕對不會漏步驟。

最棒的是冪等性！我可以放心重複執行 playbook，不用擔心搞壞環境。以前手動操作時，總是提心吊膽，深怕重複執行某個指令導致問題。

現在我們的流程：
1. GitLab CI 建置 JAR
2. Ansible 自動部署到 Staging
3. 測試通過後，手動觸發部署到 Production
4. 全程 10 分鐘

整個團隊都輕鬆多了。

搭配上週學的 Terraform：
- **Terraform**：管理基礎設施（VPC、EC2、RDS）
- **Ansible**：管理機器上的軟體和設定

兩個工具各司其職，完美！

下週要研究 Kubernetes 的自動擴展功能（HPA），讓應用能自動應對流量變化。
