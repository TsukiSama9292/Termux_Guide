# glibc Loader

在 Android 上執行官方 Linux ARM64 二進制檔案的核心挑戰：**Bionic vs glibc**。

## 1. 背景知識

### 1.1 為什麼直接執行會失敗？

- **Android** 用 **Bionic libc**（精簡、為手機優化）
- **一般 Linux 發行版** 用 **glibc**（完整、相容性廣）
- 兩者**二進位不相容**，Linux 版 binary 在 Termux 裡會直接報錯：

```bash
$ ./linux-binary
bash: ./linux-binary: cannot execute binary file: Exec format error

# 或是這種騙人的訊息（檔案明明存在）
bash: ./binary: No such file or directory
```

第二種其實是動態連結器路徑不對：ELF 裡寫 `/lib/ld-linux-aarch64.so.1`，但 Termux 裡根本沒那個路徑。

### 1.2 三種解法

| 方法 | 效能 | 易用性 | 適合誰 |
|------|------|--------|--------|
| glibc-runner (`grun`) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 新手、一般用途（推薦先試這個） |
| Bionilux | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 想要 `bionilux <cmd>` 一行搞定 |
| 手動 glibc + patchelf | ⭐⭐⭐⭐ | ⭐⭐⭐ | 進階、要自己打包 / 寫 wrapper |

> 不在本篇範圍：PRoot（效能差很多）、QEMU（模擬別的架構）、自己重新編譯（效能最好但最麻煩）。

### 1.3 前置檢查

```bash
# 確認架構，一定要是 aarch64
uname -m
# 預期：aarch64

# 確認手上的 binary 真的是 ARM64
file ./your-binary
# 預期：ELF 64-bit LSB executable, ARM aarch64 ...
# 如果顯示 x86-64，那 glibc-loader 也救不了你，要去找 arm64 版
```

---

## 2. 方法一：glibc-runner（推薦）

來自 `glibc-repo`，是目前最省事的官方路徑。

### 2.1 安裝

```bash
# 安裝 glibc-repo 來源
pkg install glibc-repo -y

# 安裝 runner
pkg install glibc-runner -y

# 測試
grun --help
glibc-runner --help
```

> `grun` 就是 `glibc-runner` 的短指令，兩個一樣。

### 2.2 基本用法

```bash
# 直接執行 glibc binary
grun ./linux-binary

# 帶參數也一樣
grun ./linux-binary --version

# 執行系統命令（會走 glibc 那套）
grun ls -la
grun gcc --version
```

### 2.3 常見坑

```bash
# 坑 1：LD_PRELOAD 衝突
# 症狀：找不到 libdl.so / 載入失敗
# 解法：先清掉再跑
unset LD_PRELOAD
grun ./linux-binary

# 坑 2：interpreter 路徑不對
# 症狀：No such file or directory
# 解法：用 patchelf 改掉（詳見第 4 節）
pkg install patchelf -y
patchelf --set-interpreter $PREFIX/glibc/lib/ld-linux-aarch64.so.1 ./linux-binary
grun ./linux-binary
```

---

## 3. 方法二：Bionilux（更完整）

Bionilux 把 glibc 環境包得更完整，適合一次跑一整套工具鏈。

### 3.1 安裝

```bash
curl -sL theonuverse.github.io/bionilux/setup | bash
```

### 3.2 基本用法

```bash
# 直接執行
bionilux ./linux-binary

# 當前綴用
bionilux ls -la
bionilux node --version
bionilux redis-server --version
```

### 3.3 環境變數

| 變數 | 預設值 | 說明 |
|------|--------|------|
| `BIONILUX_GLIBC_LIB` | `$PREFIX/glibc/lib` | glibc ARM64 函式庫路徑 |
| `BIONILUX_GLIBC_LOADER` | `$PREFIX/glibc/lib/ld-linux-aarch64.so.1` | glibc 動態連結器 |
| `BIONILUX_DEBUG` | （未設定） | 設為 `1` 開啟偵錯輸出 |
| `BIONILUX_WAKELOCK` | `0` | 設為 `1` 會順便下 `termux-wake-lock` |

除錯範例：

```bash
# 跑不起來時先開 debug 看缺什麼
BIONILUX_DEBUG=1 bionilux ./linux-binary
```

### 3.4 測試安裝

```bash
bionilux ls -la
bionilux node --version
```

---

## 4. 方法三：手動設定 glibc（進階）

想理解原理、或要自己寫啟動腳本時用。

### 4.1 安裝 glibc 本體

```bash
pkg install glibc-repo -y
pkg install glibc -y

# 確認 loader 真的存在
ls -lh $PREFIX/glibc/lib/ld-linux-aarch64.so.1
ls -lh $PREFIX/glibc/lib/libc.so.6
```

### 4.2 補上 lib64 連結（很多 binary 會找這裡）

