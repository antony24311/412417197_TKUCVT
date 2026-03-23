## 一、 虛擬機器 (VM) vs. 容器 (Container) 多維度對照

|  | 虛擬機器 (VM) | 容器 (Container) |
| -------- | -------- | -------- |
| 隔離     | 硬體級隔離：每個 VM 擁有獨立的 Guest OS 核心，隔離邊界由 Hypervisor 定義     | 作業系統級隔離：共用 Host OS 的 Kernel，僅透過 Namespace/Cgroups 隔離程序     |
| 啟動     | 較慢：需經過虛擬 BIOS、引導載入程式     | 僅需啟動應用程式程序，不需啟動核心     |
| 資源消耗     | 每個實例需預留固定的 CPU/RAM 與 GB 等級的磁碟空間存放 OS     | 僅佔用執行所需的記憶體與 MB 等級的映像檔空間     |
| 環境一致性     | 封裝了從核心到應用的所有組件，適合跨異質平台     | 封裝應用與函式庫，但極度依賴 Host Kernel 的版本與特性。     |


## 二、 本課選擇「VM 內跑 Docker」的理由（設計決策）
1. 建立「標準化」
透過 VMware 建立 Ubuntu VM，把所有人的「底層」標準化了，後續的 Docker 指令才具有 100% 的重現性
2. 復原
獲得了利用 Snapshot 進行災難復原的能力。當 Docker Daemon 設定改爛（如軟體源失效）或執行高風險攻擊測試時，只需短時間內即可還原至原始狀態

## 三、 Hypervisor Type 1 vs. Type 2 的差異與選擇
1. Type 1
直接安裝在實體硬體上，本身就是作業系統，因此效能極高、延遲極低。範例：VMware ESXi、Proxmox。適用於企業資料中心
2. Type 2
Type 2 安裝在現有的 Host OS (Windows/macOS) 之上，這讓我們能同時開啟教學講義、瀏覽器查閱資料，並在同一個螢幕操作虛擬機。範例：VMware Workstation、VirtualBox
