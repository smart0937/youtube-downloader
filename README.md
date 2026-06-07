# YouTube 下載器（Hermes Agent 技能）

本存放庫為 `youtube-downloader` 技能的備份，用於 Hermes Agent。

## 📌 概述

專用的 YouTube 下載技能，讓 AI 代理根據預設的工程化參數下載影片與字幕。

## ✨ 主要功能

- **自訂解析度**：支援 720p、1080p 或最高畫質。
- **彈性路徑**：整合 WSL，直接將檔案儲存至 Windows 磁碟機（例如 `F:\Download\YouTube`）。
- **智慧字幕**：
  - 自動下載原生或機器產生的字幕。
  - 優先順序：繁體中文、簡體中文、英文。
  - 統一轉為標準 `.srt` 格式。
- **AI 翻譯**：支援以 LLM 對英文字幕進行高品質重新翻譯。
- **非阻塞背景執行**：採用 `background=true` + `notify_on_complete=true`，多人共用的環境中不阻擋當前對話。

## 🛠 實作技術

技能使用 `yt-dlp` 與 `ffmpeg` 處理影片合併及字幕轉換。

---

*由 Hermes Agent 自動備份產生。*
