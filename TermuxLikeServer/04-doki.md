# 在 Termux 運行 Doki

用 Doki 在未 Root 的 Android 手機上跑真正的 OCI 容器（Docker Hub 的 image 直接 `pull` 來跑），不用 proot-distro 包 tarball，也不用自己搞 glibc。

> 前置閱讀：[01 先決條件](./01-prerequisites.md) → [02 Termux + SSH](./02-install-termux.md) → [03 glibc Loader](./03-glibc-loader.md)（觀念有幫助，但跑 Doki 不需要你手動裝 glibc）。

## 1. 為什麼用 Doki？

| 之前的做法 | 痛點 | Doki 怎麼解 |
|------------|------|-------------|
| proot-distro | 只能裝發行版 tarball，不是 OCI image | `doki pull nginx:alpine` 直接跑 Docker Hub |
| glibc-loader 手動跑 binary | 一個一個 binary 修 interpreter / rpath | image 層自動解開 + 選對 runner（預設 proot） |
| 想用 Docker Compose / K8s YAML | 手機上沒有 daemon | `doki-compose`、`doki kube play` 單 binary 搞定 |

Doki 重點特性（v0.12.0）：

- **13MB binary，idle 12MB，啟動 <15ms，零依賴**
- **Rootless-first，無 daemon 需求**（`dokid` 自己就是 daemon）
- **Docker Engine API v1.54 + Podman API v5** 同一個 socket
- **12 層隔離自動選**：手機沒 root 預設走 `proot`，有 KVM / namespace 的機器會自動往上選
- **Termux 專門處理過**：host network fallback、DNS 改 `8053`、socat 做 port forwarding

## 2. 前置檢查

```bash
# 架構（ARM64 最順，ARMv7 也能跑）
uname -m
# 預期：aarch64（ARMv7 顯示 armv7l 也行，只是 image 要找 arm32 版）

# Termux 一定要 F-Droid 版，Play 版太舊
termux-info | head -20

# 先更新 + 裝基本工具（02 章做過可跳過）
pkg update && pkg upgrade -y
pkg install -y wget curl git vim openssh proot socat
```

> `proot`、`socat` 是 Doki 在 Termux 上的左右手：沒 namespace 就靠 proot 隔離，沒 iptables 就靠 socat 轉 port。

## 3. 安裝 Doki

### 3.1 一鍵腳本（推薦）

```bash
# 一定要用 bash，不能用 sh
# 腳本第 2 行是 set -Eeuo pipefail，sh（dash/mksh）不支援會直接報錯
curl -sL https://dok1.xyz | bash

# 驗證
doki version
dokid --help
doki-compose --help
```

> ❌ 錯誤寫法：`curl -sL https://dok1.xyz | sh`
> 會出現 `sh: 2: set: Illegal option -o pipefail`，因為 Termux 的 `sh` 不是 `bash`。

### 3.2 手動下載（腳本失敗時）

