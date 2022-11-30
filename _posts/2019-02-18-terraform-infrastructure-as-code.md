---
layout: post
title: "Terraform Infrastructure as Code 基礎"
date: 2019-02-18 10:00:00 +0800
categories: [DevOps, Infrastructure]
tags: [Terraform, IaC, Infrastructure as Code, AWS, Cloud]
---

上週研究了容器安全掃描（參考 [容器安全掃描與漏洞管理](/posts/2019/02/11/container-security-vulnerability-scanning/)），這週來解決另一個痛點：基礎設施管理。

目前我們的 Kubernetes 叢集是手動建立的：
1. 登入 AWS Console
2. 點點點建立 EC2 instances
3. 手動安裝 Kubernetes
4. 手動設定網路、安全群組
5. 寫一堆文件記錄步驟

問題：
- **無法重現**：換個人操作可能漏掉某些設定
- **沒有版本控制**：不知道誰改了什麼
- **災難恢復困難**：叢集掛了要花很久重建
- **環境不一致**：Dev、Staging、Prod 設定有微妙差異

這週研究 Terraform，用程式碼管理基礎設施。

> 使用版本：Terraform 0.11.11（2019 年初的穩定版）

## Infrastructure as Code 的價值

把基礎設施寫成程式碼：

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "WebServer"
  }
}
```

好處：
1. **版本控制**：放進 Git，知道誰改了什麼
2. **可重現**：執行 `terraform apply` 就能建立一模一樣的環境
3. **文件即程式碼**：不用寫安裝文件，看程式碼就知道
4. **審查機制**：透過 Pull Request review 基礎設施變更
5. **災難恢復**：快速重建整個環境

## Terraform 是什麼

Terraform 是 HashiCorp 開發的 IaC 工具。

特色：
- **多雲支援**：AWS、Azure、GCP、Kubernetes 等
- **宣告式語法**：描述「想要什麼」，不是「怎麼做」
- **執行計畫**：改變前先預覽會發生什麼
- **狀態管理**：追蹤實際資源狀態
- **模組化**：可重複使用的配置

## 安裝 Terraform

```bash
# macOS
brew install terraform@0.11

# Linux
wget https://releases.hashicorp.com/terraform/0.11.11/terraform_0.11.11_linux_amd64.zip
unzip terraform_0.11.11_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# 驗證
terraform version
# Terraform v0.11.11
```

## 第一個 Terraform 配置

### 設定 AWS 憑證

```bash
# 安裝 AWS CLI
brew install awscli

# 設定憑證
aws configure
# AWS Access Key ID: YOUR_ACCESS_KEY
# AWS Secret Access Key: YOUR_SECRET_KEY
# Default region name: ap-northeast-1
# Default output format: json
```

### 建立 EC2 Instance

建立 `main.tf`：

```hcl
# 指定 provider
provider "aws" {
  region = "ap-northeast-1"
}

# 建立 EC2 instance
resource "aws_instance" "web" {
  ami           = "ami-0c3fd0f5d33134a76"  # Amazon Linux 2
  instance_type = "t2.micro"
  
  tags = {
    Name = "MyFirstTerraformInstance"
  }
}
```

### 初始化 Terraform

```bash
terraform init

# Initializing provider plugins...
# - Checking for available provider plugins on https://releases.hashicorp.com...
# - Downloading plugin for provider "aws" (2.2.0)...
# 
# Terraform has been successfully initialized!
```

這會下載 AWS provider plugin。

### 執行計畫

```bash
terraform plan

# Terraform will perform the following actions:
# 
#   + aws_instance.web
#       ami:                      "ami-0c3fd0f5d33134a76"
#       instance_type:            "t2.micro"
#       tags.%:                   "1"
#       tags.Name:                "MyFirstTerraformInstance"
# 
# Plan: 1 to add, 0 to change, 0 to destroy.
```

`+` 表示會建立新資源。

### 套用變更

```bash
terraform apply

