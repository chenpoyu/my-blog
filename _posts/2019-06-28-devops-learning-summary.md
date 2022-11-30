---
layout: post
title: "DevOps 半年學習總結與展望"
date: 2019-06-28 10:00:00 +0800
categories: [DevOps, Summary]
tags: [Learning, Retrospective, DevOps Culture]
---

從去年 11 月開始學習 DevOps（參考 [從傳統開發到 DevOps](/posts/2018/11/12/devops-intro/)），到現在剛好半年。這週來總結這段時間的學習心得。

> 時間：2018-11-12 ~ 2019-06-28（33 週）

## 學習歷程回顧

### 階段一：基礎設施即代碼（2018-11~12）

**Week 1-6**：
- [DevOps 入門](/posts/2018/11/12/devops-intro/)
- [版本控制 Git](/posts/2018/11/19/git-advanced/)
- [基礎設施即代碼 Terraform](/posts/2018/11/26/terraform-intro/)
- [配置管理 Ansible](/posts/2018/12/03/ansible-intro/)
- [持續整合 Jenkins](/posts/2018/12/10/jenkins-ci/)
- [持續部署與交付](/posts/2018/12/17/cd-strategies/)

**收穫**：
- 基礎設施不再手動建立，全部用程式碼管理
- 部署從手動變成自動化
- 時間：手動部署 2 小時 → 自動化 10 分鐘

**挑戰**：
- Terraform state 管理（學會用 S3 + DynamoDB lock）
- Ansible playbook 除錯困難（學會用 `-vvv`、`debug` module）
- Jenkins Pipeline 語法（從 Freestyle 轉到 Pipeline）

### 階段二：容器化（2019-01）

**Week 7-10**：
- [Docker 容器化](/posts/2018/12/24/docker-intro/)
- [Docker Compose](/posts/2018/12/31/docker-compose/)
- [Docker 網路與儲存](/posts/2019/01/07/docker-network-storage/)
- [容器安全與最佳實踐](/posts/2019/01/14/docker-security/)

**收穫**：
- 應用程式打包成容器，解決「在我機器上可以跑」問題
- 開發、測試、生產環境一致
- 資源利用率提升（VM → Container，省 60% 成本）

**挑戰**：
- Image 太大（學會 multi-stage build，3GB → 150MB）
- 資料持久化（學會 volume 和 bind mount 差異）
- 網路理解（bridge、host、overlay）

### 階段三：容器編排（2019-01~02）

**Week 11-17**：
- [Kubernetes 架構](/posts/2019/01/21/kubernetes-intro/)
- [Kubernetes Workloads](/posts/2019/01/28/kubernetes-workloads/)
- [Kubernetes 網路](/posts/2019/02/11/kubernetes-networking/)
- [Kubernetes 儲存](/posts/2019/02/18/kubernetes-storage/)
- [Helm 套件管理](/posts/2019/02/25/helm-package-manager/)

**收穫**：
- 容器編排自動化（自動重啟、擴展、負載平衡）
- 宣告式管理（描述想要的狀態，K8s 自動達成）
- 微服務部署變簡單

**挑戰**：
- 學習曲線陡峭（概念多：Pod、Service、Deployment、StatefulSet、DaemonSet...）
- YAML 地獄（上百行配置）
- 網路複雜（Service、Ingress、NetworkPolicy）

**轉折點**：理解「宣告式 vs 命令式」後，突然開竅。不是告訴 K8s 怎麼做，而是告訴它我要什麼。

### 階段四：可觀測性（2019-03）

**Week 18-21**：
- [日誌管理 ELK Stack](/posts/2019/03/04/elk-stack-logging/)
- [監控 Prometheus](/posts/2019/03/11/prometheus-monitoring/)
- [可視化 Grafana](/posts/2019/03/18/grafana-visualization/)
- [分散式追蹤 Jaeger](/posts/2019/03/25/distributed-tracing/)

**收穫**：
- 系統不再是黑盒子，所有指標、日誌、追蹤都可見
- 問題排查時間：1-2 小時 → 10-20 分鐘
- 提前發現問題（告警比使用者投訴早 5-10 分鐘）

