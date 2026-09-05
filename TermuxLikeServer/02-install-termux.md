# 安裝 Termux

## 初次設定

打開 Termux 後，執行以下命令：

```bash
# 更新套件管理器
pkg update && pkg upgrade

# 安裝基本工具
pkg install -y wget curl git nano vim

# 安裝 termux-services (服務管理器)
pkg install termux-services
```

**重要**：安裝 termux-services 後，必須**重新打開 Termux** 才能啟用 runit。

## 設定 SSH

**強烈建議**：使用 SSH 進行所有操作，搭配 Tailscale ( Google Play 安裝) 實現遠端存取。

### 1. 安裝並登入 Tailsacle

### 2. 設定密碼

```bash
# 設定登入密碼
passwd
```

### 3. 啟動 sshd

```bash
echo "sshd" >> ~/.profile
```

### 4. 連線方式

```bash
# 從其他設備連線 (使用 Tailscale IP)
ssh <tailscale-ip>

# 或從同一網路連線
ssh localhost
```

## 安裝 glibc 支援

### 選擇一：使用 glibc-repo (推薦)

```bash
# 安裝 glibc-repo
pkg install glibc-repo -y

# 安裝 glibc-runner
pkg install glibc-runner -y

# 測試
grun --help
```

### 選擇二：使用 Bionilux (更完整)

```bash
# 快速安裝
curl -sL theonuverse.github.io/bionilux/setup | bash
```

### 選擇三：手動安裝 glibc

```bash
# 安裝 glibc
pkg install glibc-repo -y
pkg install glibc -y

# 設定環境
echo 'export GLIBC_PREFIX="$PREFIX/glibc"' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH="$GLIBC_PREFIX/lib:$LD_LIBRARY_PATH"' >> ~/.bashrc
```

## 測試安裝

### 1. 測試基本命令

```bash
# 測試 ls
ls -la

# 測試 wget
wget --version

# 測試 git
git --version
```

### 2. 測試 glibc (如果已安裝)

```bash
# 使用 glibc-runner
glibc-runner --help
```

## 常見問題

### 1. 無法更新套件

```bash
# 清除快取
pkg clean

# 重新設定套件
pkg install apt-transport-https
pkg update
```

### 2. 無法安裝 glibc

```bash
# 確認架構
uname -m

# 應該顯示 aarch64

# 如果不是，可能需要安裝 arm 版本
pkg install glibc-repo-arm -y
```

### 3. 服務無法啟動

```bash
# 檢查日誌
ls -la $PREFIX/var/log/sv/

# 查看特定服務日誌
cat $PREFIX/var/log/sv/sshd/current
```

## 下一步

完成 Termux 安裝後，請繼續 [安裝服務](./03-install-services.md)。