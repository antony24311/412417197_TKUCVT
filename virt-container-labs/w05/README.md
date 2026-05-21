# W05｜把容器拆開來看：Namespace / Cgroups / Union FS / OCI

## Docker 環境

- Storage Driver：overlayfs
- Cgroup Version：2
- Cgroup Driver：systemd
- Default Runtime：io.containerd.runc.v2 runc

## Namespace 觀察

### 六種 namespace 用途（用自己的話）
- PID：隔離 Process ID 空間，讓容器內的 process 以為自己是 PID 1，看不到 host 上的其他行程
- NET：隔離網路協定堆疊，使容器擁有獨立的網卡 (如 eth0, lo)、IP 位址與路由表
- MNT：隔離檔案系統掛載點，讓容器有專屬的根目錄 `/`，看不到 host 的檔案系統
- UTS：隔離 Hostname 與 Domain name，容器可以設定自己的主機名稱而不影響 host
- IPC：隔離行程間的通訊機制 (如共用記憶體)，確保容器間的資源不互撞
- USER：隔離使用者與群組 ID，允許容器內的 root 在 host 上實際上只是一般使用者，以提升安全性

### Host vs 容器 inode 對照

| Namespace | Host PID 1 inode | 容器 sleep inode | 一樣嗎？ |
|---|---|---|---|
| pid | pid:[4026531836] | pid:[4026532598] | 否 |
| net | net:[4026531833] | net:[4026532600] | 否 |
| mnt | mnt:[4026531832] | mnt:[4026532595] | 否 |
| uts | uts:[4026531838] | uts:[4026532596] | 否 |
| ipc | ipc:[4026531839]| ipc:[4026532597] | 否 |
| user| user:[4026531837] | user:[4026531837] | 是（預設未開 User Namespace 隔離）|
### 容器內 `ps aux` 輸出

只看到兩個 process (包含 `sleep` 為 PID 1)，因為 **PID Namespace** 將容器的視角隔離了，它無法看見 host 上多個 process。

## Cgroups 實驗

### 容器內讀到的限制
- memory.max：268435456
- cpu.max：50000 100000

### Host 端對照（用 `docker inspect -f '{{.HostConfig.CgroupParent}}'` 動態取得路徑）
- memory.max：268435456
- cpu.max：50000 100000
- memory.current（執行時某一刻）：393216

### OOM 故障三階段
| 項目 | 故障前 | 故障中（memory=32m + dd 200m）| 回復後（memory=256m）|
|---|---|---|---|
| 容器 exit code | - | 137| 0|
| OOMKilled | - | true| false|
| dmesg 關鍵字 | 無 OOM | Memory cgroup out of memory| 無 OOM |

## Image 分層

### `docker image inspect nginx:1.27-alpine` layer 數量
8 個

### 兩個同源 image 共享 layer 的證據
從 docker image inspect 的輸出觀察，nginx:1.27-alpine 與 nginx:1.26-alpine 的每一層sha256 雜湊值皆完全不相同
```
["sha256:994456c4fd7b2b87346a81961efb4ce945a39592d32e0762b38768bca7c7d085"

"sha256:aad7be8b43d91f43cdc23af3440b13eea7c2957feec9c46c977cb256e92481f6"

"sha256:49c50d3fe9320c2fc37d1aee38488bad246a680333a20746a5ef63f21d074c67"

"sha256:ed2f467e1cfcfea2cff2f48b21b86e763979ee599591f3632b44899f26ce583b"

"sha256:6f197061abd698a3eaf862a101d043b50b9162024cdf830e7cfb75131a9f3725"

"sha256:51b6aefac2f5df9fa2c24d782ef818b0b96238af2511eb60f79a58d1c839513a"

"sha256:6dba76576010ad0450285be4d174f5084b0bf597a68f31f8ad597fab0f032f3d"

"sha256:a0636672c7fc32af4d1022152a8e32256abd648fb01f48f33023839e65c6d1cb"]
```

