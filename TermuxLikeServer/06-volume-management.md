# 卷管理

在沒有 Docker 的情況下，使用目錄和配置管理持久化儲存。

## 概念

### 為什麼需要卷管理？

- 持久化儲存資料
- 方便備份和還原
- 隔離不同服務的資料
- 易於管理和遷移

### 目錄結構

```
~/server/
├── volumes/
│   ├── nginx/
│   │   └── html/          # 網頁檔案
│   ├── redis/             # Redis 資料
│   ├── postgresql/        # PostgreSQL 資料
│   └── apps/              # 應用程式資料
├── configs/
│   ├── nginx/
│   ├── redis/
│   └── postgresql/
└── backups/
    ├── daily/
    ├── weekly/
    └── monthly/
```

## 建立卷目錄

### 基本目錄

```bash
# 建立主目錄
mkdir -p ~/server/volumes

# 建立服務目錄
mkdir -p ~/server/volumes/{nginx,redis,postgresql,apps}

# 建立配置目錄
mkdir -p ~/server/configs/{nginx,redis,postgresql}

# 建立備份目錄
mkdir -p ~/server/backups/{daily,weekly,monthly}
```

### 設定權限

```bash
# 設定目錄權限
chmod -R 755 ~/server/volumes
chmod -R 755 ~/server/configs
chmod -R 700 ~/server/backups
```

## PostgreSQL 卷管理

### 設定資料目錄

```bash
# 建立 PostgreSQL 資料目錄
mkdir -p ~/server/volumes/postgresql

# 設定環境變數
export PGDATA=~/server/volumes/postgresql

# 初始化資料庫
initdb $PGDATA
```

### 備份資料庫

```bash
# 建立備份腳本
cat > ~/server/scripts/backup-postgresql.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh

BACKUP_DIR=~/server/backups/daily
DATE=$(date +%Y%m%d_%H%M%S)
PGDATA=~/server/volumes/postgresql

# 建立備份目錄
mkdir -p $BACKUP_DIR

# 備份 PostgreSQL
pg_dumpall > $BACKUP_DIR/postgresql_$DATE.sql

# 壓縮
gzip $BACKUP_DIR/postgresql_$DATE.sql

# 刪除 7 天前的備份
find $BACKUP_DIR -name "postgresql_*.sql.gz" -mtime +7 -delete

echo "備份完成：postgresql_$DATE.sql.gz"
EOF

chmod +x ~/server/scripts/backup-postgresql.sh
```

### 還原資料庫

```bash
# 建立還原腳本
cat > ~/server/scripts/restore-postgresql.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh

BACKUP_FILE=$1
PGDATA=~/server/volumes/postgresql

if [ -z "$BACKUP_FILE" ]; then
    echo "用法：$0 <backup_file>"
    exit 1
fi

# 停止 PostgreSQL
sv down postgresql

# 備份現有資料
mv $PGDATA $PGDATA.backup.$(date +%Y%m%d)

# 初始化新資料庫
initdb $PGDATA

# 啟動 PostgreSQL
sv up postgresql

# 等待啟動
sleep 5

# 還原資料
gunzip -c $BACKUP_FILE | psql -U postgres

echo "還原完成"
EOF

chmod +x ~/server/scripts/restore-postgresql.sh
```

## Redis 卷管理

### 設定資料目錄

```bash
# 建立 Redis 資料目錄
mkdir -p ~/server/volumes/redis

# 設定 redis.conf
cat > ~/server/configs/redis/redis.conf << 'EOF'
bind 127.0.0.1
port 6379
daemonize no
dir ~/server/volumes/redis
dbfilename dump.rdb
save 900 1
save 300 10
save 60 10000
EOF
```

### 備份 Redis

```bash
# 建立備份腳本
cat > ~/server/scripts/backup-redis.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh

BACKUP_DIR=~/server/backups/daily
DATE=$(date +%Y%m%d_%H%M%S)
REDIS_DIR=~/server/volumes/redis

# 建立備份目錄
mkdir -p $BACKUP_DIR

# 觸發 Redis 備份
redis-cli BGSAVE

# 等待備份完成
while [ $(redis-cli LASTSAVE) -eq $(redis-cli LASTSAVE) ]; do
    sleep 1
done

# 複製備份檔案
cp $REDIS_DIR/dump.rdb $BACKUP_DIR/redis_$DATE.rdb

# 壓縮
gzip $BACKUP_DIR/redis_$DATE.rdb

# 刪除 7 天前的備份
find $BACKUP_DIR -name "redis_*.rdb.gz" -mtime +7 -delete

echo "備份完成：redis_$DATE.rdb.gz"
EOF

chmod +x ~/server/scripts/backup-redis.sh
```

### 還原 Redis

