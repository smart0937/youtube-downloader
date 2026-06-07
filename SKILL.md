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
- **字幕設定 (Subtitles)**: 預設一併下載中文字幕，格式統一為 `.srt`。

## 實作細節

### 1. 路徑轉換 (Path Resolution)
若使用者提供 Windows 路徑 (例如 `F:\...`)，需將其轉換為 WSL 格式：
- `F:\` 轉換為 `/mnt/f/`
- `D:\` 轉換為 `/mnt/d/`
- 使用 `mkdir -p` 確保目標目錄已存在。

### 2. 畫質選擇 (Quality Selection)
- **720p**: `-f "bestvideo[height<=720]+bestaudio/best[height<=720]"`
- **1080p**: `-f "bestvideo[height<=1080]+bestaudio/best[height<=1080]"`
- **最高畫質 (Best)**: `-f "bestvideo+bestaudio/best"`
- **僅音訊 (Audio only)**: `-x --audio-format mp3`
- **指定 ID**: 若使用者透過 `-F` 清單選擇，則使用 `-f [ID]`。

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

### 5. 執行指令 (Execution Command)
基礎指令格式（包含字幕）：
`~/.local/bin/yt-dlp [QUALITY_FILTER] --merge-output-format mp4 --write-subs --write-auto-subs --sub-langs "zh-Hant,zh-Hans,en" --convert-subs srt -o "[OUTPUT_PATH]/%(title)s.%(ext)s" [URL]`

## 執行流程 (Workflow)
1. **解析請求**: 確認影片網址 (URL)、要求的解析度、存放路徑及是否需要特殊翻譯。
2. **套用預設值**: 
   - 若未指定解析度：使用 `720p`。
   - 若未指定路徑：使用 `/mnt/f/Download/YouTube`。
   - 若未指定格式：使用 `.mp4`。
3. **確保目錄存在**: 執行 `mkdir -p [OUTPUT_PATH]`（前景，秒回）。
4. **背景下載 (非阻塞)**: 透過 `terminal(background=True, notify_on_complete=True)` 呼叫 yt-dlp。**立即回覆使用者「已開始下載」**，待完成後系統會自動通知結果。
5. **後處理 (選配)**: 若使用者要求 AI 翻譯，則對下載的字幕檔執行翻譯流程（同樣使用背景執行）。
6. **交付結果**: 通知使用者最終檔案路徑與字幕狀態。

## 注意事項 (Pitfalls)
- **非阻塞優先**: 多人環境中，長時間任務（下載 >10s、轉錄等）**必須**用 `background=true`，避免被其他使用者的聊天打斷。
- **權限問題**: 絕對不要使用 `sudo` 執行 `yt-dlp`，請直接使⽤ `~/.local/bin/` 下的二進位檔。
- **FFmpeg 依賴**: 高畫質影片與字幕轉換需要 `ffmpeg`，執行前請確認 `ffmpeg` 已安裝。
- **特殊字元**: YouTube 標題常包含特殊字元，`yt-dlp` 會自動處理，但請確保在 shell 指令中對路徑使用引號 (`" "`).
