# 安裝 Termux + SSH 設定

## 1. 初次設定

打開 Termux 後，執行以下命令：

```bash
# 更新套件管理器
pkg update && pkg upgrade

# 安裝基本工具 + OpenSSH
pkg install -y wget curl git vim openssh
```

## 2. 設定 SSH

建議使用 SSH 進行所有操作，搭配 Tailscale（從 Google Play 安裝）可實現遠端存取。

### 2.1 設定密碼

```bash
# 設定登入密碼
passwd
```

### 2.2 啟動 sshd

```bash
# 啟動一次測試
sshd

# 設定每次開啟 Termux 自動啟動
echo "sshd" >> ~/.profile
```

### 2.3 連線方式

```bash
# 查看手機 IP（區網用）
ifconfig
# 或
ip addr show wlan0

# 從其他設備連線（區網 IP 或 Tailscale IP）
ssh -p 8022 <手機-ip>

# Termux 預設 SSH port 是 8022，不是 22
```

預設使用者名稱可用 `whoami` 查看，連線時例如：

```bash
ssh -p 8022 u0_a123@192.168.1.100
```

## 3. 測試安裝

```bash
# 測試基本命令
ls -la
wget --version
git --version
ssh -V
```

連線成功就完成了，基礎環境到此結束。

## 下一步

繼續 [glibc Loader](./03-glibc-loader.md)。