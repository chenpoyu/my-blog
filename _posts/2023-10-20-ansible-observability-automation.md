---
layout: post
title: "Ansible 自動化可觀測性部署"
date: 2023-10-20 09:00:00 +0800
categories: [Observability, Infrastructure as Code]
tags: [Ansible, Automation, IaC, Configuration Management]
---

手動部署 Prometheus、Grafana、Jaeger 很簡單，但要在 100 台機器上部署呢？

今天我們來談談如何用 **Ansible** 自動化可觀測性的部署。

## Ansible 簡介

Ansible 是一個配置管理工具，特點：
- **無 Agent**：不需要在目標機器上安裝任何東西
- **宣告式**：描述「想要的狀態」，而不是「如何做」
- **冪等性**：執行多次結果相同

## 安裝 Ansible

```bash
pip install ansible
```

## 第一個 Playbook：部署 Node Exporter

### hosts 檔案

```ini
[servers]
server1 ansible_host=192.168.1.10 ansible_user=ubuntu
server2 ansible_host=192.168.1.11 ansible_user=ubuntu
server3 ansible_host=192.168.1.12 ansible_user=ubuntu
```

### Playbook

```yaml
# deploy-node-exporter.yml
---
- name: Deploy Node Exporter
  hosts: servers
  become: yes
  
  vars:
    node_exporter_version: "1.6.1"
  
  tasks:
    - name: Download Node Exporter
      get_url:
        url: "https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz"
        dest: /tmp/node_exporter.tar.gz
    
    - name: Extract Node Exporter
      unarchive:
        src: /tmp/node_exporter.tar.gz
        dest: /usr/local/bin
        remote_src: yes
        extra_opts: [--strip-components=1]
    
    - name: Create systemd service
      copy:
        content: |
          [Unit]
          Description=Node Exporter
          After=network.target
          
          [Service]
          Type=simple
          ExecStart=/usr/local/bin/node_exporter
          Restart=always
          
          [Install]
          WantedBy=multi-user.target
        dest: /etc/systemd/system/node_exporter.service
    
    - name: Start Node Exporter
      systemd:
        name: node_exporter
        state: started
        enabled: yes
        daemon_reload: yes
```

### 執行

```bash
ansible-playbook -i hosts deploy-node-exporter.yml
```

**結果**：在 3 台機器上同時部署 Node Exporter。

## 部署完整的可觀測性架構

### 目錄結構

```
observability/
├── hosts
├── group_vars/
│   ├── all.yml
│   └── prometheus.yml
├── roles/
│   ├── prometheus/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   │   └── prometheus.yml.j2
│   │   └── handlers/
│   │       └── main.yml
│   ├── grafana/
│   ├── node_exporter/
│   └── jaeger/
└── site.yml
```

### site.yml

```yaml
---
- name: Deploy Observability Stack
  hosts: all
  roles:
    - node_exporter

- name: Deploy Prometheus
  hosts: prometheus_servers
  roles:
    - prometheus

- name: Deploy Grafana
  hosts: grafana_servers
  roles:
    - grafana

- name: Deploy Jaeger
  hosts: jaeger_servers
  roles:
    - jaeger
```

### roles/prometheus/tasks/main.yml

```yaml
---
- name: Download Prometheus
  get_url:
    url: "https://github.com/prometheus/prometheus/releases/download/v{{ prometheus_version }}/prometheus-{{ prometheus_version }}.linux-amd64.tar.gz"
    dest: /tmp/prometheus.tar.gz

- name: Extract Prometheus
  unarchive:
    src: /tmp/prometheus.tar.gz
    dest: /opt
    remote_src: yes

- name: Create Prometheus config
  template:
    src: prometheus.yml.j2
    dest: /opt/prometheus/prometheus.yml
  notify: restart prometheus

- name: Create systemd service
  copy:
    content: |
      [Unit]
      Description=Prometheus
      After=network.target
      
      [Service]
      Type=simple
      ExecStart=/opt/prometheus/prometheus \
        --config.file=/opt/prometheus/prometheus.yml \
        --storage.tsdb.path=/opt/prometheus/data
      Restart=always
      
      [Install]
      WantedBy=multi-user.target
    dest: /etc/systemd/system/prometheus.service

- name: Start Prometheus
  systemd:
    name: prometheus
    state: started
    enabled: yes
    daemon_reload: yes
```

