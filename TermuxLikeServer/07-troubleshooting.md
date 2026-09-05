# 故障排除

本指南包含常見問題和解決方案。

## 安裝問題

### 問題一：無法安裝套件

**症狀**：
```bash
E: Unable to locate package <package-name>
```

**解決方案**：
```bash
# 更新套件列表
pkg update

# 清除快取
pkg clean

# 重新安裝
pkg install <package-name>

# 如果仍然失敗，嘗試更換鏡像
termux-change-repo
```

### 問題二：套件相依性問題

**症狀**：
```bash
The following packages have unmet dependencies:
  <package> depends on <dependency>
```

**解決方案**：
```bash
# 強制安裝
pkg install -f

# 或手動安裝相依套件
pkg install <dependency>
pkg install <package>
```

### 問題三：磁碟空間不足

**症狀**：
```bash
E: You don't have enough free space in /data/data/com.termux/files/usr
```

**解決方案**：
```bash
# 清除快取
pkg clean

# 清除不必要的套件
pkg autoremove

# 檢查空間
df -h

# 清除日誌
find ~/. -name "*.log" -delete
```

## 服務問題

### 問題一：服務無法啟動

**症狀**：
```bash
$ sv up postgresql
timeout: down: postgresql: 3 seconds, normally up
```

**解決方案**：
```bash
# 檢查日誌
cat $PREFIX/var/log/sv/postgresql/current

# 手動測試
pg_ctl -D $PREFIX/var/lib/postgresql start

# 檢查權限
ls -la $PREFIX/var/lib/postgresql

# 修正權限
chmod 700 $PREFIX/var/lib/postgresql
```

### 問題二：服務崩潰

**症狀**：
```bash
$ sv status postgresql
postgres: (pid 1234) 5 seconds
```

**解決方案**：
```bash
# 檢查崩潰原因
cat $PREFIX/var/log/sv/postgresql/current

# 查看錯誤訊息
grep -i error $PREFIX/var/log/sv/postgresql/current

# 重啟服務
sv restart postgresql
```

### 問題三：服務無法連線

**症狀**：
```bash
$ psql -U postgres
psql: could not connect to server: Connection refused
```

**解決方案**：
```bash
# 檢查服務是否運行
sv status postgresql

# 檢查連接埠
netstat -tlnp | grep 5432

# 檢查防火牆
iptables -L -n

# 測試連線
telnet localhost 5432
```

## glibc 問題

### 問題一：找不到動態連結器

**症狀**：
```bash
bash: ./binary: No such file or directory
```

**解決方案**：
```bash
# 檢查動態連結器
ls -la $PREFIX/glibc/lib/ld-linux-aarch64.so.1

# 如果不存在，建立連結
mkdir -p $PREFIX/glibc/lib64
ln -sf $PREFIX/glibc/lib/ld-linux-aarch64.so.1 $PREFIX/glibc/lib64/

# 使用 patchelf
patchelf --set-interpreter $PREFIX/glibc/lib/ld-linux-aarch64.so.1 ./binary
```

### 問題二：找不到函式庫

**症狀**：
```bash
error while loading shared libraries: libxxx.so: cannot open shared object file
```

**解決方案**：
```bash
# 檢查函式庫
ls -la $PREFIX/glibc/lib/

# 設定 LD_LIBRARY_PATH
export LD_LIBRARY_PATH=$PREFIX/glibc/lib:$LD_LIBRARY_PATH

# 或使用 RPATH
patchelf --set-rpath $PREFIX/glibc/lib ./binary
```

### 問題三：架構不符

**症狀**：
```bash
cannot execute binary file: Exec format error
```

**解決方案**：
```bash
# 檢查檔案類型
file ./binary

# 應該顯示：ELF 64-bit LSB executable, ARM aarch64

# 如果不是，下載正確版本
# 例如：node-v20.11.0-linux-arm64.tar.xz
```

