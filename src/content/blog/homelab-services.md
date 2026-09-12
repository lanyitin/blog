---
title: "Homelab 上的那些服務：從一張自簽憑證開始"
description: "四段 OVN 網路、Incus 的 network forward、每段一台 Traefik、一個自己的憑證中心、一台權威 DNS、一套收攏所有登入的身分驗證——以及它們怎麼互相撐住彼此。"
pubDate: "Sep 12 2026"
heroImage: "/homelab-services.png"
---

這是 Homelab 系列的第三篇，共三篇：

1. [硬體選型](/blog/homelab-hardware/)
2. [為什麼最後停在 Incus](/blog/homelab-why-incus/)
3. **跑在上面的那些服務**（本篇）

---

前兩篇講的是一台機器和一個平台。這篇講上面實際跑著什麼。

但我不打算從「有哪些好玩的 app」開始寫。因為這座 lab 真正有意思的部分不是那些有畫面的東西——裝過的人都知道它們長什麼樣。真正花時間、也真正改變體驗的，是底下那些沒有畫面的：**封包怎麼進得來、憑證從哪裡來、名字誰負責解、以及「你是誰」由誰回答**。

它們單獨看都很無聊，湊在一起之後，整座 lab 的手感完全不一樣。

## 先講網路：四段，一個 hub

所有服務跑在四個 OVN 網段上：

| 網段 | 子網 | 放什麼 |
| --- | --- | --- |
| `infra` | `10.0.0.0/24` | 基礎設施：DNS、CA、身分驗證、對外通道 |
| `dev` | `10.1.0.0/24` | 開發：Git、CI、程式碼掃描 |
| `app` | `10.2.0.0/24` | 應用：自用的各種服務 |
| `home` | `10.3.0.0/24` | 家用 |

`infra` 是 hub，跟其餘三段**雙向 peering**；三個 spoke 彼此**不通**。這個形狀是刻意的——每一段都需要 DNS 和憑證（所以要能連到 `infra`），但 `dev` 沒有理由碰得到 `app` 的東西。

這樣切完之後馬上會遇到一個問題：這些網段對家裡的 LAN 來說是不存在的。我筆電上的瀏覽器不知道 `10.0.0.2` 怎麼走。那要怎麼用？

## Incus Network Forward：我最喜歡的那個功能

這就是 Incus 的 **network forward** 在做的事，而它是整套系統裡我用得最順手的一個功能——順手到我覺得它比 Kubernetes 的 Ingress 好用太多。

概念只有一句：**在 uplink 上挑一個 IP，把指定的協定和埠，轉給 OVN 網段裡的某個位址。**

實際長這樣（這是我那台 DNS 的 forward，直接從機器上撈下來的）：

```yaml
description: technitium
listen_address: 192.168.2.155
ports:
  - protocol: udp
    listen_port: "53"
    target_port: "53"
    target_address: 10.0.0.3
  - protocol: tcp
    listen_port: 5380,53
    target_port: 5380,53
    target_address: 10.0.0.3
```

就這樣。LAN 上任何一台機器把 DNS 指到 `192.168.2.155`，查詢就會送到 OVN `infra` 網段裡那台 Technitium。在 Terraform 裡它也只是一個資源，三個欄位。

### 為什麼我說它比 Ingress 強

我用過 Kubernetes 的 Ingress，也用過 Proxmox 手接 iptables。這是 network forward 贏的地方：

**一、它是 L4，不是 HTTP-only。**

這是最大的差別。Ingress 的世界只有 HTTP 和 HTTPS——那是它的規格定義的。上面那個 DNS 的例子裡有 **UDP/53**，在 Kubernetes 裡你根本沒辦法用 Ingress 表達它。你的選項是：開一個 NodePort（拿到一個三萬多的怪埠，然後在前面再補一層），或是弄一個 LoadBalancer（家裡沒有雲端負載平衡器，於是要裝 MetalLB），或是引進 Gateway API 加 `TCPRoute`（又是一整套要學要裝的東西）。

Incus 這邊，UDP 就只是 `protocol: udp`。

**二、每個服務有自己的 IP，埠號維持原樣。**

Ingress 的核心假設是「大家共用 :443，靠 Host header 和路徑分流」。這對網站很合理，對其他東西就很扭曲——你得做路徑改寫，得處理應用程式不知道自己被掛在某個子路徑底下的問題。

