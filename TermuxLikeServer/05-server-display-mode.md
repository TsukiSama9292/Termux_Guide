# ROG 5s Server Display Mode（長時間開機實踐）

目標：插著電長期當小型電腦，同時滿足 Bypass Charging、螢幕保持亮著、Termux 持續運作、OLED 不烙印、功耗盡量低。

## 1. 架構總覽

```text
USB 電源
   │
   ▼
ROG Phone 5s ── 遊戲中心 (Game Genie) ── OLED Guard ── 路旁充電（整機外部供電）
   │
   ├── OLED Guard（原生 App，前台畫面）
   │      ├── 沉浸式全螢幕：收掉通知列 / 導航列 / 手勢 pill
   │      ├── 極暗緩慢色場，無文字無按鈕
   │      ├── 視窗最低亮度 + 保持亮屏
   │      └── 60 Hz 面板
   │
   └── Termux（背景跑 server，不需開在前景）
          ├── Wake Lock（CPU 不睡）
          ├── sshd（見 02）
          ├── dokid（見 04）
          └── Tailscale / HTTP server
```

六個條件一次看：

| # | 條件 | 做法 |
|---|------|------|
| 1 | Bypass Charging | §3，OLED Guard 加入遊戲庫，從遊戲啟動後開路旁充電 |
| 2 | 螢幕保持亮 | OLED Guard `FLAG_KEEP_SCREEN_ON`，開著 App 就亮著 |
| 3 | Termux 持續運作 | §5，Wake Lock + `~/.profile` 自啟，背景跑即可 |
| 4 | 防烙印 | §2，OLED Guard：無文字色場 + 收掉系統列 |
| 5 | 降螢幕功耗 | §4，60 Hz + 系統亮度最低 + App 視窗最低亮度 |
| 6 | 降實際刷新率 | §4，系統設定鎖 60 Hz（不是靠 App 少更新） |

關鍵觀念：**App 每秒只畫 0.5 次 ≠ 面板變成 0.5 Hz**。面板刷新率由系統決定，
所以做法是「面板鎖 60 Hz + App 約 0.5 FPS」，兩個分開處理。

## 2. OLED Guard（螢幕保護核心）

