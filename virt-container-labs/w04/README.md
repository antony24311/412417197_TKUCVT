# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑 | FHS 定義 | Docker 用途 |
|---|---|---|
| /etc/docker/ | 系統級設定檔目錄 | 存放 daemon.json 設定檔，用於自定義 Docker 運行的參數 |
| /var/lib/docker/ | 程式持久性狀態資料 | 存放映像層（Layers）、容器層、Volume 資料 |
| /usr/bin/docker | 使用者可執行檔 | Docker CLI 工具，與使用者互動 |
| /run/docker.sock | 執行期暫存 | Unix Socket 檔案，是 CLI 傳送指令給 Daemon 的唯一管道 |

## Docker 系統資訊

- Storage Driver：overlayfs
- Docker Root Dir：/var/lib/docker
- 拉取映像前 /var/lib/docker/ 大小：276K
- 拉取映像後 /var/lib/docker/ 大小：280K

## 權限結構

### Docker Socket 權限解讀
srw-rw---- 1 root docker 0 Apr 20 20:36 /run/docker.sock
* s 表示 Socket 檔案
* root (owner) 有 rw
* docker (group) 有 rw
* 其他人 (others) 是 --- 沒權限
### 使用者群組
uid=1000(antony) gid=984(docker) groups=984(docker),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),114(lpadmin),1000(antony)
目前的輸出中 並包含 docker 群組。

### 安全意涵
因為 Docker Daemon 是以 root 執行，只要你能叫得動它（在 docker group 裡），你就能掛載 Host 的敏感檔案到容器內讀取，這等同於root 權限。

## 程序與服務管理

### systemctl status docker
![image](https://hackmd.io/_uploads/HkxtZoXaZe.png)


### journalctl 日誌分析
![image](https://hackmd.io/_uploads/HJG2boXaZx.png)
```
Apr 20 20:30:32 bastion systemd[1]: Starting docker.service - Docker Application Container Engine...
Apr 20 20:30:33 bastion dockerd[1667]: time="..." level=info msg="..."
```
第一行顯示 systemd[1] 發出了 Starting docker.service 的指令。這代表 Docker 不是自己跳起來的，而是由 Linux 的總管（systemd）根據設定（可能是開機自啟或手動 systemctl start）發起啟動請求
日誌中的 dockerd[1667] 代表 Docker Daemon 的 PID (Process ID) 是 1667。這是一個典型的 Process（程序） 實例

### CLI vs Daemon 差異
執行`sudo systemctl stop docker docker.socket` 後
取消服務後依然可以利用docker version 取得版本資訊，它直接印出程式碼裡的字串，完全不需要與後台 Daemon 溝通。
只有像 docker ps 或 docker run 這種需要操作容器的指令，才需要確認 Daemon 是否活著。

## 環境變數

- $PATH：`/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin`
- which docker：`/usr/bin/docker`
- 容器內外環境變數差異觀察：（簡述）

## 故障場景一：停止 Docker Daemon

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| systemctl status docker | active | inactive | active |
| docker --version | 正常 | 正常 | 正常 |
| docker ps | 正常 | Cannot connect | 正常 |
| ps aux grep dockerd | 有 process | 無（只剩 grep 自己的程序） | 有 process |

## 故障場景二：破壞 Socket 權限

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ls -la docker.sock 權限 | srw-rw---- | srw------- | srw-rw---- |
| docker ps（不加 sudo） | 正常 | permission denied | 正常 |
| sudo docker ps | 正常 | 正常 | 正常 |
| systemctl status docker | active | active | active |

## 錯誤訊息比較

| 錯誤訊息 | 根因 | 診斷方向 |
|---|---|---|
| Cannot connect to the Docker daemon | 後台服務 (Daemon) 沒開，或是被強行關閉 | 檢查服務狀態：systemctl status docker |
| permission denied…docker.sock | 服務開著，但你的使用者帳號沒拿到進入 Socket 的門票 | 檢查權限與群組：ls -la /run/docker.sock 與 id |

Cannot connect 指的是「門關著（服務沒開）」；permission denied 指的是「門開著但你沒有鑰匙（權限不足）」

## 排錯紀錄
- 症狀：執行 ls -la docker.sock 嘗試觀察權限時，系統回報 ls: cannot access 'docker.sock': No such file or directory
- 診斷：（你首先查了什麼？）
    * 首先檢查目前所在路徑（使用 pwd），發現位於 /home/antony/Desktop。
    * 根據 FHS 標準，.sock 這類執行期暫存檔案不應存在於使用者桌面或家目錄。
    * 推測 Docker 的通訊管道應位於系統級目錄。首先檢查目前所在路徑（使用 pwd），發現位於 /home/antony/Desktop。

- 修正：放棄使用相對路徑，改用 FHS 規定的絕對路徑 /run/docker.sock 進行存取
- 驗證：執行 ls -la /run/docker.sock 後，成功看到檔案屬性 srw-rw----，證明檔案路徑正確且 Daemon 正在運行。

## 設計決策
* 取捨與優點：
在教學與開發環境中，流暢度（UX）非常重要。若每次執行 Docker 指令都要提權並輸入密碼，會增加操作負擔。透過將帳號加入 docker group，可以讓使用者像執行一般程式一樣操作容器，這符合 DevOps 自動化與快速迭代的精神。

* 風險與代碼安全性：
這等同於賦予該使用者無限制的 root 權限。 由於 Docker Daemon 是以 root 身份執行，擁有 socket 存取權的使用者可以輕易發動「掛載攻擊」。例如執行 docker run -v /:/host_root alpine，就能在容器內存取並修改 Host 主機的所有系統檔案（包括 /etc/shadow）。在生產環境（Production）中，通常會傾向使用 sudo 並搭配具體的權限限制（sudoers），或採用 Rootless Docker 來降低風險。