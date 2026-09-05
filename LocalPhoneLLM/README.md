# LocalPhoneLLM — ROG Phone 5s 本地 LLM 計畫

> 目標：ROG Phone 5s 在 Termux 跑 `llama.cpp`，offload 到 Adreno 660，
> 以 `llama-server` 提供 OpenAI-compatible API（`127.0.0.1:8080`）。
> 不 Root、不解鎖 Bootloader。

```text
ROG Phone 5s (Android 11, SD 888+, Adreno 660)
  └── Termux
        ├── llama.cpp（預編譯包 + Vulkan backend）
        ├── Mesa Turnip → /dev/kgsl-3d0
        └── Gemma 4 E2B-it Q4_0 → llama-server :8080
```

## 1. 現況（2026-09-05 實機驗證）

| 項目 | 實測值 | 狀態 |
| --- | --- | --- |
| 機型 / Android / ABI | `ASUS_I005DC` / 11 / `arm64-v8a`（`lahaina`） | ✅ |
| GPU | Adreno 660，`/dev/kgsl-3d0` 存在 | ✅ |
| RAM / Swap / 儲存 | 11GB（可用約 6.3GB）/ 4GB / 約 213GB 可用 | ✅ |
| Termux + sshd（`rog5s`） | 正常 | ✅ |
| `vulkan-tools`、`vulkan-loader-generic` | 已安裝 | ✅ |
| Turnip ICD（`freedreno_icd.aarch64.json`） | ✅ 已安裝（`mesa-vulkan-icd-freedreno 26.0.6-3`） | ✅ 2026-09-05 |
| `llama-cpp` / `llama-cpp-backend-vulkan` | ✅ 已安裝（`v0.3.0-0`） | ✅ 2026-09-05 |
| `tmux`（背景任務用） | ✅ 已安裝（`3.7c`） | ✅ 2026-09-05 |
| `~/models` | ✅ `gemma-4-E2B-it-Q4_0.gguf`（2841481184 bytes，官方值一致） | ✅ 2026-09-05 |

**Phase 1 Gate 全過**：`vulkaninfo` → `Turnip Adreno (TM) 660`（`DRIVER_ID_MESA_TURNIP`）；
`llama-cli --list-devices` → `Vulkan0: Turnip Adreno (TM) 660（8433 MiB，5650 MiB free）`。

結論：Phase 1 完成，下一步 **Phase 2 下載模型**。

**Phase 2 Gate 全過（2026-09-05，用戶本機驗證）**：模型 2841481184 bytes；
`llama_prepare_model_devices: using device Vulkan0 (Turnip Adreno (TM) 660)`；
`load_tensors: offloaded 36/36 layers to GPU`
（CPU buffer 2076 MiB / Vulkan0 buffer 1026.76 MiB，KV cache 亦在 Vulkan0）；
`/v1/chat/completions` 回 200 正常回答。下一步 **Phase 3 Benchmark（待用戶指示）**。

## 2. 已定案的決策（含舊版錯誤更正）

| 決策 | 內容 |
| --- | --- |
| 引擎 | Termux 預編譯 `llama-cpp`＋`llama-cpp-backend-vulkan`（免編譯，`pkg` 直裝） |
| GPU driver | **直接用 Turnip**（`mesa-vulkan-icd-freedreno`，x11-repo）。高通原廠 driver 跑 Q4_K 會 pipeline creation 失敗 → segfault，不測 |
| 模型首測 | `ggml-org/gemma-4-E2B-it-GGUF` 的 `Q4_0`（**2.84GB**，舊版誤寫 1.4GB）。E2B＝effective 2.3B（總計約 5.1B），128K context，先用 `-c 4096` |
| Snapdragon backend | 第三階段比較用（Docker cross-compile＋adb），不是起點 |
| 不碰 | Ollama / MLC / MediaPipe |
| loader | 沿用 `vulkan-loader-generic`，**勿裝**互斥的 `vulkan-loader-android` |
| 預設 backend | **CPU（`-ngl 0`）**：decode 8.80 t/s vs GPU 4.81 t/s，聊天場景快近 2 倍；GPU 僅 prefill 佔優，列為備選 |

