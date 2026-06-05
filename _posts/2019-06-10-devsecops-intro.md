---
layout: post
title: "DevSecOps 入門"
date: 2019-06-10 10:00:00 +0800
categories: [DevOps, Security]
tags: [Docker, Security, DevOps]
---

上週優化了雲端成本（參考 [雲端成本優化](/posts/2019/06/03/cloud-cost-optimization/)），這週來談一個重要但常被忽視的主題：安全。

我們的痛點：安全總是事後才考慮。

上個月的事件：
1. 資安團隊掃描發現 Log4j 漏洞（CVE-2021-44228）
2. 要求所有服務立即更新
3. 我們花了 3 天找出哪些服務用了 Log4j
4. 又花了 2 天測試和部署更新
5. 損失慘重

需要 **DevSecOps**：把安全整合到 DevOps 流程，而不是事後補救。

> 工具：SonarQube、OWASP Dependency-Check、Trivy、Falco

## DevSecOps 是什麼

傳統流程：
```
開發 → 測試 → 部署 → (出事) → 資安掃描 → 修復
```

DevSecOps 流程：
```
開發 → 安全掃描 → 測試 → 安全掃描 → 部署 → 持續監控
     ↑ 每個環節都有安全
```

核心理念：**Shift Left**（安全左移）
- 越早發現問題，修復成本越低
- 開發階段發現：1x 成本
- 測試階段發現：10x 成本
- 生產階段發現：100x 成本

## SAST（靜態應用安全測試）

分析原始碼，找出安全漏洞。

### SonarQube

#### 安裝

```bash
# 使用 Docker
docker run -d --name sonarqube \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:8.2-community
```

開啟 http://localhost:9000（預設帳密：admin/admin）

#### 掃描專案

安裝 Scanner：
```bash
# macOS
brew install sonar-scanner

# 或下載
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.0.0.1744.zip
```

配置專案：

`sonar-project.properties`：
```properties
sonar.projectKey=order-service
sonar.projectName=Order Service
sonar.projectVersion=1.0
sonar.sources=src/main/java
sonar.java.binaries=target/classes
sonar.host.url=http://localhost:9000
sonar.login=your-token
```

執行掃描：
```bash
mvn clean verify sonar:sonar
```

#### 查看結果

Dashboard 顯示：

**Bugs**: 18
- 🔴 Blocker: 2
- 🔴 Critical: 5
- 🟠 Major: 11

**Vulnerabilities**: 7
- 🔴 Critical: 3（SQL Injection）
- 🟠 Major: 4（XSS）

**Code Smells**: 234
- 🔴 Critical: 12
- 🟠 Major: 89
- 🟡 Minor: 133

**Security Hotspots**: 15
- 需要人工審查

#### 修復範例

**SQL Injection**：

掃描發現：
```java
// 危險
public List<Order> findOrders(String userId) {
    String sql = "SELECT * FROM orders WHERE user_id = '" + userId + "'";
    return jdbcTemplate.query(sql, new OrderRowMapper());
}
```

SonarQube 警告：
```
SQL Injection 風險
嚴重性：Critical
建議：使用 PreparedStatement
```

修復：
```java
// 安全
public List<Order> findOrders(String userId) {
    String sql = "SELECT * FROM orders WHERE user_id = ?";
    return jdbcTemplate.query(sql, new OrderRowMapper(), userId);
}
```

**XSS（跨站腳本攻擊）**：

掃描發現：
```java
// 危險
@GetMapping("/search")
public String search(@RequestParam String keyword) {
    return "<h1>Search results for: " + keyword + "</h1>";
}
```

攻擊者輸入：
```
keyword=<script>alert('XSS')</script>
```

修復：
```java
// 安全（自動 escape）
@GetMapping("/search")
public String search(@RequestParam String keyword, Model model) {
    model.addAttribute("keyword", keyword);
    return "search";  // 使用 template engine
}
```

### 整合 CI/CD