**挑戰**：
- 日誌量暴增（學會 sampling、索引優化）
- 指標太多不知道看哪個（學會定義 SLI）
- 分散式追蹤 sampling 比例調整

**心得**：可觀測性是 DevOps 的眼睛，沒有它就像閉著眼開車。

### 階段五：高可用與可靠性（2019-04~05）

**Week 22-28**：
- [負載平衡與高可用](/posts/2019/04/01/load-balancing-ha/)
- [自動擴展 HPA](/posts/2019/04/08/autoscaling/)
- [服務網格 Istio](/posts/2019/04/15/service-mesh-istio/)
- [服務網格流量管理](/posts/2019/04/22/service-mesh-traffic-management/)
- [混沌工程](/posts/2019/04/29/chaos-engineering-intro/)
- [資料庫遷移策略](/posts/2019/05/06/database-migration-strategy/)
- [備份與災難恢復](/posts/2019/05/13/backup-disaster-recovery/)

**收穫**：
- 可用性：99% → 99.9%（年停機時間從 3.65 天降到 8.76 小時）
- 自動擴展應對流量高峰（雙 11、促銷活動）
- Service Mesh 統一處理服務間通訊（重試、超時、熔斷）

**挑戰**：
- Istio 複雜度高（學了一個月才真正理解）
- 混沌工程初期很可怕（故意搞壞系統）
- 資料庫遷移零停機（花最多心力規劃）

**心得**：高可用不是買更好的機器，而是假設一切都會壞，提前準備。

### 階段六：性能與成本（2019-05~06）

**Week 29-31**：
- [性能測試策略](/posts/2019/05/20/performance-testing-strategy/)
- [SRE 與錯誤預算](/posts/2019/05/27/sre-error-budget-slo/)
- [雲端成本優化](/posts/2019/06/03/cloud-cost-optimization/)

**收穫**：
- 找到並修復性能瓶頸（API 延遲 P95 從 800ms 降到 180ms）
- 錯誤預算量化了可靠性和速度的平衡
- 雲端成本降 33%（$12,000 → $8,000/月）

**挑戰**：
- 性能測試需要模擬真實流量（學會用 Gatling 寫複雜場景）
- 說服團隊接受錯誤預算概念（「bug 可以不修？」）
- 成本優化不能影響性能

**心得**：性能和成本看似矛盾，實際上很多浪費可以避免（過度配置、閒置資源）。

### 階段七：安全與整合（2019-06）

**Week 32-33**：
- [DevSecOps](/posts/2019/06/10/devsecops-intro/)
- [多雲架構](/posts/2019/06/17/multi-cloud-architecture/)
- [工具鏈整合](/posts/2019/06/24/devops-toolchain-integration/)

**收穫**：
- 安全左移（開發階段就掃描漏洞，而不是上線後才發現）
- 多雲策略讓我們不被單一廠商鎖定
- 工具鏈整合讓開發到部署全自動化

**挑戰**：
- 說服開發接受安全掃描（會增加 build 時間）
- 多雲複雜度（決定先做 Cloud-Agnostic，不是真正多雲）
- 工具太多整合困難（花很多時間打通 API）

## 技術棧演進

### 2018-11（開始時）

```
應用層：Spring Boot (單體應用)
       ↓
部署：手動 scp 到伺服器
     手動 systemctl restart
     
監控：沒有（靠使用者回報）

基礎設施：手動在 AWS Console 點擊建立
```

**問題**：
- 部署慢且容易出錯
- 出問題不知道
- 環境不一致（開發和生產不同）

### 2019-01（容器化後）

```
應用層：Spring Boot (微服務)
       ↓
容器：Docker
     ↓
部署：Jenkins CI/CD
     docker-compose
     
監控：簡單的 Cloudwatch
```

**改善**：
- 部署變快（10 分鐘）
- 環境一致
- 基本監控

### 2019-03（Kubernetes 後）