## PostgreSQL 問題

### 問題一：無法初始化資料庫

**症狀**：
```bash
$ initdb $PREFIX/var/lib/postgresql
initdb: could not create directory "/data/data/com.termux/files/usr/var/lib/postgresql": No such file or directory
```

**解決方案**：
```bash
# 建立目錄
mkdir -p $PREFIX/var/lib/postgresql

# 建立日誌目錄
mkdir -p $PREFIX/var/log/postgresql

# 重新初始化
initdb $PREFIX/var/lib/postgresql
```

### 問題二：無法建立鎖定檔案

**症狀**：
```bash
FATAL: could not create lock file "/tmp/.s.PGSQL.5432.lock": No such file or directory
```

**解決方案**：
```bash
# 建立 tmp 目錄
mkdir -p $PREFIX/tmp

# 設定權限
chmod 777 $PREFIX/tmp

# 啟動 PostgreSQL
pg_ctl -D $PREFIX/var/lib/postgresql start
```

### 問題三：無法連結執行檔

**症狀**：
```bash
CANNOT LINK EXECUTABLE "./postgres": library "libexecinfo.so" not found
```

**解決方案**：
```bash
# 安裝缺少的函式庫
pkg install libandroid-execinfo

# 重新安裝 PostgreSQL
pkg reinstall postgresql
```

## Redis 問題

### 問題一：無法啟動 Redis

**症狀**：
```bash
$ redis-server
Could not create server TCP listening socket *:6379: bind: Address already in use
```

**解決方案**：
```bash
# 查找占用連接埠的程序
netstat -tlnp | grep 6379

# 終止程序
kill <pid>

# 或更改連接埠
redis-server --port 6380
```

### 問題二：連線被拒絕

**症狀**：
```bash
$ redis-cli
Could not connect to Redis at 127.0.0.1:6379: Connection refused
```

**解決方案**：
```bash
# 檢查 Redis 是否運行
ps aux | grep redis

# 啟動 Redis
redis-server

# 檢查 redis.conf
cat $PREFIX/etc/redis.conf | grep bind
```

## Nginx 問題

### 問題一：無法綁定連接埠

**症狀**：
```bash
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
```

**解決方案**：
```bash
# 查找占用連接埠的程序
netstat -tlnp | grep :80

# 終止程序
kill <pid>

# 或更改連接埠
# 編輯 nginx.conf
nano $PREFIX/etc/nginx/nginx.conf
# 修改 listen 8080;
```

### 問題二：權限不足

**症狀**：
```bash
nginx: [emerg] bind() to 0.0.0.0:8080 failed (13: Permission denied)
```

**解決方案**：
```bash
# 使用高位連接埠 (1024 以上)
# 編輯 nginx.conf
nano $PREFIX/etc/nginx/nginx.conf
# 修改 listen 8080;

# 或更改權限
sudo chown root:root /usr/sbin/nginx
sudo chmod +s /usr/sbin/nginx
```

## 網路問題

### 問題一：無法連線到服務

**症狀**：
```bash
$ curl localhost:8080
curl: (7) Failed to connect to localhost port 8080: Connection refused
```

**解決方案**：
```bash
# 檢查服務是否運行
sv status nginx

# 檢查連接埠
netstat -tlnp | grep :8080

# 測試連線
telnet localhost 8080

# 檢查防火牆
iptables -L -n
```

### 問題二：無法從外部連線

**症狀**：
從其他設備無法連線到服務。

**解決方案**：
```bash
# 檢查手機 IP
ifconfig

# 確認在同一網路
ping <phone-ip>

# 檢查防火牆
iptables -A INPUT -p tcp --dport 8080 -j ACCEPT

# 使用高位連接埠
# 有些路由器封鎖低位連接埠
```

## 效能問題

### 問題一：服務回應緩慢

**症狀**：
服務回應時間過長。

