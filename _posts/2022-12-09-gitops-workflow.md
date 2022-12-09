---
layout: post
title: "團隊規範制定：GitOps Workflow"
date: 2022-12-09 09:00:00 +0800
categories: [DevOps, GitOps]
tags: [GitOps, Workflow, Code Review, Change Management, Best Practices]
---

經過四個月的實踐,我們建立了完整的 GitOps 基礎設施。但技術只是一部分,更重要的是:**如何讓團隊按照統一的流程工作?**

這週要制定 **GitOps Workflow 規範**,包括 MR 審核機制、部署流程、變更管理。讓 GitOps 從「少數人的工具」變成「團隊的標準」。

---

## 本週目標

建立完整的 GitOps 工作流程規範,確保團隊成員能安全、高效地協作。

---

## 技術實作重點

### 1. GitOps 的三層流程

```
Layer 1: 開發流程 (Application Repository)
  ├─ 功能開發
  ├─ Code Review
  ├─ CI 建置與測試
  └─ 推送 Image 到 Registry

Layer 2: 配置流程 (Manifest Repository)
  ├─ 更新 Manifest (自動或手動)
  ├─ Manifest Review
  └─ Merge 到目標環境分支

Layer 3: 部署流程 (ArgoCD)
  ├─ 偵測 Git 變更
  ├─ 自動或手動 Sync
  └─ 驗證部署結果
```

### 2. Manifest Repository 的分支策略

**策略一:Environment Branches (我們的選擇)**

```
main (Production)
  ↑
staging (Staging)
  ↑
dev (Development)
```

**流程:**

```
1. 開發者在 dev 分支更新 Manifest
2. 測試通過後,MR 到 staging
3. Staging 驗證通過,MR 到 main (Production)
```

**優點:**

- ✅ 清晰的環境隔離
- ✅ 可以獨立回滾某個環境
- ✅ 符合直覺

**缺點:**

- ❌ Merge 衝突多 (dev → staging → main)
- ❌ Hotfix 流程複雜 (需要同時更新三個分支)

---

**策略二:Directory-based (備選方案)**

```
main
├── dev/
├── staging/
└── production/
```

**流程:**

```
1. 所有變更都在 main 分支
2. 修改不同目錄代表不同環境
```

**優點:**

- ✅ 沒有 Merge 衝突
- ✅ Hotfix 簡單

**缺點:**

- ❌ 難以追蹤「哪些變更尚未推到 Production」
- ❌ 容易誤改 Production 配置

---

**我們的決定:Environment Branches (配合嚴格的 Code Review)**

### 3. Merge Request 審核機制

#### CODEOWNERS 配置

```bash
# .github/CODEOWNERS (或 .gitlab/CODEOWNERS)

# Production 分支:需要 Tech Lead 審核
* @tech-lead @ops-lead

# Staging 分支:需要至少 1 位資深工程師審核
* @senior-engineer-1 @senior-engineer-2

# Dev 分支:任何團隊成員都能審核
* @team
```

#### Branch Protection Rules (GitHub)

**For `main` (Production):**

```yaml
Settings → Branches → Branch protection rules
Branch name pattern: main

✅ Require a pull request before merging
  ✅ Require approvals: 2
  ✅ Require review from Code Owners
✅ Require status checks to pass before merging
  ✅ GitHub Actions CI
  ✅ ArgoCD Diff Check
✅ Require conversation resolution before merging
✅ Require signed commits
✅ Include administrators (連 Admin 也要遵守規則)
```

**For `staging`:**

```
✅ Require approvals: 1
```

**For `dev`:**

```
✅ Require approvals: 0 (但建議至少 1 人 Review)
```

### 4. MR Template (規範變更描述)

#### .github/pull_request_template.md

