# 先決條件

## 硬體需求

### 最低需求
- **處理器**：ARM64 (AArch64) 架構
- **記憶體**：4GB RAM 以上
- **儲存空間**：10GB 可用空間
- **Android 版本**：7.0 (Nougat) 以上

### 推薦配置
- **處理器**：Snapdragon 8 系列或同等級
- **記憶體**：8GB RAM 以上
- **儲存空間**：64GB 以上
- **Android 版本**：12 (Material You) 以上

### 測試設備
本指南在以下設備上測試通過：
- ASUS ROG Phone 5s (ZS676KS)
- Snapdragon 888+
- 16GB RAM
- Android 13

## 軟體準備

### 1. 安裝 Termux

**重要**：請從 F-Droid 安裝，不要從 Google Play Store 安裝。

#### 為什麼不從 Play Store 安裝？
- Play Store 版本過舊
- 可能包含廣告或追蹤器
- F-Droid 版本更新、更穩定

#### 安裝步驟
1. 下載 [F-Droid](https://f-droid.org/packages/com.termux/)
2. 安裝 F-Droid APK
3. 在 F-Droid 中搜尋 "Termux"
4. 安裝 Termux

### 2. 安裝 Termux:Boot (可選)

如果希望開機自動啟動服務：
1. 從 F-Droid 安裝 [Termux:Boot](https://f-droid.org/packages/com.termux.boot/)
2. 打開 Termux:Boot 一次（建立資料夾）
3. 建立啟動腳本

### 3. 安裝 Termux:API (可選)

如果需要存取 Android 功能：
- 從 F-Droid 安裝 [Termux:API](https://f-droid.org/packages/com.termux.api/)
- 安裝套件：`pkg install termux-api`

## 更新套件管理器

```bash
pkg update && pkg upgrade
```

## 建立目錄結構

```bash
# 建立伺服器目錄
mkdir -p ~/server/{bin,services,configs,volumes,cache,backups}

# 建立子目錄
mkdir -p ~/server/bin/{native,glibc}
mkdir -p ~/server/services/{nginx,redis,postgresql,node}
mkdir -p ~/server/configs/{nginx,redis,postgresql,node}
mkdir -p ~/server/volumes/{nginx,redis,postgresql,apps}
mkdir -p ~/server/cache/{nix,oci,downloads}
```

## 網路設定

### 本地端存取

預設情況下，所有服務都監聽 `127.0.0.1` (localhost)。

### 區網存取

如果需要從其他設備存取：

```bash
# 查看手機 IP
ifconfig

# 或
ip addr show wlan0
```

### 外網存取

**警告**：不建議直接暴露服務到外網。

如果需要，請考慮：
1. 使用 VPN (如 WireGuard)
2. 使用反向代理 (如 ngrok)
3. 使用 Cloudflare Tunnel

## 安全建議

### 1. 使用 SSH 金鑰

```bash
# 安裝 OpenSSH
pkg install openssh

# 產生金鑰
ssh-keygen -t ed25519

# 複製公鑰
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
```

### 3. 定期更新

```bash
# 更新所有套件
pkg update && pkg upgrade

# 更新特定套件
pkg upgrade postgresql
```

## 下一步

完成上述設定後，請繼續 [安裝 Termux](./02-install-termux.md)。