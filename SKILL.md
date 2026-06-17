---
name: youtube-downloader
description: '依照使用者偏好下載 YouTube 影片及字幕（預設 720p, 存至 F:\\Download\\YouTube）。'
version: 1.1.0
created_by: agent
---

# YouTube Downloader (YouTube 下載器)

## 適用場景
當使用者請求下載 YouTube 影片或音訊時使用此技能。

## 預設設定 (Defaults)
- **解析度 (Resolution)**: `720p` (除非使用者另有指定)。
- **存放路徑 (Output Path)**: `/mnt/f/Download/YouTube` (對應 Windows `F:\Download\YouTube`)。
- **檔案格式 (Format)**: `.mp4` (透過 ffmpeg 合併)。
- **字幕設定 (Subtitles)**: 
  - **標準模式**: 預設一併下載中文字幕，格式統一為 `.srt`。
  - **安全模式 (Safe Mode)**: 僅下載影片主體，不下載字幕。適用於大量下載或遇到 **HTTP 429 (Too Many Requests)** 時，可有效繞過伺服器限制。

## 實作細節

### 1. 路徑轉換 (Path Resolution)
若使用者提供 Windows 路徑 (例如 `F:\...`)，需將其轉換為 WSL 格式：
- `F:\` 轉換為 `/mnt/f/`
- `D:\` 轉換為 `/mnt/d/`
- 使用 `mkdir -p` 確保目標目錄已存在。

### 2. 畫質與格式選擇 (Quality & Format Selection)
- **JS Runtime 啟動**: 為了確保能偵測到 720p 以上的高畫質格式，指令必須包含 `--js-runtimes node`。
- **畫質選擇器 (Resolution Selectors)**:
  - **720p (相容直式/橫式)**: `-f "bestvideo[width=720]+bestaudio/bestvideo[height=720]+bestaudio/best"` (注意：此設定通常會回傳 **WebM** 格式，因為其品質較佳)。
  - **1080p**: `-f "bestvideo[width=1080]+bestaudio/bestvideo[height=1080]+bestaudio/best"`
  - **最高畫質 (Best)**: `-f "bestvideo+bestaudio/best"`
  - **僅音訊 (Audio only)**: `-x --audio-format mp3`

**⚠️ 格式與畫質的權衡陷阱 (The Format vs. Resolution Trade-off)**:
YouTube 的 MP4 (H.264) 軌道通常最高僅提供至 360p 或 480p。若強行指定 `[ext=mp4]`，`yt-dlp` 會為了滿足格式要求而**自動降級畫質**。

- **需求 A：只要 MP4，畫質隨緣** $\rightarrow$ 使用 `-f "bestvideo[height<=720][ext=mp4]+bestaudio[ext=m4a]/best[height<=720][ext=mp4]"`。
- **需求 B：必須 720p 以上且必須是 MP4** $\rightarrow$ **唯一可靠路徑**：
  1. 下載 WebM 版本 (保證有 720p+) $\rightarrow$ 2. 使用 `ffmpeg` 轉碼為 MP4。
  - **轉碼指令範例**: `ffmpeg -i input.webm -c:v libx264 -preset fast -crf 22 -c:a aac -b:a 128k output.mp4`

### 2. 畫質與格式選擇 (Quality & Format Selection)
... [Existing content] ...

**⚠️ 檔名陷阱 (Naming Pitfall)**:
禁止使用 Shell 變數（如 `$(yt-dlp --get-title)`）來獲取標題並命名檔案。若 `yt-dlp` 輸出警告訊息（WARNING），該訊息會被誤抓入變數，導致檔名變成日期或其他錯誤文字。
- **正確做法**：始終使用 `yt-dlp` 內建的輸出模板：`-o "/mnt/f/Download/YouTube/%(title)s.%(ext)s"`。

**⚠️ 直式影片 (Shorts) 陷阱**:
... [Existing content] ...
直式影片的解析度是 `寬x高` (例如 720x1280)。若僅指定 `height=720` 會導致篩選失敗並回退至低畫質。請務必使用 `[width=720]` 或上述的相容選擇器。詳情參閱 `references/youtube-shorts-resolution.md`。

### 3. 字幕處理 (Subtitle Processing)
- **下載邏輯**: 優先下載原生中文字幕，其次是自動生成中文字幕，最後是自動翻譯為中文。
- **對應參數**:
  - `--write-subs`: 下載原生字幕。
  - `--write-auto-subs`: 下載自動生成字幕。
  - `--sub-langs "zh-Hant,zh-Hans,en"`: 優先語言順序。
  - `--convert-subs srt`: 統一轉換為 `.srt` 格式。
- **高品質翻譯 (Option B)**: 若使用者要求「AI 重新翻譯」，則下載英文 `.srt`，由 LLM 翻譯後覆蓋原檔。

### 4. 背景執行 (Non-blocking Execution) ⚡
**多人使用環境 — 下載任務一律採用 `background=true` + `notify_on_complete=true`，絕對不阻塞當前對話。**

```
terminal(
  command="~/.local/bin/yt-dlp ...",
  background=true,
  notify_on_complete=true,
  timeout=600
)
```

### 6. 片段截取與暫存檔清理 (Video Trimming & Cleanup) ♻️
當使用者要求截取影片特定時間段（例如 `06:35-07:24`）時，必須執行「閉環清理」以避免大檔案佔用磁碟空間：
- **暫存邏輯**: 將完整影片下載至 `/tmp/` 目錄（或其他臨時路徑）。
- **截取邏輯**: 使用 `ffmpeg` 從暫存檔截取片段 $\rightarrow$ 輸出至最終目標路徑（如 `F:\Download\YouTube`）。
- **清理邏輯**: 截取完成後，**必須立即**執行 `rm` 刪除 `/tmp/` 中的完整暫存檔。
- **指令範例**: `yt-dlp ... -o "/tmp/temp_vid.mp4" [URL] && ffmpeg -i /tmp/temp_vid.mp4 -ss [START] -to [END] -c copy [FINAL_PATH] && rm /tmp/temp_vid.mp4`

## 執行警告 (Execution Warning)
**⚠️ 避免直覺偏差 (Intuition Trap)**：即使對 `yt-dlp` 或相關工具非常熟悉，也必須首先載入此 Skill 並對照 SOP。禁止在已記錄的已知陷阱（如 JS Runtime 導致的高畫質遺失、直式影片解析度判定）上進行重複的「試錯 $\rightarrow$ 修正」循環。優先依賴 SOP 中的「精確打擊」方案而非即時推理。

## 執行流程 (Workflow)
1. **解析請求**: 確認影片網址 (URL)、要求的解析度、存放路徑及是否需要特殊翻譯。**若涉及時間截取，需啟用閉環清理流程。**
2. **套用預設值**: 
   - 若未指定解析度：使用 `720p`。
   - 若未指定路徑：使用 `/mnt/f/Download/YouTube`。
   - 若未指定格式：使用 `.mp4`。
3. **確保目錄存在**: 執行 `mkdir -p [OUTPUT_PATH]`（前景，秒回）。
4. **背景執行 (非阻塞)**: 透過 `terminal(background=True, notify_on_complete=True)` 呼叫 yt-dlp 或截取指令鏈。**立即回覆使用者「已開始下載/處理」**，待完成後系統會自動通知結果。
5. **後處理 (選配)**: 若使用者要求 AI 翻譯，則對下載的字幕檔執行翻譯流程（同樣使用背景執行）。
6. **交付結果**: 通知使用者最終檔案路徑與字幕狀態。

### 4. 故障排除與 429 錯誤應對 (Troubleshooting)
- **HTTP 429 (Too Many Requests)**: 當 YouTube 伺服器偵測到過多請求（尤其是字幕端點）時會觸發封鎖。
- **應對策略**:
    1. **優先順序降級**: 若包含字幕的完整指令失敗，立即嘗試「僅下載影片」模式（移除 `--write-subs` 與 `--write-auto-subs`），優先確保主體檔案下載。
    2. **簡化格式篩選**: 使用 `-f "best"` 取代複雜的格式過濾器，減少與 API 的交互次數。
    3. **等待重試 (Back-off)**: 若所有模式均失敗，建議等待 5-10 分鐘後再試。
- **JS Runtime 警告**: 若出現 `No supported JavaScript runtime could be found`，可能會導致 `yt-dlp` 在自動篩選時無法偵測到 720p 及以上的高畫質格式（被 YouTube 隱藏在 JS 端點後）。詳見 `references/js-runtime-resolution-issue.md`。
    - **解決方案 (精確打擊)**：執行 `yt-dlp -F` 找出高品質格式的具體 ID $\rightarrow$ 使用 `-f "[VideoID]+[AudioID]"` 強制指定下載，繞過自動篩選邏輯。
