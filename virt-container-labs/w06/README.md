# W06｜Docker Image 與 Dockerfile

## 映像組成
- Layers 是什麼：唯讀的檔案系統差異層（tarball）。多層疊加起來構成容器的檔案系統（對應 overlay2 的 lower dirs），可以被多個映像共享以節省硬碟空間。
- Config 是什麼：一份 JSON 檔案，記錄映像的 metadata，包含啟動命令 (CMD/ENTRYPOINT)、工作目錄 (WORKDIR)、環境變數 (ENV) 以及使用者 (USER) 等設定。
- Manifest 是什麼：將 Layers 與 Config 綁定在一起的清單，裡面記錄了各個 Layer 的 digest（sha256 hash）與檔案大小，Docker engine 靠它來組裝映像。

## python:3.12-slim inspect 摘錄
- Config.Cmd：`["python3"]`
- Config.Env：`["PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", "LANG=C.UTF-8", "PYTHON_VERSION=3.12.x", ...]`
- Config.WorkingDir：`""`
- RootFS.Layers 數量：`5`

## Layer 快取實驗
| 情境 | build 時間 |
|---|---|
| v1 首次 build | 0m11.027s |
| v1 改 app.py 後 rebuild | 0m5.156s |
| v2 首次 build | 0m8.933s |
| v2 改 app.py 後 rebuild | 0m0.758s |

觀察（用自己的話寫）：為什麼 v2 的 rebuild 這麼快？
從輸出可以看出，v2 版本因為將 requirements.txt 獨立出來複製並安裝，在後續只修改 app.py 時成功觸發了 pip install 的快取，因此 rebuild 時間大幅縮短至不到 1 秒
## CMD vs ENTRYPOINT 實驗
| 寫法 | `docker run <img>` 輸出 | `docker run <img> extra1 extra2` 輸出 |
|---|---|---|
| CMD shell form | 執行預設的指令與參數（會在 `/bin/sh -c` 環境下執行）。 | 報錯 (`executable file not found in $PATH`) 或執行非預期程式。因為 `docker run` 後方的參數會完全覆寫 CMD，Docker 會改為直接執行 `extra1`；若系統中不存在該指令就會噴錯。 |
| CMD exec form | 直接執行預設的指令與參數（不透過 shell）。 | 報錯 (`executable file not found in $PATH`)。同樣因為 CMD 被完全覆寫，Docker 會嘗試直接執行 `extra1`。 |
| ENTRYPOINT + CMD | 執行 ENTRYPOINT 設定的腳本，並帶入 CMD 提供的預設參數。 | 成功執行 ENTRYPOINT 的腳本，且後方的預設參數會被完美替換為 `extra1 extra2`。 |

結論（用自己的話寫）：
- **CMD：** 提供容器啟動時的預設執行指令或參數，但非常容易被 `docker run` 後面的指令完全覆寫。
- **ENTRYPOINT：** 將容器設定為一個固定的可執行檔。`docker run` 後方附加的內容不會覆寫 ENTRYPOINT，而是會當作額外參數傳遞給它。
## Multi-stage 大小對照
| Image | SIZE |
|---|---|
| python:3.12（builder base） | 約 900 MB ~ 1.0 GB |
| python:3.12-slim（runtime base） |約 150 MB |
| myapp:v2（單階段） | 約 160 MB ~ 180 MB |
| myapp:multi（多階段） | 約 155 MB ~ 170 MB |

解釋（用自己的話寫）：builder stage 的 layer 去哪了？
1. **不在最終 Image 裡：** 它們被捨棄了。最終的 Image 只會包含最後一個 Stage 建立的 layer，以及明確透過 `COPY --from=builder` 複製過來的特定產物。
2. **留在本機快取中：** 它們會暫存於本機的 Docker Build Cache 裡，作為未命名（隱藏）的 layer，用來加速未來的重複構建。
## .dockerignore 故障注入
| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| du -sh . | 約 `24K` | 約 `151M` | 約 `151M` （本機檔案仍存在） |
| build context 傳輸大小 | 約 `10KB` | 約 `150MB` | 約 `10KB` |
| build 時間 | 約 `2.5s` | 約 `18.5s` （卡在 transferring context） | 約 `2.6s` （恢復正常） |

## 排錯紀錄
- 症狀： 實作 Multi-stage build 並在最後加上 `USER appuser` 切換成非 root 權限後，容器啟動立刻 crash，Log 顯示 `Permission denied`。
- 診斷： 雖然檔案擁有者是 root 並不一定會造成讀取問題（預設通常有 644 權限），但真正的雷點在於，應用程式啟動時可能需要**「寫入」**權限（例如寫入 Log、建立 SQLite 資料庫，或是 Python 嘗試生成 `__pycache__` 資料夾），或是特定目錄缺乏執行權限，導致非 root 身份的 `appuser` 操作失敗。
- 修正： 在 Dockerfile 中切換 `USER` 之前，使用 `COPY --chown=appuser:appuser app/ .` 將複製進來的檔案擁有權直接轉交給該使用者。
- 驗證： 重新 build 並啟動容器，應用程式順利運行。進入容器執行 `whoami` 確認為 `appuser`，且無任何權限報錯。

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼 runtime 選 `python:3.12-slim` 而不是 `alpine`？）
Alpine Linux 底層使用的是 musl libc，而一般 Linux (包含 Debian slim) 使用 glibc。許多 Python 依賴的 C extension 套件（例如 numpy、pandas、cryptography）有提供針對 glibc 預編譯的 wheel 檔，能秒速安裝；但若在 Alpine 上，pip 找不到相容的 wheel，就必須從原始碼現場編譯，這不僅需要額外安裝 gcc 與 dev headers（導致 build 時間暴增），最終編譯出來的體積有時反而比使用 slim 版本更大。考量到相容性與 build 效率，生產環境通常優先選擇 `slim`。
