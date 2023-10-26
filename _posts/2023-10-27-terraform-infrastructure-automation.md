---
layout: post
title: "Terraform 基礎設施自動化與可觀測性"
date: 2023-10-27 09:00:00 +0800
categories: [Observability, Infrastructure as Code]
tags: [Terraform, IaC, Cloud, Automation]
---

Ansible 適合配置管理，但如果要在雲端（AWS、Azure、GCP）部署資源，**Terraform** 更適合。

今天我們來談談如何用 Terraform 自動化可觀測性基礎設施。

## Terraform 簡介

Terraform 是 HashiCorp 推出的 Infrastructure as Code 工具，特點：
- **宣告式**：描述「想要的狀態」
- **多雲支援**：支援 AWS、Azure、GCP、Kubernetes 等
- **狀態管理**：追蹤實際資源與配置的差異

## 安裝 Terraform

```bash
brew install terraform  # macOS
# 或
wget https://releases.hashicorp.com/terraform/1.5.7/terraform_1.5.7_linux_amd64.zip
unzip terraform_1.5.7_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```

## 第一個 Terraform 配置

### main.tf

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "prometheus" {
  ami           = "ami-0c55b159cbfafe1f0"  # Ubuntu 20.04
  instance_type = "t3.medium"
  
  tags = {
    Name = "prometheus-server"
  }
  
  user_data = <<-EOF
              #!/bin/bash
              wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
              tar xvfz prometheus-*.tar.gz
              cd prometheus-*
              ./prometheus &
              EOF
}

output "prometheus_ip" {
  value = aws_instance.prometheus.public_ip
}
```

### 執行

```bash
terraform init      # 初始化
terraform plan      # 預覽變更
terraform apply     # 應用變更
```

## 完整的可觀測性架構

### 目錄結構

```
observability-infra/
├── main.tf
├── variables.tf
├── outputs.tf
├── modules/
│   ├── prometheus/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── grafana/
│   └── jaeger/
└── environments/
    ├── staging/
    │   └── terraform.tfvars
    └── production/
        └── terraform.tfvars
```

### modules/prometheus/main.tf

```hcl
resource "aws_instance" "prometheus" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id
  
  vpc_security_group_ids = [aws_security_group.prometheus.id]
  
  user_data = templatefile("${path.module}/user_data.sh", {
    prometheus_version = var.prometheus_version
  })
  
  tags = {
    Name        = "prometheus-${var.environment}"
    Environment = var.environment
    Service     = "observability"
  }
}