```
應用層：Spring Boot (微服務)
       ↓
容器：Docker
     ↓
編排：Kubernetes
     ├─ Deployment (無狀態服務)
     ├─ StatefulSet (資料庫)
     └─ Service、Ingress (網路)
     
CI/CD：GitLab CI
       ↓
       Harbor (Container Registry)
       ↓
       ArgoCD (GitOps)

監控：Prometheus + Grafana
日誌：ELK Stack
追蹤：Jaeger
```

**改善**：
- 自動擴展
- 自我修復
- 可觀測性完整

### 2019-06（現在）

```
應用層：Spring Boot (微服務)
       ↓
容器：Docker
     ↓
編排：Kubernetes
     ├─ Deployment、StatefulSet
     └─ Service Mesh (Istio)
         ├─ 流量管理
         ├─ 安全 (mTLS)
         └─ 可觀測性
     
基礎設施：Terraform (IaC)

CI/CD：GitLab CI
       ├─ SonarQube (程式碼品質)
       ├─ OWASP (依賴掃描)
       ├─ Trivy (容器掃描)
       └─ ArgoCD (GitOps)

可觀測性：
  監控：Prometheus + Grafana
  日誌：ELK Stack
  追蹤：Jaeger
  告警：Alertmanager → PagerDuty/Slack

可靠性：
  自動擴展：HPA
  混沌工程：定期演練
  備份：Velero (K8s)、pgBackRest (DB)
```

**達成**：
- 部署：10 分鐘，全自動
- 可用性：99.9%
- 監控：完整
- 安全：左移
- 成本：優化 33%

## 關鍵數據對比

| 指標 | 2018-11 | 2019-06 | 改善 |
|------|---------|---------|------|
| 部署時間 | 2 小時 | 10 分鐘 | **92%** ↓ |
| 部署頻率 | 每週 1 次 | 每天 5-10 次 | **50x** ↑ |
| 失敗率 | 20% | 2% | **90%** ↓ |
| MTTR (平均修復時間) | 2 小時 | 15 分鐘 | **87%** ↓ |
| 可用性 | 99% | 99.9% | **10x** ↑ (停機時間) |
| 資源利用率 | 30% | 70% | **133%** ↑ |
| 雲端成本 | $12,000/月 | $8,000/月 | **33%** ↓ |
| 線上 bug 發現時間 | 使用者回報 (平均 30 分鐘) | 告警 (5 分鐘內) | **83%** ↓ |

## 文化轉變

### Dev 與 Ops 關係

**以前**：
```
Dev：「我寫好了，丟給你們部署」
Ops：「你的程式有問題，跑不起來」
Dev：「在我電腦可以跑啊」
Ops：「那是你的環境」
→ 互相指責
```

**現在**：
```
Dev：寫好程式碼，push
CI/CD：自動建置、測試、部署
失敗：Slack 通知 Dev
→ Dev 自己修，再 push
→ 快速迭代
```

Dev 和 Ops 的界線模糊了，大家都對整個流程負責。

### 失敗文化

**以前**：
- 出問題 → 找戰犯 → 懲罰
- 結果：大家隱瞞問題、不敢創新

**現在**：
- 出問題 → Blameless Postmortem → 改進流程
- 混沌工程：故意搞壞系統，提前發現問題
- 結果：大家願意分享失敗、快速學習

### 自動化思維

**以前**：
- 手動做事
- 覺得「自動化太複雜，手動比較快」

**現在**：
- 做超過 2 次的事就自動化
- 時間投資：寫自動化腳本可能要 2 小時，但之後每次省 30 分鐘
- 人做機器該做的事是浪費

## 重要教訓

### 1. 不要追求完美

**錯誤**：想一次做到完美（Kubernetes + Istio + ELK + Prometheus...）

**結果**：太複雜，團隊跟不上。

**正確**：逐步迭代
- Week 1-10：Docker + Jenkins
- Week 11-20：Kubernetes
- Week 21-30：監控、日誌
- Week 31-33：Service Mesh、安全

每個階段穩定後再加新東西。

### 2. 工具不是重點，問題才是

**錯誤**：「我們要用 Kubernetes，因為大家都在用」

