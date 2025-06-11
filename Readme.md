# 在本地電腦使用 Ollama 和 AnythingLLM 建立聊天機器人

本教學將引導你如何在自己的 **Ubuntu 電腦** 上，使用 Docker 來運行 Ollama 服務（並利用你的 NVIDIA GPU ），然後搭配 AnythingLLM 建立一個功能強大且支援 RAG (檢索增強生成) 的聊天機器人。

**目標：**

1.  成功在本地 Ubuntu 系統上安裝 Docker 並啟用 NVIDIA GPU 支援。
2.  成功在本地啟動 Ollama 服務並下載語言模型（利用 GPU 加速）。
3.  成功在本地啟動 AnythingLLM 服務，並將其與 Ollama 連結。
4.  了解如何在 AnythingLLM 中上傳文件，以實現 RAG 功能。

**重要提醒：**
* **請仔細按照每一步操作，並確認命令的輸出。**
* 你將主要在 **Ubuntu 終端機** 中執行命令。
* 你需要一個 **NVIDIA GPU** 來利用此教學中的 GPU 加速功能。

---

## 準備工作：在 Ubuntu 上安裝 Docker 與 NVIDIA GPU 支援

這一步將在你的 Ubuntu 系統上安裝 Docker Community Edition (CE) 和 NVIDIA Docker Container Runtime，以確保你的 Docker 容器能夠訪問 NVIDIA GPU。

**安裝 NVIDIA 設備驅動：**

1.  **確保你的 Ubuntu 系統已安裝 NVIDIA 顯卡驅動。** 如果尚未安裝，請參考 NVIDIA 官方文檔或你的 Ubuntu 版本指引來安裝最新的顯卡驅動。
2.  **重啟你的電腦。**
3.  在終端機中，輸入 `nvidia-smi` 並檢查輸出，確認驅動已正確安裝並顯示你的 GPU 資訊。

    ```bash
    user@host:~$ nvidia-smi
    ```

**安裝 Docker Community Edition (CE)：**

```bash
# 安裝 Docker CE
user@host:~$ sudo apt install docker.io -y

# 將當前用戶添加到 docker 組，這樣你就可以無需使用 sudo 來運行 docker 命令
user@host:~$ sudo gpasswd -a $USER docker

# 重新登入你的用戶會話以應用組變更 (或簡單地重啟終端機)
# 為了立即生效，你可以使用以下命令重新登入到當前用戶會話：
user@host:~$ sudo su - $USER
```
* **檢查點：** 驗證 Docker 是否可以運行。
    ```bash
    user@host:~$ docker run -it --rm bash echo NVIDIA GPUs rock!
    ```
    你應該會看到輸出 `NVIDIA GPUs rock!`，表示 Docker 已成功安裝並運行。

**安裝 NVIDIA Docker Container Runtime (啟用 GPU 支援)：**

```bash
# 添加 NVIDIA Docker 的 GPG 金鑰
user@host:~$ curl -s -L [https://nvidia.github.io/nvidia-docker/gpgkey](https://nvidia.github.io/nvidia-docker/gpgkey) | sudo apt-key add -

# 添加 NVIDIA Docker 的軟件源列表 (自動檢測你的 Ubuntu 版本)
user@host:~$ distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
user@host:~$ curl -s -L [https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list](https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list) | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list

# 更新 apt 軟體包列表並安裝 nvidia-docker2
user@host:~$ sudo apt update && sudo apt install nvidia-docker2 -y

# 重啟 Docker daemon 以使其感知到 GPU 擴展
user@host:~$ sudo systemctl restart docker
```
* **檢查點：** 驗證 NVIDIA Docker Container Runtime 是否能順利運行 NGC 映像並訪問 GPU。
    ```bash
    user@host:~$ docker run --rm --gpus all nvcr.io/nvidia/cuda:11.0-base nvidia-smi
    ```
    你應該會看到 `nvidia-smi` 在 Docker 容器內部的輸出，顯示你的 GPU 資訊，表示 GPU 支援已成功啟用。