resource "aws_security_group" "prometheus" {
  name        = "prometheus-${var.environment}"
  description = "Allow Prometheus traffic"
  
  ingress {
    from_port   = 9090
    to_port     = 9090
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_ebs_volume" "prometheus_data" {
  availability_zone = aws_instance.prometheus.availability_zone
  size              = var.storage_size
  type              = "gp3"
  
  tags = {
    Name = "prometheus-data-${var.environment}"
  }
}

resource "aws_volume_attachment" "prometheus_data" {
  device_name = "/dev/sdf"
  volume_id   = aws_ebs_volume.prometheus_data.id
  instance_id = aws_instance.prometheus.id
}
```

### modules/prometheus/variables.tf

```hcl
variable "environment" {
  description = "Environment name"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.medium"
}

variable "ami_id" {
  description = "AMI ID"
  type        = string
}

variable "subnet_id" {
  description = "Subnet ID"
  type        = string
}

variable "prometheus_version" {
  description = "Prometheus version"
  type        = string
  default     = "2.45.0"
}

variable "storage_size" {
  description = "Storage size in GB"
  type        = number
  default     = 100
}
```

### main.tf (root)

```hcl
module "prometheus" {
  source = "./modules/prometheus"
  
  environment        = var.environment
  ami_id             = var.ami_id
  subnet_id          = var.subnet_id
  instance_type      = var.prometheus_instance_type
  prometheus_version = var.prometheus_version
  storage_size       = var.prometheus_storage_size
}

module "grafana" {
  source = "./modules/grafana"
  
  environment   = var.environment
  ami_id        = var.ami_id
  subnet_id     = var.subnet_id
  instance_type = var.grafana_instance_type
  
  prometheus_url = "http://${module.prometheus.private_ip}:9090"
}

module "jaeger" {
  source = "./modules/jaeger"
  
  environment   = var.environment
  ami_id        = var.ami_id
  subnet_id     = var.subnet_id
  instance_type = var.jaeger_instance_type
}
```

### environments/production/terraform.tfvars

```hcl
environment = "production"
ami_id      = "ami-0c55b159cbfafe1f0"
subnet_id   = "subnet-12345678"

prometheus_instance_type  = "t3.large"
prometheus_version        = "2.45.0"
prometheus_storage_size   = 500

grafana_instance_type     = "t3.medium"

jaeger_instance_type      = "t3.medium"
```

## Kubernetes 資源管理

### main.tf

```hcl
provider "kubernetes" {
  config_path = "~/.kube/config"
}

resource "kubernetes_namespace" "observability" {
  metadata {
    name = "observability"
  }
}

resource "kubernetes_deployment" "prometheus" {
  metadata {
    name      = "prometheus"
    namespace = kubernetes_namespace.observability.metadata[0].name
  }
  
  spec {
    replicas = 1
    
    selector {
      match_labels = {
        app = "prometheus"
      }
    }
    
    template {
      metadata {
        labels = {
          app = "prometheus"
        }
      }
      
      spec {
        container {
          name  = "prometheus"
          image = "prom/prometheus:v2.45.0"
          
          port {
            container_port = 9090
          }
          
          volume_mount {
            name       = "config"
            mount_path = "/etc/prometheus"
          }
          
          volume_mount {
            name       = "data"
            mount_path = "/prometheus"
          }
        }
        
        volume {
          name = "config"
          config_map {
            name = kubernetes_config_map.prometheus_config.metadata[0].name
          }
        }
        
        volume {
          name = "data"
          persistent_volume_claim {
            claim_name = kubernetes_persistent_volume_claim.prometheus_data.metadata[0].name
          }
        }
      }
    }
  }
}

resource "kubernetes_config_map" "prometheus_config" {
  metadata {
    name      = "prometheus-config"
    namespace = kubernetes_namespace.observability.metadata[0].name
  }
  
  data = {
    "prometheus.yml" = file("${path.module}/prometheus.yml")
  }
}

resource "kubernetes_persistent_volume_claim" "prometheus_data" {
  metadata {
    name      = "prometheus-data"
    namespace = kubernetes_namespace.observability.metadata[0].name
  }
  
  spec {
    access_modes = ["ReadWriteOnce"]
    resources {
      requests = {
        storage = "100Gi"
      }
    }
  }
}

resource "kubernetes_service" "prometheus" {
  metadata {
    name      = "prometheus"
    namespace = kubernetes_namespace.observability.metadata[0].name
  }
  
  spec {
    selector = {
      app = "prometheus"
    }
    
    port {
      port        = 9090
      target_port = 9090
    }
    
    type = "LoadBalancer"
  }
}
```

## Grafana Provisioning

### 自動建立 Dashboard

```hcl
resource "grafana_dashboard" "node_exporter" {
  config_json = file("${path.module}/dashboards/node-exporter.json")
}

resource "grafana_dashboard" "jvm" {
  config_json = file("${path.module}/dashboards/jvm.json")
}

resource "grafana_data_source" "prometheus" {
  type = "prometheus"
  name = "Prometheus"
  url  = "http://${module.prometheus.private_ip}:9090"
  
  json_data {
    http_method = "GET"
  }
}

resource "grafana_folder" "observability" {
  title = "Observability"
}

resource "grafana_alert_notification" "slack" {
  name = "Slack"
  type = "slack"
  
  settings = {
    url = var.slack_webhook_url
  }
}
```

## 狀態管理

### 本地狀態（不建議用於團隊）

```bash
terraform apply  # 狀態儲存在 terraform.tfstate
```

### 遠端狀態（建議）

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "observability/terraform.tfstate"
    region = "us-east-1"
    
    dynamodb_table = "terraform-locks"  # 用於鎖定
    encrypt        = true
  }
}
```

## Workspace（多環境）

```bash
# 建立 workspace
terraform workspace new staging
terraform workspace new production

# 切換 workspace
terraform workspace select production

# 查看當前 workspace
terraform workspace show
```

```hcl
locals {
  environment = terraform.workspace
  
  instance_type = {
    staging    = "t3.small"
    production = "t3.large"
  }
}

resource "aws_instance" "prometheus" {
  instance_type = local.instance_type[local.environment]
}
```

## CI/CD 整合

### GitLab CI

```yaml
stages:
  - validate
  - plan
  - apply

validate:
  stage: validate
  script:
    - terraform init -backend=false
    - terraform fmt -check
    - terraform validate

plan-staging:
  stage: plan
  script:
    - terraform init
    - terraform workspace select staging
    - terraform plan -out=staging.tfplan
  artifacts:
    paths:
      - staging.tfplan
  only:
    - develop

apply-staging:
  stage: apply
  script:
    - terraform init
    - terraform workspace select staging
    - terraform apply staging.tfplan
  dependencies:
    - plan-staging
  only:
    - develop

plan-production:
  stage: plan
  script:
    - terraform init
    - terraform workspace select production
    - terraform plan -out=production.tfplan
  artifacts:
    paths:
      - production.tfplan
  only:
    - main

apply-production:
  stage: apply
  script:
    - terraform init
    - terraform workspace select production
    - terraform apply production.tfplan
  dependencies:
    - plan-production
  only:
    - main
  when: manual  # 需要手動核准
```

## 實戰建議

### 1. 使用 Modules 重用程式碼

```hcl
module "prometheus_staging" {
  source = "./modules/prometheus"
  environment = "staging"
}

module "prometheus_production" {
  source = "./modules/prometheus"
  environment = "production"
}
```

### 2. 使用 Variables 參數化配置

```hcl
variable "environment" {
  type = string
}

variable "instance_type" {
  type    = string
  default = "t3.medium"
}
```

### 3. 使用 Data Sources 查詢現有資源

```hcl
data "aws_vpc" "default" {
  default = true
}

resource "aws_security_group" "prometheus" {
  vpc_id = data.aws_vpc.default.id
}
```

---

**Terraform 可以讓你用程式碼管理整個可觀測性基礎設施。**

當你把基礎設施程式碼化，你就能版本控制、自動測試、自動部署，大幅提升可靠性和一致性。