## 3. 執行計畫

操作一律經 SSH（alias 定義見 `.opencode/prompts/ROG5s.md`）：

```bash
ssh rog5s "bash -l -c '<command>'"
```

### Phase 1 — 讓 GPU 可見（Gate：`llama-cli --list-devices` 看到 660 才往下）

```bash
# 1-1 更新＋裝 Turnip
ssh rog5s "bash -l -c 'pkg update && pkg upgrade -y && pkg install -y x11-repo tur-repo && pkg install -y mesa-vulkan-icd-freedreno'"
# 1-2 確認 ICD 出現：期待 freedreno_icd.aarch64.json
ssh rog5s "bash -l -c 'ls \$PREFIX/share/vulkan/icd.d/'"
# 1-3 確認 Vulkan 看到硬體而非 llvmpipe：期待 Turnip Adreno (TM) 660
ssh rog5s "bash -l -c 'VK_ICD_FILENAMES=\$PREFIX/share/vulkan/icd.d/freedreno_icd.aarch64.json vulkaninfo --summary 2>&1 | grep -i adreno'"
# 1-4 裝 llama.cpp
ssh rog5s "bash -l -c 'pkg install -y llama-cpp llama-cpp-backend-vulkan'"
# 1-5 確認 llama.cpp 看到 GPU：期待 ggml_vulkan: 0 = Turnip Adreno (TM) 660
ssh rog5s "bash -l -c 'VK_ICD_FILENAMES=\$PREFIX/share/vulkan/icd.d/freedreno_icd.aarch64.json llama-cli --list-devices'"
```

出錯才用的 workarounds：`ASAN_OPTIONS=detect_leaks=0`、
`EXECUTABLE_DISABLE_MTE=1`、`KMP_AFFINITY=disabled`、`--no-warmup`、`-t 3`。

### Phase 2 — 模型＋server＋驗證 offload

```bash
# 2-1 下載模型（約 2.84GB，`-c` 斷點續傳＋無限重試，中斷後重跑同一指令即續傳）
ssh rog5s "bash -l -c 'mkdir -p ~/models && cd ~/models && wget -c --tries=0 --read-timeout=30 -O gemma-4-E2B-it-Q4_0.gguf https://huggingface.co/ggml-org/gemma-4-E2B-it-GGUF/resolve/main/gemma-4-E2B-it-Q4_0.gguf'"
# 2-2 啟動 server（tmux 背景跑，>30s 任務）
ssh rog5s "tmux new-session -d -s opencode-llama 'bash -l -c \"VK_ICD_FILENAMES=\\\$PREFIX/share/vulkan/icd.d/freedreno_icd.aarch64.json KMP_AFFINITY=disabled EXECUTABLE_DISABLE_MTE=1 llama-server -m ~/models/gemma-4-E2B-it-Q4_0.gguf -ngl 99 -t 3 -c 4096 --no-mmproj --no-warmup --host 127.0.0.1 --port 8080\"'"
# 2-3 看 log 取樣
ssh rog5s "tmux capture-pane -pt opencode-llama -S -50"
```

**GPU 驗證標準**（三項全滿足，不是只看 `-ngl 99`）：

```text
ggml_vulkan: 0 = Turnip Adreno (TM) 660
load_tensors: offloaded XX/XX layers to GPU   # 全 offload，非 0/XX
```

API 驗證：`curl http://127.0.0.1:8080/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"gemma-4-e2b","messages":[{"role":"user","content":"Hello"}]}'`，手機瀏覽器開 `:8080` 亦有 Web UI。

### 推薦運行指令（CPU＋關 thinking＋關多模態）

benchmark 結論：聊天是 decode-heavy，CPU（8.80 t/s）比 GPU（4.81 t/s）快近 2 倍，
故預設用 CPU；GPU 指令保留作 prefill-heavy 場景備選。

