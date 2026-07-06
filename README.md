# 訓練紀錄

簡單、快速的健身訓練 log 工具。

🔗 **https://fittinglog.jimmyhusaas.workers.dev**

## 功能

- 紀錄每日訓練動作、組數、重量、次數
- 自動計算當日總訓練量
- 常用動作快速選單
- 歷史紀錄瀏覽與編輯
- 資料存於本地瀏覽器（localStorage），無需帳號

## 技術

純靜態單頁應用，無 build step。

- React 18（CDN）
- localStorage 持久化
- 部署於 Cloudflare Workers（原 GitHub Pages 因 backend stuck state 遷離）