### roles/prometheus/templates/prometheus.yml.j2

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  
  - job_name: 'node_exporter'
    static_configs:
      - targets:
{% for host in groups['servers'] %}
          - '{{ hostvars[host].ansible_host }}:9100'
{% endfor %}
```

### roles/prometheus/handlers/main.yml

```yaml
---
- name: restart prometheus
  systemd:
    name: prometheus
    state: restarted
```

## 動態生成配置

### 從 Consul 動態生成 Prometheus 配置

```yaml
# roles/prometheus/tasks/main.yml
- name: Get services from Consul
  uri:
    url: "http://consul:8500/v1/catalog/services"
    return_content: yes
  register: consul_services

- name: Generate Prometheus config
  template:
    src: prometheus.yml.j2
    dest: /opt/prometheus/prometheus.yml
  notify: restart prometheus
```

```yaml
# roles/prometheus/templates/prometheus.yml.j2
scrape_configs:
{% for service in consul_services.json.keys() %}
  - job_name: '{{ service }}'
    consul_sd_configs:
      - server: 'consul:8500'
        services: ['{{ service }}']
{% endfor %}
```

## Grafana 自動化

### 自動建立 Data Source

```yaml
# roles/grafana/tasks/main.yml
- name: Create Prometheus data source
  uri:
    url: "http://{{ grafana_host }}:3000/api/datasources"
    method: POST
    user: admin
    password: admin
    force_basic_auth: yes
    body_format: json
    body:
      name: "Prometheus"
      type: "prometheus"
      url: "http://{{ prometheus_host }}:9090"
      access: "proxy"
      isDefault: true
    status_code: [200, 409]  # 409 = already exists
```

### 自動匯入 Dashboard

```yaml
- name: Import Dashboards
  uri:
    url: "http://{{ grafana_host }}:3000/api/dashboards/import"
    method: POST
    user: admin
    password: admin
    force_basic_auth: yes
    body_format: json
    body:
      dashboard: "{{ lookup('file', 'dashboards/node-exporter.json') | from_json }}"
      overwrite: true
    status_code: [200, 412]  # 412 = already exists
```

## 實戰：一鍵部署完整環境

### 執行

```bash
ansible-playbook -i hosts site.yml
```

**結果**：
- 在所有機器上部署 Node Exporter
- 在 prometheus_servers 上部署 Prometheus
- 在 grafana_servers 上部署 Grafana
- 在 jaeger_servers 上部署 Jaeger
- 自動配置 Prometheus 抓取所有 Node Exporter
- 自動在 Grafana 中建立 Data Source 和 Dashboard

### 驗證

```bash
# 檢查 Node Exporter
curl http://192.168.1.10:9100/metrics

# 檢查 Prometheus
curl http://192.168.1.20:9090/-/healthy

# 檢查 Grafana
curl http://192.168.1.30:3000/api/health
```

## 版本控制與 GitOps

### 目錄結構

```
observability-infra/
├── ansible/
│   ├── hosts
│   ├── site.yml
│   └── roles/
├── .gitlab-ci.yml
└── README.md
```

### .gitlab-ci.yml

```yaml
stages:
  - validate
  - deploy

validate:
  stage: validate
  script:
    - ansible-playbook --syntax-check site.yml
    - ansible-lint site.yml

deploy-staging:
  stage: deploy
  script:
    - ansible-playbook -i hosts-staging site.yml
  only:
    - develop

deploy-production:
  stage: deploy
  script:
    - ansible-playbook -i hosts-production site.yml
  only:
    - main
  when: manual
```

## 實戰建議

### 1. 使用 Ansible Vault 加密敏感資訊

```bash
ansible-vault encrypt group_vars/all.yml
```

```yaml
# group_vars/all.yml
grafana_admin_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ...
```

### 2. 使用 Tags 選擇性執行

```yaml
- name: Deploy Prometheus
  hosts: prometheus_servers
  roles:
    - role: prometheus
      tags: [prometheus]
```

```bash
# 只部署 Prometheus
ansible-playbook -i hosts site.yml --tags prometheus
```

### 3. 使用 Check Mode 預覽變更

```bash
ansible-playbook -i hosts site.yml --check --diff
```

---

**Ansible 可以讓你在幾分鐘內，在數百台機器上部署完整的可觀測性架構。**

當你能用程式碼管理基礎設施，你就能確保一致性、可重複性、可追溯性。