自製原生 Android App：[TsukiSama9292/OLED-Guard](https://github.com/TsukiSama9292/OLED-Guard)。

### 2.1 為什麼不用 Termux 方案

之前試過兩版 Termux 終端機方案（`rog-dashboard.sh` 文字看板、
`oled-distributor.py` 終端機色場，留在使用紀錄見 §8），實測卡在同一個天花板：

- Android 的**通知列和導航列** terminal 收不掉，長期亮著就是兩條固定烙印帶
- Termux v0.118.3（2025 更新）本身更新極慢，導航列相關問題回報了也沒修沒編譯
- 終端機拿不到視窗級最低亮度，也做不到真正的沉浸式全螢幕

這些只有原生 App 解得掉，所以螢幕這層改用 OLED Guard，Termux 退回只跑 server。

### 2.2 運作方式

- 4 個大型柔邊暗色場（暗藍 / 暗紫 / 暗綠 / 暗紅）在極暗中性灰底（RGB 6）上疊加，
  無銳利邊緣、無純黑區、無可辨識圖形
- 每個色場獨立慢速隨機漫步，每 ~30–80 秒換一次目標，目標可稍微超出螢幕邊緣掃過角落
- 變色：每 ~3–6 分鐘只選一個色場、每通道 ±3 內微調，無彩虹循環、無突變
- 更新率 ~0.5 FPS（每 2 秒一幀），醒來畫一幀就休眠，無 busy loop
- 所有顏色箝在 RGB 4–18，疊加不可能產生亮像素，不用 HDR
- App 強制**視窗最低亮度**（非 root，只作用於本視窗），系統亮度另外拉到最低（見 §4）
- 啟動即沉浸式全螢幕 + 常亮，無需 root / ADB / 網路；正常運作時畫面零文字零按鈕

### 2.3 安裝

```bash
# 取 APK（自己編譯，編譯指南見專案 README；或用已編好的）
app/build/outputs/apk/debug/app-debug.apk
```

裝到手機上，從 launcher 啟動即可，**無需任何操作**，放著就是保護畫面。
可調參數（更新間隔、色場數、亮度箝制、漫步/變色週期）都在
`OledDistributorView.java` 頂部的 `static final`，改完重編就行。

## 3. 遊戲中心設定（關鍵）

只把 App 裝好不夠，要讓整機進入外部供電 + 遊戲模式，設定一次即可：

```text
1. 遊戲中心 / Armoury Crate → 遊戲庫 → 把 OLED Guard 加進去
2. 從遊戲庫啟動 OLED Guard（一定要從這裡啟動，遊戲設定才會套用）
3. 滑出 Game Genie 工具列 → 開啟「路旁充電」（Bypass Charging）
4. 同一列 → 關閉「防誤觸」（不碰螢幕時才不會跳出防誤觸遮罩）
```

驗證：插著電放 10 分鐘，電量百分比**不上升**就是路旁充電生效中，
整機由外部電源供電，背景的 Termux server 不耗電池。

## 4. 螢幕：鎖 60 Hz + 亮度最低

### 4.1 刷新率鎖 60 Hz

```text
設定 → 顯示 → 螢幕更新率 → 60 Hz
```

或建一個專用模式：

```text
設定 → 電池 → System modes → 新增 SERVER 模式
  → Screen refresh rate = 60 Hz
  → Color scheme = 深色
  → Screen sleep = 永不（OLED Guard 本來就會常亮，這是雙保險）
  → Brightness = 最低
```

ROG 5s 面板最高 144 Hz，原生提供 60 / 90 / 120 / 144 Hz，
當 server 用 60 Hz 就好，90 以上沒意義。

### 4.2 亮度拉到最低

App 已經強制視窗最低亮度，系統亮度也手動拉到底，兩層疊加才是最暗。

### 4.3 驗證真的跑 60 Hz

```text
設定 → 關於手機 → 連點版本號碼 7 次 → 開啟開發人員選項
開發人員選項 → Show refresh rate（在螢幕上顯示目前 Hz）
```

看到 60 就對了。這個開關只是顯示用，確認完可以關掉。

## 5. Termux 保活：Wake Lock + 自啟（背景即可）

螢幕亮著不代表 CPU 不會睡，server 一定要拿 Wake Lock。
OLED Guard 開在前景時，Termux 縮到背景跑就行：

```bash
pkg install -y termux-api
termux-wake-lock
```

確認通知欄出現 Termux wakelock 通知即成功。重開機後要重新下一次，
要在手機本機下（SSH 進來下是沒用的）。
sshd + dokid 的自啟寫法見
[02](./02-install-termux.md) 和 [04 §4.1](./04-doki.md#41-開機--每次進-termux-自動啟profile-正確寫法)。

日常操作開另一個 SSH 連線進來，手機螢幕保持 OLED Guard 全螢幕，互不干擾。

## 6. 功耗驗證（建議做一次）

固定條件（最低亮度、OLED Guard 色場 + 路旁充電），只換刷新率，
看 USB 電表或掉電速度：

| 實驗 | 刷新率 | 亮度 | 畫面 | 預期 |
|------|--------|------|------|------|
| 1 | 144 Hz | 最低 | 色場 | 最耗電 |
| 2 | 120 Hz | 最低 | 色場 | 次之 |
| 3 | 90 Hz | 最低 | 色場 | 中等 |
| 4 | 60 Hz | 最低 | 色場 | **推薦，最省** |

結論應該是：60 Hz + 最低亮度 + OLED Guard（0.5 FPS）+ 路旁充電最接近最佳點。

## 7. 疑難排解

- 邊緣手勢喚出的系統列：App 會自動重新隱藏，不用理它；
  若 SystemUI 硬要顯示手勢 pill，屬系統行為，不 root 無解，影響很小。
- `dns 8053 / 7432 bind: address already in use`：舊 dokid 還在，
  見 [04 §4.1](./04-doki.md#41-開機--每次進-termux-自動啟profile-正確寫法)，
  `~/.profile` 要加 `doki ping` 判斷，不要每次 SSH 多開一個。
- 螢幕還是會睡：檢查 System modes / 省電模式有沒有覆蓋掉螢幕逾時
 （OLED Guard 常亮 + 永不休眠是雙保險，兩個都設）。
- 防誤觸遮罩跳出來：回 §3 第 4 步，確認是從遊戲庫啟動且防誤觸已關。

## 8. 退役紀錄（Termux 螢幕方案）

- `scripts/rog-dashboard.sh`：文字看板版。死因：Android 10+ 擋 `/proc/stat`
 （CPU 永遠 0.0%）、thermal sentinel 假值（125°C）、網卡名稱不固定，
  且固定文字本身就是烙印風險。
- `scripts/oled-distributor.py`：終端機色場版（Python，RGB 4–22，0.5 FPS，
  隨機漫步 + synchronized output）。邏輯驗證通過，但死於天花板：
  通知列 / 導航列收不掉，Termux v0.118.3 相關問題長期沒修。
- `~/.termux/termux.properties` 的 `fullscreen=true` / `extra-keys = []`：
  OLED Guard 自己做沉浸式，不再需要。

以上三者都已被原生 App 取代，留檔備查，不用再傳到手機上。
