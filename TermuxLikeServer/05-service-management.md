# 服務管理

使用 runit 透過 termux-services 管理服務。

## 基本概念

### 什麼是 runit？

- 簡單、輕量的 init 系統
- 每個服務都是獨立的處理程序
- 自動重啟崩潰的服務
- 記錄所有輸出到日誌

### 服務目錄結構

```
$PREFIX/var/service/
├── sshd/
│   ├── run          # 啟動腳本
│   └── log/
│       └── run      # 日誌腳本
├── postgresql/
│   ├── run
│   └── log/
│       └── run
└── redis/
    ├── run
    └── log/
        └── run
```

## 基本操作

### 1. 安裝 termux-services

```bash
# 安裝
pkg install termux-services

# 重啟 shell
exit
```

### 2. 服務控制命令

```bash
# 啟動服務
sv up <service>

# 停止服務
sv down <service>

# 重啟服務
sv restart <service>

# 查看狀態
sv status <service>

# 查看日誌
tail -f $PREFIX/var/log/sv/<service>/current
```

### 3. 設定開機自動啟動

```bash
# 啟用開機自動啟動
sv-enable <service>

# 停用開機自動啟動
sv-disable <service>
```

## 建立自訂服務

### 步驟一：建立服務目錄

```bash
# 建立服務目錄
mkdir -p $PREFIX/var/service/sv/my-service

# 建立日誌目錄
mkdir -p $PREFIX/var/service/sv/my-service/log
```

### 步驟二：建立 run 腳本

```bash
cat > $PREFIX/var/service/sv/my-service/run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh

# 設定環境
export HOME=/data/data/com.termux/files/home
export PATH=/data/data/com.termux/files/usr/bin:$PATH

# 進入工作目錄
cd ~/my-service

# 執行程式
exec ./my-program
EOF

# 設定權限
chmod +x $PREFIX/var/service/sv/my-service/run
```

### 步驟三：建立日誌腳本

```bash
cat > $PREFIX/var/service/sv/my-service/log/run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh

exec svlogd -tt $PREFIX/var/log/sv/my-service/
EOF

# 設定權限
chmod +x $PREFIX/var/service/sv/my-service/log/run
```

### 步驟四：啟用服務

```bash
# 啟動服務
sv up my-service

# 設定開機自動啟動
sv-enable my-service
```

## 範例：PostgreSQL 服務

### 建立服務

```bash
# 建立目錄
mkdir -p $PREFIX/var/service/sv/postgresql/log

# 建立 run 腳本
cat > $PREFIX/var/service/sv/postgresql/run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh

export PGDATA=$PREFIX/var/lib/postgresql
export PGLOG=$PREFIX/var/log/postgresql

# 建立日誌目錄
mkdir -p $PGLOG

# 啟動 PostgreSQL
exec pg_ctl -D $PGDATA -l $PGLOG/postgresql.log start -w
EOF

# 建立日誌腳本
cat > $PREFIX/var/service/sv/postgresql/log/run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh
exec svlogd -tt $PREFIX/var/log/sv/postgresql/
EOF

# 設定權限
chmod +x $PREFIX/var/service/sv/postgresql/run
chmod +x $PREFIX/var/service/sv/postgresql/log/run
```

### 啟動服務

```bash
# 啟動
sv up postgresql

# 檢查狀態
sv status postgresql

# 查看日誌
tail -f $PREFIX/var/log/sv/postgresql/current
```

## 範例：Redis 服務

### 建立服務

```bash
# 建立目錄
mkdir -p $PREFIX/var/service/sv/redis/log

# 建立 run 腳本
cat > $PREFIX/var/service/sv/redis/run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh

export REDIS_CONF=$PREFIX/etc/redis.conf
export REDIS_LOG=$PREFIX/var/log/redis.log

# 啟動 Redis
exec redis-server $REDIS_CONF --daemonize no
EOF

# 建立日誌腳本
cat > $PREFIX/var/service/sv/redis/log/run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh
exec svlogd -tt $PREFIX/var/log/sv/redis/
EOF

# 設定權限
chmod +x $PREFIX/var/service/sv/redis/run
chmod +x $PREFIX/var/service/sv/redis/log/run
```

### 啟動服務

```bash
# 啟動
sv up redis

# 檢查狀態
sv status redis
```

## 範例：Nginx 服務

### 建立服務

```bash
# 建立目錄
mkdir -p $PREFIX/var/service/sv/nginx/log

# 建立 run 腳本
cat > $PREFIX/var/service/sv/nginx/run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh

# 執行 Nginx
exec nginx -g 'daemon off;'
EOF

# 建立日誌腳本
cat > $PREFIX/var/service/sv/nginx/log/run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh
exec svlogd -tt $PREFIX/var/log/sv/nginx/
EOF

# 設定權限
chmod +x $PREFIX/var/service/sv/nginx/run
chmod +x $PREFIX/var/service/sv/nginx/log/run
```

### 啟動服務

```bash
# 啟動
sv up nginx

# 檢查狀態
sv status nginx
```

## 範例：Node.js 應用服務

### 建立服務

```bash
# 建立目錄
mkdir -p $PREFIX/var/service/sv/node-app/log

# 建立 run 腳本
cat > $PREFIX/var/service/sv/node-app/run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh

export NODE_ENV=production
export PORT=3000

cd ~/my-app

exec node app.js
EOF

# 建立日誌腳本
cat > $PREFIX/var/service/sv/node-app/log/run << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh
exec svlogd -tt $PREFIX/var/log/sv/node-app/
EOF

# 設定權限
chmod +x $PREFIX/var/service/sv/node-app/run
chmod +x $PREFIX/var/service/sv/node-app/log/run
```

## 管理所有服務

### 啟動所有服務

```bash
# 啟動 PostgreSQL
sv up postgresql

# 啟動 Redis
sv up redis

# 啟動 Nginx
sv up nginx

# 啟動 Node.js 應用
sv up node-app
```

### 停止所有服務

```bash
# 停止所有服務
for svc in postgresql redis nginx node-app; do
    sv down $svc
done
```

### 查看所有服務狀態

```bash
# 查看所有服務狀態
for svc in postgresql redis nginx node-app; do
    echo "=== $svc ==="
    sv status $svc
done
```

## 日誌管理

### 查看日誌

```bash
# 即時查看日誌
tail -f $PREFIX/var/log/sv/postgresql/current

# 查看最近 100 行
tail -100 $PREFIX/var/log/sv/postgresql/current

# 搜尋錯誤
grep -i error $PREFIX/var/log/sv/postgresql/current
```

### 日誌輪替

runit 會自動輪替日誌，但可以手動控制：

```bash
# 停止日誌
sv stop postgresql/log

# 啟動日誌
sv start postgresql/log
```

## 故障排除

### 問題一：服務無法啟動

```bash
# 檢查日誌
cat $PREFIX/var/log/sv/<service>/current

# 檢查權限
ls -la $PREFIX/var/service/sv/<service>/run

# 手動測試
$PREFIX/var/service/sv/<service>/run
```

### 問題二：服務崩潰後無法重啟

```bash
# 檢查 runit 狀態
ps aux | grep runsv

# 重啟 runit
killall runsv
```

### 問題三：日誌無法寫入

```bash
# 檢查日誌目錄權限
ls -la $PREFIX/var/log/sv/<service>/

# 修正權限
chmod -R 755 $PREFIX/var/log/sv/<service>/
```

## 下一步

完成服務管理設定後，請繼續 [卷管理](./06-volume-management.md)。