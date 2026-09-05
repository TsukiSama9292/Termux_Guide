# 安裝服務

本指南將安裝以下服務：
- PostgreSQL (資料庫)
- Redis (快取)
- Nginx (網頁伺服器)
- Node.js (執行環境)

## PostgreSQL

### 安裝

```bash
# 安裝 PostgreSQL
pkg install postgresql

# 初始化資料庫
initdb $PREFIX/var/lib/postgresql

# 啟動 PostgreSQL
pg_ctl -D $PREFIX/var/lib/postgresql start
```

### 設定服務

```bash
# 使用 termux-services
sv up postgresql

# 設定開機自動啟動
sv-enable postgresql
```

### 基本操作

```bash
# 連接到 PostgreSQL
psql -U postgres

# 建立新使用者
createuser -P myuser

# 建立新資料庫
createdb -O myuser mydb

# 備份資料庫
pg_dump mydb > backup.sql

# 還原資料庫
psql mydb < backup.sql
```

### 設定連線

```bash
# 編輯 postgresql.conf
nano $PREFIX/var/lib/postgresql/postgresql.conf

# 修改以下設定：
# listen_addresses = '*'  (允許所有連線)
# port = 5432

# 編輯 pg_hba.conf
nano $PREFIX/var/lib/postgresql/pg_hba.conf

# 加入以下行 (允許本地連線)：
# local   all   all   trust
# host    all   all   127.0.0.1/32   trust
```

### 常見問題

```bash
# 問題：無法建立鎖定檔案
# 解決方案：
mkdir -p $PREFIX/tmp
chmod 777 $PREFIX/tmp

# 問題：無法連結執行檔
# 解決方案：
pkg install libandroid-execinfo
```

## Redis

### 安裝

```bash
# 安裝 Redis
pkg install redis

# 啟動 Redis
redis-server &
```

### 設定服務

```bash
# 使用 termux-services
sv up redis

# 設定開機自動啟動
sv-enable redis
```

### 基本操作

```bash
# 連接到 Redis
redis-cli

# 測試連線
ping

# 設定值
set mykey "Hello"

# 取得值
get mykey

# 刪除值
del mykey
```

### 設定 redis.conf

```bash
# 編輯設定檔
nano $PREFIX/etc/redis.conf

# 常用設定：
# bind 127.0.0.1        (只允許本地連線)
# port 6379             (預設連接埠)
# requirepass mypassword (設定密碼)
# maxmemory 256mb       (最大記憶體)
# maxmemory-policy allkeys-lru (淘汰策略)
```

### 常見問題

```bash
# 問題：無法啟動 Redis
# 解決方案：
redis-server --daemonize yes

# 問題：連線被拒絕
# 解決方案：
redis-cli shutdown
redis-server
```

## Nginx

### 安裝

```bash
# 安裝 Nginx
pkg install nginx

# 啟動 Nginx
nginx
```

### 設定服務

```bash
# 使用 termux-services
sv up nginx

# 設定開機自動啟動
sv-enable nginx
```

### 基本操作

```bash
# 測試設定檔
nginx -t

# 重新載入設定
nginx -s reload

# 停止 Nginx
nginx -s stop

# 查看處理程序
ps aux | grep nginx
```

### 設定虛擬主機

```bash
# 建立網站目錄
mkdir -p ~/server/volumes/nginx/html

# 建立 index.html
cat > ~/server/volumes/nginx/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Termux Server</title>
</head>
<body>
    <h1>歡迎來到 Termux 伺服器</h1>
    <p>這是在 Android 手機上運行的伺服器。</p>
</body>
</html>
EOF
```

### 設定 nginx.conf

```bash
# 編輯設定檔
nano $PREFIX/etc/nginx/nginx.conf

# 修改 http 區塊：
http {
    server {
        listen 8080;
        server_name localhost;

        root /data/data/com.termux/files/home/server/volumes/nginx/html;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

### 常見問題

```bash
# 問題：無法綁定連接埠
# 解決方案：
# 使用高位連接埠 (1024 以上)
# Nginx 預設使用 8080

# 問題：權限不足
# 解決方案：
chmod -R 755 ~/server/volumes/nginx
```

## Node.js

### 安裝

```bash
# 安裝 Node.js
pkg install nodejs

# 安裝 npm
pkg install npm

# 驗證安裝
node --version
npm --version
```

### 使用 glibc 版本 (可選)

如果需要官方 Linux ARM64 版本：

```bash
# 使用 Bionilux
bionilux node --version

# 或使用 glibc-runner
grun node --version
```

### 基本操作

```bash
# 建立新專案
mkdir ~/myapp && cd ~/myapp
npm init -y

# 安裝套件
npm install express

# 執行腳本
node app.js

# 使用 nodemon (開發用)
npm install -g nodemon
nodemon app.js
```

### 設定服務

建立 Node.js 服務腳本：

```bash
# 建立服務目錄
mkdir -p $PREFIX/var/service/sv/node-app

# 建立 run 腳本
cat > $PREFIX/var/service/sv/node-app/run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh
cd ~/myapp
exec node app.js
EOF

# 設定權限
chmod +x $PREFIX/var/service/sv/node-app/run

# 啟動服務
sv up node-app
```

## Docker Image 作為 Binary 來源 (進階)

如果需要從 Docker image 提取二進制檔案：

```bash
# 安裝必要的工具
pkg install docker.io  # 如果可用

# 或使用 skopeo + umoci
pkg install skopeo umoci
```

### 提取步驟

```bash
# 1. 下載 Docker image
skopeo copy docker://redis:alpine dir:~/cache/oci/redis

# 2. 提取 layer
umoci unpack --image ~/cache/oci/redis ~/cache/oci/redis-unpacked

# 3. 複製二進制檔案
cp ~/cache/oci/redis-unpacked/rootfs/usr/local/bin/redis-server ~/server/bin/glibc/
```

## 服務總覽

| 服務 | 連接埠 | 預設設定檔 | 資料目錄 |
|------|--------|------------|----------|
| PostgreSQL | 5432 | $PREFIX/var/lib/postgresql/postgresql.conf | $PREFIX/var/lib/postgresql |
| Redis | 6379 | $PREFIX/etc/redis.conf | /data/data/com.termux/files/usr/var/lib/redis |
| Nginx | 8080 | $PREFIX/etc/nginx/nginx.conf | ~/server/volumes/nginx |
| Node.js | - | - | ~/myapp |

## 下一步

完成服務安裝後，請繼續 [glibc Loader](./04-glibc-loader.md)。