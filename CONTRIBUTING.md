# GIT 作業標準

制定日期：2026-11-06

本文件規範公司專案的 Git 分支策略、日常開發、發行與緊急修復流程，**所有專案一律採用 Fork 流程。** 指令中的 `<git-server>` 為 REPO 位址，`example/Project` 為公司 Repo，`member/Project` 為個人 Fork，請換成實際名稱。

## 0. 五條鐵則

1. **單向流動**：所有變更只能走 `工作分支 → dev → main`，不可逆流或跳層，Hotfix 也不例外（見第 6 節）。
2. **不直接 commit 或 push 到 `dev`、`main`**：一律透過 PR。
3. **一個 PR 只做一件事**：工作分支要短命，合併後立即刪除，不重複使用。
4. **勤同步**：`dev` 有更新就併進工作分支，趁衝突還小時解決。
5. **不用 Squash，已推送的分支不用 Rebase**：合併一律使用 merge commit。

## 1. 分支模型

```text
feat/* · fix/* · refactor/* · release/* · hotfix/*  ──PR──▶  dev  ──PR──▶  main
```

| 分支 | 角色 | 寫入規則 |
|---|---|---|
| `main` | 正式穩定版，每個 commit 都可出貨 | 只接受來自 `dev` 的 PR；禁止直接 push、force push |
| `dev` | 開發整合與驗收 | 只透過 PR 合併；禁止直接 push、force push |
| `feat/*` | 新功能 | 開發者推到自己的 Fork |
| `fix/*` | 一般 Bug 修復 | 同上 |
| `refactor/*` | 重構 | 同上 |
| `release/*` | 發行準備（改版本號等） | 同上，PR 進 `dev` |
| `hotfix/*` | 線上緊急修復 | 同上，從 `main` 分出、PR 進 `dev` |
| `sync/*` | 把 `main` 併回 `dev` | 同上，PR 進 `dev` |

分支名稱描述工作內容，全小寫並以連字號分隔，例如 `feat/user-auth`、`fix/login-timeout`。

> **關於「main 不領先 dev」**：`dev → main` 的 merge commit 只存在於 `main`，所以平台可能顯示 main 領先幾個 commit，這是正常現象。真正要守住的是：**除了 merge commit，`main` 不能有 `dev` 沒有的 commit。** 可用以下指令檢查，結果應為空白：
>
> ```bash
> git fetch upstream
> git log upstream/dev..upstream/main --no-merges --oneline
> ```

## 2. 環境初始化（每人每專案一次）

1. 在公司 Repo 頁面按 **Fork**，複製到個人帳號。
2. Clone 個人 Repo，並把公司 Repo 設為 `upstream`：

```bash
git clone https://<git-server>/member/Project.git
cd Project
git remote add upstream https://<git-server>/example/Project.git
git fetch upstream
```

3. 若本地還沒有 `dev`，建立一條作為公司 `dev` 的鏡子，不在上面開發：

```bash
git switch -c dev upstream/dev
```

## 3. 日常開發流程

### Step 1：從最新 `dev` 開分支

```bash
git fetch upstream
git switch -c feat/login-page upstream/dev
```

### Step 2：開發與提交

小步提交，每個 commit 專注一項變更。`git add` 前先用 `git status` 確認，避免誤提交無關檔案或機密資料。

```bash
git status
git add <檔案>
git commit -m "feat: 新增使用者登入頁"
```

### Step 3：推送到自己的 Fork

```bash
git push -u origin feat/login-page
```

### Step 4：開 PR

- **Base**：`example/Project` 的 `dev`
- **Head**：`member/Project` 的 `feat/login-page`
- 指定至少 1 位 Reviewer
- PR 描述須說明改了什麼，以及**如何驗證**（有 CI 看 CI；無 CI 則註明實測環境與版本）

### Step 5：開發與審查期間同步 `dev`

`dev` 有更新，或 PR 出現衝突時，在工作分支上併入：

```bash
git fetch upstream
git merge upstream/dev
# 有衝突時：解完後 git add <衝突檔案>，再 git commit
git push origin feat/login-page
```

**不要用 Rebase 或 force push 處理已推送的分支。** Rebase 只用於整理本地尚未推送的 commit。

### Step 6：合併

由管理者確認 Review 與測試通過後，以 **merge commit** 合併進 `dev`。

### Step 7：合併後清理

```bash
git fetch upstream
git switch dev
git merge --ff-only upstream/dev
git branch -d feat/login-page
git push origin --delete feat/login-page
git fetch origin --prune
```

> 來自 Fork 的 PR，公司 Repo 的自動刪除設定不會刪除 Fork 上的分支，所以預設要手動刪除。

**若 `--ff-only` 失敗**，代表本地 `dev` 被動過。先確認沒有要保留的未提交變更，把舊內容備份成分支，再還原：

```bash
git status
git branch backup/dev-local dev
git reset --hard upstream/dev
```

`git reset --hard` 會丟棄所有未提交的變更，執行前務必確認。

## 4. Commit Message 規範

格式為 `<type>: <描述>`，type 用英文，描述用繁體中文。

| type | 用途 | 範例 |
|---|---|---|
| `feat` | 新增功能 | `feat: 新增匯出報表按鈕` |
| `fix` | 修復錯誤 | `fix: 修正手機版導覽列破版` |
| `docs` | 文件修訂 | `docs: 更新 README` |
| `style` | 不影響邏輯的格式調整 | `style: 統一縮排` |
| `refactor` | 重構，未新增功能也未修 Bug | `refactor: 拆分權限檢查模組` |
| `test` | 增修測試 | `test: 補上登入逾時測試` |
| `chore` | 版本號、建置、雜項 | `chore: 版本號更新為 1.2.0` |

