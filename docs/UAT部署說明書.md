# UAT / 正式機 部署說明書

本專案用 GitHub Actions 實現兩段式部署：

| 階段 | 觸發方式 | 對應檔案 |
|------|---------|---------|
| **部署到 UAT 測試** | 在 PR 點選 `deploy-uat` 標籤 | `.github/workflows/uat-2-label.yml` |
| **部署到正式機** | 把 PR merge 進 `main` | `.github/workflows/deploy-prod.yml` |

日常流程：
**開分支 → 發 PR → 點 `deploy-uat` 標籤上 UAT 測 → 測 OK → merge 進 main → 自動上正式機。**

---

## 一、UAT 部署檔逐段解說（`uat-2-label.yml`）

```yaml
name: UAT Label 部署
```
- **`name`**：這個 workflow 的顯示名稱，會出現在 GitHub 的 Actions 頁與 PR 的 Checks 區。改它只影響顯示，不影響功能。

```yaml
on:
  pull_request:
    types: [labeled]
```
- **`on`**：定義「什麼事件會觸發這個 workflow」。
- **`pull_request: types: [labeled]`**：只有在 PR「**被加上標籤**」這個動作時才觸發。
  - ⚠️ 重要：`pull_request` 類事件，GitHub 讀的是 **PR 來源分支（head）裡的 workflow 檔**，不是 main 的版本。所以改完這支檔案要生效，PR 的分支也得有新版（從最新 main 開分支即可）。

```yaml
concurrency:
  group: uat-deploy
  cancel-in-progress: false
```
- **`concurrency`**：控制「同時最多跑幾個」。同一個 `group` 名稱底下，同一時間只會有一個在跑。
- **`group: uat-deploy`**：因為 UAT 只有一套環境，給它一個固定群組名，確保**多個分支不會同時打 UAT**。
- **`cancel-in-progress: false`**：後來觸發的會**排隊等前一個跑完**，而不是把進行中的取消掉（避免部署到一半被中斷）。

```yaml
permissions:
  contents: read
  pull-requests: write
```
- **`permissions`**：這個 workflow 拿到的 `GITHUB_TOKEN` 權限。
- **`contents: read`**：可以 checkout 程式碼。
- **`pull-requests: write`**：可以操作 PR（這裡用來**移除標籤**）。

```yaml
jobs:
  deploy-uat:
```
- **`jobs`**：一個 workflow 由一或多個 job 組成。
- **`deploy-uat`**：這個 job 的 ID（自己取名）。

```yaml
    if: github.event.label.name == 'deploy-uat'
```
- **`if`**：條件，不成立就跳過整個 job。
- 這行表示：**只有當「被加上的標籤名稱」是 `deploy-uat` 時才執行**。加其他標籤不會觸發部署。

```yaml
    runs-on: ubuntu-latest
```
- **`runs-on`**：在哪種機器上跑。`ubuntu-latest` 是 GitHub 提供的 Linux 虛擬機。其他常見：`windows-latest`、`macos-latest`，或自架 runner `self-hosted`。

```yaml
    steps:
      - name: Checkout 該 PR 的分支
        uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref }}
```
- **`steps`**：job 裡依序執行的步驟。
- **`uses: actions/checkout@v4`**：套用官方的「把程式碼抓下來」動作。
- **`with: ref: ${{ github.head_ref }}`**：指定抓**這個 PR 的來源分支**（而不是 main），這樣部署的才是你要測的那份程式碼。

```yaml
      - name: 部署到 UAT
        run: |
          echo "✅ 部署分支 ${{ github.head_ref }} 到 UAT"
          # 這裡放你真正的 UAT 部署指令
```
- **`run`**：執行 shell 指令。`|` 表示底下可以寫多行。
- 目前只是 `echo` 示意，**這裡就是你要改成真正部署指令的地方**（詳見下方〈二〉）。
- **`${{ github.head_ref }}`**：內建變數，PR 的來源分支名稱。

```yaml
      - name: 移除標籤（標記已處理，可再點一次重新部署）
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.removeLabel({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              name: 'deploy-uat'
            }).catch(() => {});
```
- **`if: always()`**：**不論前面成功或失敗都會執行**這一步（確保標籤一定被移除，下次才能再點一次重新部署）。
- **`actions/github-script@v7`**：讓你用 JavaScript 直接呼叫 GitHub API。
- **`github.rest.issues.removeLabel(...)`**：移除 `deploy-uat` 標籤。
- **`.catch(() => {})`**：就算標籤已經不在、移除失敗，也不要讓整個 job 報錯。

