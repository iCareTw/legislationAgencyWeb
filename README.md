# 立法委員質詢統計視覺化網頁

立法委員在立法院會議中的質詢發言統計與視覺化工具。使用 Vue 3 + Vite 建構的單頁式應用程式 (SPA)。

## 功能特色

- 依委員會篩選立委發言記錄 (多選)
- 依發言次數、總時長、平均時長排序
- 即時計算篩選後的統計數據
- 響應式設計，支援桌面與行動裝置

## 快速開始

### 開發模式

```bash
# 安裝依賴
npm install

# 啟動開發伺服器 (http://localhost:3000)
npm run dev
```

### 生產建置

```bash
# 建置生產版本
npm run build

# 預覽建置結果
npm run preview
```

## 資料依賴

本專案顯示**預先聚合的統計資料**，資料更新流程如下：

```
legislationAgencyCrawler (爬取原始資料)
    ↓
legislationAgencyAnalytics (生成統計)
    ↓
public/data/aggregated.json (本專案讀取)
```

**資料檔案**：
- `public/data/aggregated.json` - 立委發言統計 (由 Analytics 生成)
- `public/data/committees.json` - 委員會清單 (symlink 到 `configs/committees.json`)

**更新資料**：
```bash
# 1. 爬取最新會議資料
cd ../legislationAgencyCrawler
python crawl_meeting.py update

# 2. 生成統計資料
cd ../legislationAgencyAnalytics
python run_speech_stat.py update
python aggregate_stats.py

# 3. 重新整理網頁即可看到更新
```

## 專案結構

```
legislationAgencyWeb/
├── src/
│   ├── App.vue       # 主應用程式元件 (包含所有功能)
│   ├── main.js       # Vue 應用程式初始化
│   └── style.css     # 全域樣式
├── public/
│   └── data/         # 靜態 JSON 資料
│       ├── aggregated.json    # 立委統計資料
│       └── committees.json    # 委員會清單 (symlink)
├── vite.config.js    # Vite 建置設定
└── package.json      # 專案設定
```

## 技術架構

- **Vue 3** - Composition API (`<script setup>`)
- **Vite** - 快速建置工具
- 無路由 (單頁式應用)
- 無 CSS 框架 (使用 Vanilla CSS)

## 部署說明

建置後的檔案位於 `dist/` 目錄，可部署至任何靜態網站託管服務：

```bash
npm run build
# 上傳 dist/ 目錄內容至伺服器
```

**注意事項**：
- `vite.config.js` 已設定 `base: './'` 支援部署至子目錄
- 確保 `public/data/` 目錄的 JSON 檔案存在且為最新

## 開發指引

開發者請參考 [CLAUDE.md](CLAUDE.md) 了解：
- Vue 元件架構
- 資料流程與篩選邏輯
- 與其他子專案的整合方式
