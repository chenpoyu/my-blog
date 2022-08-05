---
layout: post
title: "解構傳統 Jenkins 部署的侷限"
date: 2022-08-05 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [Jenkins, ArgoCD, GitOps, Kubernetes, CI/CD]
---

在過去五年裡,我們團隊用 Jenkins Pipeline 部署了數十個微服務。從表面上看,一切運作順利——開發者推送程式碼,Pipeline 自動觸發,最後 `kubectl apply` 完成部署。但隨著系統規模擴大,我們開始意識到:「自動化」不等於「可控制」。

這篇文章不是要批判 Jenkins,而是要從架構思維的角度,探討為什麼我們需要從 **指令式部署 (Imperative)** 轉向 **宣告式部署 (Declarative)**,以及為什麼 GitOps 會是更好的選擇。

> **時空背景:** 2022 年,Kubernetes 已經成熟,但 GitOps 剛開始在企業落地。

---

## 本週目標

檢討現有 Jenkins CI/CD Pipeline 的維護成本與技術債,為接下來的轉型奠定理論基礎。

---

## 技術實作重點

### 1. 盤點現有 Pipeline 的問題

我們分析了團隊中 50 個 Microservices 的 Jenkinsfile,發現以下模式:

```groovy
stage('Deploy to K8s') {
    steps {
        sh 'kubectl apply -f k8s/deployment.yaml'
        sh 'kubectl set image deployment/my-app my-app=${IMAGE_TAG}'
    }
}
```

這種寫法看似簡單,但隱藏了幾個嚴重問題:

#### 問題一:狀態漂移 (Configuration Drift)

當有人手動在 Kubernetes 上執行 `kubectl edit` 或 `kubectl patch` 時,實際運行的配置與 Git 中的版本已經不一致。但 Jenkins 無法察覺這點,下次部署時可能會覆蓋掉這些變更,或是產生衝突。

**案例:**  
某次線上出問題,值班工程師緊急調整了 Pod 的 CPU Limit。隔天有同事推送了新版本,Pipeline 自動部署,結果把昨天的臨時修改覆蓋掉了,系統又掛了。

#### 問題二:權限管理混亂

Jenkins 需要擁有 `kubectl` 的 Cluster-Admin 權限才能部署。這意味著:

- 所有能觸發 Pipeline 的人,間接擁有了集群的最高權限
- 缺乏細粒度的 RBAC 控制
- 審計困難——無法追溯「誰在什麼時候部署了什麼」

#### 問題三:部署狀態不透明

Pipeline 執行完 `kubectl apply` 後就結束了,但這不代表應用程式真的啟動成功。我們需要額外寫腳本去檢查 Pod 的 Rollout 狀態:

```bash
kubectl rollout status deployment/my-app --timeout=5m
```

但這種檢查很脆弱,遇到 ImagePullBackOff 或 CrashLoopBackOff 時,Pipeline 可能會誤判為成功。

### 2. Imperative vs. Declarative 的本質差異

| 特性 | Imperative (Jenkins) | Declarative (GitOps) |
|------|---------------------|---------------------|
| 部署方式 | 執行一系列指令 | 描述期望的最終狀態 |
| 狀態追蹤 | 無法保證一致性 | 持續監控並自動修正 |
| 回滾機制 | 需手動寫反向指令 | Git Revert 即可 |
| 審計能力 | 依賴 CI 日誌 | 所有變更都在 Git History |

**關鍵洞察:**  
Jenkins 告訴系統「做什麼 (Do)」,GitOps 告訴系統「變成什麼 (Be)」。前者無法保證最終狀態,後者則能持續確保狀態收斂。

---

## 遇到的挑戰與對策

### 挑戰一:團隊成員的心理阻力

當我提出要遷移到 GitOps 時,有同事質疑:「Jenkins 用得好好的,為什麼要換?學習成本誰來承擔?」

**對策:**  
我沒有強推,而是先做了一份「事故統計報告」,列出過去半年因為 Jenkins Pipeline 導致的生產問題:

- 因權限過大導致的誤操作:3 次
- 因狀態漂移導致的部署失敗:7 次
- 回滾失敗需要手動介入:5 次

**數據會說話。** 當大家看到這些數字,就理解這不是技術潮流,而是實際痛點的解方。

### 挑戰二:如何定義「轉型成功」的指標

我們需要量化的目標,而不是模糊的「改善」。最終定義了這些 KPI:

- **Mean Time to Recovery (MTTR):** 從發現問題到回滾的時間,目標從 30 分鐘降到 5 分鐘
- **Deployment Frequency:** 每天部署次數,希望能從 5 次提升到 20 次
- **Change Failure Rate:** 部署後需要緊急修復的比例,從 15% 降到 5%

---

## 給團隊的觀念分享

### 自動化不代表正確

很多團隊以為「寫了 CI/CD Pipeline 就是 DevOps」,但其實只是把手動操作自動化而已。真正的 DevOps 要追求的是:

> **可觀測性 (Observability)** + **可預測性 (Predictability)** + **快速恢復能力 (Resilience)**

Jenkins Pipeline 解決了自動化,但沒有解決「狀態的一致性」。當系統出問題時,我們無法快速回答:

- 現在線上跑的,到底是哪個版本?
- 這個配置是誰改的?為什麼改?
- 如果要回到三天前的狀態,要怎麼做?

GitOps 的核心價值,就是讓 **Git 成為唯一的真相來源 (Single Source of Truth)**。

### 技術選型的思考框架

在評估要不要導入新技術時,我會問自己三個問題:

1. **它解決了什麼根本問題?** (不是「它很酷」或「大家都在用」)
2. **它的學習曲線能否被團隊吸收?** (不能只有我一個人會)
3. **它的維護成本能否被長期承擔?** (不能變成技術債)

對於 GitOps 這個選擇,我的答案是:

1. 解決了「狀態一致性」與「權限管控」的根本問題
2. 學習曲線可控——團隊已經會 Git 和 K8s,只是換個工作流程
3. 維護成本低——ArgoCD 本身很輕量,且社群活躍

---

## 下週預告

既然決定要轉型,下一個問題就是:**為什麼是 ArgoCD?**

市面上 GitOps 工具不少 (Flux, Jenkins X, Spinnaker),為什麼我們選擇 ArgoCD?它的架構有什麼特點?Push-based 和 Pull-based 的本質差異是什麼?

下週將深入分析 GitOps 的核心原則,以及 ArgoCD 的技術優勢。

---

**作者的話:**  
轉型從來不是一蹴可幾,而是一連串的權衡與取捨。這篇文章是我們團隊思考的起點,希望也能給你一些啟發。如果你也在思考類似的問題,歡迎留言交流。

**Tags:** #Jenkins #GitOps #ArgoCD #Kubernetes #DevOps #技術債 #架構思維
