---
layout: post
title: "GitOps 宣言：為什麼是 ArgoCD？"
date: 2022-08-12 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [ArgoCD, GitOps, Kubernetes, Pull-based, Push-based]
---

上週我們討論了 Jenkins Pipeline 的侷限,這週要回答一個更本質的問題:**為什麼選擇 ArgoCD?**

在 GitOps 的世界裡,工具選擇不只是「哪個功能多」,而是「哪個架構最符合 GitOps 的核心原則」。今天我們要深入探討 Push-based 與 Pull-based 的差異,以及為什麼 ArgoCD 的設計哲學更適合企業級場景。

---

## 本週目標

建立對 GitOps 核心原則的理解,並確立 ArgoCD 作為我們轉型工具的理論基礎。

---

## 技術實作重點

### 1. GitOps 的四大原則

在選擇工具之前,先理解 GitOps 到底在追求什麼。根據 [OpenGitOps](https://opengitops.dev/) 的定義:

#### 原則一:宣告式 (Declarative)

系統的期望狀態必須以宣告式方式描述 (通常是 YAML)。

```yaml
# 這是宣告式:描述「應該是什麼樣子」
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: my-app:v1.2.3
```

**反例 (指令式):**
```bash
kubectl scale deployment my-app --replicas=3
kubectl set image deployment/my-app app=my-app:v1.2.3
```

#### 原則二:版本控制 (Versioned and Immutable)

所有配置必須存在 Git 中,且每次變更都有完整的 commit history。

**為什麼重要?**  
當線上出問題時,你可以立刻回答:
- 這個配置是誰改的? → `git blame`
- 為什麼改? → Commit message 與 PR 描述
- 如何回到上一個穩定版本? → `git revert`

#### 原則三:自動拉取 (Pulled Automatically)

系統應該**主動**從 Git 拉取最新狀態,而不是被動地接受外部指令推送。

**這是 ArgoCD 最關鍵的設計差異。**

#### 原則四:持續調和 (Continuously Reconciled)

系統應該持續監控實際狀態與期望狀態的差異,並自動修正 (Self-Healing)。

---

### 2. Push-based vs. Pull-based:誰更安全?

這是選擇 GitOps 工具時最核心的問題。

#### Push-based 模式 (例如:傳統 Jenkins)

```
[Jenkins] --kubectl apply--> [Kubernetes Cluster]
```

**特徵:**
- CI 系統需要擁有 Kubernetes 的寫入權限
- 部署指令從外部發起 (從 Jenkins 推送到 K8s)
- 需要暴露 K8s API Server 給 CI 系統

**安全風險:**
- Jenkins 被入侵 → 攻擊者可以直接操作生產環境
- 需要在 Jenkins 中儲存 Kubeconfig (高權限憑證)
- 網路邊界複雜 (CI 系統需要能連到 K8s API)

#### Pull-based 模式 (ArgoCD 的做法)

```
[Git Repository] <--polls-- [ArgoCD in K8s] --applies--> [Kubernetes Cluster]
```

**特徵:**
- ArgoCD 運行在 Kubernetes 集群內部
- 它主動去 Git 拉取配置,而不是被動接受指令
- CI 系統**不需要** Kubernetes 權限,只需要能推送到 Git

**安全優勢:**
- 即使 CI 系統被入侵,攻擊者也無法直接操作 K8s
- Kubeconfig 不會離開集群邊界
- 所有變更都必須經過 Git (有 audit trail)

**類比:**  
Push-based 像「你家大門鑰匙給了快遞員」,Pull-based 像「快遞員把包裹放門口,你自己去拿」。哪個更安全?顯而易見。

---

### 3. 為什麼是 ArgoCD,而不是 Flux 或 Spinnaker?

我們評估了三個主流工具:

| 特性 | ArgoCD | Flux CD | Spinnaker |
|------|--------|---------|-----------|
| **架構** | Pull-based | Pull-based | Push-based |
| **UI** | 強大的圖形化介面 | 僅 CLI | 複雜的 Web UI |
| **學習曲線** | 中等 | 陡峭 | 非常陡峭 |
| **多集群管理** | 原生支援 | 需額外配置 | 支援但複雜 |
| **社群活躍度** | 高 (CNCF 孵化) | 高 (CNCF 孵化) | 中 (Netflix 開源) |

**最終選擇 ArgoCD 的三個理由:**

1. **視覺化排錯:** ArgoCD 的 Resource Tree 可以清楚看到每個資源的狀態,對於 K8s 新手很友善
2. **企業級功能:** SSO 整合、RBAC、多集群管理都是內建,不需要自己拼湊
3. **社群成熟度:** 2022 年已經有大量企業採用案例,踩坑的人多,文件也完善

---

## 遇到的挑戰與對策

### 挑戰一:「Pull-based 會不會很慢?」

有同事擔心:「如果 ArgoCD 每 3 分鐘才去 Git 拉一次,那部署不就延遲了?」

**對策:**  
我解釋了兩個機制:

1. **Webhook 通知:** Git 有變更時可以主動通知 ArgoCD,立刻觸發同步
2. **輪詢間隔可調:** 預設 3 分鐘,但可以改成 30 秒甚至更短

**實測結果:**  
配置 Webhook 後,從 Git Commit 到 Pod 更新,平均耗時 **45 秒**,完全可以接受。

### 挑戰二:「那 CI 要做什麼?」

以前 Jenkins Pipeline 負責 Build + Deploy,現在 CD 交給 ArgoCD,那 Jenkins 要做什麼?

**對策:**  
這是一個**觀念轉換**:

- **CI 的職責:** Build Image → Run Tests → Push to Registry → **Update Git Manifest**
- **CD 的職責:** ArgoCD 監測到 Manifest 變更 → 自動同步到 K8s

**範例:**
```groovy
// Jenkinsfile (只負責 CI)
stage('Update Manifest') {
    steps {
        sh """
            git clone https://github.com/my-org/k8s-manifests.git
            cd k8s-manifests
            sed -i 's|image: my-app:.*|image: my-app:${IMAGE_TAG}|' deployment.yaml
            git commit -am "Update image to ${IMAGE_TAG}"
            git push
        """
    }
}
```

**關鍵洞察:**  
CI 不再直接碰 Kubernetes,它只是「更新 Git 中的期望狀態」。真正的部署由 ArgoCD 負責。

---

## 給團隊的觀念分享

### 不要為了工具而工具

這週花了很多時間在「說服自己」為什麼選 ArgoCD。但更重要的是,我們要確保這個選擇是基於:

1. **問題驅動,而非潮流驅動**  
   我們要解決的是「狀態一致性」和「權限安全」,不是「因為 CNCF 推薦」。

2. **團隊能力匹配**  
   ArgoCD 的學習曲線在可承受範圍內,不會變成只有我一個人會用的「黑魔法」。

3. **長期維護成本**  
   社群活躍、文件完善,五年後這個工具還會被維護。

### 架構決策要能被挑戰

在技術選型會議上,我刻意邀請持懷疑態度的同事來「攻擊」這個方案:

- 「萬一 ArgoCD 掛了怎麼辦?」  
  → 答:ArgoCD 本身跑在 K8s 上,可以做 HA 部署。而且即使掛了,也不影響現有應用運行,只是暫時無法部署新版本。

- 「如果 Git Repository 被刪了怎麼辦?」  
  → 答:Git 本身就有 backup 機制,而且我們會配置多個 remote (GitHub + 內部 GitLab)。

**這些質疑讓方案更完善,而不是阻礙。**

---

## 下週預告

理論講完了,接下來要動手了。下週的主題:**Jenkins 瘦身計畫:回歸 CI 本質**。

我們要開始重構現有的 Jenkinsfile,把所有 `kubectl` 指令移除,讓 Jenkins 專注做好「建置與測試」。同時也會介紹如何用 Shared Library 來標準化這個過程。

---

**作者的話:**  
選擇工具不是選美比賽,而是找到最適合團隊現況的解決方案。這週的分析過程,讓我更確信 ArgoCD 是正確的方向。但真正的挑戰,從下週開始。

**Tags:** #ArgoCD #GitOps #Pull-based #Push-based #安全架構 #技術選型