**正確**：「我們的問題是手動部署太慢，需要自動化。Kubernetes 可以解決嗎？」

先定義問題，再選工具。

我們差點導入 Service Mesh，後來發現問題不是服務間通訊，而是資料庫慢。最後加個 index 就解決了。

### 3. 文檔和知識分享

**錯誤**：只有一個人懂（單點故障）

**結果**：那個人離職或休假，系統掛了沒人會修。

**正確**：
- 寫文檔（Runbook、Troubleshooting Guide）
- 定期分享會（每週五下午）
- Pair programming、Pair operations
- On-call rotation（強迫每個人學）

### 4. 監控比你想的重要

**錯誤**：功能做完就上線，監控「之後再說」

**結果**：出問題不知道，靠使用者回報。

**正確**：上線前先做監控
- 健康檢查（liveness、readiness）
- 關鍵指標（RPS、延遲、錯誤率）
- 告警規則

監控不是錦上添花，是必需品。

### 5. 安全不能事後補

**錯誤**：先上線，安全「之後再說」

**結果**：Log4Shell 漏洞爆發，花一週緊急修補。

**正確**：DevSecOps
- CI/CD 整合安全掃描
- 發現漏洞就擋住部署
- 定期更新依賴

安全左移，越早發現越便宜。

## 團隊成長

### 技能樹

**2018-11**：
- 後端開發：Spring Boot
- 資料庫：MySQL
- 部署：不太懂

**2019-06**：
- 後端開發：Spring Boot、微服務
- 資料庫：MySQL、PostgreSQL、Redis
- 容器：Docker、Docker Compose
- 編排：Kubernetes、Helm
- CI/CD：GitLab CI、Jenkins、ArgoCD
- 基礎設施：Terraform、Ansible
- 監控：Prometheus、Grafana、ELK、Jaeger
- 雲端：AWS (EC2、RDS、S3、VPC...)
- 程式語言：Java、Shell、Python、YAML

技能樹擴展很多，但也更雜。

### 角色轉變

**以前**：後端工程師

**現在**：Full-stack？DevOps Engineer？SRE？

界線模糊了，現在的工作是：
- 寫程式碼
- 建立基礎設施
- 配置 CI/CD
- 寫監控告警
- 處理線上問題
- 成本優化

這是好事（掌握全局），也是挑戰（要學的太多）。

### 學習方法

**有效**：
1. **官方文檔**：最權威，但有時太技術
2. **實際操作**：看再多文章，不如自己動手
3. **出錯學習**：搞壞測試環境，理解為什麼
4. **寫部落格**：教學相長，寫出來才真正懂
5. **社群**：CNCF Slack、Kubernetes Forum、Reddit

**無效**：
1. 只看不做
2. 盲目跟風（看到新工具就想用）
3. 不問為什麼（複製貼上 StackOverflow）

## 未來展望

### 接下來要學的

**短期（3 個月）**：
- **GitOps**：ArgoCD 深入（目前只用基本功能）
- **Istio 進階**：流量管理更複雜的場景
- **Kubernetes Operator**：自動化複雜應用（資料庫、消息佇列）
- **FinOps**：成本優化持續進行

**中期（6 個月）**：
- **機器學習整合**：ML 模型部署到 Kubernetes（KubeFlow？）
- **多雲實踐**：目前只是 Cloud-Agnostic，考慮真正多雲
- **邊緣運算**：IoT 裝置管理（K3s？）
- **Serverless**：某些場景可能更適合（AWS Lambda、Knative）

**長期（1 年）**：
- **平台工程**：建立內部 PaaS，讓開發更容易
- **AI Ops**：用 ML 做異常檢測、根因分析
- **eBPF**：更底層的可觀測性和安全

### 技術趨勢觀察

**確定會火**：
- **Kubernetes**：已經是標準，只會更普及
- **Service Mesh**：複雜度高，但解決真實問題
- **GitOps**：宣告式管理是未來
- **可觀測性**：系統越來越複雜，必須可見