# ... (顯示計畫)
# 
# Do you want to perform these actions?
#   Terraform will perform the actions described above.
#   Only 'yes' will be accepted to approve.
# 
#   Enter a value: yes
# 
# aws_instance.web: Creating...
# aws_instance.web: Still creating... (10s elapsed)
# aws_instance.web: Still creating... (20s elapsed)
# aws_instance.web: Creation complete after 25s (ID: i-0abc123def456789)
# 
# Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

去 AWS Console 看，EC2 instance 真的建立了！

### 查看狀態

```bash
terraform show

# aws_instance.web:
#   id = i-0abc123def456789
#   ami = ami-0c3fd0f5d33134a76
#   instance_type = t2.micro
#   ...
```

### 銷毀資源

```bash
terraform destroy

# ... (顯示計畫)
# 
# Do you really want to destroy all resources?
#   Enter a value: yes
# 
# aws_instance.web: Destroying... (ID: i-0abc123def456789)
# aws_instance.web: Destruction complete after 30s
# 
# Destroy complete! Resources: 1 destroyed.
```

## Terraform 語法基礎

### Resource

定義要建立的資源：

```hcl
resource "resource_type" "resource_name" {
  argument1 = "value1"
  argument2 = "value2"
}
```

### Variable

定義變數：

`variables.tf`：
```hcl
variable "instance_type" {
  description = "EC2 instance type"
  default     = "t2.micro"
}

variable "region" {
  description = "AWS region"
  default     = "ap-northeast-1"
}
```

使用變數：

`main.tf`：
```hcl
provider "aws" {
  region = var.region
}

resource "aws_instance" "web" {
  ami           = "ami-0c3fd0f5d33134a76"
  instance_type = var.instance_type
}
```

執行時覆蓋變數：
```bash
terraform apply -var="instance_type=t2.small"
```

或建立 `terraform.tfvars`：
```hcl
instance_type = "t2.small"
region        = "us-west-2"
```

### Output

輸出資源資訊：

`outputs.tf`：
```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.web.public_ip
}
```

執行後會顯示：
```bash
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

instance_id = i-0abc123def456789
public_ip = 13.230.123.45
```

查看 output：
```bash
terraform output
terraform output public_ip
```

### Data Source

查詢現有資源：

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"
}
```

這樣會自動使用最新的 Amazon Linux 2 AMI。

## 實際案例：建立完整的 Web 應用環境

建立包含 VPC、Subnet、Security Group、EC2 的環境。

`main.tf`：
```hcl
provider "aws" {
  region = var.region
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "${var.project_name}-igw"
  }
}

# Public Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "${var.region}a"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.project_name}-public-subnet"
  }
}

# Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Security Group
resource "aws_security_group" "web" {
  name        = "${var.project_name}-web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  # SSH
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  # HTTP
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  # HTTPS
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  # Outbound
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${var.project_name}-web-sg"
  }
}

# EC2 Instance
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public.id
  
  vpc_security_group_ids = [aws_security_group.web.id]
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y docker
              service docker start
              usermod -a -G docker ec2-user
              docker run -d -p 80:80 nginx
              EOF
  
  tags = {
    Name = "${var.project_name}-web-${count.index + 1}"
  }
}

# Data source for AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

`variables.tf`：
```hcl
variable "region" {
  default = "ap-northeast-1"
}

variable "project_name" {
  default = "myshop"
}

variable "instance_type" {
  default = "t2.micro"
}

variable "instance_count" {
  default = 2
}
```

`outputs.tf`：
```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "instance_ids" {
  value = aws_instance.web[*].id
}

output "instance_public_ips" {
  value = aws_instance.web[*].public_ip
}
```

執行：
```bash
terraform init
terraform plan
terraform apply
```

Terraform 會：
1. 建立 VPC
2. 建立 Internet Gateway
3. 建立 Subnet
4. 建立 Route Table 並關聯
5. 建立 Security Group
6. 建立 2 個 EC2 instances
7. 在 instances 上安裝 Docker 並執行 Nginx

全自動！

## 狀態管理

Terraform 會在本地產生 `terraform.tfstate` 檔案，記錄資源狀態。

### 遠端狀態儲存

