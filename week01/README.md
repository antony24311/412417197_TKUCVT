# W01｜虛擬化概論、環境建置與 Snapshot 機制

## 環境資訊
- Host OS：（例：Windows 11 / macOS 14）
- VM 名稱：（例：vct-w01-41012345）
- Ubuntu 版本：（貼上 `lsb_release -a` 輸出）
- Docker 版本：（貼上 `sudo docker --version` 輸出）
- Docker Compose 版本：（貼上 `docker compose version` 輸出）

## VM 資源配置驗證

| 項目 | VMware 設定值 | VM 內命令 | VM 內輸出 |
|---|---|---|---|
| CPU | 2 vCPU | `lscpu \| grep "^CPU(s)"` | 2 |
| 記憶體 | 4 GB | `free -h \| grep Mem` | 3.8Gi |
| 磁碟 | 40 GB | `df -h /` | 40G |
| Hypervisor | VMware | `lscpu \| grep Hypervisor` | VMware |

## 四層驗收證據
- [x] ① Repository：`cat /etc/apt/sources.list.d/docker.list` 輸出
![image](https://hackmd.io/_uploads/rJdef2vqWl.png)

- [x] ② Engine：`dpkg -l | grep docker-ce` 輸出
![image](https://hackmd.io/_uploads/BJzSM3vcbx.png)

- [x] ③ Daemon：`sudo systemctl status docker` 顯示 active
![image](https://hackmd.io/_uploads/rJjwfnw5Zx.png)

- [x] ④ 端到端：`sudo docker run hello-world` 成功輸出
![image](https://hackmd.io/_uploads/rkfJmnPcbx.png)

- [x] Compose：`docker compose version` 可執行
![image](https://hackmd.io/_uploads/rkDjMhPcZg.png)



## 容器操作紀錄
- [x] nginx：`sudo docker run -d -p 8080:80 nginx` + `curl localhost:8080` 輸出
![image](https://hackmd.io/_uploads/HkLyWRw5-l.png)

- [x] alpine：`sudo docker run -it --rm alpine /bin/sh` 內部命令與輸出
![image](https://hackmd.io/_uploads/Hk4HbAP5Zg.png)


- [x] 映像列表：`sudo docker images` 輸出
![image](https://hackmd.io/_uploads/ryRUbRvqbl.png)

## Snapshot 清單

| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
|---|---|---|---|
| clean-baseline | 03/18 03:21 | 系統基礎環境基線。代表作業系統（Kali）已完成初始設定、網路通訊正常，且 Docker 引擎已正確安裝但尚未載入特定專案映像檔的純淨狀態。 | hostnamectl (主機識別)、ip route (網路連通)、docker --version (工具就緒)、systemctl status docker (服務運作)、docker run hello-world (功能閉環) |
| docker-ready | 03/18 03:34 | 容器化開發就緒狀態。必要的基礎映像檔（如 Nginx, Alpine）已預先拉取完成 | systemctl status docker (持續運作)、docker run hello-world (二次確認)、docker images (確認 Nginx、Alpine 映像檔已存在於本地端) |

## 故障演練三階段對照

| 項目 | 故障前（基線） | 故障中（注入後） | 回復後 |
|---|---|---|---|
| docker.list 存在 | 是 | 否 | 是  |
| apt-cache policy 有候選版本 | 是 | 否 | 是  |
| docker 重裝可行 | 是 | 否 | 是 |
| hello-world 成功 | 是 | N/A | 是  |
| nginx curl 成功 | 是 | N/A | 是  |

## 手動修復 vs Snapshot 回復

| 面向 | 手動修復 | Snapshot 回復 |
|---|---|---|
| 所需時間 | 一分鐘 | 30秒 |
| 適用情境 | 清晰且簡單的錯誤  | 複雜且還原後損失較少的情況 |
| 風險 | 高 | 低 |

## Snapshot 保留策略
- 新增條件：安裝工具或是重大更新時，且在新增前確保所有的功能是正常的
- 保留上限：最多三個活躍的snapshot
- 刪除條件：刪除最舊的

## 最小可重現命令鏈
1. 修改前
```
ls /etc/apt/sources.list.d/
apt-cache policy docker-ce | head -10
```
2. 注入
```
sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken
sudo apt update 
```
3. 修改後
```
ls /etc/apt/sources.list.d/
apt-cache policy docker-ce | head -10 
sudo apt -y install docker-ce 2>&1 | tail -5
```

## 排錯紀錄
- 症狀：
透過 `apt-cache policy docker-ce` 觀察到 Version table 異常。原本應指向https://download.docker.com/linux/ubuntu 的來源消失，僅剩本地已安裝的快取資訊（Priority 100）。
嘗試執行安裝指令時，系統提示「0 upgraded, 0 newly installed」，無法獲取遠端新版本。
![image](https://hackmd.io/_uploads/H1XeBAA9be.png)

- 診斷：
`ls /etc/apt/sources.list.d/ `檢查軟體源配置目錄
發現原本應為 `docker.list` 的設定檔被更名為 `docker.list.broken`
- 修正：
```bash=
sudo mv /etc/apt/sources.list.d/docker.list.broken /etc/apt/sources.list.d/docker.list
sudo apt update
```
- 驗證：
執行 `apt-cache policy docker-ce`，確認 Version table 中的優先級（Priority）回到 500，且顯示來源為 https://download.docker.com/linux/ubuntu
`ls /etc/apt/sources.list.d/ `目錄下已恢復正確的 `docker.list `檔案且docker可以正常運作
```bash=
sudo systemctl status docker --no-pager
sudo docker --version
docker compose version
sudo docker run --rm hello-world
sudo docker images
```
能夠正常輸出