到 [Doki releases](https://github.com/OpceanAI/Doki/releases) 找對應版：

- Android ARM64 (Termux)：`doki-android-arm64`、`dokid-android-arm64`…
- Android ARMv7 (Termux)：`*-android-armv7`（Go 用 `GOOS=linux` 編的，走 proot 跑）

```bash
mkdir -p ~/bin
# 以 v0.12.0 為例，檔名請以 releases 頁為準
wget -O ~/bin/doki <release-url>/doki-android-arm64
wget -O ~/bin/dokid <release-url>/dokid-android-arm64
chmod +x ~/bin/doki ~/bin/dokid
export PATH="$HOME/bin:$PATH"
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.profile
doki version
```

### 3.3 PATH 說明：Termux 預設就找得到，不用加

`/data/data/com.termux/files/usr/bin` 本來就在 Termux 的 `$PATH` 裡，
所以這行是多餘的，拿掉也沒差：

```bash
# 不需要加這行，刪掉即可
export PATH="/data/data/com.termux/files/usr/bin:$PATH"
```

你的 `~/.profile` 只留 `sshd` 就好：

```
sshd
```

唯一需要加 PATH 的情況，是 3.2 手動裝到 `~/bin` 的人：

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.profile
source ~/.profile
command -v doki
```

## 4. 啟動 dokid

```bash
# 預設 socket，直接背景跑
dokid &

# 明確指定 socket（推薦寫死，避免 Docker CLI 找不到）
dokid --host unix:///data/data/com.termux/files/usr/var/run/doki.sock &

# 需要從區網 / Tailscale 連進來才開 TCP（預設不要開）
dokid --host tcp://0.0.0.0:2375 &

# 看 log
jobs
tail -f ~/.doki/dokid.log
# 或
doki info
doki ping
```

### 4.1 開機 / 每次進 Termux 自動啟（~/.profile 正確寫法）

`~/.profile` 是每次 login shell 都會跑一次（開 Termux、每次 `ssh` 進來都會跑），
所以不能裸寫 `dokid ... &`，否則會發生兩件事（你 log 裡兩個都有）：

1. 第二個 `dokid` 搶不到 port：`dns ... 8053: bind: address already in use`、`7432: bind: address already in use`
2. `dokid` 綁在那次 SSH session 底下，你一 Ctrl-C / 斷線它就收到 `signal interrupt` 跟著死掉

正確寫法：加判斷（有在跑就不要再啟）+ `nohup` + `disown` 脫離 session：

```sh
sshd
mkdir -p /data/data/com.termux/files/usr/var/run
mkdir -p /data/data/com.termux/files/home/.doki
export PATH="$HOME/bin:$PATH"

# dokid 沒在跑才啟動，避免每次 ssh 都多開一個
if ! doki ping >/dev/null 2>&1; then
    nohup dokid --host unix:///data/data/com.termux/files/usr/var/run/doki.sock >>~/.doki/dokid.log 2>&1 &
    disown
fi
```

常用 Termux 路徑（寫 config / 除錯會用到）：

| 用途 | Termux 路徑 |
|------|-------------|
| data_dir | `/data/data/com.termux/files/usr/var/lib/doki` |
| socket | `/data/data/com.termux/files/usr/var/run/doki.sock` |
| config | `~/.doki/config.json` |
| emulation 偏好 | `~/.doki/emulation.json` |

## 5. 第一次跑容器

```bash
# 拉 image（多架構自動選 arm64）
doki pull alpine
doki images

# 跑起來
doki run alpine echo "Hello from Doki"

# 互動式
doki run --rm -it alpine sh

# 背景跑 + 取名 + port
doki run -d --name web -p 8080:80 nginx:alpine

# 查看
doki ps
doki logs web
doki exec web sh -c 'cat /etc/os-release'
doki stop web
doki rm web
```

跑起來就代表隔離鏈是通的。Termux 上預設是：

```
image → rootfs → runner=proot → host network namespace（Termux App 的那份）
```

想強制指定 runner：

```bash
doki run --runtime proot alpine echo "always proot"
doki run --runtime native alpine echo "no isolation，純測速用"
```

## 6. Docker CLI / SDK 照樣用

Doki 講 Docker Engine REST API，socket 指過去就行：

```bash
export DOCKER_HOST=unix:///data/data/com.termux/files/usr/var/run/doki.sock
docker ps
docker images
docker run alpine echo "via docker cli"

# 寫進 profile 省得每次 export
echo 'export DOCKER_HOST=unix:///data/data/com.termux/files/usr/var/run/doki.sock' >> ~/.profile
```

Python / Node SDK 同理，指到同一個 socket：

```python
import docker
client = docker.DockerClient(base_url="unix:///data/data/com.termux/files/usr/var/run/doki.sock")
client.containers.run("alpine", "echo hello")
```

## 7. Compose / K8s（手機上也行）

### 7.1 Compose

```yaml
# compose.yaml
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
  api:
    image: python:3-alpine
    command: python -m http.server 8000
    depends_on:
      web:
        condition: service_started
```

```bash
doki-compose up -d
doki-compose ps
doki-compose logs
doki-compose down
```

支援 networks / volumes / secrets / healthcheck / `depends_on`（60s poll）等完整 spec，手機上跑兩三個小 service 沒問題。

### 7.2 K8s YAML（小部署不用開 cluster）

```bash
doki kube play my-app.yaml
doki-kubectl get pods
doki-kubectl logs <pod>
```

`doki-kube` 是單 binary control plane（apiserver + scheduler + controllers + kube-proxy + CoreDNS），`dokid` 順便當 CRI 用。單節點玩玩可以，正式環境還是回雲上。

## 8. Termux 專屬設定

### 8.1 Storage：用 fuse-overlayfs

rootless + 沒 overlay kernel module 的環境，`fuse-overlayfs` 最穩：

```json
// ~/.doki/config.json
{
  "data_dir": "/data/data/com.termux/files/usr/var/lib/doki",
  "socket": "/data/data/com.termux/files/usr/var/run/doki.sock",
  "storage_driver": "fuse-overlayfs",
  "rootless": true,
  "dns": {
    "listen": "127.0.0.11:8053"
  }
}
```

### 8.2 DNS：53 被 SELinux 擋，改 8053

- Linux 預設 `127.0.0.11:53`，**Termux 預設 `127.0.0.11:8053`**
- 上游 DNS 從 `getprop net.dns1..net.dns4` 讀，不用手動設
- 改位址：`DOKI_DNS_LISTEN=127.0.0.11:8053` 或 config 的 `dns.listen`
- 容器裡 `ndots:0`，`forgejo` 這種短名直接解，不會去重試 `forgejo.local`

### 8.3 網路：host + socat

- `/proc/sys/net` 沒權限時自動 fallback 到 host network via proot（跟 Termux App 共用 network namespace）
- port mapping 沒 iptables 就用 `socat`（rootless 模式），所以前面才要 `pkg install socat`
- IPv6 bridge 是 dual-stack，macOS Native 模式不適用（手機不用管）

```bash
# 常用的 port 寫法都支援
doki run -p 8080:80 nginx:alpine
doki run -p 127.0.0.1:8080:80 nginx:alpine
doki run -P nginx:alpine
```

### 8.4 跨架構：x86 image 在 ARM 上跑

```bash
# 看現在偏好
doki emu show
# 掃描可用的後端
doki emu detect
# 強制選：auto | qemu | fex | box64
doki emu set auto
```

- ARM64 主機自動偏好 FEX-Emu → Box64 → QEMU user-mode
- 存檔在 `~/.doki/emulation.json`，環境變數 `DOKI_EMULATION_MODE` / `DOKI_EMULATOR` 可覆蓋
- 效能損耗 `FEX ~30% / QEMU ~50%`，能找 arm64 image 就不要跨架構

### 8.5 環境變數速查（Termux 常用）

| 變數 | 用途 | 預設 |
|------|------|------|
| `DOKI_HOST` | daemon socket | 依平台，Termux 建議寫死上面那串 |
| `DOKI_DATA_DIR` | 資料目錄 | `~/.doki/data` |
| `DOKI_STORAGE_DRIVER` | 儲存驅動 | `fuse-overlayfs`（Termux 用這個） |
| `DOKI_DNS_LISTEN` | DNS 監聽 | `127.0.0.11:8053`（Termux） |
| `DOKI_RUNTIME` | 強制 runner | 自動偵測（手機=proot） |
| `DOKI_EMULATION_MODE` | 跨架構後端 | `auto` |
| `DOKI_DEBUG` | 開 debug（pprof `:6060`） | 未設定 |
| `DOKI_LOG_LEVEL` | `debug/info/warn/error` | `info` |

## 9. 診斷工具

```bash
# 環境體檢
doki doctor
doki deps ls
doki deps check   # CI gate 用，缺東西就非零退出
doki deps install socat
```

## 10. 故障排除

### 10.1 dokid 起不來

```bash
# 看是不是 socket 被舊的卡住
ls -lh /data/data/com.termux/files/usr/var/run/doki.sock
rm -f /data/data/com.termux/files/usr/var/run/doki.sock
dokid --host unix:///data/data/com.termux/files/usr/var/run/doki.sock &

# 看 port / 權限
doki doctor
```

### 10.2 pull 很慢 / 失敗

```bash
# 先測網路 + DNS
getprop net.dns1
ping 8.8.8.8
doki pull alpine
# 換 registry mirror（寫 config.json 的 registry_mirrors）
```

### 10.3 容器 DNS 解不到

```bash
# 確認 container 的 resolv.conf 是 127.0.0.11
doki exec <name> cat /etc/resolv.conf
# 確認 daemon 的 listen 是 8053 不是 53
cat ~/.doki/config.json | grep -A2 dns
```

### 10.4 空間不夠（手機常見）

```bash
df -h
du -sh ~/.doki/* /data/data/com.termux/files/usr/var/lib/doki/*
doki system df
doki system prune -f
doki images
doki rmi <不要的image>
pkg clean
```

### 10.5 想看隔離到底選了什麼

```bash
doki info | grep -i -A2 runtime
doki run --runtime proot alpine echo ok
doki run --runtime native alpine echo ok
```

## 11. 參考資源

- [Doki GitHub](https://github.com/OpceanAI/Doki)
- [Doki releases（抓 Android binary）](https://github.com/OpceanAI/Doki/releases)
- [Termux Wiki](https://wiki.termux.com/)