```markdown
## 變更類型
- [ ] 新增應用
- [ ] 更新 Image Tag
- [ ] 修改配置 (資源限制、環境變數等)
- [ ] 刪除資源
- [ ] Hotfix (緊急修復)

## 影響範圍
- **環境:** Dev / Staging / Production
- **應用:** app-a, app-b, ...
- **預期影響:** (例如:Pod 會重啟、流量會增加 20% 等)

## 變更原因
(為什麼要做這個變更?解決什麼問題?)

## 測試計畫
- [ ] 已在 Dev 環境測試
- [ ] 已通過 ArgoCD Diff 檢查
- [ ] 已通知相關團隊

## 回滾計畫
(如果部署失敗,如何回滾?)

## Checklist
- [ ] Manifest 語法正確 (helm template / kustomize build 驗證)
- [ ] Image Tag 正確 (沒有 typo)
- [ ] Secret 已正確加密 (Sealed Secrets)
- [ ] 資源限制合理 (不會 OOM)
- [ ] 已通知 On-call 工程師 (Production 變更)
```

### 5. 自動化檢查 (GitHub Actions)

#### .github/workflows/manifest-validation.yml

```yaml
name: Manifest Validation

on:
  pull_request:
    branches:
      - main
      - staging
      - dev

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    # 1. Helm Template 驗證
    - name: Validate Helm Charts
      run: |
        for chart in helm-charts/*; do
          echo "Validating $chart..."
          helm template $chart --values $chart/values-dev.yaml > /dev/null
        done
    
    # 2. Kustomize Build 驗證
    - name: Validate Kustomize
      run: |
        for overlay in apps/*/overlays/*; do
          echo "Building $overlay..."
          kustomize build $overlay > /dev/null
        done
    
    # 3. YAML Lint
    - name: YAML Lint
      uses: ibiqlik/action-yamllint@v3
      with:
        file_or_dir: .
        config_data: |
          extends: default
          rules:
            line-length: disable
    
    # 4. ArgoCD Diff Check
    - name: ArgoCD Diff
      env:
        ARGOCD_SERVER: argocd.example.com
        ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_TOKEN }}
      run: |
        argocd app diff app-a-prod --server $ARGOCD_SERVER --auth-token $ARGOCD_AUTH_TOKEN
```

### 6. 部署流程標準化

#### Dev 環境 (自動化)

```
1. 開發者推送代碼到 Application Repo
2. Jenkins CI 自動建置 & 測試
3. Image Updater 自動更新 Manifest (dev 分支)
4. ArgoCD 自動 Sync
5. 通知 Slack: "✅ app-a deployed to Dev (v1.2.4)"
```

**完全自動,無需人工介入。**

---

#### Staging 環境 (半自動)

```
1. MR: dev → staging
2. 至少 1 位資深工程師審核
3. Merge 後,ArgoCD 自動 Sync
4. 執行自動化測試 (E2E, Performance)
5. 測試通過 → 準備推 Production
6. 測試失敗 → 回滾並修復
```

**需要人工審核,但部署自動。**

---

#### Production 環境 (手動觸發)

```
1. MR: staging → main
2. 需要 2 位 Tech Lead 審核
3. 通知 On-call 工程師
4. 選擇低流量時段 (或使用 Blue-Green)
5. Merge 後,ArgoCD 不自動 Sync (需手動觸發)
6. 在 ArgoCD UI 中點擊 "SYNC" (或用 CLI)
7. 監控部署過程 (Pod 狀態、錯誤日誌)
8. 驗證健康檢查通過
9. 通知團隊: "✅ app-a v1.2.4 deployed to Production"
```

**完全手動控制,確保安全。**

### 7. 變更管理規範

#### 變更分類

| 類型 | 定義 | 審核要求 | 部署時機 |
|------|------|----------|----------|
| **Standard Change** | 常規更新 (Image Tag 更新) | 1 位審核者 | 工作時間 |
| **Normal Change** | 配置變更 (資源限制、環境變數) | 2 位審核者 | 低流量時段 |
| **Emergency Change** | 線上故障緊急修復 | 事後補審核 | 立即執行 |
| **Major Change** | 架構變更 (新增服務、資料庫遷移) | Tech Lead + Ops Lead | 預先排定維護窗口 |

