# glibc Loader

在 Android 上執行 Linux 二進制檔案的核心挑戰：**Bionic vs glibc**。

## 背景知識

### 問題根源

- **Android** 使用 **Bionic libc** (精簡、優化)
- **Linux** 使用 **glibc** (完整、相容)
- 兩者 **不二進位相容**

```bash
# 嘗試直接執行 Linux 二進制檔案會失敗
$ ./linux-binary
bash: ./linux-binary: cannot execute binary file: Exec format error
```

### 解決方案

有三種主要方法：

1. **Native Bionic Port**：重新編譯 (效能最高，但需要編譯)
2. **glibc Loader**：使用 glibc 動態連結器 (效能高，可直接使用)
3. **PRoot**：使用者空間模擬 (效能低，但最簡單)

## 方法一：使用 glibc-repo + glibc-runner

### 安裝

```bash
# 安裝 glibc-repo
pkg install glibc-repo -y

# 安裝 glibc-runner
pkg install glibc-runner -y

# 測試
grun --help
```

### 使用方式

```bash
# 直接執行 glibc 二進制檔案
grun ./linux-binary

# 設定環境變數後執行
grun --set ./linux-binary

# 執行系統命令
grun ls -la
grun gcc --version
```

### 設定環境

```bash
# 加入 .bashrc
echo 'export GLIBC_PREFIX="$PREFIX/glibc"' >> ~/.bashrc
echo 'export PATH="$GLIBC_PREFIX/bin:$PATH"' >> ~/.bashrc

# 重啟 shell
source ~/.bashrc
```

### 常見問題

```bash
# 問題：找不到 libdl.so
# 解決方案：
unset LD_PRELOAD
grun ./linux-binary

# 問題：找不到動態連結器
# 解決方案：
patchelf --set-interpreter $PREFIX/glibc/lib/ld-linux-aarch64.so.1 ./linux-binary
```

## 方法二：使用 Bionilux

### 安裝

```bash
# 快速安裝
curl -sL theonuverse.github.io/bionilux/setup | bash
```

### 使用方式

```bash
# 直接執行
bionilux ./linux-binary

# 執行系統命令
bionilux ls -la
bionilux node --version
```

### 環境變數

| 變數 | 預設值 | 說明 |
|------|--------|------|
| `BIONILUX_GLIBC_LIB` | `$PREFIX/glibc/lib` | glibc ARM64 函式庫路徑 |
| `BIONILUX_GLIBC_LOADER` | `$PREFIX/glibc/lib/ld-linux-aarch64.so.1` | glibc 動態連結器 |
| `BIONILUX_DEBUG` | *(未設定)* | 設為 1 啟用偵錯輸出 |
| `BIONILUX_WAKELOCK` | 0 | 設為 1 執行 termux-wake-lock |

### 測試安裝

```bash
# 測試執行 ls
bionilux ls -la

# 測試執行 node
bionilux node --version

# 測試執行 redis-server
bionilux redis-server --version
```

## 方法三：手動設定 glibc 環境

### 安裝 glibc

```bash
# 安裝 glibc-repo
pkg install glibc-repo -y

# 安裝 glibc
pkg install glibc -y

# 確認安裝
ls -la $PREFIX/glibc/lib/
```

### 設定動態連結器

```bash
# 建立連結器路徑
mkdir -p $PREFIX/glibc/lib64
ln -sf $PREFIX/glibc/lib/ld-linux-aarch64.so.1 $PREFIX/glibc/lib64/
```

### 使用 patchelf

```bash
# 安裝 patchelf
pkg install patchelf

# 修改二進制檔案的 interpreter
patchelf --set-interpreter $PREFIX/glibc/lib/ld-linux-aarch64.so.1 ./linux-binary

# 設定 RPATH
patchelf --set-rpath $PREFIX/glibc/lib ./linux-binary
```

### 建立包裝腳本

```bash
# 建立通用包裝腳本
cat > ~/server/bin/glibc-run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh

GLIBC_PREFIX="$PREFIX/glibc"
GLIBC_LOADER="$GLIBC_PREFIX/lib/ld-linux-aarch64.so.1"
GLIBC_LIBS="$GLIBC_PREFIX/lib"

# 清除 LD_PRELOAD
unset LD_PRELOAD

# 執行 glibc 二進制檔案
exec "$GLIBC_LOADER" --library-path "$GLIBC_LIBS" "$@"
EOF

chmod +x ~/server/bin/glibc-run

# 使用方式
~/server/bin/glibc-run ./linux-binary
```

## 實際案例

### 案例一：執行 Node.js

```bash
# 下載官方 Linux ARM64 版本
wget https://nodejs.org/dist/v20.11.0/node-v20.11.0-linux-arm64.tar.xz

# 解壓縮
tar -xf node-v20.11.0-linux-arm64.tar.xz

# 使用 glibc-runner 執行
grun ./node-v20.11.0-linux-arm64/bin/node --version

# 或使用 Bionilux
bionilux ./node-v20.11.0-linux-arm64/bin/node --version
```

### 案例二：執行 Redis

```bash
# 下載 Redis 原始碼
wget https://download.redis.io/releases/redis-7.2.4.tar.gz

# 解壓縮
tar -xf redis-7.2.4.tar.gz
cd redis-7.2.4

# 編譯 (需要 glibc 環境)
grun make

# 執行
grun ./src/redis-server
```

### 案例三：執行 Python

```bash
# 下載 Python 原始碼
wget https://www.python.org/ftp/python/3.12.1/Python-3.12.1.tgz

# 解壓縮
tar -xf Python-3.12.1.tgz
cd Python-3.12.1

# 編譯
grun ./configure
grun make

# 執行
grun ./python
```

## 效能比較

| 方法 | 效能 | 易用性 | 適用場景 |
|------|------|--------|----------|
| Native Bionic | ⭐⭐⭐⭐⭐ | ⭐⭐ | 效能要求高 |
| glibc-runner | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 一般用途 |
| Bionilux | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 一般用途 |
| 手動 glibc | ⭐⭐⭐⭐ | ⭐⭐⭐ | 進階使用者 |
| PRoot | ⭐⭐ | ⭐⭐⭐⭐⭐ | 開發測試 |

## 故障排除

### 問題一：找不到動態連結器

```bash
# 錯誤訊息
bash: ./binary: No such file or directory

# 解決方案
ls -la $PREFIX/glibc/lib/ld-linux-aarch64.so.1
patchelf --set-interpreter $PREFIX/glibc/lib/ld-linux-aarch64.so.1 ./binary
```

### 問題二：找不到函式庫

```bash
# 錯誤訊息
error while loading shared libraries: libxxx.so: cannot open shared object file

# 解決方案
export LD_LIBRARY_PATH=$PREFIX/glibc/lib:$LD_LIBRARY_PATH
```

### 問題三：權限不足

```bash
# 錯誤訊息
Permission denied

# 解決方案
chmod +x ./binary
```

### 問題四：架構不符

```bash
# 錯誤訊息
cannot execute binary file: Exec format error

# 解決方案
file ./binary
# 應該顯示：ELF 64-bit LSB executable, ARM aarch64

# 如果不是 aarch64，需要下載正確版本
```

## 下一步

完成 glibc Loader 設定後，請繼續 [服務管理](./05-service-management.md)。