### `docker diff` 輸出範例與解讀
 | 狀態代碼 | 對應檔案/路徑 | 操作解讀 | 
 |---|---|---|
 | C  | /tmp | Changed ：因為在 /tmp 目錄下新增了檔案 | 
 | A  | /tmp/hello.txt | Added：在容器執行期間，新增了 hello.txt 這個檔案 | 
 | C  | '/etc' '/etc/nginx' '/etc/nginx/conf.d'| Changed ：路徑中的目錄被標記為 Changed，因為其子目錄或檔案內容發生了變動（刪除或新增） | 
 | A  | /etc/nginx/conf.d/custom.conf | f	Added ：新增了自定義的 nginx 設定檔 | 
 | D  | /etc/nginx/conf.d/default.conf | Deleted：容器內原有的預設設定檔被移除了 | 


## OCI 呼叫鏈

（用自己的話說明 dockerd → containerd → containerd-shim → runc 各自負責什麼，以及 OCI Runtime Spec `config.json` 裡哪些欄位對應到 namespace / cgroup 設定）
- **dockerd**：負責高層邏輯處理與接收使用者的 API 請求 (`docker run`)。
- **containerd**：負責 image 的管理與容器的生命週期，作為 OCI Runtime 的上層管理者。
- **containerd-shim**：每個容器一支，負責接住 runc spawn 出來的 process、持有 stdio，讓 containerd 即使重啟也不會影響運行中的容器。
- **runc**：符合 OCI Runtime Spec 的參考實作，真正負責呼叫 `clone()` 建立 namespaces 與設定 cgroups 的底層工具。
- OCI `config.json` 欄位對應：`linux.namespaces` 負責對應到要開啟的 namespace 陣列；`linux.resources` 對應到 cgroup 的限制 (如 cpu, memory, pids)。

## 排錯紀錄
- 症狀：故障注入 `docker run --memory=32m ... dd count=200` 卻沒有 OOM，dd 竟然跑完了
- 診斷：若把 dd 輸出目標寫成 `/tmp/fill`，資料是寫入磁碟的可寫層，不會算在 memory cgroup 的範圍內
- 修正：將寫入目標改為 `/dev/shm/big`，因為 tmpfs 是掛載在記憶體上，會算進 cgroup
- 驗證：重新執行命令，容器順利被 SIGKILL，拿到正確的 `ExitCode 137`

## 想一想（回答 3 題）
1. 容器裡的 PID 1 跟 host PID 1 是同一支 process 嗎？`kill -9 1`（在容器內）會發生什麼？
不是。Host 的 PID 1 通常是 `systemd`，而容器內的 PID 1 是容器啟動的第一個應用程式 (例如 nginx 或 sleep)。在容器內若執行 `kill -9 1`，該應用程式會被強制終止，因為 PID 1 一死，容器就會跟著關閉退出。

2. 兩個容器都基於 `ubuntu:24.04`，磁碟空間是吃兩份還是共用？怎麼驗證？
是共用的。透過 Union FS 機制共享了唯讀層 (lowerdir)。驗證方式為使用 `docker image inspect` 觀察 layer hash，或是觀察 `/var/lib/docker/overlay2/` 的總體積，在啟動第二個相同基礎的容器時，實體容量的增長幾乎可以忽略，只有各自獨立的 upperdir 在佔用新空間。

3. 如果 host 的 kernel 爆漏洞，容器還能稱為「隔離」嗎？這個限制跟 VM 差在哪？
隔離會被打破，因為容器本質上就是共用 host 的同一顆 kernel。VM 的限制較小，因為每個 VM 有獨立的 guest OS kernel，就算 VM 的 kernel 爆洞，也很難直接打穿 Hypervisor 影響到實體機或其他 VM。這也是 Kata Containers 或 Firecracker 這種微型 VM (強隔離) 存在的原因