> 💡 部署成功 / 失敗的回饋，直接看 PR 的 **Checks 綠勾 ✅ / 紅叉 ❌**（這是刻意設計的：不發留言、不寄 email，只用 check 當回饋）。

---

## 二、最常見的修改：填入真正的部署指令

把「部署到 UAT」這一步的 `run` 換成你的實際指令。範例：

**例 1：SSH 到伺服器拉最新程式**
```yaml
      - name: 部署到 UAT
        run: |
          ssh user@uat-server "cd /var/www/app && git fetch && git checkout ${{ github.head_ref }} && git pull && pm2 restart app"
```

**例 2：建置後用 rsync 上傳**
```yaml
      - name: 部署到 UAT
        run: |
          npm ci
          npm run build
          rsync -avz ./dist/ user@uat-server:/var/www/app/
```

**例 3：觸發雲端服務（如 Docker / K8s）**
```yaml
      - name: 部署到 UAT
        run: |
          docker build -t myapp:uat .
          docker push myregistry/myapp:uat
          kubectl set image deployment/myapp myapp=myregistry/myapp:uat -n uat
```

### 用到密碼 / 金鑰時：用 Secrets，不要寫死

1. 到 GitHub repo → **Settings → Secrets and variables → Actions → New repository secret** 新增（例如 `UAT_SSH_KEY`、`UAT_HOST`）。
2. 在 workflow 用 `${{ secrets.名稱 }}` 取用：
```yaml
      - name: 部署到 UAT
        env:
          SSH_KEY: ${{ secrets.UAT_SSH_KEY }}
          HOST: ${{ secrets.UAT_HOST }}
        run: |
          echo "$SSH_KEY" > key.pem && chmod 600 key.pem
          ssh -i key.pem user@"$HOST" "deploy.sh"
```
- **`env`**：設定這一步可用的環境變數。

---

## 三、其他常見調整

| 想做的事 | 怎麼改 |
|---------|--------|
| **換標籤名稱**（例如改成 `deploy-test`） | 同時改 `if: github.event.label.name == 'xxx'` 和移除標籤的 `name: 'xxx'`，並到 repo 建立同名標籤 |
| **改用 Windows 機器跑** | `runs-on: windows-latest` |
| **多個 UAT 環境並存**（不想排隊） | 把 `concurrency.group` 改成會變動的值，例如 `group: uat-${{ github.head_ref }}`（每個分支各自一條隊伍） |
| **加「人工核准」才部署** | 在 job 加 `environment: uat`，並到 Settings → Environments 設 Required reviewers |
| **限制只有某些人能部署** | 在部署前加一步用 `github-script` 檢查 `github.actor` 的權限（write 以上才放行） |
| **部署後想自動留言通知** | 加一步 `github.rest.issues.createComment(...)`（注意：會發 email、會在 PR 留言） |
| **指定要部署的 commit 而非分支最新** | checkout 的 `ref` 改成 `${{ github.event.pull_request.head.sha }}` |

---

## 四、正式機部署檔（`deploy-prod.yml`）

```yaml
name: 正式機部署（merge 到 main 觸發）
on:
  push:
    branches: [main]
jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: 部署到正式機
        run: echo "🚀 部署 main 到正式機"
```
- **`on: push: branches: [main]`**：只要有東西被推進 / merge 進 `main` 就觸發（所以 PR merge 後會自動跑）。
- **修改方式**：把「部署到正式機」的 `run` 換成你的正式機部署指令（同〈二〉的寫法，但通常指向正式機伺服器 / 正式環境的 secrets）。

---

## 五、注意事項

1. **workflow 檔的生效範圍**
   - `pull_request` 類（如 UAT 這支）：讀 **PR 來源分支** 的版本 → 改完要從最新 main 開分支才會用到新版。
   - `push` 類（如正式機這支）：讀 **被推送分支（main）** 的版本。

2. **checks 就是回饋**：拿掉留言與 email 後，部署成功與否一律看 PR 的 Checks（綠勾／紅叉），點 Details 可看完整 log。

3. **共用 UAT 會排隊**：因為 `concurrency.group: uat-deploy` 固定，多人同時點標籤時會依序部署，後者覆蓋前者（這是預期行為）。

4. **Windows 本機用 gh 留言的坑**：在 Git Bash 用 `gh pr comment --body "/xxx"` 時，開頭的 `/xxx` 可能被路徑轉換弄亂；在 GitHub 網頁操作則無此問題（本專案目前用標籤觸發，不受影響）。