```bash
# 建立還原腳本
cat > ~/server/scripts/restore-redis.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh

BACKUP_FILE=$1
REDIS_DIR=~/server/volumes/redis

if [ -z "$BACKUP_FILE" ]; then
    echo "用法：$0 <backup_file>"
    exit 1
fi

# 停止 Redis
sv down redis

# 備份現有資料
mv $REDIS_DIR/dump.rdb $REDIS_DIR/dump.rdb.backup.$(date +%Y%m%d)

# 還原資料
gunzip -c $BACKUP_FILE > $REDIS_DIR/dump.rdb

# 啟動 Redis
sv up redis

echo "還原完成"
EOF

chmod +x ~/server/scripts/restore-redis.sh
```

## Nginx 卷管理

### 設定網頁目錄

```bash
# 建立網頁目錄
mkdir -p ~/server/volumes/nginx/html

# 建立預設頁面
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

### 備份 Nginx

```bash
# 建立備份腳本
cat > ~/server/scripts/backup-nginx.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/sh

BACKUP_DIR=~/server/backups/daily
DATE=$(date +%Y%m%d_%H%M%S)
NGINX_DIR=~/server/volumes/nginx

# 建立備份目錄
mkdir -p $BACKUP_DIR

# 備份 Nginx 配置
tar -czf $BACKUP_DIR/nginx_config_$DATE.tar.gz $PREFIX/etc/nginx/

# 備份網頁檔案
tar -czf $BACKUP_DIR/nginx_html_$DATE.tar.gz $NGINX_DIR/html/

# 刪除 7 天前的備份
find $BACKUP_DIR -name "nginx_*.tar.gz" -mtime +7 -delete

echo "備份完成"
EOF

chmod +x ~/server/scripts/backup-nginx.sh
```

## 自動備份

### 使用 cron

```bash
# 安裝 cron
pkg install cronie

# 啟動 cron
crond

# 設定定時任務
crontab -e
```

### 加入定時備份

```bash
# 編輯 crontab
cat > ~/server/configs/crontab << 'EOF'
# 每天凌晨 2 點備份 PostgreSQL
0 2 * * * ~/server/scripts/backup-postgresql.sh

# 每天凌晨 3 點備份 Redis
0 3 * * * ~/server/scripts/backup-redis.sh

# 每天凌晨 4 點備份 Nginx
0 4 * * * ~/server/scripts/backup-nginx.sh

# 每週日備份所有資料
0 1 * * 0 ~/server/scripts/backup-all.sh
EOF

crontab ~/server/configs/crontab
```

## 遷移資料

### 移動 PostgreSQL 資料

```bash
# 停止 PostgreSQL
sv down postgresql

# 移動資料目錄
mv ~/server/volumes/postgresql ~/server/volumes/postgresql.old

# 建立新目錄
mkdir -p ~/server/volumes/postgresql

# 複製資料
cp -r ~/server/volumes/postgresql.old/* ~/server/volumes/postgresql/

# 更新設定
# 編輯 PostgreSQL 配置指向新目錄
```

### 移動 Redis 資料

```bash
# 停止 Redis
sv down redis

# 移動資料目錄
mv ~/server/volumes/redis ~/server/volumes/redis.old

# 建立新目錄
mkdir -p ~/server/volumes/redis

# 複製資料
cp ~/server/volumes/redis.old/dump.rdb ~/server/volumes/redis/

# 啟動 Redis
sv up redis
```

## 磁碟空間管理

### 檢查空間使用

```bash
# 檢查總體空間
df -h

# 檢查目錄大小
du -sh ~/server/volumes/*

# 找出大檔案
find ~/server/volumes -type f -size +100M -exec ls -lh {} \;
```

### 清理舊備份

```bash
# 刪除 30 天前的備份
find ~/server/backups -name "*.sql.gz" -mtime +30 -delete
find ~/server/backups -name "*.rdb.gz" -mtime +30 -delete
find ~/server/backups -name "*.tar.gz" -mtime +30 -delete

# 刪除日誌
find ~/server/volumes -name "*.log" -mtime +7 -delete
```

## 故障排除

### 問題一：磁碟空間不足

```bash
# 檢查空間
df -h

# 清理快取
pkg clean

# 清理日誌
find ~/server/volumes -name "*.log" -mtime +3 -delete

# 清理備份
find ~/server/backups -mtime +30 -delete
```

### 問題二：權限問題

```bash
# 檢查權限
ls -la ~/server/volumes/*

# 修正權限
chmod -R 755 ~/server/volumes
chown -R $(id -u):$(id -g) ~/server/volumes
```

### 問題三：備份失敗

```bash
# 檢查備份腳本
ls -la ~/server/scripts/backup-*.sh

# 測試備份腳本
~/server/scripts/backup-postgresql.sh

# 檢查錯誤訊息
cat ~/server/backups/daily/*.log
```

## 下一步

完成卷管理設定後，請繼續 [故障排除](./07-troubleshooting.md)。