`.gitlab-ci.yml`：
```yaml
stages:
  - build
  - security-scan
  - test
  - deploy

sonarqube_scan:
  stage: security-scan
  image: maven:3.6-jdk-11
  script:
    - mvn clean verify sonar:sonar
      -Dsonar.host.url=$SONAR_URL
      -Dsonar.login=$SONAR_TOKEN
  only:
    - merge_requests
    - master

quality_gate:
  stage: security-scan
  script:
    - |
      STATUS=$(curl -u $SONAR_TOKEN: "$SONAR_URL/api/qualitygates/project_status?projectKey=order-service" | jq -r '.projectStatus.status')
      if [ "$STATUS" != "OK" ]; then
        echo "Quality gate failed: $STATUS"
        exit 1
      fi
  only:
    - merge_requests
    - master
```

Quality Gate 失敗 → CI 失敗 → 不能合併 → 強制修復。

## 依賴套件掃描

檢查第三方套件是否有已知漏洞（CVE）。

### OWASP Dependency-Check

```bash
# 安裝
brew install dependency-check

# 掃描 Maven 專案
dependency-check --project order-service \
  --scan . \
  --format HTML \
  --out ./reports
```

報告範例：
```
=== Dependency-Check Report ===

commons-collections-3.2.1.jar
  CVE-2015-6420 (CVSS: 7.5) - High
  描述：反序列化漏洞，可遠端執行程式碼
  解決：升級到 3.2.2 或 4.0

log4j-core-2.14.0.jar
  CVE-2021-44228 (CVSS: 10.0) - Critical  ← Log4Shell!
  描述：JNDI 注入漏洞
  解決：立即升級到 2.17.0

spring-web-5.2.8.RELEASE.jar
  CVE-2020-5421 (CVSS: 8.6) - High
  描述：RFD 攻擊
  解決：升級到 5.2.9
```

### Maven Plugin

自動化整合：

`pom.xml`：
```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.owasp</groupId>
      <artifactId>dependency-check-maven</artifactId>
      <version>5.3.2</version>
      <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>  <!-- CVSS >= 7 就失敗 -->
        <suppressionFile>suppressions.xml</suppressionFile>
      </configuration>
      <executions>
        <execution>
          <goals>
            <goal>check</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

CI 執行：
```bash
mvn verify  # 會自動執行 dependency-check
```

### Suppression（抑制誤報）

有些 CVE 不適用於我們的使用情境：

`suppressions.xml`：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">
  <suppress>
    <notes>
      這個 CVE 只影響 Tomcat embedded，我們用 standalone Tomcat
    </notes>
    <gav regex="true">^org\.springframework:spring-webmvc:.*$</gav>
    <cve>CVE-2020-5398</cve>
  </suppress>
</suppressions>
```

### 自動更新依賴

使用 Dependabot（GitHub）或 Renovate：

`.github/dependabot.yml`：
```yaml
version: 2
updates:
  - package-ecosystem: "maven"
    directory: "/"
    schedule:
      interval: "weekly"
    reviewers:
      - "security-team"
    labels:
      - "dependencies"
      - "security"
```

每週自動檢查依賴更新，建立 Pull Request。

## 容器安全

### Trivy

掃描 Docker image 的漏洞。

安裝：
```bash
brew install aquasecurity/trivy/trivy
```

掃描 image：
```bash
trivy image order-service:latest
```

報告：
```
order-service:latest (alpine 3.11.6)
===================================

Total: 58 (UNKNOWN: 0, LOW: 15, MEDIUM: 28, HIGH: 12, CRITICAL: 3)

+---------+------------------+----------+-------------------+---------------+
| LIBRARY | VULNERABILITY ID | SEVERITY | INSTALLED VERSION | FIXED VERSION |
+---------+------------------+----------+-------------------+---------------+
| musl    | CVE-2020-28928   | CRITICAL | 1.1.24-r2         | 1.1.24-r3     |
| openssl | CVE-2021-3449    | HIGH     | 1.1.1g-r0         | 1.1.1k-r0     |
| curl    | CVE-2021-22876   | MEDIUM   | 7.67.0-r0         | 7.67.0-r4     |
+---------+------------------+----------+-------------------+---------------+
```

### 修復

**方法一：更新 base image**

```dockerfile
# 修改前
FROM alpine:3.11.6

# 修改後
FROM alpine:3.13.5  # 更新版本
```

**方法二：減少套件**

```dockerfile
# 修改前（包含很多不需要的套件）
FROM ubuntu:18.04
RUN apt-get update && apt-get install -y \
    curl wget git vim gcc make ...

# 修改後（只裝必要的）
FROM alpine:3.13.5
RUN apk add --no-cache \
    openjdk11-jre-headless  # 只需要 JRE
```