團隊協作時，狀態應該存在遠端（S3）。

先建立 S3 bucket 和 DynamoDB table（用於鎖定）：

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "myshop/terraform.tfstate"
    region         = "ap-northeast-1"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}
```

初始化：
```bash
terraform init
# Terraform will now migrate your state to the S3 backend.
```

現在狀態存在 S3，多人協作不會衝突。

## Terraform 模組

模組讓配置可重複使用。

### 建立模組

`modules/web-server/main.tf`：
```hcl
variable "instance_name" {}
variable "instance_type" { default = "t2.micro" }
variable "subnet_id" {}
variable "security_group_ids" { type = list }

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_group_ids
  
  tags = {
    Name = var.instance_name
  }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

output "instance_id" {
  value = aws_instance.web.id
}

output "public_ip" {
  value = aws_instance.web.public_ip
}
```

### 使用模組

`main.tf`：
```hcl
module "web_server_1" {
  source = "./modules/web-server"
  
  instance_name       = "web-1"
  instance_type       = "t2.micro"
  subnet_id           = aws_subnet.public.id
  security_group_ids  = [aws_security_group.web.id]
}

module "web_server_2" {
  source = "./modules/web-server"
  
  instance_name       = "web-2"
  instance_type       = "t2.small"
  subnet_id           = aws_subnet.public.id
  security_group_ids  = [aws_security_group.web.id]
}

output "web1_ip" {
  value = module.web_server_1.public_ip
}

output "web2_ip" {
  value = module.web_server_2.public_ip
}
```

## 整合到 CI/CD

在 GitLab CI 中自動執行 Terraform：

`.gitlab-ci.yml`：
```yaml
image:
  name: hashicorp/terraform:0.11.11
  entrypoint: [""]

stages:
  - validate
  - plan
  - apply

variables:
  TF_ROOT: ${CI_PROJECT_DIR}/terraform
  TF_IN_AUTOMATION: "true"

before_script:
  - cd ${TF_ROOT}
  - terraform init

validate:
  stage: validate
  script:
    - terraform validate

plan:
  stage: plan
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 week

apply:
  stage: apply
  script:
    - terraform apply -auto-approve tfplan
  dependencies:
    - plan
  when: manual
  only:
    - master
```

## 遇到的問題

### 問題一：State 衝突

兩個人同時執行 `terraform apply`，狀態衝突。

解決方法：使用 S3 backend with DynamoDB locking。

### 問題二：不小心刪除重要資源

執行 `terraform destroy` 時不小心確認，整個環境都刪了。

解決方法：
1. 使用 `prevent_destroy` lifecycle
```hcl
resource "aws_instance" "critical" {
  # ...
  
  lifecycle {
    prevent_destroy = true
  }
}
```

2. 使用不同的 workspace
```bash
terraform workspace new prod
terraform workspace new staging
terraform workspace select prod
```

3. 設定 IAM 權限，限制誰能刪除

### 問題三：狀態漂移

手動在 AWS Console 改了設定，Terraform 不知道。

解決方法：
```bash
# 檢查差異
terraform plan

# 更新狀態
terraform refresh

# 或套用 Terraform 配置覆蓋手動變更
terraform apply
```

最好的做法：**所有變更都透過 Terraform，禁止手動修改**。

## 心得

Terraform 真的改變了我們管理基礎設施的方式。以前建環境要花一整天，現在 `terraform apply` 十分鐘搞定。

而且最棒的是，現在基礎設施的變更也可以 code review 了！同事要改網路設定，提 MR，我可以看 diff：
```diff
- cidr_block = "10.0.1.0/24"
+ cidr_block = "10.0.2.0/24"
```

一目了然，不會有人偷偷改設定導致問題。

災難恢復也變簡單了。上週測試環境掛了，以前要花半天重建，現在：
```bash
terraform destroy
terraform apply
```

十分鐘重建完成！

下週要研究 Ansible，用來自動化配置管理。Terraform 管理基礎設施，Ansible 管理機器上的軟體安裝和設定，兩者搭配使用。
