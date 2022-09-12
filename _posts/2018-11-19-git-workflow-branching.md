---
layout: post
title: "Git 分支策略與團隊協作"
date: 2018-11-19 14:20:00 +0800
categories: [DevOps, 版本控制]
tags: [Git, Git-Flow, Branching, Collaboration]
---

上週開始學習 DevOps（參考 [DevOps 基礎概念](/posts/2018/11/12/devops-introduction/)），第一步就是要把 Git 用好。以前寫個人專案時，Git 對我來說就是 `git add .`、`git commit -m "update"`、`git push`。但在團隊協作中，這樣根本行不通。

## 我們團隊的 Git 災難

上個月發生了一次慘痛的教訓。團隊五個人同時開發不同功能：

- 小陳在開發使用者登入功能
- 我在實作購物車
- 小王在修復支付頁面的 bug
- 小李在重構資料庫層
- PM 突然說有個緊急修復要上線

結果：
1. 大家都在 `master` 分支上開發
2. 要上緊急修復時，發現 `master` 上有一堆未完成的功能
3. 想 cherry-pick 出需要的 commit，結果改動太多根本挑不出來
4. 最後只好手動比對程式碼，花了三小時才把緊急修復推上去

主管火大：「你們到底有沒有在用版本控制？」

## Git Flow 工作流程

研究了幾天後，發現業界普遍使用 Git Flow 這套分支管理策略。概念是用不同的分支處理不同的任務。

### 主要分支

**master（現在叫 main）**
- 永遠保持穩定，可以隨時部署到生產環境
- 只能從 `release` 或 `hotfix` 分支合併進來
- 每次合併都要打上版本號的 tag，如 `v1.0.0`

**develop**
- 開發分支，整合所有完成的功能
- 最新的開發進度都在這裡
- 功能開發完成後合併到這個分支

### 輔助分支

**feature branches（功能分支）**

從 `develop` 分支出來，開發完成後合併回 `develop`。

```bash
# 開始開發新功能
git checkout develop
git checkout -b feature/user-login

# 開發過程中
git add .
git commit -m "Add login form UI"
git commit -m "Implement authentication logic"

# 開發完成，合併回 develop
git checkout develop
git merge --no-ff feature/user-login
git branch -d feature/user-login
git push origin develop
```

命名規則：`feature/功能描述`，例如：
- `feature/user-authentication`
- `feature/shopping-cart`
- `feature/payment-integration`

**release branches（發布分支）**

準備發布新版本時，從 `develop` 切出來。在這個分支上只能做：
- 修復 bug
- 更新版本號
- 準備發布文件

不能加入新功能！

```bash
# 準備發布 1.2.0 版本
git checkout develop
git checkout -b release/1.2.0

# 修復發現的小問題
git commit -m "Fix typo in error message"
git commit -m "Update version to 1.2.0"

# 合併到 master 並打 tag
git checkout master
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"

# 合併回 develop（可能有 bug 修復）
git checkout develop
git merge --no-ff release/1.2.0

# 刪除 release 分支
git branch -d release/1.2.0
```

**hotfix branches（緊急修復分支）**

生產環境出問題時使用，從 `master` 切出來，修復後合併回 `master` 和 `develop`。

```bash
# 生產環境掛了，緊急修復！
git checkout master
git checkout -b hotfix/fix-payment-bug

# 修復問題
git commit -m "Fix null pointer in payment service"

# 合併到 master
git checkout master
git merge --no-ff hotfix/fix-payment-bug
git tag -a v1.2.1 -m "Hotfix version 1.2.1"

# 合併回 develop
git checkout develop
git merge --no-ff hotfix/fix-payment-bug

# 刪除 hotfix 分支
git branch -d hotfix/fix-payment-bug
```

## 簡化版：GitHub Flow

Git Flow 對小團隊來說有點複雜，GitHub 推薦一種更簡單的流程：

1. `master` 分支永遠可部署
2. 要開發新功能，從 `master` 切出新分支
3. 定期 commit 並 push 到遠端
4. 開 Pull Request 進行 code review
5. review 通過後合併到 `master`
6. 合併後立即部署

適合持續部署（Continuous Deployment）的團隊。

## Commit Message 的藝術

以前我的 commit message：

```
update
fix bug
修改
test
...
```

完全看不出來改了什麼。現在學習使用 Conventional Commits 格式：

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Type 類型

- **feat**：新功能
- **fix**：bug 修復
- **docs**：文件更新
- **style**：程式碼格式調整（不影響功能）
- **refactor**：重構
- **test**：測試相關
- **chore**：建置流程或工具變動

### 範例

```
feat(auth): add JWT token authentication

- Implement token generation with 24h expiration
- Add refresh token mechanism
- Update login API to return JWT token

Closes #123
```

```
fix(payment): handle null pointer in checkout process

Fix crash when user clicks checkout without selecting payment method.
Add validation to ensure payment method is selected.

Fixes #456
```

## 實際應用在我們團隊

重新規劃後的分支策略：

```
master (v1.0.0)
  |
  +-- develop
  |     |
  |     +-- feature/user-profile (小陳)
  |     +-- feature/shopping-cart (我)
  |     +-- feature/payment-fix (小王)
  |
  +-- hotfix/urgent-security-patch (緊急修復)
```

現在如果有緊急修復：
1. 從 `master` 切 `hotfix` 分支
2. 修復並測試
3. 合併回 `master` 和 `develop`
4. 部署

不會再被其他人的半成品功能影響了！

## 好用的 Git 指令

### 查看分支圖

```bash
git log --oneline --graph --all --decorate
```

我設定了 alias：

```bash
git config --global alias.lg "log --oneline --graph --all --decorate"
# 之後只要 git lg 就好
```

### Interactive Rebase 整理 commit

開發完一個功能後，commit 歷史可能很亂：

```
feat: add login form
fix: typo
fix: another typo
refactor: clean up code
fix: forgot to save file
...
```

可以用 interactive rebase 整理：

```bash
git rebase -i HEAD~5
```

會進入編輯器，可以：
- `pick`：保留這個 commit
- `squash`：合併到前一個 commit
- `reword`：修改 commit message
- `drop`：刪除這個 commit

整理後變成：

```
feat(auth): implement user login functionality
```

乾淨多了！

### Stash 暫存變更

開發到一半突然要切到其他分支：

```bash
# 暫存當前變更
git stash save "WIP: implementing shopping cart"

# 切換分支處理其他事情
git checkout hotfix/urgent-fix

# 回來繼續開發
git checkout feature/shopping-cart
git stash pop
```

## 遇到的坑

### 坑一：忘記切分支

在 `develop` 上直接開始寫程式碼，寫到一半才發現。

解決方法：

```bash
# 把變更移到新分支
git stash
git checkout -b feature/new-feature
git stash pop
```

### 坑二：Merge Conflict

多人同時修改同一個檔案，合併時產生衝突。

```
<<<<<<< HEAD
// 我的修改
=======
// 別人的修改
>>>>>>> feature/other
```

沒有捷徑，只能手動解決衝突，然後：

```bash
git add .
git commit
```

## 心得

建立清楚的分支策略後，團隊協作順暢很多。雖然一開始覺得麻煩，但習慣後發現：

1. **職責分明**：每個分支都有明確的用途
2. **可追溯**：清楚知道每個功能的開發歷程
3. **風險降低**：不會因為一個人的錯誤影響整個專案
4. **容易回滾**：出問題可以快速恢復到穩定版本

下週要來研究 Jenkins，把建置和測試自動化，就不用每次都手動打包了。
