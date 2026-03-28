# PowerToys Awake Support — #26708

Issue: https://github.com/raycast/extensions/issues/26708

## Overview

為 PowerToys Tool Runner 擴展新增 Awake 工具支援，讓用戶可以透過 Raycast 控制電腦的休眠行為。

## Background

- 擴展安裝量：1,329
- 現有 12 個工具，全部透過 Windows Named Event 觸發（`EventWaitHandle.OpenExisting().Set()`）
- **Awake 不同**：沒有 Named Event，需要透過修改 settings.json 或呼叫 CLI 控制

## Awake 運作方式

PowerToys Awake 有 4 種模式：

| Mode | 說明 |
|---|---|
| `0` - Passive | 不介入，系統正常休眠 |
| `1` - Indefinite | 永遠不休眠 |
| `2` - Timed | 保持清醒指定時間 |
| `3` - Expirable | 保持清醒到指定時間點 |

**Settings 檔案位置：** `%LOCALAPPDATA%\Microsoft\PowerToys\Awake\settings.json`

Settings JSON 結構（`properties` 裡面）：
- `mode` — 模式（0/1/2/3）
- `keepDisplayOn` — 是否同時保持螢幕亮
- `IntervalHours` / `IntervalMinutes` — Timed 模式的時間
- `ExpirationDateTime` — Expirable 模式的到期時間

Awake 進程透過 `FileSystemWatcher` 監聽 settings.json 變化，寫入即生效。

**CLI 方式：** `PowerToys.Awake.exe --time-limit <seconds> --display-on`

## Implementation Plan

### 新增 3 個命令

#### 1. `awake-keep-awake.tsx` — Keep Awake (Indefinite)
- mode: `no-view`
- PowerShell 讀取 settings.json → 改 mode 為 `1` → 寫回
- 同時設定 `keepDisplayOn: true`

#### 2. `awake-keep-awake-timed.tsx` — Keep Awake (Timed)
- mode: `view`（需要表單讓用戶選擇時間）
- 提供預設選項：30 min / 1 hr / 2 hr / custom
- 寫入 mode `2` + `IntervalHours` / `IntervalMinutes`

#### 3. `awake-off.tsx` — Turn Off Awake
- mode: `no-view`
- 寫入 mode `0`（Passive）

### 共用邏輯

新增 `src/utils/awakeSettings.ts`：
- `readAwakeSettings()` — 讀取 settings.json
- `writeAwakeSettings(mode, options)` — 寫入 settings.json
- `isAwakeRunning()` — 檢查 Awake 進程是否在執行
- 路徑：`$env:LOCALAPPDATA\Microsoft\PowerToys\Awake\settings.json`

### package.json 新增

```json
{
  "name": "awake-keep-awake",
  "title": "Keep Awake",
  "description": "Keep your PC awake indefinitely using PowerToys Awake",
  "mode": "no-view",
  "subtitle": "PowerToys",
  "icon": "Awake.png"
},
{
  "name": "awake-keep-awake-timed",
  "title": "Keep Awake (Timed)",
  "description": "Keep your PC awake for a specified duration using PowerToys Awake",
  "mode": "view",
  "subtitle": "PowerToys"
},
{
  "name": "awake-off",
  "title": "Turn Off Awake",
  "description": "Stop keeping your PC awake and return to normal sleep behavior",
  "mode": "no-view",
  "subtitle": "PowerToys",
  "icon": "Awake.png"
}
```

### Assets

需要從 PowerToys repo 取得 Awake icon，或自己做一個符合風格的 icon。

## 注意事項

- 這是擴展中**第一個不用 Named Event 的工具**，用 PowerShell 改 settings.json
- 需要確認 Awake 進程正在執行（`Get-Process -Name 'PowerToys.Awake'` 或只檢查 `PowerToys`）
- Settings.json 可能不存在（用戶從未開啟 Awake），需要處理這個 edge case
- Timed 命令需要 `mode: "view"` 搭配 Raycast Form，打破現有全 `no-view` 的模式

---

## Windows Setup

### 1. Clone（只拿需要的檔案，不帶 git history）

```powershell
# 建立空的 sparse repo
git clone --depth 1 --filter=blob:none --sparse https://github.com/raycast/extensions.git raycast-extensions-powertoys
cd raycast-extensions-powertoys

# 只拉 powertoys-tool-runner 擴展
git sparse-checkout set extensions/powertoys-tool-runner
```

這樣只會下載 `extensions/powertoys-tool-runner/` 目錄的檔案，不含完整 history 和其他幾百個擴展。

### 2. 安裝依賴

```powershell
cd extensions/powertoys-tool-runner
npm install
```

### 3. 開發模式

```powershell
npm run dev
```

Raycast 會自動偵測到本地開發版本。

### 4. 測試前置條件

- 安裝 [Microsoft PowerToys](https://github.com/microsoft/PowerToys/releases)
- 確保 PowerToys 正在執行
- 在 PowerToys Settings 中啟用 Awake 模組
- 確認 settings.json 存在於 `%LOCALAPPDATA%\Microsoft\PowerToys\Awake\settings.json`

### 5. 手動驗證 Awake settings 路徑

```powershell
# 確認檔案存在
Test-Path "$env:LOCALAPPDATA\Microsoft\PowerToys\Awake\settings.json"

# 查看目前設定
Get-Content "$env:LOCALAPPDATA\Microsoft\PowerToys\Awake\settings.json" | ConvertFrom-Json | ConvertTo-Json -Depth 5
```
