---
title: "一台塞得進行李的 Homelab"
description: "整座 Homelab 跑在一台迷你電腦上。這篇寫它的完整規格，以及兩個真正決定選型的條件：要帶得走，還要過得了 IncusOS 的硬性門檻。"
pubDate: "Sep 10 2026"
heroImage: "/homelab-hardware.png"
---

這是 Homelab 系列的第一篇，共三篇：

1. **硬體選型**（本篇）
2. [為什麼最後停在 Incus](/blog/homelab-why-incus/)
3. [跑在上面的那些服務](/blog/homelab-services/)

---

我的 Homelab 只有一台機器。

不是「主力一台、備援一台」，也不是「三台組叢集」。就是一台放在桌子底下的迷你電腦，上面跑著 DNS、憑證中心、身分驗證、Git、CI、幾套自用的應用服務，還有幾套工作用的實驗環境。寫這篇的時候它的記憶體用掉了 22 GB，硬碟用掉 360 GB，風扇聽不太到。

會變成這樣，是因為選型的時候我只在意兩件事。

## 兩個真正的條件

**第一，它得帶得走。**

我的生活在移動，機器不能是那種「裝好就永遠釘在那個房間」的東西。這一條直接把塔式機殼、二手伺服器、NAS 準系統全部刪掉——不管它們多划算。剩下的選項只有迷你電腦，而且要是能收進行李、插上電就能開的那種。

**第二，它得過得了 IncusOS 的門檻。**

這一條看起來像是倒果為因——硬體不是應該先選，再決定裝什麼系統嗎？但 IncusOS 的完整安全模式有幾項硬性要求，一旦決定要用它，機器的可選範圍就被切掉一大半：

| 需求 | 為什麼 |
| --- | --- |
| CPU 支援 x86-64-v3 | IncusOS 的底線，Haswell 以後才有 |
| TPM 2.0 | 磁碟加密的金鑰要有地方放 |
| UEFI Secure Boot | 完整模式的前提，關掉只能跑 degraded |

很多便宜的迷你電腦卡在第二、三項——不是沒有 TPM，就是 BIOS 陽春到沒得開。所以「能跑 IncusOS」實際上比「CPU 快不快」更早被拿來篩。

至於功耗、噪音、內顯，老實說當時都不是主要考量。它們後來確實都派上用場了，但那是紅利，不是理由。

## 完整規格

現在這台機器是 GMKtec NucBox M3 Ultra。以下是從 `incus info --resources` 抓下來的實機資料，不是型錄：

| 項目 | 規格 |
| --- | --- |
| 機器 | GMKtec NucBox M3 Ultra（Mini PC 機殼） |
| CPU | Intel Core i7-12700H，14 核 20 執行緒（6 P-core + 8 E-core），24 MB L3 |
| 記憶體 | 32 GB（系統回報 31.88 GiB） |
| 系統碟 | TWSC TSC3AN1T0-F6Q10S，1 TB NVMe |
| 外接碟 | Toshiba MQ04UBD200，2 TB，USB |
| 內顯 | Intel Iris Xe（Alder Lake-P GT2） |
| 有線網路 | Intel I226-V，2.5 GbE |
| 無線網路 | Realtek RTL8852BE，Wi-Fi 6 |
| 不斷電系統 | APC Back-UPS RS 1200SI，USB 連接 |

幾個值得拿出來講的地方。

### CPU：大小核在容器上意外地好用

i7-12700H 是 Alder Lake 的行動版，6 個 P-core 加 8 個 E-core。桌機作業系統上大小核排程曾經是災難，但在這裡它的形狀剛好對：Homelab 大部分容器是「醒著但沒事做」的——DNS、CA、reverse proxy 一天到晚閒著，偶爾爆一下。E-core 接走這些常駐的雜事，P-core 留給真的需要算的（SonarQube 掃描、CI 跑測試、影音硬體轉檔）。

### 記憶體：最先見底的是它

32 GB 是目前最吃緊的資源。寫這篇的當下：

```
Memory:
  Used:  22.18GiB
  Free:   9.69GiB
  Total: 31.88GiB
```

不是因為服務多，是因為有幾個特別胖的。SonarQube 自己就要 2.5 GB（它內嵌一個 Elasticsearch），Authentik 那一套（PostgreSQL + Redis + server + worker）也要 2.6 GB，另一台放應用服務的容器 2.7 GB。

再加上這台機器不只跑容器——上面還躺著幾組工作用的實驗環境，是完整的虛擬機，平常關著，要用才開。那是另一個把 32 GB 吃光的來源。

如果重來一次，我會直接上 64 GB。CPU 到現在都還有餘裕，記憶體沒有。