**重啟你的系統：**

完成所有驅動和 Docker 安裝後，**重啟你的電腦**，以確保所有更改都已正確應用。

---

## 步驟 0：清理舊的容器 (重要！)

為確保順利開始，我們將停止並刪除任何可能殘留的 Docker 容器和佔用端口的進程。

1.  **關閉所有終端視窗：**
    * 關閉所有開啟的終端視窗。這可以避免潛在的環境變量或進程殘留問題。

2.  **打開一個新的終端視窗：**
    * 使用你的 **Ubuntu 終端機**。
    * **確認：** 確保你不是在任何 Docker 容器的內部（例如，提示符不是 `root@<容器ID>:/#` 這樣的）。你應該看到像 `user@host:~$` 這樣的本地主機提示符。

3.  **停止並刪除可能的舊容器：**
    執行以下命令。如果容器不存在，你會看到 `Error: No such container`，這是正常的。

    ```bash
    # 嘗試停止所有可能的舊容器名稱
    docker stop ollama anythingllm tender_heyrovsky strange_austin

    # 嘗試刪除所有可能的舊容器名稱
    docker rm ollama anythingllm tender_heyrovsky strange_austin
    ```

4.  **釋放 3001 端口 (如果它被佔用)：**
    如果之前遇到「port is already allocated」錯誤，可能是 3001 端口被其他非 Docker 進程佔用。

    * 在你的 Ubuntu 終端中執行以下命令來查找佔用 3001 端口的進程 PID：

        ```bash
        sudo netstat -tulpn | grep 3001
        ```

        你會看到類似 `tcp        0      0 0.0.0.0:3001            0.0.0.0:* LISTEN      1234/program_name` 的輸出。記下 `/` 前面的數字（例如 `1234`），這是 PID。

    * **終止該進程：** 將 `1234` 替換為你實際找到的 PID。

        ```bash
        sudo kill -9 1234
        ```

---

## 步驟 1：啟動 Ollama 服務並下載語言模型

這一步將在你的本地電腦上運行 Ollama，並下載一個大型語言模型。

1.  **啟動 Ollama 容器（啟用 GPU 支援）：**
    回到你的 **Ubuntu 終端**，執行以下命令。**請注意，我們添加了 `--gpus all` 參數來啟用 GPU 加速。**

    ```bash
    docker run -d --gpus all -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
    ```
    * `-d`: 在後台運行容器。
    * `--gpus all`: 這是啟用 NVIDIA GPU 支援的關鍵參數，讓容器可以訪問所有可用的 GPU。
    * `-v ollama:/root/.ollama`: 將主機上的 `ollama` 數據卷掛載到容器內部的 `/root/.ollama`，用於持久化模型數據。
    * `-p 11434:11434`: 將主機的 11434 端口映射到容器的 11434 端口。
    * `--name ollama`: 為容器指定一個名稱為 `ollama`。

    * **檢查點：** 執行此命令後，你應該會看到一串長長的容器 ID。這表示容器已成功啟動。
    * 你可以運行 `docker ps` 來確認 `ollama` 容器正在運行。

2.  **進入 Ollama 容器並下載模型：**
    現在，我們將進入剛剛啟動的 Ollama 容器內部，下載一個語言模型。

    ```bash
    docker exec -it ollama bash
    ```
    * **檢查點：** 執行此命令後，你的終端提示符會變成類似 `root@<容器ID>:/#` 的樣子。這說明你已經成功進入了容器內部。

    在容器內部，執行以下命令下載 `llama3.2:1b` 模型 (建議使用此模型，大小較小，易於測試)：

    ```bash
    ollama pull llama3.2:1b
    ```
    * 這會從 Ollama 的模型庫下載指定模型。下載速度取決於你的網絡帶寬。
    * **檢查點：** 等待下載完成，直到你看到模型已成功下載的提示。

    下載完成後，輸入 `exit` 退出容器的 shell：

    ```bash
    exit
    ```
    * **檢查點：** 你會回到你本地主機的終端提示符。

