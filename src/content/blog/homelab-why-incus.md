---
title: "從 k3s 到 Talos 到 Proxmox，最後停在 Incus"
description: "三代 Homelab 的取捨：為什麼離開 Kubernetes，為什麼離開 Proxmox，以及為什麼是 IncusOS 而不是「隨便一個 Linux 發行版再裝 Incus」。"
pubDate: "Sep 11 2026"
heroImage: "/homelab-why-incus.png"
---

這是 Homelab 系列的第二篇，共三篇：

1. [硬體選型](/blog/homelab-hardware/)
2. **為什麼最後停在 Incus**（本篇）
3. [跑在上面的那些服務](/blog/homelab-services/)

---

我的 Homelab 換過三代平台：先是 Kubernetes（k3s，後來換成 Talos），接著是 Proxmox，現在是 Incus + IncusOS。

三代裡面，這一代是體驗最好的一次。好到我願意花時間寫三篇文章說明它——而前兩代我連筆記都懶得補完。

這篇想講清楚的是：前兩代到底哪裡不對，以及 Incus 做對了什麼。

## 第一代：Kubernetes（k3s → Talos）

會從 Kubernetes 開始，理由很現實：那是工作上用的東西。在家裡跑一套，等於用自己的時間買工作上的熟練度。

k3s 裝起來很快，Talos 更是漂亮——不可變、API-only、整台機器沒有 shell，宣告式到底。我到現在都還覺得 Talos 的設計很優雅。

但在單機 Homelab 上，它有四個問題，而且每一個都是結構性的、不是調一調參數就能解的。

### 有狀態服務真的很痛

Homelab 裡幾乎每一個服務都是有狀態的。Git repository、資料庫、憑證、DNS zone、使用者目錄——沒有一個是「壞了重開一個就好」的無狀態 pod。

而 Kubernetes 的整套設計是繞著無狀態轉的。要放資料，你得先理解 PV、PVC、StorageClass、CSI driver；要備份，你得再引進一套工具；要在單節點上做這些，你得裝一個本地 CSI，然後發現它解決的問題其實是「假裝這台機器是叢集」。

我花在「讓資料能好好躺著」的時間，遠多於花在那些服務本身上。

### 什麼都得先容器化

這一條是最致命的。

Kubernetes 的世界裡只有 pod。你想跑一台「像機器的東西」——可以 SSH 進去、可以裝套件、可以跑 systemd、可以當成一台 Linux 用——沒有這個選項。所有東西都得先被塞進一個無狀態容器的形狀裡，不管它本來適不適合。

但 Homelab 裡就是有一半的東西不適合。實驗環境、要模擬客戶機房的測試機、想裝來玩三天就刪掉的軟體——這些東西的自然形狀就是「一台機器」，硬要 Helm chart 化只是自找麻煩。

### 維運成本吃掉了樂趣

版本升級要看 deprecation、CNI 要選、Ingress controller 要挑、憑證要 cert-manager、密鑰要 sealed-secrets 或 external-secrets、監控要一整套 stack。每一個決定都合理，加起來就是：我在維運一座叢集，而不是在用我的 Homelab。

我開始發現自己週末在處理的問題，跟這台機器上實際跑的服務完全無關。

### 單節點根本拿不到 Kubernetes 的好處

Kubernetes 的價值在排程、自癒、水平擴展、滾動更新。這四件事在一台機器上：

- 排程——只有一個節點，沒得排。
- 自癒——節點掛了就是全掛，沒有地方可以遷。
- 水平擴展——CPU 就這麼多。
- 滾動更新——新舊版本要同時跑，記憶體不夠。

付了全額的複雜度，拿到的好處接近零。這就是我離開的原因：不是 Kubernetes 不好，是它解的問題我沒有。

## 第二代：Proxmox

從 Kubernetes 逃出來之後，我往反方向跑：Proxmox。

這個方向本身是對的。Proxmox 有 LXC 和 KVM，既能跑輕量容器也能跑完整虛擬機，「一台機器」這個形狀終於回來了。有一段時間我用得挺開心。

但兩件事最後把我趕走。

### IaC 支援不好，UI 先行

Proxmox 是一個 Web UI 產品。它有 API，也有社群的 Terraform provider，但整個產品的設計重心在那個介面上——文件用 UI 講、教學用 UI 講、遇到問題查到的答案也是「點哪裡哪裡」。

結果就是：我的環境描述不完整。一部分在 Terraform 裡，一部分在我腦袋裡，一部分在某次半夜點了什麼卻沒記下來的 UI 操作裡。要驗證「這套設定是不是真的能重建」，唯一的方法是砍掉重來，而我從來不敢。

### host OS 會飄移

Proxmox 底層是一個你可以完全掌控的 Debian。聽起來是優點，實際上是慢性病。

裝一個工具、改一次 `/etc/` 底下的設定、apt 升級留下的一堆設定檔問句、為了解決某個問題加上去的 systemd unit——這些東西一件一件累積，半年後這台機器上有什麼、為什麼有，沒有人知道，包括我自己。

這就是所謂的 configuration drift。它不會讓你的系統當掉，它只會讓你漸漸不敢動它。而「不敢動」對 Homelab 是死刑——Homelab 存在的意義就是可以隨便動。

## 第三代：Incus

Incus 是 LXD 的社群分支，做的事情很單純：用一套 API 管理**系統容器**和**虛擬機**。