```bash
mkdir -p $PREFIX/glibc/lib64
ln -sf $PREFIX/glibc/lib/ld-linux-aarch64.so.1 $PREFIX/glibc/lib64/
```

### 4.3 用 patchelf 修 binary

```bash
pkg install patchelf -y

# 改 interpreter
patchelf --set-interpreter $PREFIX/glibc/lib/ld-linux-aarch64.so.1 ./linux-binary

# 改 RPATH（讓它優先找 glibc 的 lib）
patchelf --set-rpath $PREFIX/glibc/lib ./linux-binary

# 驗證
patchelf --print-interpreter ./linux-binary
patchelf --print-rpath ./linux-binary
```

### 4.4 寫一個通用 wrapper

之後就不用每次打一長串 loader 路徑：

```bash
mkdir -p ~/bin
cat > ~/bin/glibc-run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh
GLIBC_PREFIX="$PREFIX/glibc"
GLIBC_LOADER="$GLIBC_PREFIX/lib/ld-linux-aarch64.so.1"
GLIBC_LIBS="$GLIBC_PREFIX/lib"

# Termux 的 LD_PRELOAD 會污染 glibc，一定要清掉
unset LD_PRELOAD

exec "$GLIBC_LOADER" --library-path "$GLIBC_LIBS" "$@"
EOF

chmod +x ~/bin/glibc-run

# 用法
~/bin/glibc-run ./linux-binary --version
```

---

## 5. 實戰案例

### 5.1 案例一：跑官方 Node.js（最常用來驗證）

```bash
# 下載官方 Linux ARM64 版
wget https://nodejs.org/dist/v20.11.0/node-v20.11.0-linux-arm64.tar.xz
tar -xf node-v20.11.0-linux-arm64.tar.xz

# 用 grun 跑
grun ./node-v20.11.0-linux-arm64/bin/node --version
# 預期：v20.11.0

# 或用 Bionilux
bionilux ./node-v20.11.0-linux-arm64/bin/node --version

# 或用手動 wrapper
~/bin/glibc-run ./node-v20.11.0-linux-arm64/bin/node --version
```

### 5.2 案例二：跑 Python 官方版

```bash
wget https://www.python.org/ftp/python/3.12.1/Python-3.12.1.tgz
tar -xf Python-3.12.1.tgz
cd Python-3.12.1

# configure + make 都要包在 grun 裡
grun ./configure
grun make -j$(nproc)

# 執行
grun ./python --version
```

### 5.3 案例三：編譯跑 Redis

```bash
wget https://download.redis.io/releases/redis-7.2.4.tar.gz
tar -xf redis-7.2.4.tar.gz
cd redis-7.2.4

grun make -j$(nproc)
grun ./src/redis-server --version
```

---

## 6. 故障排除

### 6.1 `No such file or directory`（檔案明明存在）

原因：ELF 的 interpreter 是 `/lib/ld-linux-aarch64.so.1`，Termux 沒那個路徑。

```bash
# 看它想要什麼
readelf -l ./binary | grep interpreter

# 修掉
patchelf --set-interpreter $PREFIX/glibc/lib/ld-linux-aarch64.so.1 ./binary
```

### 6.2 `error while loading shared libraries: libxxx.so`

原因：找不到 glibc 的函式庫。

```bash
# 臨時解
export LD_LIBRARY_PATH=$PREFIX/glibc/lib:$LD_LIBRARY_PATH
grun ./binary

# 永久解：寫死 RPATH
patchelf --set-rpath $PREFIX/glibc/lib ./binary
```

### 6.3 `cannot execute binary file: Exec format error`

原因：架構不對，拿了 x86_64 的 binary。

```bash
file ./binary
# 如果顯示 x86-64 → 去找 aarch64 / arm64 版，沒別的辦法
uname -m
# 手機這邊要是 aarch64
```

### 6.4 `Permission denied`

```bash
chmod +x ./binary
ls -l ./binary
```

### 6.5 還是不行？開追蹤

```bash
# 看缺什麼 lib
BIONILUX_DEBUG=1 bionilux ./binary

# 看系統呼叫卡在哪
pkg install strace -y
grun strace -f ./binary 2>&1 | tail -50
```

---

## 7. 建議流程

1. 先試 `grun ./binary`，能跑就不要搞複雜的。
2. 跑不起來 → `file` + `readelf -l` 看架構和 interpreter。
3. 修 `interpreter` + `rpath`，再包成 `~/bin/glibc-run`。
4. 要長期跑、或要給別人用 → 換 Bionilux，環境變數好控制。
5. 真的要效能 → 回去找 Termux 原生套件（`pkg install xxx`），Bionic 直跑還是最快。

## 參考資源

- [glibc-packages（glibc-runner）](https://github.com/termux-pacman/glibc-packages)
- [Bionilux](https://github.com/theonuverse/bionilux)
- [Termux Wiki](https://wiki.termux.com/)

## 下一步

繼續 [在 Termux 運行 Doki](./04-doki.md)。