```bash
# tmux 背景常駐（推薦，log 寫 server-cpu.log）
ssh rog5s "tmux new-session -d -s opencode-llama 'bash -l -c \"KMP_AFFINITY=disabled EXECUTABLE_DISABLE_MTE=1 llama-server -m ~/models/gemma-4-E2B-it-Q4_0.gguf --alias gemma4-e2b-q4_0 --no-mmproj --reasoning off -ngl 0 -t 3 -c 4096 --no-warmup --host 0.0.0.0 --port 8080 > server-cpu.log 2>&1\\"\'"
```

```bash
# 本機 Termux 前景（除錯用）
KMP_AFFINITY=disabled EXECUTABLE_DISABLE_MTE=1 llama-server -m models/gemma-4-E2B-it-Q4_0.gguf --alias gemma4-e2b-q4_0 --no-mmproj --reasoning off -ngl 0 -t 3 -c 4096 --no-warmup --host 0.0.0.0 --port 8080
```

旗標：`-ngl 0` 跑 CPU；`--reasoning off` 關 thinking（回應無 `reasoning_content`）；
`--no-mmproj` 關多模態；`--api-key` 需用 `Authorization: Bearer <key>` 存取。
Key 優先順序（`llama-start.sh` 實作）：裝置 env `LLAMA_API_KEY` ＞ `~/.env` 內
`LLAMA_API_KEY=...` ＞ 預設 `sk-1234`。實測驗證：decode 8.77 t/s（與 bench 8.80 一致）、thinking 已關。

手機本機另有快捷腳本（`~/llama-start.sh` 啟動常駐、`~/llama-stop.sh` 停止並完全卸載，
含 port 釋放驗證），本機直接 `bash ~/llama-start.sh` / `bash ~/llama-stop.sh` 即可。

### Phase 3 — Benchmark（實測後填）

| 測試 | CPU（`-ngl 0`） | Turnip（`-ngl 99`） |
| --- | --- | --- |
| 載入時間 | | |
| Prompt 處理 tok/s（pp128） | 25.99 | 32.71 |
| Decode tok/s（tg64） | 8.80 | 4.81 |
| 連續 10 分鐘穩定性 / 溫度 | 未測 | 未測 |

> 方法：`llama-bench -m models/gemma-4-E2B-it-Q4_0.gguf -t 3 -p 128 -n 64`，
> GPU 組另加 Turnip `VK_ICD_FILENAMES`。結論：GPU 贏 prefill，CPU 贏 decode 近 2 倍
>（unified memory 頻寬競爭，符合社群經驗）。server（`-ngl 99`）實測 decode 4.77 t/s
> 與 GPU bench 一致。server 另加 `--reasoning off --no-mmproj`，回應已無 `reasoning_content`。

通過標準：GPU offload 全層成功、decode 可用、10 分鐘不 crash。

### Phase 4（選做）— Snapdragon backend 比較

官方 Docker toolchain（`GGML_OPENCL=ON`＋`GGML_HEXAGON=ON`）cross-compile
後 `adb push`，比較 CPU / Adreno OpenCL / Hexagon NPU。
文件：`docs/backend/snapdragon/README.md`。

## 4. 風險與回退

| 風險 | 回退 |
| --- | --- |
| OOM / 太慢 | 降級小模型（社群 `Q4_K_M`、Gemma 3 270M）或降 quant（`Q3_K_M`）、縮 `-c` |
| Turnip 在 660 上不穩 | 先跑 CPU baseline；再考慮 Termux 內自編譯（`git cmake clang ninja`＋`vulkan-headers shaderc`，`-DGGML_VULKAN=ON`） |
| 誤裝 `vulkan-loader-android` | 移除，保留 `vulkan-loader-generic` |

## 5. 參考

- 官方 Android（Termux no-root）：`ggml-org/llama.cpp` → `docs/android.md`
- 官方 Snapdragon backend：同 repo → `docs/backend/snapdragon/README.md`
- Termux＋Turnip 實戰：`qapdex-maker/llama-cpp-vulkan-termux`、`sanatani-hackers/Llama.cpp-termux`
- 模型：`huggingface.co/ggml-org/gemma-4-E2B-it-GGUF`（Q4_0 2.84GB / Q8_0 4.97GB / BF16 9.31GB）
