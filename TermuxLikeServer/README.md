# Android Phone Server Guide

在未 Root 的 Android 手機上，使用 Termux 運行伺服器軟體，達到接近原生速度。

## 核心原則

- **無 Root**：不需要解鎖 bootloader 或 root 權限
- **無 PRoot**：不使用 proot-distro 或類似工具
- **無 QEMU**：不使用虛擬機
- **原生速度**：直接使用 Android Bionic 或 glibc loader 執行 ELF 二進制檔案

## 架構概覽

```
Android Phone (ARM64)
    │
    ├── Termux (Bionic libc)
    │   ├── 原生套件 (PostgreSQL, Redis, Nginx...)
    │   └── glibc loader (Bionilux/glibc-runner)
    │       └── Linux ARM64 二進制檔案
    │
    ├── Service Manager (runit/termux-services)
    │
    └── Data Layer (~/server/volumes/)
```

## 文件導覽

1. [先決條件](./01-prerequisites.md) - 硬體需求、軟體準備
2. [安裝 Termux](./02-install-termux.md) - 基本設定與最佳化
3. [安裝服務](./03-install-services.md) - PostgreSQL, Redis, Nginx 等
4. [glibc Loader](./04-glibc-loader.md) - 執行 Linux ARM64 二進制檔案
5. [服務管理](./05-service-management.md) - 使用 runit 管理服務
6. [卷管理](./06-volume-management.md) - 持久化儲存與備份
7. [故障排除](./07-troubleshooting.md) - 常見問題與解決方案

## 快速開始

```bash
# 1. 安裝 Termux (從 F-Droid)
# 2. 更新套件
pkg update && pkg upgrade

# 3. 安裝基本服務
pkg install postgresql redis nginx termux-services

# 4. 初始化 PostgreSQL
initdb $PREFIX/var/lib/postgresql

# 5. 啟動服務
sv up postgresql
sv up redis
sv up nginx
```

## 參考資源

- [Termux Wiki](https://wiki.termux.com/)
- [Termux Services](https://github.com/termux/termux-services)
- [Bionilux](https://github.com/theonuverse/bionilux)
- [glibc-runner](https://github.com/termux-pacman/glibc-packages)

## 授權條款

本指南採用 [MIT License](LICENSE)。