---
layout: post
title: "年度回顧:GitOps 轉型五個月總結"
date: 2022-12-23 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [GitOps, Retrospective, Metrics, DORA, Lessons Learned]
---

從 2022 年 8 月開始,我們花了五個月時間完成了 **從 Jenkins 到 ArgoCD 的 GitOps 轉型**。

這週不是技術文,而是 **回顧這段旅程的成果、教訓、以及 2023 年的計劃**。希望這些經驗能幫助其他團隊避開我們踩過的坑。

---

## 轉型時間軸

```
2022-08
├─ W1: 分析 Jenkins 的問題
├─ W2: 選擇 ArgoCD 作為 GitOps 工具
├─ W3: 重構 Jenkins (CI Only)
└─ W4: 深入理解 ArgoCD 架構

2022-09
├─ W1: 安裝 ArgoCD
├─ W2: 設計 Manifest Repository 結構
├─ W3: 建立第一條 CI/CD Pipeline
└─ W4: 實作 Auto-Sync 與 Self-Heal

2022-10
├─ W1: Helm 與 ArgoCD 整合
├─ W2: App-of-Apps 模式
├─ W3: Kustomize 多環境管理
└─ W4: 故障排除與監控

2022-11
├─ W1: SSO 整合 (GitHub OAuth)
├─ W2: Sealed Secrets 密碼管理
├─ W3: Multi-cluster 管理
└─ W4: Image Updater 自動化

2022-12
├─ W1: 第一個真實微服務遷移
├─ W2: 團隊 GitOps Workflow 規範
└─ W3: 效能優化與資源調校
```

**總共 20 週,五個月。**

---

## 關鍵成果

### 1. DORA Metrics 改善

**Lead Time for Changes (程式碼到生產的時間)**

| 環境 | 轉型前 (Jenkins) | 轉型後 (ArgoCD) | 改善 |
|------|-----------------|-----------------|------|
| Dev | 20 分鐘 | 5 分鐘 | 75% ⬇️ |
| Staging | 1 小時 | 10 分鐘 | 83% ⬇️ |
| Production | 4 小時 | 20 分鐘 | 92% ⬇️ |

**Deployment Frequency (部署頻率)**

| 環境 | 轉型前 | 轉型後 | 改善 |
|------|--------|--------|------|
| Dev | 5 次/天 | 20 次/天 | 4x ⬆️ |
| Staging | 2 次/週 | 10 次/週 | 5x ⬆️ |
| Production | 1 次/週 | 3 次/週 | 3x ⬆️ |

**Change Failure Rate (變更失敗率)**

```
轉型前:15% (每 7 次部署有 1 次失敗)
轉型後:5% (每 20 次部署有 1 次失敗)
改善:67% ⬇️
```

**Time to Restore (故障恢復時間)**

```
轉型前:平均 45 分鐘 (需要找 Jenkins Job,重新部署)
轉型後:平均 5 分鐘 (Git Revert + ArgoCD Sync)
改善:89% ⬇️
```

---

### 2. 團隊效率提升

**開發者回饋 (匿名問卷,N=15):**

| 問題 | 滿意度 (1-5) |
|------|-------------|
| 部署速度 | 4.6 ⭐ |
| 部署信心 | 4.8 ⭐ |
| 問題排查效率 | 4.5 ⭐ |
| 學習曲線 | 3.2 ⭐ (需改進) |
| 整體滿意度 | 4.7 ⭐ |

**定性回饋:**

> 「再也不用擔心『我剛剛部署了什麼?』」 - 後端工程師 A

> 「回滾從『恐懼』變成『按一個按鈕』」 - DevOps 工程師 B

> 「現在可以放心地在 Friday 部署了」 - 前端工程師 C

---

### 3. 可觀測性提升

**Before (Jenkins):**

```
問題:Production 出現 Bug
排查:
1. 去 Jenkins 找部署記錄 (哪個 Job?哪個 Build?)
2. 找 Git Commit (可能已經被覆蓋)
3. 猜測「當時的配置是什麼」
時間:平均 30 分鐘
```

**After (ArgoCD):**

```
問題:Production 出現 Bug
排查:
1. 打開 ArgoCD UI
2. 看 "Last Sync" 的 Git Commit
3. 看 Diff (知道改了什麼)
4. Git Revert 回滾
時間:平均 3 分鐘
```

---

### 4. 成本節省

**Infrastructure Cost:**