---

## 步驟 2：設置 AnythingLLM 的儲存並啟動服務

AnythingLLM 將作為你的聊天機器人前端應用。

1.  **設置 AnythingLLM 的儲存位置：**
    在你的**本地主機終端**中，執行以下三行命令。這將在你主目錄下創建一個 `anythingllm` 文件夾，並準備一個 `.env` 文件。

    ```bash
    export STORAGE_LOCATION=$HOME/anythingllm
    mkdir -p "$STORAGE_LOCATION"
    touch "$STORAGE_LOCATION/.env"
    ```
    * `export STORAGE_LOCATION=$HOME/anythingllm`: 定義一個環境變量，指向你希望 AnythingLLM 儲存數據的目錄。
    * `mkdir -p "$STORAGE_LOCATION"`: 創建這個目錄（如果它不存在）。
    * `touch "$STORAGE_LOCATION/.env"`: 創建一個空的 `.env` 文件。

    * **檢查點：** 執行這些命令後，通常不會有輸出。你可以通過 `echo $STORAGE_LOCATION` 來確認變量已設置。

2.  **啟動 AnythingLLM 容器：**
    現在，在你的**本地主機終端**中，執行以下命令來啟動 AnythingLLM 容器：

    ```bash
    docker run -d -p 3001:3001 \
    --cap-add SYS_ADMIN \
    -v "${STORAGE_LOCATION}:/app/server/storage" \
    -v "${STORAGE_LOCATION}/.env:/app/server/.env" \
    -e STORAGE_DIR="/app/server/storage" \
    mintplexlabs/anythingllm
    ```
    * `-p 3001:3001`: 將主機的 3001 端口映射到容器的 3001 端口。
    * `-v "${STORAGE_LOCATION}:/app/server/storage"`: 將主機上你創建的 `anythingllm` 目錄掛載到容器中。

    * **檢查點：** 執行此命令後，你應該會看到一串長長的容器 ID，表示容器正在後台啟動。
    * 你可以運行 `docker ps` 來確認 `anythingllm` 容器正在運行（其名稱通常是隨機生成的，例如 `tender_heyrovsky`）。

---

## 步驟 3：在瀏覽器中配置 AnythingLLM

現在，你的 Ollama 和 AnythingLLM 服務都在運行了。最後一步是在 AnythingLLM 的網頁介面中配置它們的連接。

1.  **訪問 AnythingLLM 界面：**
    打開你的網頁瀏覽器，訪問以下地址：

    ```
    http://localhost:3001
    ```
    * **檢查點：** 你應該會看到 AnythingLLM 的歡迎界面、登錄界面或初始設置嚮導。

2.  **完成 AnythingLLM 的初始設置：**
    * 如果你是首次訪問，AnythingLLM 會引導你完成一些初始設置，例如創建管理員用戶名和密碼。按照屏幕上的提示操作。

3.  **配置 AnythingLLM 連接 Ollama：**
    * 登錄 AnythingLLM 後，導航到左側菜單中的 **"Settings" (設置)** 頁面。
    * 在設置頁面中，找到並點擊 **"LLM Preference" (LLM 首選項)**。
    * 在模型提供者列表中，選擇 **"Ollama"**。

    * **Ollama Base URL (Ollama 基礎 URL):**
        * 在輸入框中，**手動輸入**以下地址：
            ```
            [http://host.docker.internal:11434](http://host.docker.internal:11434)
            ```
        * **重要說明：** `host.docker.internal` 是一個特殊的 Docker 網絡別名。AnythingLLM 容器需要通過這個地址來訪問運行在你主機上的 Ollama 容器。在 Windows 上透過 WSL2 運行 Docker Desktop 時，這個地址通常有效。對於**純 Linux 系統**上的 Docker Engine，你可能需要嘗試 `http://172.17.0.1:11434` (這是 Docker 默認橋接網絡的網關 IP)，或者確保兩個容器位於相同的自定義 Docker 網絡中。
        * **不要點擊 "Auto-Detect"**，手動輸入能確保連接正確。
    * **Ollama Model (Ollama 模型):**
        * 在你輸入了正確的 "Ollama Base URL" 後，稍等片刻，AnythingLLM 會自動檢測到 Ollama 中可用的模型。
        * 從下拉菜單中選擇你下載的模型，例如 `llama3.2:1b`。
    * 其他設置（如 `Max Tokens`, `Performance Mode` 等）可以保持默認或根據需要調整。

