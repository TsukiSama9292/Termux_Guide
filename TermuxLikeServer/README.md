# Android Phone Server Guide（基礎版）

在未 Root 的 Android 手機上，使用 Termux + SSH 建立可遠端連線的環境。

## 核心原則

- **無 Root**：不需要解鎖 bootloader 或 root 權限
- **基礎 + 容器**：先搞定安裝 + SSH + glibc 觀念，再用 Doki 跑真正的 OCI 容器

## 文件導覽

1. [先決條件](./01-prerequisites.md) - 硬體需求、從 F-Droid 安裝 Termux
2. [安裝 Termux + SSH 設定](./02-install-termux.md) - 基本設定、密碼、sshd、連線
3. [glibc Loader](./03-glibc-loader.md) - 執行 Linux ARM64 二進制檔案
4. [在 Termux 運行 Doki](./04-doki.md) - 拉 OCI image、跑容器、Compose / K8s
5. [ROG 5s Server Display Mode](./05-server-display-mode.md) - 長時間開機：OLED-Guard 原生 App + 路旁充電 + 60 Hz
6. [在 Termux 跑 opencode](./06-opencode.md) - Bun binary 實戰：seccomp 問題與 shim 方案

## 快速開始

```bash
# 1. 從 F-Droid 安裝 Termux
# 2. 更新套件
pkg update && pkg upgrade

# 3. 安裝基本工具 + SSH
pkg install -y wget curl git vim openssh

# 4. 設定密碼並啟動 sshd
passwd
sshd
```

## 參考資源

- [Termux Wiki](https://wiki.termux.com/)

## 授權條款

本指南採用 [MIT License](LICENSE)。