## 5. 合併策略與發行流程

### 合併方式

| 情境 | 方式 | 原因 |
|---|---|---|
| 工作分支 → `dev` | **merge commit** | 保留開發脈絡，避免 squash 後歷史對不上 |
| `dev` → `main` | **merge commit** | 清楚標示版本交界 |
| `sync/*` → `dev` | **merge commit** | 同上 |
| `upstream/dev` → 本地 `dev` | `git merge --ff-only` | 本地 `dev` 只是鏡子，不產生分岔 |
| `upstream/dev` → 工作分支 | `git merge` | 不改寫已推送的歷史 |

### 發行步驟（由管理者執行）

1. **更新版本號**：從 `upstream/dev` 開 `release/<版本>`，改完後 PR 進 `dev`。

```bash
git fetch upstream
git switch -c release/1.2.0 upstream/dev
# 修改版本號並 commit
git push -u origin release/1.2.0
```

2. **驗收 `dev`**：有 CI 則跑自動化測試；無 CI 則由管理者取得最新 `dev`，編譯、部署到測試環境，人工確認無回歸問題。
3. **開 Release PR**：`dev → main`，描述列出此次的新功能與修復項目。
4. **合併**：Review 後以 merge commit 合併。
5. **打 Tag**：在 `main` 的新 commit 上標記語意化版本（`v主版號.次版號.修訂號`）。

```bash
git fetch upstream
git tag -a v1.2.0 -m "Release 1.2.0" upstream/main
git push upstream v1.2.0
```

## 6. 緊急 Hotfix（唯一可從 `main` 開分支的情況）

僅限線上緊急修復。Hotfix 從 `main` 開分支，讓修正建立在線上版本上；但和其他變更一樣，**必須先 PR 進 `dev` 測試，再由 `dev` PR 到 `main`**，不可直接 PR 進 `main`。

### Step 1：從 `main` 開分支並修正

```bash
git fetch upstream
git switch -c hotfix/login-crash upstream/main
# 修正並 commit（版本號可在此一併更新為修訂版號）
git push -u origin hotfix/login-crash
```

### Step 2：PR 進 `dev`

- **Base**：`example/Project` 的 `dev`
- **Head**：`member/Project` 的 `hotfix/login-crash`
- 以 merge commit 合併。若與 `dev` 衝突，依第 3 節 Step 5 在 hotfix 分支上併入 `upstream/dev` 解決。

### Step 3：測試 `dev`

依第 5 節發行步驟第 2 步驗收，確認修正有效且沒有回歸問題。

### Step 4：`dev` PR 到 `main` 並打 Tag

依第 5 節發行步驟第 3～5 步，開 `dev → main` 的 PR、以 merge commit 合併，再打修訂版 Tag（例：`v1.2.1`）。

> **注意**：`dev → main` 會一併帶上 `dev` 中所有尚未發行的變更。合併前須確認 `dev` 上的其他變更也已驗收、可以出貨。

## 7. 修復既有分歧（`main` 已有 `dev` 沒有的 commit）

若第 1 節的檢查指令有輸出，代表發生分歧。開一條 `sync/*` 分支，把 `main` 併回 `dev`：

```bash
git fetch upstream
git switch -c sync/main-to-dev upstream/dev
git merge upstream/main
git push -u origin sync/main-to-dev
# 開 PR 進 dev，以 merge commit 合併
```

合併後即恢復單向流動。之後要查明分歧來源，並確認分支保護有正確設定。

## 8. 平行開發與 AI 協作（Git Worktree）

需要同時處理多項工作，或同時使用多個 AI 工具時，不要在同一個目錄反覆切換分支，改用 `git worktree` 為每項工作建立獨立目錄：

```bash
git fetch upstream
git worktree add ../project-login  -b feat/login-page upstream/dev
git worktree add ../project-report -b feat/report-export upstream/dev
```

- 每個目錄用獨立的 IDE 視窗開啟，一個 AI 工具只在一個目錄內作業。
- 分支依**工作內容**命名，不依工具命名。
- 事先分配檔案或模組，避免多項工作同時修改同一份檔案。
- 每項工作仍是獨立分支、獨立 PR，適用本標準所有規則。
- AI 產出的變更同樣須經開發者確認與 Review，才可合併。

PR 合併後清理（回到主目錄執行）：

```bash
git worktree remove ../project-login
git branch -d feat/login-page
git push origin --delete feat/login-page
```

## 9. 平台設定清單（由 Repo 管理者設定）

**合併方式**

- [ ] 只允許 **merge commit**，關閉 squash 與 rebase

**分支保護**（`main` 與 `dev` 各建一條規則）

- [ ] **必須透過 PR 才能合併**（至少 1 位 Reviewer）
- [ ] 有新 commit 推送時，原有的核准失效
- [ ] 所有討論都解決後才能合併
- [ ] **禁止 force push**
- [ ] **保護規則對管理員同樣有效，不可略過**
- [ ] **限制可 push 的人員**：名單只放負責合併的管理者
- [ ] 必須通過 CI 檢查（有 CI 的專案才勾選）
- [ ] （選用）分支須與目標分支同步後才能合併

> **設定完務必實測**：用一般成員帳號試 push 到 `dev`，應被拒絕；再用管理者帳號合併一個測試 PR，應能成功。

## 10. 專案附加規範

本標準適用所有專案。專案特有的規則寫在該專案自己的文件，並且不可與本標準牴觸。