**方法三：Multi-stage build**

```dockerfile
# Build stage
FROM maven:3.6-jdk-11 AS build
COPY src /app/src
COPY pom.xml /app
RUN mvn -f /app/pom.xml clean package

# Runtime stage（更小、更安全）
FROM openjdk:11-jre-slim
COPY --from=build /app/target/order-service.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

Runtime image 只包含 JRE 和應用程式，沒有編譯工具。

### CI/CD 整合

```yaml
# .gitlab-ci.yml
security_scan_image:
  stage: security-scan
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - merge_requests
    - master
```

發現 Critical/High 漏洞 → CI 失敗。

### 運行時安全（Falco）

掃描只能發現已知漏洞，運行時安全監控異常行為。

安裝 Falco（Kubernetes）：
```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace
```

Falco 監控：
- 異常的檔案存取
- 異常的網路連線
- 異常的系統呼叫
- 異常的指令執行

規則範例：

`falco_rules.local.yaml`：
```yaml
# 偵測在容器內執行 shell
- rule: Shell Spawned in Container
  desc: Detect shell spawned inside a container
  condition: >
    container.id != host and
    proc.name in (bash, sh, zsh)
  output: >
    Shell spawned in container (user=%user.name container=%container.name
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline)
  priority: WARNING

# 偵測存取敏感檔案
- rule: Read sensitive file untrusted
  desc: Detect read of sensitive file
  condition: >
    open_read and
    sensitive_files and
    not trusted_containers
  output: >
    Sensitive file read (user=%user.name file=%fd.name container=%container.name)
  priority: WARNING
```

查看告警：
```bash
kubectl logs -n falco -l app=falco

# 輸出
18:42:12.345678901: Warning Shell spawned in container (user=root container=order-service-7d8c9f-x7k2m shell=bash parent=java cmdline=bash -c "curl http://evil.com")
```

有人在容器內執行 shell → 可能被攻擊！

## Secrets 管理

### 問題：敏感資訊洩漏

常見錯誤：
```java
// 硬編碼在程式碼
String dbPassword = "MyP@ssw0rd123";

// 提交到 Git
# application.yml
spring:
  datasource:
    password: MyP@ssw0rd123
```

### 解決方案

**1. 環境變數**：
```yaml
# application.yml
spring:
  datasource:
    password: ${DB_PASSWORD}
```

```bash
# Kubernetes Secret
kubectl create secret generic db-credentials \
  --from-literal=password='MyP@ssw0rd123'

# Deployment
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
```

**2. HashiCorp Vault**：

```java
// 從 Vault 讀取
@Configuration
public class VaultConfig {
    @Bean
    public DataSource dataSource(VaultTemplate vaultTemplate) {
        VaultResponse response = vaultTemplate
            .read("secret/data/database");
        
        String password = (String) response.getData().get("password");
        
        return DataSourceBuilder.create()
            .password(password)
            .build();
    }
}
```

**3. AWS Secrets Manager**：

```java
import com.amazonaws.services.secretsmanager.*;
import com.amazonaws.services.secretsmanager.model.*;

public String getSecret() {
    AWSSecretsManager client = AWSSecretsManagerClientBuilder.standard()
        .withRegion("ap-southeast-1")
        .build();
    
    GetSecretValueRequest request = new GetSecretValueRequest()
        .withSecretId("prod/db/password");
    
    GetSecretValueResult result = client.getSecretValue(request);
    return result.getSecretString();
}
```

### Git Secrets 掃描

使用 git-secrets 防止提交敏感資訊：

```bash
# 安裝
brew install git-secrets

# 初始化
cd my-repo
git secrets --install
git secrets --register-aws  # AWS credentials pattern
```

新增自訂規則：
```bash
git secrets --add 'password\s*=\s*["\'].+["\']'
git secrets --add 'api[_-]?key\s*=\s*["\'].+["\']'
```

提交時自動檢查：
```bash
git commit -m "Add feature"

# 如果包含敏感資訊
[ERROR] Matched one or more prohibited patterns
password = "MyP@ssw0rd123"  ← 不允許提交
```

### 已經提交的怎麼辦？

使用 BFG Repo-Cleaner 清除歷史：

```bash
# 安裝
brew install bfg