4.  **保存設置：**
    點擊頁面底部的 **"Save Settings" (保存設置)** 按鈕。

---

## 步驟 4：使用 AnythingLLM (RAG 功能)

AnythingLLM 內建了 RAG (檢索增強生成) 功能，你可以上傳自己的文件，讓聊天機器人參考這些文件進行回答。

1.  **創建一個工作區 (Workspace)：**
    * 在 AnythingLLM 的主界面，點擊 **"Workspaces"** (工作區)。
    * 點擊 **"New Workspace"** (新建工作區) 來創建一個新的聊天環境。給它一個名字。

2.  **上傳文件：**
    * 在你的工作區中，你會看到一個選項來上傳文件（例如 PDF、TXT、Markdown 文件等）。
    * 點擊上傳按鈕，選擇你想要聊天機器人學習的文件。AnythingLLM 會自動處理這些文件並建立索引。

3.  **開始對話：**
    * 文件處理完成後，你就可以在工作區的聊天框中與聊天機器人對話了。
    * 嘗試提出與你上傳文件內容相關的問題，觀察聊天機器人是否能參考文件中的資訊來回答。

---

## 結語與常見問題排除

恭喜你！你已經成功地在本地電腦上搭建並運行了一個基於 Ollama 和 AnythingLLM 的中文聊天機器人！

**常見問題與解決方案：**

* **`docker: error during connect`：**
    * **解決方案：** 確保 **Docker Daemon 服務** 正在運行。在 Ubuntu 上，你可以檢查 `sudo systemctl status docker`，如果沒有運行，使用 `sudo systemctl start docker` 啟動它。
* **`Bind for 0.0.0.0:3001 failed: port is already allocated.`：**
    * **解決方案：** 3001 端口被佔用。請回到 **步驟 0** 中的「釋放 3001 端口」部分，按照指示查找並終止佔用該端口的進程。
* **`bash: docker: command not found`：**
    * **解決方案：** 你可能在 Docker 容器內部（提示符為 `root@<容器ID>:/#`）執行了 Docker 命令。請輸入 `exit` 退回你的本地主機終端，然後再執行命令。
* **AnythingLLM 顯示 "Ollama instance could not be reached"：**
    * **解決方案：**
        1.  **Ollama 容器是否運行？** 在終端執行 `docker ps`，確認 `ollama` 容器狀態是 `Up ... (healthy)`。
        2.  **Ollama API 是否可訪問？** 在**本地主機終端**執行 `curl http://localhost:11434`。如果看到 JSON 輸出（即使是錯誤訊息），表示 Ollama 正在響應。如果 `Connection refused`，Ollama 服務可能沒有正確啟動。
        3.  **AnythingLLM 配置正確嗎？** 在 AnythingLLM 的 LLM Preference 頁面，確認 Ollama Base URL **精確地** 是 `http://host.docker.internal:11434` (或對於純 Linux 系統，`http://172.17.0.1:11434`)。
        4.  **模型是否下載成功？** 在 Ollama 的模型下拉菜單中，是否看到了 `llama3.2:1b`？如果沒有，請確認在 Ollama 容器內部成功執行了 `ollama pull llama3.2:1b`。


