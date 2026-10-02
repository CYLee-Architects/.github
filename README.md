標準的 Forking 工作流配合 GitHub 分支保護機制，能確保團隊程式碼品質與發布穩定度。以下為完整的流程設計與保護設定：

---

### 〇、核心鐵則（先讀，違反必衝突）

1. **`dev` / `main` 一律不直接 commit**：這兩個分支只是「公司最新狀態的鏡子」，只拿來同步，不拿來開發。
2. **所有開發都在短命分支**：`feature/*`、`fix/*`、`refactor/*`，一個 PR 對應一條分支。
3. **分支合併後立即刪除、不重複使用**：尤其公司端若曾 squash，舊分支的歷史已和 `dev` 對不上，續用必撞衝突。
4. **勤同步**：開發途中若公司 `dev` 前進了，就把最新 `dev` 併進你的 feature 分支，趁早分批解小衝突，別拖到發 PR 才一次解一大包。

---

### 一、團隊 Git 開發流程（Forking Workflow）

#### 1. 環境初始化（組員一次性設定）

1. **Fork 專案**：組員至公司 Repo (`[github.com/example/Project](https://github.com/example/Project)`) 點選右上角 **Fork**，複製一份到個人 GitHub 帳號。
2. **Clone 至本地並設定 Remote**：

```bash
# Clone 個人 Repo
git clone https://github.com/member/Project.git
cd Project

# 新增公司 Repo 為 upstream 遠端來源
git remote add upstream https://github.com/example/Project.git
```

#### 2. 日常開發步驟

1. **同步最新程式碼**：開發新功能前，先把本地 `dev` 對齊公司 Repo 的 `dev` 分支。

```bash
git fetch upstream
git checkout dev
git merge --ff-only upstream/dev   # 只允許快轉；若失敗代表本地 dev 被動過
                                   # → 以 git reset --hard upstream/dev 還原成鏡子
```

2. **建立功能分支**：一個 PR 對應一條短命分支，合併後即刪除、不重複使用（可避免 squash 後歷史對不上而衝突）。

```bash
git checkout -b feature/login-page
```

3. **提交與推送**：完成開發後，Push 到個人 Repo。

```bash
git add .
git commit -m "feat: add user login page"
git push origin feature/login-page
```

4. **（開發途中）同步公司最新 `dev`**：若開發期間公司 `dev` 有新變更，於 feature 分支上併入，趁早解小衝突。

```bash
git fetch upstream
git merge upstream/dev        # 在 feature 分支上解衝突，而非 dev
git push origin feature/login-page
```

5. **發送 PR**：
   * 前往 GitHub 個人 Repo 頁面，點選 **Compare & pull request**。
   * **base repository**: `example/Project` | **base**: `dev`
   * **head repository**: `member/Project` | **compare**: `feature/login-page`
   * 等待 Code Review 與 CI 檢查通過後，由 Leader 合併至公司的 `dev` 分支。
   * **合併方式**：以 **Create a merge commit** 合入 `dev`（與 Release 一致，**避免 squash**，否則同一份改動會被壓成新 commit，與原分支歷史對不上而在後續衝突）。

6. **合併後收尾**：PR 合併後，切回 `dev` 對齊公司最新，並刪除已合併的 feature 分支（本地與個人 Repo 皆刪）。

```bash
git checkout dev
git fetch upstream
git merge --ff-only upstream/dev
git branch -d feature/login-page            # 刪本地分支
git push origin --delete feature/login-page # 刪個人 Repo 分支
git push origin dev                         # 讓個人 Repo 的 dev 也跟上（選用）
```

---

### 二、DEV 併完後併回 MAIN 的標準步驟（Release SOP）

當 `dev` 分支累積的功能通過測試、準備進行階段性發布時，由管理者執行 Merge 至 `main`：

1. **建立 Release PR**：
   * 在公司 Repo 點選 **New Pull Request**。
   * 設定來源與目標：**base:** `main` ← **compare:** `dev`。
   * 撰寫 Release Note，列出此次更新的 Features 與 Bug fixes。

2. **執行整合測試**：
   * 觸發 CI/CD 自動化測試（如單元測試、E2E 測試、Staging 環境部署）。

3. **執行合併**：
   * 審核無誤後，選擇 **Create a merge commit**（建議保留 Merge 節點以清楚識別版本交界，避免使用 Squash）。

4. **標記版本 Tag（建議）**：
   * 合併完成後，在 `main` 打上語意化版本號（Semantic Versioning）：

```bash
git checkout main
git pull upstream main
git tag -a v1.0.0 -m "Release version 1.0.0"
git push upstream v1.0.0
```

   * 或直接在 GitHub 頁面的 **Releases** 頁面點選 "Draft a new release"，選擇剛合併的 `main` 並發布。

---

### 三、分支合併方式速查

| 合併情境 | 建議方式 | 原因 |
| --- | --- | --- |
| feature → `dev` | **Create a merge commit** | 保留原始 commit，與個人分支歷史一致，避免 squash 後衝突 |
| `dev` → `main`（Release） | **Create a merge commit** | 保留 Merge 節點，清楚識別版本交界 |
| 同步 upstream → 本地 `dev` | **Fast-forward only** (`--ff-only`) | `dev` 只當鏡子，不產生額外 merge 節點、不分岔 |

> ⚠️ **全程避免對 `dev` 使用 Squash merge**：squash 會把多個 commit 壓成一筆新 commit，Git 無法對應回原分支，導致同一份改動在後續被視為不同歷史而反覆衝突。