```
Jenkins Agents 資源消耗:
- 轉型前:10 個 Jenkins Agent (每個 2C4G) = 20 Core 40GB
- 轉型後:5 個 Jenkins Agent (只跑 CI) = 10 Core 20GB
節省:50% ⬇️
```

**Human Cost:**

```
On-call 工程師處理部署問題的時間:
- 轉型前:每週 8 小時
- 轉型後:每週 2 小時
節省:75% ⬇️
```

---

## 遇到的主要挑戰

### 挑戰一:團隊學習曲線陡峭

**問題:**

很多開發者不熟悉 Kubernetes Manifest,更不用說 Kustomize、Helm。

**對策:**

1. **內部培訓:**
   - 每週一次「GitOps Office Hour」
   - 提供範例 Template
   - Pair Programming (一起寫 Manifest)

2. **文件化:**
   - 建立 Confluence 頁面:《GitOps 最佳實踐》
   - 記錄常見問題 (FAQ)

3. **工具支援:**
   - 提供 VS Code 擴展 (YAML Auto-completion)
   - 建立 Manifest Generator (Web UI 自動生成)

**結果:**

從「完全不懂」到「能獨立寫 Manifest」,平均需要 2-3 週。

---

### 挑戰二:Secret 管理複雜

**問題:**

GitOps 要求「所有配置都在 Git」,但 Secret 不能明文存放。

**嘗試過的方案:**

1. ❌ **Git-crypt:** 加密 Secret 檔案
   - 問題:團隊成員需要共享 GPG Key,管理困難

2. ❌ **External Secrets Operator:** 從 Vault 拉取 Secret
   - 問題:引入新的依賴 (Vault),增加複雜度

3. ✅ **Sealed Secrets:** 最終選擇
   - 優點:簡單、原生整合、無額外依賴

**教訓:**

不要過度設計,選擇「夠用就好」的方案。

---

### 挑戰三:跨團隊協作

**問題:**

不同團隊對「什麼該放 Git」有不同看法:

- 後端團隊:「所有配置都要在 Git」
- 前端團隊:「部分配置可以用 kubectl apply」
- SRE 團隊:「Secret 不能放 Git」

**對策:**

召開 **Architecture Decision Record (ADR)** 會議,達成共識:

```
ADR-001: GitOps 範圍界定

決策:
1. 所有應用配置必須在 Git (Deployment, Service, Ingress)
2. Secret 使用 Sealed Secrets 加密後存放
3. 暫時性的測試配置可以用 kubectl (但生命週期 < 1 天)
4. 基礎設施配置 (Ingress Controller, Cert-Manager) 也要在 Git

例外:
- 緊急 Hotfix 可以先 kubectl,事後補 Git
```

---

### 挑戰四:效能瓶頸

**問題:**

當應用數量超過 100 個時,ArgoCD Controller CPU 飆高,Sync 變慢。

**對策:**

1. 增加 Controller 資源 (2C4G → 4C8G)
2. 調整 Reconcile 間隔 (3m → 10m)
3. 啟用 Git Cache
4. 拆分 Repository
5. 使用 ApplicationSet 合併相似應用

**結果:**

Sync 時間從 45 秒降到 12 秒 (73% 改善)。

---

### 挑戰五:如何說服管理層?

**問題:**

管理層的疑問:「為什麼要花這麼多時間重構?現在的 Jenkins 也能用啊?」

**對策:**

提供 **量化數據** 而非空談:

1. **Lead Time 降低 92%** (4 小時 → 20 分鐘)
2. **Change Failure Rate 降低 67%** (15% → 5%)
3. **On-call 時間減少 75%** (8 小時/週 → 2 小時/週)
4. **預計節省成本:** 每年 $50,000 (Agent 資源 + 人力成本)

**結果:**

管理層批准了額外的 Kubernetes 資源預算。

---

## 教訓總結

### 1. 不要一次全部遷移

**錯誤做法:**

「這週末一次把所有應用從 Jenkins 遷到 ArgoCD!」

**正確做法:**

1. 先遷 1 個非關鍵應用 (例如內部工具)
2. 觀察 2 週,收集反饋
3. 優化流程
4. 再遷 5 個應用
5. 重複直到全部遷移完成

**我們的進度:**

- 第 1 個月:2 個應用
- 第 2 個月:10 個應用
- 第 3 個月:30 個應用
- 第 4 個月:50 個應用
- 第 5 個月:120 個應用 (全部)

---

### 2. 流程比工具重要

ArgoCD 只是工具,真正的價值是 **GitOps 的流程**:

