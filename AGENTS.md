# AGENTS.md — Local Web Karaoke & 線上點歌本開發規範

## 1. 專案目的 (Project Purpose)
本專案為個人日K系統 **「Local Web Karaoke & Syllable Studio」** 的純靜態展示與線上點歌系統，部署於 GitHub Pages。
核心目標是將**「Denmoku 點歌查詢介面（`index.html`）」**與**「雙行卡拉OK掃描舞台（`player.html`）」**完全解耦，並制定標準化的 JSON 音節資料結構與自動化入庫 SOP，讓 AI 助手能依據同一套規範，快速將新歌曲的 `.lrc` 或音節字幕轉換並擴充至線上歌本中。

---

## 2. 系統架構與目錄結構 (Architecture & File Structure)
本專案採用純靜態前端架構（原生 HTML5 / CSS3 / Vanilla JS + Tailwind CSS CDN），零後端與零編譯依賴，確保可直接於 GitHub Pages 運行：

```text
/
├── AGENTS.md                   # 本專案開發與新歌入庫規範文件
├── index.html                  # 點歌本首頁（讀取 data/songbook.json 提供搜尋與分類篩選）
├── player.html                 # 卡拉OK主舞台與 DAW 音節編輯器（透過 ?file=...&title=... 載入歌曲）
└── data/
    ├── songbook.json           # 全局歌本索引資料庫（模擬靜態資料表）
    └── lyrics/                 # 統一格式的單曲音節時間軸 JSON 庫
        └── <songId>-<slug>.json
```

---

## 3. 視覺風格與 UI 規範 (Design System)
- **整體氛圍**：日系卡拉OK點歌機（Denmoku）暗色風格。
- **色彩配置**：
  - 主背景：深藍漸層 `radial-gradient(circle at top, #1b263b 0%, #070d18 100%)`
  - 卡片容器：`bg-slate-900/90` 搭配 `border-slate-800`
  - 強調色：琥珀金（`#ffaa00` / `amber-400`）用於歌詞掃描填充與曲號標示；緋紅（`#e53e3e` / `rose-600`）用於播放與編輯主控按鈕。
- **字體排版**：日文歌詞區塊優先使用 `"Hiragino Kaku Gothic ProN", "Meiryo", "Noto Sans JP", sans-serif`，確保 `<ruby>` 與 `<rt>` 振假名上下行不重疊。

---

## 4. 統一資料格式規範 (Unified JSON Schema)
為維持播放器載入效能與前端程式碼簡潔，所有歌曲在入庫至 `data/lyrics/` 前，**必須統一轉換為專案標準 JSON 陣列格式**（不可直接將 `.lrc` 或 `.ass` 原始檔作為歌本索引目標）：

```json
[
  {
    "start": 32.05,
    "syllables": [
      { "text": "止", "ruby": "と", "dur": 0.28 },
      { "text": "ま", "ruby": "", "dur": 0.28 },
      { "text": "　", "ruby": "", "dur": 0.35 }
    ]
  }
]
```

### 欄位填寫與轉換規則：
1. **振假名（Ruby）規則**：僅**漢字或熟字訓**需要填寫 `"ruby"` 讀音（例如 `"text": "秒", "ruby": "びょう"`）；平假名、片假名、英數字與標點符號的 `"ruby"` 欄位**一律保持空字串 `""`**。
2. **空白字元規則**：句中用於換氣停頓的全形空白（`　`）或半形空白必須保留為獨立音節物件（`{ "text": "　", "ruby": "", "dur": ... }`），且 `"ruby"` 必須為空。
3. **時間軸保護原則**：轉換人工打軸的 `.lrc` 或 `.ass` 時，必須精準保留原始起始時間 `start`（單位：秒），並依行間時長合理分配單字 `dur`，**嚴禁 AI 自行捏造或篡改已校準的時間標籤**。

---

## 5. 新增歌曲標準作業程序 (SOP for Adding a Song)
當使用者提供新歌曲資訊與歌詞檔案（`.lrc`、`.ass` 或純文字）並要求「入庫新歌」時，AI **必須**依序執行以下三個步驟：

### 步驟 1：分配曲號與檔案命名
- 讀取 `data/songbook.json` 取得目前最大的 `songId`，自動遞增 `1` 作為新曲號（起始編號為 `10001`）。
- 檔名一律採用 `<5位數曲號>-<小寫英文連字號>.json`（kebab-case），例如：`10002-felt-start.json`。

### 步驟 2：轉檔並寫入 `data/lyrics/<songId>-<slug>.json`
- 過濾純間奏空行（無文字內容的時間標籤）。
- 依照第 4 節的 Unified JSON Schema 轉換為標準 JSON 陣列並儲存至 `data/lyrics/` 目錄。

### 步驟 3：同步更新歌本資料庫 (`data/songbook.json`)
- 在 `data/songbook.json` 陣列最前方插入新曲目的 Metadata 物件：
```json
{
  "songId": "10003",
  "title": "歌曲完整名稱",
  "artist": "歌手或社團名稱",
  "category": "東方同人",
  "source": "原曲或作品出處",
  "format": "LRC 人工精校軸",
  "hasRuby": false,
  "lyricFile": "data/lyrics/10003-slug.json",
  "addedDate": "2026-11-22"
}
```

---

## 6. AI 協作與修改禁忌 (Constraints & Rules)
- **前後端職責分離**：新增歌曲時，AI 僅可修改 `data/songbook.json` 與新增 `data/lyrics/*.json`，**嚴禁更動 `player.html` 的 Web Audio 音訊引擎、雙行 `clip-path` 掃描渲染與 DAW 編輯器核心邏輯**。
- **確保相對路徑正確**：`data/songbook.json` 中的 `lyricFile` 必須使用相對於根目錄的路徑（`data/lyrics/...`），確保部署至 GitHub Pages 子路徑時可正常透過 `fetch()` 載入。