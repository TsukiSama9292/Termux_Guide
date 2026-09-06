# 在 Termux 跑 opencode（Bun binary 實戰）

opencode 不是用 Node.js 跑的，它是 Bun `build --compile` 打出來的單一
glibc binary（`opencode-linux-arm64`，內嵌 Bun runtime）。所以這篇不是
「用 grun 跑 node」，而是「讓 Bun 編譯產物在 Termux 活下來」。
前置知識（Bionic vs glibc、`grun`、patchelf）見 [03](./03-glibc-loader.md)。

## 1. 為什麼 `grun` 不夠

`grun` 只能解決 libc / interpreter 層。Bun binary 在 Android 10/11 上還有
第三道牆：seccomp 會擋 `statx` 等 syscall（Bun 用 inline `svc` 繞過 libc
直接呼叫），結果是開局即死：

```bash
$ grun ./opencode --version
Bad system call
$ echo $?
159   # 128 + 31 (SIGSYS)
```

以下是在 ROG 5s（Android 11）上的實測紀錄。

## 2. 實測一：C04-wq/opencode-termux（musl 包，失敗）

```bash
npm install -g opencode-termux@latest
opencode
```

結果：

```text
  ◇ Setting up Termux dependencies…
  ✓ Termux dependencies are ready
  ◇ Preparing OpenCode for Termux…
Error: Command failed: /data/data/com.termux/files/usr/tmp/opencode-termux-bvRMYq/files/opencode --version
```

對照 `install.js` 流程：下載、SHA 驗證、解包、6 檔檢查、`patchelf` 全過，
死在 staging 裡的 `--version` 冒煙測試，`~/.opencode/` 連建都沒建起來。
手動重現（輸出不再被藏起來）：

```bash
mkdir -p ~/oc-debug && cd ~/oc-debug
# 下載 Releases 的 opencode-termux-aarch64.tar.gz 並解包（略）
file files/opencode
# ELF 64-bit LSB executable, ARM aarch64 ... interpreter /lib/ld-musl-aarch64.so.1

patchelf --set-interpreter $HOME/oc-debug/files/ld-musl-aarch64.so.1 $HOME/oc-debug/files/opencode
LD_PRELOAD=$HOME/oc-debug/files/ld-musl-aarch64.so.1 \
LD_LIBRARY_PATH=$HOME/oc-debug/files \
SSL_CERT_FILE=$PREFIX/etc/tls/cert.pem \
$HOME/oc-debug/files/opencode --version; echo "exit=$?"
# Bad system call
# exit=159
```

結論：musl 路線在 Android 11 同樣死於 seccomp，換什麼 loader 都沒用，
死的是 syscall 層。另注意該包還有 issue #4：wrapper 會 `export LD_PRELOAD`
指向 musl loader，導致 opencode 生出的 bash/rg 子進程全部 segfault——
就算裝起來了 agent 的 shell 工具也是壞的。

## 3. 實測二：HanSoBored/opencode-termux（成功）

做法是 launcher + 自編 SIGSYS shim：seccomp 把 trapped syscall 交給
userspace，shim 用 `fstatat` 模擬 `statx`，其餘回 `-ENOSYS` 讓 caller
退回舊 syscall。專門為 Android 10/11 設計。

```bash
# 先清掉舊的（C04 的 npm wrapper 會蓋掉 opencode 指令），外加清 debug 目錄
npm uninstall -g opencode-termux
rm -rf ~/oc-debug

pkg install -y glibc-repo glibc glibc-runner clang curl git
git clone https://github.com/HanSoBored/opencode-termux.git
cd opencode-termux
./install.sh
# 預期最後印出版本號（安裝程式自帶驗證），例如 1.18.29
```

裝完收尾（`~/.opencode/bin` 不在 PATH，且 `$PREFIX/bin/opencode`
可能殘留斷掉的 symlink）：

```bash
rm -f $PREFIX/bin/opencode
echo 'export PATH="$HOME/.opencode/bin:$PATH"' >> ~/.profile
source ~/.profile
hash -r
command -v opencode
# /data/data/com.termux/files/home/.opencode/bin/opencode
opencode --version
# 1.18.29
```

進 TUI 後務必實測 bash 和 glob 工具（這正是 C04 那包壞掉的地方）。
這個 launcher 啟動時就清掉了有問題的 `LD_PRELOAD`，理論上沒那個坑，
但以實測為準。

## 4. 其他路線（備查）

| 方案 | 做法 | 结论 |
|------|------|------|
| retired64/opencode-termux | 一鍵腳本 + C bootstrapper 走 glibc linker | 無 shim，Android 12+ 可用，11 上撞同一道牆 |
| Hope2333/opencode-termux | bun-termux-loader 包成 .deb | 同 glibc 路線，要看 seccomp 處理 |
| guysoft/opencode-termux | 從源碼交叉編譯原生 Bionic 版 Bun + OpenCode | 最徹底，但要自己編整條工具鏈 |
| 上游 PR anomalyco/opencode#41695 | npm 包 `--ignore-scripts` + grun shim | 官方還沒合，TUI 在新版 Android 才穩 |

## 參考資源

- [HanSoBored/opencode-termux](https://github.com/HanSoBored/opencode-termux)
- [C04-wq/opencode-termux](https://github.com/C04-wq/opencode-termux)（issue #1、#4）
- [anomalyco/opencode#10504](https://github.com/anomalyco/opencode/issues/10504)（官方 binary 在 Termux 跑不起來的原始回報）
- [03 glibc Loader](./03-glibc-loader.md)