「系統容器」是關鍵字。它不是 Docker 那種一個 process 的應用容器，而是一整個 Linux 使用者空間——有 init、有 systemd、有 package manager，可以 SSH 進去，行為上就是一台機器，只是共用 host 的 kernel，所以又輕又快。

這一句話同時解掉了前兩代的問題：

| 前兩代的問題 | Incus 的答案 |
| --- | --- |
| 什麼都得先容器化（k8s） | 系統容器就是一台機器，不用重塑形狀 |
| 有狀態服務很痛（k8s） | 容器有自己的檔案系統，資料就放在裡面 |
| 單節點拿不到好處（k8s） | 它本來就是設計給單機也合理的工具 |
| IaC 支援不好（Proxmox） | 官方 Terraform provider，API 是第一公民 |

我這台機器現在同時跑著九個系統容器（DNS、CA、reverse proxy host、CI runner⋯⋯），以及一批平常關著的完整虛擬機——那是工作上要模擬客戶機房的實驗環境。**同一個 CLI、同一個 API、同一份 Terraform 描述**，容器和虛擬機都管。這正是 Kubernetes 給不了、而 Proxmox 給得很彆扭的東西。

再加上 OVN。Incus 內建 OVN 支援，讓我可以在一台機器上切出真正隔離的軟體定義網路——我切了四段（`infra` / `dev` / `app` / `home`），以 `infra` 當 hub 跟其餘三段雙向 peering。這在 Kubernetes 裡要靠 NetworkPolicy 加一堆心智負擔，在 Proxmox 裡要自己接 bridge 和防火牆規則。

## 為什麼是 IncusOS，而不是 Debian 再裝 Incus

Incus 裝在任何一個發行版上都能跑。所以這個決定要單獨解釋。

理由只有一句：**它是「host OS 會飄移」那個問題的正面解法。**

IncusOS 是一個不可變的作業系統。它：

- **沒有 shell，不能 SSH**——沒有互動式登入這回事。
- **不能 `apt install`**——根本沒有 package manager 給你用。
- **不能改設定檔**——檔案系統是唯讀的。
- 所有事情只能透過**已認證的 Incus API** 做。
- 開機要求 **UEFI Secure Boot + TPM 2.0**，整顆碟是**加密 ZFS**。

第一次讀到這些限制，直覺是「這也太綁手綁腳」。但它們正是我要的東西——**沒有 shell，就沒有人能偷偷改東西；不能裝套件，就不會有人記不得裝過什麼**。這台機器的狀態不可能飄移，因為它沒有地方可以飄。

所有真正的設定都只剩一個來源：那份 Terraform。這也是為什麼這一代是體驗最好的一次——不是因為 Incus 的功能比 Proxmox 多，是因為我終於能**相信**我的 IaC 描述了整個系統。要驗證？把容器全砍掉重跑一次 `tofu apply` 就好，而我現在真的敢。

## 誠實的代價

這條路不是沒有摩擦。實際踩過的：

**裝機時的「先有雞還是先有蛋」。** IncusOS 沒有 shell，所以你沒辦法在機器上生一個 trust token 給客戶端。正確做法是在**製作安裝映像時就把 client 憑證塞進 seed**，開機後 server 就直接認得你。

這裡有個雷：用官方 Web Image Builder 時，**不要選 `Incus (LTS 7.0)` 當 Image application**。那個組合（2026 年 6 月）不會把憑證寫進 seed，裝完照樣跟你要 token，嚴重一點會直接被鎖在門外。選普通的 `Incus` 才對。

**OVN 要自己接上去。** 不是打開開關就有。得先跑一個 `ovn-central` 容器裝好 NB/SB 資料庫，然後透過 IncusOS 的管理 API 打開 host 的 OVN chassis（`PUT /os/1.0/services/ovn`），最後才能設 `network.ovn.northbound_connection`。順序錯了，建網路時會得到一個看不懂的 `ovnnb_db.sock ... no such file`。

**有些 sysctl 真的改不了。** SonarQube 內嵌的 Elasticsearch 想要很高的 `vm.max_map_count`，那是 host 層級的參數，在容器裡即使特權模式也改不動。最後是靠它 dev 模式下綁 loopback、檢查不致命才繞過去。這類「容器共用 kernel」的限制，是系統容器換來的輕量的另一面。

**`systemd-resolved` 會跟你打架。** 它預設接管 `/etc/resolv.conf`，在同時有 managed bridge、OVN 網路和內部 DNS 的環境下，查詢常常走錯上游。這件事逼我把整個名稱解析架構重新想過一遍。

這些坑都不小，但它們有一個共同點：**每一個都是一次性的**。解掉就寫進 Terraform 和 cloud-init，下一次重建不會再遇到。這跟 Proxmox 時代那種「每半年慢慢長出來的、沒有人記得的技術債」，性質完全不同。

## 結論

如果要把三代濃縮成一句：

> Kubernetes 讓我維運叢集，Proxmox 讓我維運一台會慢慢走樣的機器，Incus + IncusOS 讓我維運一份 Terraform。

第三種才是我一開始想要的。

[下一篇](/blog/homelab-services/)來寫跑在上面的服務——特別是撐起整座 lab 的那三個基礎設施：自己的憑證中心、自己的 DNS、以及一套把所有服務的登入收攏在一起的身分驗證。