- ✅ 所有變更都在 Git
- ✅ Code Review 強制執行
- ✅ 自動化測試
- ✅ 回滾簡單

**即使沒有 ArgoCD,這些流程仍然有價值。**

---

### 3. 文化轉變需要時間

技術轉型容易,文化轉型困難。

**團隊從抗拒到接受的過程:**

```
Week 1-2: 抗拒
「為什麼要改?Jenkins 很好用啊!」

Week 3-4: 懷疑
「這真的會更好嗎?」

Week 5-8: 接受
「好像確實快了一些」

Week 9-12: 擁抱
「我再也不想回去 Jenkins 了」

Week 13+: 傳教士
「其他團隊也應該用 GitOps!」
```

**關鍵:** 讓團隊親身體驗到好處,而非強制推行。

---

### 4. 監控與可觀測性是基礎

沒有監控,就不知道:

- 部署是否成功
- 效能是否下降
- 哪裡出了問題

**必備工具:**

- ✅ ArgoCD UI (可視化部署狀態)
- ✅ Prometheus + Grafana (效能監控)
- ✅ Slack 通知 (部署成功/失敗)
- ✅ Git History (變更追蹤)

---

### 5. 從失敗中學習

我們犯過的錯誤:

1. ❌ **過早優化:** 一開始就設計複雜的 Multi-cluster 架構
   - 教訓:先解決單叢集,再考慮多叢集

2. ❌ **忽略安全:** 忘了設定 RBAC,所有人都是 Admin
   - 教訓:安全從第一天就要做

3. ❌ **文件滯後:** 技術跑得很快,文件跟不上
   - 教訓:每個 Sprint 都要更新文件

4. ❌ **缺乏測試環境:** 直接在 Staging 測試新功能
   - 教訓:建立獨立的 Dev 環境

**每次失敗,都是學習的機會。**

---

## 2023 年計劃

### Q1 2023:深化與優化

1. **Multi-cluster 管理完善**
   - 目標:管理 5 個 Kubernetes Cluster
   - 挑戰:跨叢集的網路與權限管理

2. **Progressive Delivery**
   - 整合 Argo Rollouts
   - 實作 Canary Deployment 和 Blue-Green Deployment

3. **安全強化**
   - 整合 SAST/DAST 工具
   - 實作 Policy-as-Code (OPA)

---

### Q2 2023:平台化

1. **Self-Service Portal**
   - 開發 Web UI,讓開發者自己建立應用
   - 自動生成 Manifest 和 ArgoCD Application

2. **GitOps-as-a-Service**
   - 將 GitOps 能力開放給其他團隊
   - 提供 Template 和最佳實踐

---

### Q3-Q4 2023:擴展與創新

1. **Multi-tenancy**
   - 支援多租戶架構
   - 每個租戶有獨立的 Namespace 和 RBAC

2. **Cost Optimization**
   - 基於 GitOps 的資源自動調度
   - 自動關閉 Dev/Staging 環境 (下班時間)

3. **AI/ML Pipeline 整合**
   - 將 ML Model 部署也納入 GitOps

---

## 最後的話

### 給準備開始 GitOps 的團隊

如果你正在考慮 GitOps,我的建議是:

1. **先搞清楚「為什麼」**
   - 不是因為「很酷」,而是因為「解決實際問題」
   - 如果 Jenkins 夠用,不需要換

2. **從小規模開始**
   - 不要一次全部遷移
   - 先遷 1-2 個非關鍵應用

3. **投資在文化和流程**
   - 工具是次要的,流程才是核心
   - Code Review、測試、監控缺一不可

4. **準備好時間和資源**
   - 這不是「一週就能完成的專案」
   - 我們花了 5 個月,而且團隊有 3 位專職負責

5. **保持學習與改進**
   - GitOps 生態系統變化很快
   - 持續關注社群的最佳實踐

---

### 給我的團隊

感謝這五個月來,團隊的每一位成員:

- 願意擁抱變化
- 不怕犯錯,勇於嘗試
- 互相支援,共同成長

**這不是結束,而是新的開始。**

2023 年,我們繼續前進 🚀

---

**作者的話:**  
這五個月是我職業生涯中最充實的時光之一。從懷疑到確信,從挫折到成就,每一步都值得。希望這 20 篇文章能幫助更多團隊走上 GitOps 之路。

**系列完結 🎉**

**Tags:** #GitOps #Retrospective #Metrics #DORA #Lessons-Learned #Year-End-Review
