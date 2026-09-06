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
- **Android 版本**：12 以上

## 軟體準備

### 1. 安裝 Termux

**重要**：請從 F-Droid 安裝，不要從 Google Play Store 安裝。

Play Store 版本過舊，F-Droid 版本更新、更穩定。

#### 安裝步驟
1. 下載 [F-Droid](https://f-droid.org/packages/com.termux/)
2. 安裝 F-Droid APK
3. 在 F-Droid 中搜尋 "Termux"
4. 安裝 Termux

### 2. 更新套件管理器

打開 Termux 後先執行：

```bash
pkg update && pkg upgrade
```

## 下一步

完成上述設定後，請繼續 [安裝 Termux + SSH 設定](./02-install-termux.md)。