#### Emergency Change 流程 (Hotfix)

```
1. 發現線上故障
2. 立刻在 main 分支建立 hotfix/xxx MR
3. 跳過審核,直接 Merge (事後補審核)
4. ArgoCD 手動 Sync
5. 驗證修復成功
6. 事後補寫 Post-mortem 文件
7. 將 Hotfix Cherry-pick 回 dev 和 staging 分支
```

**案例:**

```
2022-12-09 14:30 - Production 出現 OOM,所有 Pod Crash
2022-12-09 14:32 - 緊急調整 memory limit (512Mi → 1Gi)
2022-12-09 14:33 - Merge hotfix MR (跳過審核)
2022-12-09 14:35 - ArgoCD Sync,Pod 重啟成功
2022-12-09 14:40 - 服務恢復
2022-12-09 15:00 - 事後審核 MR,發現是新版本記憶體洩漏
```

---

## 遇到的挑戰與對策

### 挑戰一:開發者覺得「太多規則,太慢」

**反饋:**

「以前我直接 kubectl apply 就好了,現在要 MR、要審核,太麻煩!」

**對策:**

展示「規則帶來的價值」:

1. **追溯性:** 上週 Production 出問題,3 分鐘內找到是誰改的、為什麼改
2. **可回滾:** 回滾只需要 Git Revert,不需要「記得上次的配置是什麼」
3. **知識共享:** Code Review 讓團隊成員互相學習

**結果:** 團隊逐漸接受,甚至主動要求「這個變更也幫我 Review 一下」。

### 挑戰二:Merge 衝突頻繁

當多個 MR 同時更新 dev → staging 時,經常衝突。

**對策:**

1. **小步快跑:** 鼓勵頻繁、小規模的變更,而非累積一週後大爆炸
2. **自動化 Rebase:** GitHub Actions 自動 Rebase MR
3. **溝通:** 團隊每日站會同步「誰在改什麼」

### 挑戰三:如何平衡「速度」與「安全」?

**對策:**

- Dev:追求速度 (自動化)
- Staging:平衡 (半自動)
- Production:追求安全 (手動)

**不同環境,不同策略。**

---

## 給團隊的觀念分享

### 流程不是「約束」,是「保護」

很多開發者一開始抗拒流程,覺得「失去了自由」。但當第一次因為 Code Review 發現「差點把 Production 搞掛」時,他們會感謝這個流程。

**流程的目的:**

- 不是「不信任你」
- 而是「多一雙眼睛,少一次事故」

### 文化比工具重要

GitOps 的成功,70% 靠文化,30% 靠工具。

**好的文化:**

- ✅ 鼓勵提問:「為什麼這樣改?」
- ✅ 允許犯錯:「這次學到什麼?」
- ✅ 持續改進:「這個流程可以更好嗎?」

**壞的文化:**

- ❌ 責備:「誰把 Production 搞掛的?」
- ❌ 繞過流程:「我直接 kubectl apply 快一點」
- ❌ 知識壟斷:「只有我知道怎麼部署」

---

## 下週預告

團隊規範建立後,下週要探討 **效能優化與資源限制**。

主題內容:

- ArgoCD 控制器資源消耗分析
- 如何應對大量資源同步的延遲
- Repository 大小對效能的影響

當 Application 數量超過 100 個時,這些問題會變得很關鍵。

---

**作者的話:**  
這週最大的收穫是「看到團隊從抗拒到接受再到主動擁抱流程」。當有人說「謝謝你的 Review,差點忘了改這個配置」時,我知道文化已經建立起來了。

**Tags:** #GitOps #Workflow #Code-Review #Change-Management #Best-Practices
