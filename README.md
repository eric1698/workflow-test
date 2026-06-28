# GitHub Actions UAT 部署測試

測試「管理者在 PR 中決定要不要把某個分支部署到 UAT，測試 OK 後 merge main 觸發正式機部署」的三種做法。

## 前置作業（只做一次）

1. 在 GitHub 建立一個 **public** repo（private repo 的環境核准需付費方案）。
2. 把本專案推上去：
   ```bash
   git remote add origin https://github.com/<你的帳號>/<repo>.git
   git add .
   git commit -m "init: UAT deploy 三種做法"
   git push -u origin main
   ```
3. 改幾行 `app.txt`、開一個測試分支與 PR，用來實驗。

---

## 做法一：環境核准按鈕（`uat-1-environment-approval.yml`）

**設定**：Repo → Settings → Environments → New environment → 命名 `uat` → 勾選 **Required reviewers** → 加入管理者。

**測試**：開一個 PR → 到 PR 的 Checks 或 Actions 頁，會看到 **Review deployments** 綠色按鈕 → 按 **Approve and deploy** → 才會部署該分支。

**特性**：最原生、有真正的按鈕；但每個 PR 都會自動產生一筆「待核准」。

---

## 做法二：Label 標籤觸發（`uat-2-label.yml`）

**設定**：Repo → Issues/PR → Labels → 新增一個標籤 `deploy-uat`。

**測試**：在某個 PR 右側點選 `deploy-uat` 標籤 → 觸發部署 → 完成後自動移除標籤並留言。要再部署就再加一次標籤。

**特性**：點選即觸發、半個按鈕；多個 PR 各自加標籤即可挑分支。

---

## 做法三：PR 留言指令（`uat-3-comment.yml`）

**設定**：無需額外設定（建議 Settings → Actions → General → Workflow permissions 設為 Read and write）。

**測試**：在任一 PR 的留言區打 `/deploy-uat` → 部署該 PR 的分支到 UAT → 自動回留言告知目前 UAT 上是哪個分支。可隨時重複部署。

**特性**：最靈活、可重複、有完整稽核紀錄、內建權限檢查（write 以上才可部署）。

---

## 共用機制

- 三個 workflow 都用 `concurrency: uat-deploy` 確保**同一時間只部署一個分支到 UAT**（共用環境，後到的排隊）。
- 測試 OK 後 → 把 PR merge 進 `main` → 觸發 `deploy-prod.yml` 部署正式機。
"# workflow-test" 