# 清除包含 password 的檔案
bfg --delete-files passwords.txt

# 清除所有 commit 中的特定字串
bfg --replace-text replacements.txt
```

`replacements.txt`：
```
MyP@ssw0rd123  # 會被替換成 ***REMOVED***
api_key_12345
```

## 網路安全

### API Rate Limiting

防止 DDoS 和暴力破解。

使用 Spring Boot + Bucket4j：

`pom.xml`：
```xml
<dependency>
    <groupId>com.github.vladimir-bukhtoyarov</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>4.10.0</version>
</dependency>
```

實作：
```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {
    
    private final Map<String, Bucket> cache = new ConcurrentHashMap<>();
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                             HttpServletResponse response, 
                             Object handler) {
        String key = getClientIP(request);
        Bucket bucket = cache.computeIfAbsent(key, k -> createBucket());
        
        if (bucket.tryConsume(1)) {
            return true;  // 允許請求
        } else {
            response.setStatus(429);  // Too Many Requests
            response.getWriter().write("Rate limit exceeded");
            return false;
        }
    }
    
    private Bucket createBucket() {
        // 每分鐘最多 60 個請求
        Refill refill = Refill.intervally(60, Duration.ofMinutes(1));
        Bandwidth limit = Bandwidth.classic(60, refill);
        return Bucket4j.builder()
            .addLimit(limit)
            .build();
    }
}
```

### HTTPS Everywhere

**問題**：HTTP 流量未加密，可能被竊聽。

**解決**：
1. 使用 Let's Encrypt 免費憑證
2. Kubernetes Ingress 強制 HTTPS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-service
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"  # 強制 HTTPS
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
```

### 網路政策（Network Policy）

限制 Pod 之間的通訊。

預設（不安全）：
```
所有 Pod 可以互相通訊
order-service ←→ payment-service ←→ database ←→ ...
```

使用 Network Policy（零信任）：
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: order-service  # 只允許 order-service 連線
    - podSelector:
        matchLabels:
          app: payment-service
    ports:
    - protocol: TCP
      port: 5432
```

其他 Pod 嘗試連線 database → 被拒絕。

## 安全 Checklist

建立上線前的安全檢查清單：

```markdown
## 安全 Checklist

### 程式碼
- [ ] SonarQube 掃描通過（無 Critical 漏洞）
- [ ] 依賴套件無 Critical/High CVE
- [ ] 沒有硬編碼密碼/API key
- [ ] SQL 使用 PreparedStatement
- [ ] 輸入驗證（防止 XSS、SQL Injection）

### 容器
- [ ] Trivy 掃描通過
- [ ] 使用最小 base image（alpine, distroless）
- [ ] 非 root 使用者執行
- [ ] 讀寫檔案系統設為 read-only

### Kubernetes
- [ ] Resource limits 設定
- [ ] Network Policy 設定
- [ ] RBAC 最小權限
- [ ] Secrets 使用 Vault/Secrets Manager
- [ ] Ingress 強制 HTTPS

### API
- [ ] Rate limiting 啟用
- [ ] CORS 正確配置
- [ ] 認證/授權機制（OAuth2/JWT）
- [ ] API 文件不洩漏敏感資訊

### 監控
- [ ] Falco 運行時監控
- [ ] 日誌不包含敏感資訊
- [ ] 安全事件告警設定
```

## 心得

以前總覺得安全是資安團隊的事，開發只要寫功能就好。但這次 Log4Shell 事件讓我們意識到：安全是每個人的責任。

DevSecOps 不是增加負擔，是讓我們更早發現問題：
- SonarQube 在開發階段就發現 SQL Injection
- Dependency-Check 每週自動更新依賴
- Trivy 確保 Docker image 安全
- Falco 即時監控異常行為

最重要的改變：安全變成流程的一部分，不是事後檢查。

建議：
1. **自動化**：人工檢查會漏掉，CI/CD 自動化掃描
2. **Shift Left**：越早發現越好
3. **教育**：團隊要有安全意識
4. **持續改進**：定期更新工具和規則

安全不是一次性的，是持續的過程。

下週要研究多雲架構，學習如何在不同雲端平台上部署應用。
