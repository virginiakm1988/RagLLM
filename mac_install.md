# 🍏 在 macOS 上使用 Ollama 和 AnythingLLM 建立支援 RAG 的本地聊天機器人

## 目錄

- [目標](#目標)
- [注意事項](#注意事項)
- [步驟 1：安裝 Docker Desktop for Mac](#步驟-1安裝-docker-desktop-for-mac)
- [步驟 2：啟動 Ollama 服務並下載模型](#步驟-2啟動-ollama-服務並下載模型)
- [步驟 3：啟動 AnythingLLM 服務](#步驟-3啟動-anythingllm-服務)
- [步驟 4：在瀏覽器中配置 AnythingLLM](#步驟-4在瀏覽器中配置-anythingllm)
- [步驟 5：使用 AnythingLLM 的 RAG 能力](#步驟-5使用-anythingllm-的-rag-能力)
- [常見錯誤與解決方案](#常見錯誤與解決方案)

---

本教學指導你如何在 **macOS** (包括 Intel 與 Apple Silicon M1/M2/M3) 上使用 Docker 啟動 Ollama 與 AnythingLLM，建立支援文件應答 (RAG) 的本地聊天機器人系統。

---

## 步驟 1：安裝 Docker Desktop for Mac

1. 前往官方網站下載： [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)

2. 根據你的 Mac 架構選擇版本：

   - Apple Silicon（M1/M2/M3）：選擇 ARM 版
   - Intel：選擇 x86\_64 版

3. 安裝後啟動 Docker Desktop，確認右上角狀態列出現小鯨魚圖示 🐳。

4. 開啟 Terminal，確認 Docker 是否安裝成功：

```bash
docker --version
```

你應該會看到類似 `Docker version 24.x.x, build XXXXX` 的輸出。

5. 測試 Docker 是否能正常運作：

```bash
docker run hello-world
```

成功執行後，你會看到 Hello from Docker! 的訊息，表示 Docker 已經準備就緒。

---

## 步驟 2：啟動 Ollama 服務並下載模型

macOS 不支援 NVIDIA GPU，因此不需加入 `--gpus all`，Ollama 會自動使用 CPU 或 Apple Neural Engine (針對 Apple Silicon)。

```bash
docker run -d \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  --name ollama \
  ollama/ollama
```

執行後確認容器正在運作：

```bash
docker ps
```

### 下載語言模型

進入容器：

```bash
docker exec -it ollama bash
```

下載模型（建議：llama3.2:1b，測試方便）：

```bash
ollama pull llama3.2:1b
exit
```

---

## AnythingLLM 在 macOS 上的特殊設定與注意事項

- Docker Volume 掛載目錄請使用完整 `$HOME` 路徑避免權限問題。
- `.env` 檔案的權限要確保容器內能讀取（建議放空，預設也能運作）。
- 啟動命令與 Ubuntu 相同，但無需 GPU 參數：

```bash
docker run -d -p 3001:3001 \
  --cap-add SYS_ADMIN \
  -v "$HOME/anythingllm:/app/server/storage" \
  -v "$HOME/anythingllm/.env:/app/server/.env" \
  -e STORAGE_DIR="/app/server/storage" \
  mintplexlabs/anythingllm
```

---

## host.docker.internal 的使用與連接測試

macOS 原生支援 `host.docker.internal`，可用來讓 AnythingLLM 容器連接主機的 Ollama。

### 測試 Ollama 介面是否可達：

在本機 Terminal 執行：

```bash
curl http://localhost:11434
```

在容器內測試（進入 AnythingLLM 容器內）：

```bash
docker exec -it <anythingllm_container_id> bash
curl http://host.docker.internal:11434
```

若收到 JSON 回應（或錯誤 JSON）即代表可連線。

---

## 常見錯誤與解決方案

| 錯誤訊息                                                      | 解決方式                                                                                                                                                   |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `docker: command not found`                               | 請確認 Docker Desktop 已安裝且已啟動                                                                                                                             |
| `Bind for 0.0.0.0:3001 failed: port is already allocated` | 端口 3001 被佔用，執行 `lsof -i tcp:3001` 並 `kill -9 <PID>`                                                                                                    |
| AnythingLLM 顯示 "Ollama instance could not be reached"     | 1) 確保 Ollama 正常運作 (`docker ps`)2) 在 AnythingLLM 設定中手動輸入 `http://host.docker.internal:11434` 而非 Auto-Detect3) 模型是否有下載完成？可再次執行 `ollama pull llama3.2:1b` |
| Permission denied mounting volume                         | 確保 `$HOME/anythingllm` 資料夾與 `.env` 檔權限允許 Docker container 存取                                                                                           |

