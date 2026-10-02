標準的 Forking 工作流配合 GitHub 分支保護機制，能確保團隊程式碼品質與發布穩定度。以下為完整的流程設計與保護設定：

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

1. **同步最新程式碼**：開發新功能前，先更新公司 Repo 的 `dev` 分支。
```bash
git fetch upstream
git checkout dev
git merge upstream/dev

```

2. **建立功能分支**：建立獨立 Feature 分支避免 squash 失敗。
```bash
git checkout -b feature/login-page

```

3. **提交與推送**：完成開發後，Push 到個人 Repo。
```bash
git add .
git commit -m "feat: add user login page"
git push origin feature/login-page

```

4. **發送 PR**：
* 前往 GitHub 個人 Repo 頁面，點選 **Compare & pull request**。
* **base repository**: `example/Project` | **base**: `dev`
* **head repository**: `member/Project` | **compare**: `feature/login-page`
* 等待 Code Review 與 CI 檢查通過後，由 Leader 合併至公司的 `dev` 分支。

---

### 二、DEV 併完後併回 MAIN 的標準步驟（Release SOP）

當 `dev` 分支累積的功能通過測試、準備進行階段性發布時，由管理者執行 Merge 至 `main`：

1. **建立 Release PR**：
* 在公司 Repo 點選 **New Pull Request**。
* 設定來源與目標：**base: `main**` $\leftarrow$ **compare: `dev**`。
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