**解決方案**：
```bash
# 檢查 CPU 使用率
top

# 檢查記憶體使用率
free -h

# 檢查磁碟 I/O
iostat

# 優化設定
# 例如：減少 PostgreSQL 連線數
# 例如：增加 Redis 快取大小
```

### 問題二：記憶體不足

**症狀**：
```bash
Out of memory
```

**解決方案**：
```bash
# 檢查記憶體使用
free -h

# 終止不必要的程序
kill <pid>

# 增加 swap
# 在 Termux 中無法直接增加 swap
# 但可以減少服務數量
```

## 備份問題

### 問題一：備份失敗

**症狀**：
```bash
$ ~/server/scripts/backup-postgresql.sh
pg_dump: [archiver (db) connection to database "postgres" failed: FATAL: too many connections
```

**解決方案**：
```bash
# 限制連線數
pg_ctl -D $PREFIX/var/lib/postgresql -o "-c max_connections=10" start

# 重新備份
~/server/scripts/backup-postgresql.sh
```

### 問題二：還原失敗

**症狀**：
```bash
$ psql -U postgres < backup.sql
ERROR: could not create directory
```

**解決方案**：
```bash
# 檢查權限
ls -la ~/server/volumes/postgresql

# 修正權限
chmod 700 ~/server/volumes/postgresql

# 重新還原
psql -U postgres < backup.sql
```

## 日誌分析

### 查看錯誤日誌

```bash
# PostgreSQL 日誌
grep -i error $PREFIX/var/log/sv/postgresql/current

# Redis 日誌
grep -i error $PREFIX/var/log/sv/redis/current

# Nginx 日誌
grep -i error $PREFIX/var/log/sv/nginx/current
```

### 搜尋特定錯誤

```bash
# 搜尋連線錯誤
grep -i "connection refused" $PREFIX/var/log/sv/*/current

# 搜尋權限錯誤
grep -i "permission denied" $PREFIX/var/log/sv/*/current

# 搜尋記憶體錯誤
grep -i "out of memory" $PREFIX/var/log/sv/*/current
```

## 進階除錯

### 使用 strace

```bash
# 安裝 strace
pkg install strace

# 追蹤系統呼叫
strace -f ./binary

# 追蹤特定系統呼叫
strace -e trace=open,read,write ./binary
```

### 使用 gdb

```bash
# 安裝 gdb
pkg install gdb

# 除錯程式
gdb ./binary

# 在 gdb 中
(gdb) run
(gdb) backtrace
(gdb) info threads
```

## 尋求幫助

### 檢查文件

```bash
# Termux Wiki
# https://wiki.termux.com/

# Termux GitHub
# https://github.com/termux/termux-packages
```

### 搜尋問題

```bash
# 使用 Google 搜尋
# "termux <error-message>"

# 檢查 GitHub Issues
# https://github.com/termux/termux-packages/issues
```

### 回報問題

如果找到 bug，請回報：

```bash
# 收集系統資訊
termux-info

# 收集錯誤訊息
cat $PREFIX/var/log/sv/*/current > /tmp/debug.log

# 回報到 GitHub
# https://github.com/termux/termux-packages/issues/new
```

## 結語

恭喜！你已經完成在 Android 手機上設定伺服器的全部流程。

### 回顧

1. ✅ 安裝 Termux
2. ✅ 安裝 PostgreSQL、Redis、Nginx
3. ✅ 設定 glibc Loader
4. ✅ 設定服務管理
5. ✅ 設定卷管理
6. ✅ 設定自動備份

### 下一步

- 探索更多服務：MongoDB、Docker (rootless)、Kubernetes (輕量版)
- 優化效能：調整配置、監控資源
- 建立自己的伺服器發行版

### 保持更新

```bash
# 定期更新套件
pkg update && pkg upgrade

# 檢查新版本
pkg list-upgradable
```

祝你在 Android 手機上架設伺服器愉快！