network forward 讓我在 uplink 上一個服務給一個 IP：

| Listen | 服務 | 埠 |
| --- | --- | --- |
| `192.168.2.150` | infra 的 reverse proxy | 80, 443, 3552 |
| `192.168.2.151` | dev 的 reverse proxy | 80, 443, 3552 |
| `192.168.2.154` | step-ca | 443 |
| `192.168.2.155` | Technitium | 53 (tcp+udp), 5380 |

CA 就聽 443，DNS 就聽 53。沒有路徑改寫，沒有埠號魔術，服務看到的自己跟它原本的樣子一樣。

**三、沒有一個要常駐的元件。**

Ingress 是一個「規則物件」，它需要一個 **Ingress controller** 實際跑著才有意義——你得選一個（nginx？traefik？haproxy？）、裝它、給它資源、升級它、記住它那一套 annotation 的方言。

network forward 不是。它是**網路本身的屬性**，由底下的 OVN 直接實現。沒有多出來的容器，沒有要維護的 controller，沒有 annotation。

**四、它同時是一份存取控制的宣告。**

OVN 網段從 LAN 沒有路由進來，所以 forward 是**唯一一道門**。這表示「這座 lab 對家裡的網路暴露了什麼」這個問題，答案就是 forward 清單本身——`incus network forward list` 跑一次就看完了，不用去翻防火牆規則跟每個服務的設定。

把「對外開了什麼」和「怎麼開的」收斂成同一份東西，對單人維運的 Homelab 來說價值很大。

### 它跟 Traefik 的分工

有了 forward 之後，還需要 reverse proxy 嗎？需要，但兩者分工很乾淨：

```
LAN 192.168.1.x
   │
   │  network forward（L4：協定 + 埠 → OVN 位址）
   ▼
OVN 10.x.0.0/24
   │
   │  Traefik（L7：Host() 規則 → 容器）
   ▼
實際的服務
```

**forward 負責「進得來」，Traefik 負責「進來之後去哪」。** 所以 podman host 的 forward 永遠只開 80、443 和一個管理埠——後面接了幾十個服務都不必再動它。反過來說，像 DNS 和 CA 這種不是 HTTP 的東西，就直接用 forward 打到底，完全不經過 Traefik。

這個分層是我覺得最舒服的地方：**該用 L4 解的就用 L4 解，不必為了統一而把 DNS 硬塞進一個為網站設計的抽象裡。**

## step-ca：一張自己簽的憑證，換掉整片瀏覽器警告

這是投資報酬率最高的一個服務。

在這之前，內部服務要嘛跑純 HTTP，要嘛用自簽憑證然後每次都點「繼續前往（不安全）」。兩種都很糟——前者讓你養成壞習慣，後者讓安全警告變成雜訊，真的出事那天你也會照樣按過去。