### 儲存：一顆內建、一顆外掛，分工很清楚

機器上有兩個 ZFS pool：

| Pool | 位置 | 容量 | 已用 | 放什麼 |
| --- | --- | --- | --- | --- |
| `local` | 1 TB NVMe | 891 GB | 141 GB | 所有容器、虛擬機、映像、備份 |
| `hdd` | 2 TB USB 外接 | 1.76 TiB | 220 GB | 只有一個 volume：大檔案資料 |

這個分法是刻意的。系統碟要快，而且它的內容全部是「可以重建的」——容器都是 Terraform 定義的，炸了就重跑一次。外接碟要大，放的是少數重建不回來的那些大檔案。

用 USB 外接碟當大容量儲存當然不漂亮，吞吐量也比不上內接。但它符合第一個條件：拔掉就能帶走，換一台機器插上去還在。

順帶一提，IncusOS 把整顆碟都接管成加密 ZFS，這也是 TPM 那一項不能妥協的原因——沒有 TPM 就沒有地方安全地放金鑰。

### 網路：2.5 GbE 目前只跑在 1 Gb

網卡是 Intel I226-V，支援到 2.5 GbE：

```
Supported modes: ..., 1000baseT/Full, 2500baseT/Full
Link speed: 1000Mbit/s (full duplex)
```

實際協商出來是 1 Gb——因為上游的交換器只有 gigabit。這是整台機器裡唯一「硬體比環境好」的地方，換條線換台交換器就能解，但目前完全沒有感覺到瓶頸，所以一直沒動。

Wi-Fi 也是一樣，RTL8852BE 支援 Wi-Fi 6，但這台機器從裝好到現在沒用過無線——伺服器就該插線。

### 內顯：後來才變重要

Iris Xe 在選型的時候不在考慮範圍內，但它後來變成某些工作負載能不能用的關鍵。影音轉檔如果純靠 CPU，一路軟解會把 P-core 吃滿；把 iGPU 傳進容器走 VA-API 之後，同樣的轉檔幾乎不佔 CPU。

Incus 這邊做起來出乎意料簡單——因為容器和 host 共用 kernel，host 已經有 i915 了，只要把 device 丟進容器、容器內裝 userspace 驅動就好：

```bash
incus config device add c1 igpu gpu gputype=physical
incus exec c1 -- ls -l /dev/dri/          # card0 / renderD128
incus exec c1 -- apt install -y intel-media-va-driver-non-free vainfo
```

唯一的坑是非特權容器裡 `render` 群組的 GID 對不上，要用 `incus config device set c1 igpu gid=<容器內的 GID>` 修一下。

這件事也回頭解釋了為什麼是 Intel 而不是 AMD：不是效能，是 QSV／VA-API 這條路上文件跟踩雷紀錄都多得多。

### UPS：唯一不能軟體化的那個

APC Back-UPS RS 1200SI 用 USB 接在機器上。它是整套系統裡唯一一個「沒辦法用 Terraform 描述」的元件，但少了它，前面所有關於不可變、可重建的設計都撐不住一次跳電——ZFS 撐得住突然斷電，跑到一半的資料庫寫入撐不住。

## 裝機前一定要進 BIOS 做的事

這段是踩過的雷，寫下來給未來的自己。IncusOS 開機前，BIOS 要先動五個地方：

1. 打開 **Intel PTT / TPM 2.0**——這台預設是關的，藏在 Security 或 Advanced 底下。
2. 打開 **Secure Boot**，而且建議切到 **Setup Mode**（清空金鑰），讓 IncusOS 首次開機自己 enroll。
3. 關掉 **CSM / Legacy Boot**，走純 UEFI。
4. 如果 SATA 在 **RAID 模式**，改成 **AHCI**——RAID 模式下 IncusOS 開不起來。
5. 做一次 **TPM Clear**。殘留的 TPM 狀態是安裝失敗的常見原因。

漏掉其中任何一項，開機大概會停在 `currently unsupported operating mode`，而那個訊息不會告訴你是哪一項。

還有一件更重要的：**整顆碟會被加密清空**。先備份。

## 回頭看

這份規格表最有趣的地方，是它幾乎每一項都不是「效能考量」的結果。體積是因為要帶著走，CPU 世代是因為 x86-64-v3，TPM 和 Secure Boot 是因為不可變的 host OS，儲存分兩層是因為想清楚了哪些東西重建得回來。

換句話說，硬體其實是被作業系統決定的。而我為什麼選了一個沒有 shell、不能 SSH、連 `apt install` 都做不到的作業系統——那是[下一篇](/blog/homelab-why-incus/)的事。