**可能會火**：
- **WebAssembly**：容器的替代方案？
- **Serverless**：某些場景很適合
- **邊緣運算**：5G 普及後會需要

**可能是泡沫**：
- 某些「DevOps 平台」：打包一堆工具，但不解決根本問題
- 過度複雜的架構：不是每個公司都需要 Netflix 的架構

### 團隊目標

**技術**：
- 可用性：99.9% → 99.95%
- 部署：全自動化，開發不需碰 kubectl
- 監控：更主動（預測故障，而不是事後反應）
- 成本：持續優化

**文化**：
- Blameless 文化深化
- 知識分享制度化
- 自動化一切能自動化的

**組織**：
- 考慮設立 SRE 團隊（專職可靠性）
- 平台團隊（建立內部工具）

## 給新手的建議

如果你也想學 DevOps：

### 1. 先打好基礎

- **Linux**：DevOps 大多跑在 Linux，熟悉 CLI
- **網路**：TCP/IP、HTTP、DNS（很多問題出在網路）
- **程式**：至少會一種語言（Python/Go/Shell）
- **Git**：版本控制是基本功

### 2. 循序漸進

不要一次學太多，建議順序：
1. Linux + Shell Script
2. Docker
3. CI/CD (Jenkins or GitLab CI)
4. Kubernetes
5. Terraform
6. Prometheus + Grafana
7. 進階主題（Service Mesh、GitOps...）

每個至少花 2-4 週真正弄懂。

### 3. 動手做 Side Project

看 100 篇文章，不如做一個專案：

**範例**：部落格系統
1. 用 Docker 容器化
2. 寫 CI/CD 自動建置
3. 用 Terraform 建立 AWS 基礎設施
4. 部署到 Kubernetes
5. 配置監控和日誌
6. 設定告警

完整走一遍，會學到超多。

### 4. 搞壞東西

- 在測試環境故意搞壞
- 刪掉 Pod 看會怎樣
- 斷網路看監控會怎樣
- 關掉資料庫看應用會怎樣

搞壞才知道為什麼需要某個功能（自動重啟、健康檢查、重試...）。

### 5. 寫部落格

- 強迫自己整理思路
- 教學相長
- 未來自己忘記可以回來看
- 幫助其他人

這半年寫了 33 篇，自己收穫最多。

### 6. 加入社群

- Kubernetes Slack
- CNCF Meetup
- 線上論壇（Reddit、Stack Overflow）
- 參加研討會（COSCUP、DevOpsDays）

認識同好，交流經驗。

## 結語

這半年從傳統開發轉到 DevOps，學到太多東西。

最大的收穫不是技術，而是思維轉變：
- **自動化**：機器能做的不要人做
- **可觀測性**：看不見就無法改進
- **持續改進**：沒有完美，只有更好
- **擁抱失敗**：從失敗中學習
- **協作文化**：打破 Dev 和 Ops 的牆

DevOps 不只是工具，更是文化和實踐。

這個系列到這邊告一段落。未來會繼續學習，持續分享。

感謝這半年來的閱讀和支持。讓我們一起在 DevOps 的路上前進！

---

## 附錄：學習資源

**書籍**：
- 《The Phoenix Project》：DevOps 文化
- 《The DevOps Handbook》：實踐指南
- 《Site Reliability Engineering》：Google SRE
- 《Accelerate》：DevOps 指標和研究

**線上課程**：
- Udemy：Docker、Kubernetes 課程
- Coursera：Google Cloud、AWS 課程
- Linux Academy (現在的 A Cloud Guru)

**官方文檔**：
- Kubernetes：https://kubernetes.io/docs/
- Docker：https://docs.docker.com/
- Terraform：https://www.terraform.io/docs/
- Prometheus：https://prometheus.io/docs/

**社群**：
- CNCF Slack：https://slack.cncf.io/
- Kubernetes Forum：https://discuss.kubernetes.io/
- Reddit：r/kubernetes、r/devops

**部落格**：
- AWS Blog
- Google Cloud Blog
- Netflix Tech Blog
- Spotify Engineering

持續學習，一起成長！