[step-ca](https://smallstep.com/docs/step-ca/) 解掉的就是這件事：一個自己跑的憑證中心，而且**內建 ACME**——就是 Let's Encrypt 那套協定。

### 它怎麼被建起來的

整個建立過程有一個我很喜歡的細節：

1. Root CA 的憑證和私鑰是**從外面提供的**（用 age 加密後注入），不是 step-ca 自己生的。
2. 開機時用這把 root key 簽出一張 **intermediate**。
3. **簽完就把 root key 銷毀**在那台機器上。

於是 root 私鑰從來不會長期躺在任何一台線上的機器裡。日常簽發全部由 intermediate 負責，就算 CA 那台被攻破，signing chain 的根還在離線備份裡。

root CA 憑證本身則走另一條路：**每一個容器在 cloud-init 階段就把它裝進系統信任庫**。所以整座 lab 裡任何一台機器、任何一個 `curl`，都天生信任 step-ca 簽出來的憑證。不用一台一台去裝。

### 兩種簽發路徑

step-ca 上開了兩種 provisioner，對應兩種需求：

| Provisioner | 誰在用 | 怎麼認證 |
| --- | --- | --- |
| ACME | 各網段的 Traefik，走 DNS-01 | ACME challenge |
| JWK | 自己終結 TLS 的服務 | provisioner 密碼 |

**ACME 那條是主力。** Traefik 用 ACME DNS-01 自動要憑證、自動續期。加一個新服務時我什麼都不用做——掛上 label，憑證就自己出現了。跟在公網上用 Let's Encrypt 的體驗一模一樣，只是 CA 是自己的。

**JWK 那條處理例外。** 有些服務不在 Traefik 後面、要自己終結 TLS，最典型的是 SMTP relay——Postfix 在 587 埠做 STARTTLS，沒有 HTTP 前端可以讓 Traefik 掛 router。這種就在同一個 compose project 裡放一個 `step-cli` 的 sidecar：

```
首次簽發（JWK provisioner，需要密碼）
   step ca certificate  →  smtp.crt / smtp.key
                            │
持續續期（用憑證自己認證，不需要密碼）
   step ca renew --daemon ─┘   到壽命約 2/3 時自動換新
```

關鍵區別是**簽發要密碼、續期不要**——`step ca renew` 是拿「手上那張還沒過期的憑證」對 CA 做 mTLS 認證來換新的。所以常駐的 daemon 身邊不必留著 provisioner 密碼。

sidecar 簽好的憑證放在共用 volume，主服務唯讀掛入。Postfix 的 worker 是短命 process，換檔後新連線自然讀到新憑證，連 reload 都不用。

### 最後連 Incus 自己也用它

有一個收尾讓我特別滿意：**Incus daemon 自己的 API 憑證，現在也是跟 step-ca 用 ACME 要的**。

```
acme.ca_url:    https://<step-ca>/acme/acme/directory
acme.domain:    incus.infra.lan
acme.challenge: DNS-01
acme.provider:  technitium
```

管理這台機器用的那個 API 端點，憑證由跑在這台機器上的 CA 簽、DNS-01 挑戰由跑在這台機器上的 DNS 回答。整條信任鏈閉合在 lab 內部，不依賴任何外部服務。

## Technitium DNS：`*.lan` 的權威，也是 ACME 的關鍵零件

[Technitium](https://technitium.com/dns/) 是那台權威 DNS，做兩件事。

**第一件是本份工作**：解析四個 zone（`infra.lan`、`dev.lan`、`app.lan`、`home.lan`）的記錄，把服務名字指到對應的 Traefik 入口。看網址就知道東西住在哪一段。

**第二件比較有趣**：它是 ACME DNS-01 挑戰的落地點。DNS-01 的運作方式是「證明你能在某個網域底下放一筆 TXT 記錄」，所以 Traefik 必須有辦法程式化地寫 DNS——而 lego（Traefik 用的 ACME 函式庫）內建 Technitium provider，用 API token 就能寫。

前面那句「加一個服務、憑證自己出現」，靠的就是這個。**CA 和 DNS 是一組的，少一個另一個就不成立。**

### 兩個踩過的坑

**先有雞還是先有蛋。** Technitium 的 Terraform provider 在 **plan 階段**就會連線，需要一台活著的 server 加一個 API token——而全新環境兩個都沒有。所以 bootstrap 要分兩階段：先只建容器（`tofu apply -target=...`），開 Web console 手動生一個 token，再跑完整的 apply 去建 TSIG key 和 zone。

這是我對「Terraform 管 Terraform 自己要用的東西」這類循環依賴最直觀的一次體會。

**host 自己解不到自己的 DNS。** Incus daemon 要簽憑證時，lego 是跑在 **host 上**的。那台 host 可以用 TCP 連到 Technitium 的 API，但 UDP/53 打到 OVN 位址會逾時——而 lego 查 DNS 走的正是 UDP。

最後是讓 host 上的 lego 改用 LAN 路由器當 resolver，由它轉發 `*.lan` 過去。繞了一圈，但它誠實反映了一件事：**host 在 OVN 網路裡是個外人**，不能假設它跟容器有一樣的視野。

## Traefik：一個網段一台

四個網段，四台 Traefik，各自跑在該網段的容器主機上。

### 為什麼不是共用一台

一開始我也想過只架一台統一入口。但兩個理由讓我改成一段一台：

**第一，自動探索是本地的。** Traefik 最好用的地方是它會掛著容器 runtime 的 socket，自動看到同一台主機上的容器——新容器一起來、label 一掛上，route 就生效了，不用寫設定。這個能力的前提是「跟被代理的容器在同一台主機上」。

**第二，共用一台會把隔離吃掉。** 網段之間刻意不互通，如果有一台 Traefik 需要同時連到四段，那它就成了一個繞過所有隔離的萬用通道。與其開這個洞，不如每段自己有一台。

代價是四份幾乎一樣的設定——但那正是 Terraform module 該解決的問題，呼叫四次就好。

### 它負責的三件事

**一、Host 分流。** 每個網段的 forward 只開 80/443，進來之後靠 `Host()` 規則決定去哪個容器。加服務不用動 forward。

**二、憑證。** 走 ACME DNS-01 對 step-ca 要憑證，自動續期。這是前面兩節能兜起來的那個扣環。

**三、共用的 `proxy` 網路。** Traefik 自己擁有一個叫 `proxy` 的容器網路，要被代理的 app 就加入它。加入了才看得到、才代理得到——**網路成員資格本身就是「要不要對外」的開關**，不用另外寫規則。

### 探索看不到的後端怎麼辦

有些後端不在同一台容器主機上。例如 SonarQube 因為比較重，住在自己專屬的 Incus 容器裡，透過 OVN 連過去——Traefik 的 socket 探索完全看不到它。

這種就改用 Traefik 的 **file provider**：Terraform 把一段靜態 route 設定渲染進一個被監看的設定 volume（由一個 init 容器寫入），Traefik 讀到就多一條 route。等於「自動探索為主、手動宣告補洞」，而那個手動的部分照樣在 Terraform 裡。

## Podman + Arcane：這些容器實際上怎麼跑起來

講完網路和憑證，還有一個沒回答的問題：那些應用容器到底是誰在跑？

答案是每個網段一台 **rootless Podman 主機**（`podman-infra`、`podman-dev`、`podman-app`、`podman-home`），上面用 [Arcane](https://github.com/getarcaneapp/arcane) 管理 compose project。

### 為什麼是 rootless Podman

選 Podman 不是因為討厭 Docker，是因為兩個性質剛好對上這裡的需求：

- **沒有 daemon。** 容器是一般的子行程，由 systemd 管，不必養一個 root 的常駐服務。
- **預設 rootless。** 容器跑在一個普通使用者底下。這些容器本來就跑在 Incus 系統容器裡了，再加一層非 root 是很便宜的縱深。

而「一個網段一台主機」讓網段邊界變成**實體的**——`dev` 的容器跑在 `dev` 那台上，不是靠某條規則說它屬於 dev。

### 為什麼要 Arcane

因為我要 compose 的內容進 Terraform。

Arcane 是一個管理 Podman 的 Web UI，重點是它有 **API**，而且有 Terraform provider。所以 compose project、volume、network 都是 Terraform 資源——`tofu apply` 改的是真正被執行的那份 compose，不是「我 SSH 上去改了一個檔案然後希望自己記得」。

這是第二篇那句「我終於能相信我的 IaC」的最後一哩：沒有 Arcane，最後這一段又會退回手工。

管理 API 走一個固定埠（也是靠 network forward 開出來的），token 有個小規矩：**它必須以 `arc_` 開頭**，Arcane 會驗格式，不合格的 key 會讓它在啟動時直接拒絕。

### 重開機之後還活著，要做三件事

rootless 的代價是重開機之後不會自動回來——除非明確設定。踩完之後定案的三件事：

1. Incus 容器本身設 `boot.autostart`。
2. cloud-init 開 **user lingering**（`loginctl enable-linger`），讓這個使用者的 systemd service 在沒人登入時也能跑。
3. 啟用 rootless 的 `podman.socket` 和 `podman-restart.service`。

然後是這一段最值得記下來的坑：

> **`podman-restart.service` 只會啟動 `restart: always` 的容器。**

Docker 的 daemon 開機時會把 `unless-stopped` 的容器也拉起來，Podman **不會**。所以這裡每一個 compose project 都明確寫 `restart: always`——不是偏好，是不寫就開不回來。

更麻煩的是，改了模板只影響**之後**建立的容器。既有的要就地修：

```sh
podman update --restart=always <container>
```

或是把整個 project 重新 up 一次。我是在一次重開機之後才發現有幾個容器沒回來的。

### 比較重的東西走另一條路

有些 stack 太重、或需要在容器裡跑 Docker（例如 CI runner 要用 Docker socket 跑 job），這種就不塞進共用的 podman host，而是給它一台**專屬的 Incus 容器**，裡面裝 Docker。

機制是：`tofu apply` 把算好的 `compose.yaml` 推上去，一對 systemd `.path` → `.service` 監看那個檔案，內容一變就自動重佈。改一個模板、跑一次 apply，服務就換掉了——不用登入、不用手動 `docker compose up`。

（這裡也有個坑：`docker compose up -d` **不會**發現 compose 裡 inline `configs:` 的內容改了，它只會說「Container Running」然後繼續用舊的設定檔。這種服務要強制重建。）

## Authentik：一個身分，四種協定

最後一根柱子是身分。[Authentik](https://goauthentik.io/) 是這座 lab 的 IAM，一整套跑在 `infra` 上：PostgreSQL + Redis + server + worker。

會想架這個，是因為服務一多帳號就開始失控——每個服務一組密碼，改密碼要改五個地方，想關掉某個帳號得一個一個找。SSO 不是為了炫技，是為了**讓「這個人能不能進來」只有一個地方可以回答**。

有意思的是，接上 SSO 的服務用了四種不同的接法——因為它們支援的東西就是不一樣：

| 服務 | 協定 | 為什麼是這個 |
| --- | --- | --- |
| Forgejo | OIDC | 原生支援，最乾淨 |
| SonarQube | SAML | 它的 SSO 走 SAML |
| FreshRSS | 原生 OIDC | PHP 端自己實作的 OIDC client |
| Calibre-Web | **LDAP** | 它沒有通用 OIDC，只認 LDAP |

Calibre-Web 那個最麻煩：為了讓它能接上，得另外跑一個 **Authentik LDAP outpost**——把 Authentik 的使用者目錄以 LDAP `:389`／LDAPS `:636` 對外提供，Calibre-Web 再對它做 bind。等於為了一個不支援現代協定的軟體，退回去講二十幾年前的協定。

但這正是自架的現實：你不能要求每個開源專案都支援 OIDC，只能讓身分提供者去遷就它們。

而這一切——OIDC provider、SAML provider、LDAP provider、應用程式、群組、權限綁定、outpost 的 token——**全部在 Terraform 裡**。FreshRSS 的 client id 和 secret 甚至是直接從 Authentik 的 resource 屬性接過去的，不用人工複製貼上。這是我覺得最有價值的一段：SSO 設定向來最容易「只存在於某個管理介面裡」，把它寫進程式碼之後，整條串接是可以 code review 的。

## 其他跑著的東西

骨幹講完了，剩下的快速帶過。

**開發層（`dev`）**

- **Forgejo**——Git hosting，走 Authentik OIDC。這座 lab 的 IaC 原始碼自己就放在上面；GitHub 那份是唯讀的公開鏡像，還設了 pre-push hook 擋住往 GitHub 推。
- **Forgejo Actions Runner**——CI。它跑的 job 容器裡也掛了 lab 的 CA bundle，不然 `git clone` 過不了 step-ca 的憑證。
- **SonarQube**——程式碼品質掃描，走 SAML。它內嵌的 Elasticsearch 想要很高的 `vm.max_map_count`，那是 host 層級參數，在容器裡改不動——最後靠它 dev 模式綁 loopback、檢查不致命才繞過去。

**應用層（`app`）**

- **FreshRSS**——RSS 閱讀器，原生 OIDC。
- **Calibre-Web**——電子書，走 LDAP outpost。
- **RustFS**——S3 相容的物件儲存。

**基礎層（`infra`）**

- **SMTP relay**——Postfix，STARTTLS 收件、轉給外部 smarthost，讓 lab 內的服務都有地方寄通知信。憑證就是前面那個 step-cli sidecar 簽的。
- **Cloudflare Tunnel**——不開任何一個對外連接埠。connector 主動往外撥，ingress 規則在 Cloudflare 那邊管。它坐在 `infra` hub 上，所以能代理到任何一個 peered 網段的來源。

---

寫到這裡，三篇的主線其實是同一條：不是「我裝了很多東西」，而是**我終於能把整座 lab 講清楚**——為什麼選這台機器、為什麼選這個平台、每個服務為什麼長這樣、封包從 LAN 到容器中間經過哪幾層。

前兩代做不到這件事。這一代可以。這就是體驗最好的那個部分。
