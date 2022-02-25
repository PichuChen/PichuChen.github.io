# 第一次送 Push Request 到 Golang 就失手

2021 年底的時候，偶然看到 Golang 的程式碼在 Parse IP 判斷是 IPv4 還是 IPv6 是從 0 的位置開始判斷的，因為做網路的長期經驗，直覺就覺得這不是最佳的做法。

![image](https://user-images.githubusercontent.com/600238/152420097-06cd3c91-1101-4fea-a6e3-8e3aab1f0732.png)

因為通常 IP 位置的字串長度並不長，以 IPv4 來說最長大約是 15 bytes (255.255.255.255) 最短是 7 bytes (1.1.1.1)。
而 IPv6 最長會是 39 bytes (FFFF:FFFF:FFFF:FFFF:FFFF:FFFF:FFFF:FFFF) 最短則是 2 bytes (::)。
<img width="607" alt="截圖 2022-02-04 上午4 24 19" src="https://user-images.githubusercontent.com/600238/152422988-1048f754-d495-4b6b-a6f0-32b37a9fa2f1.png">



也就是說，如果從0開始判斷的話，最多要比較 5 次才能判斷是 IPv6 還是 IPv6，最少要 1 次，典型來說會落在 2 ~ 5 次。
但是從 1 開始判斷的話，最多只需要比較 4 次就可以得到結果了，比較的次數會大幅減少呢！
因為 IPv6 的最短形式是兩個冒號 (:) 的關係，所以在 IPv6 的最短狀況上會比較到第二個冒號，因此不會有問題。

<img width="591" alt="截圖 2022-02-04 上午4 26 01" src="https://user-images.githubusercontent.com/600238/152423209-186f7e8c-0031-420c-8ee9-0ed76ab3367d.png">


那既然如此的話，就試試看把這個 PR 送上去吧！

## 實作

首先是下載下來，下載以及修改 Golang 的過程基本上很簡單，按照[這邊](https://go.dev/doc/contribute#checkout_go)的步驟下載以及測試就可以了。

接著傳上自己的 GitHub，送出 PR 然後 Google 的 Bot 就會提醒你要簽 CLA 等等的東西。

送上去的 [PR 在這邊](https://github.com/golang/go/pull/50086)

接著是比較麻煩的部分，Golang 並不是透過 GitHub 來做開發的，主要的 Code Review 在 gerrit 上面，
所以需要在 gerrit 上面註冊一個帳號。

接下來就是等 Review 的訊息了。

## Review

[Review 的結果](https://go-review.googlesource.com/c/go/+/372397)基本上不太意外，簡單來說這個修改其實會降低可讀性，然後也沒有 Benchmark 來證明到底會快多少。

<img width="1269" alt="截圖 2022-02-04 上午6 20 21" src="https://user-images.githubusercontent.com/600238/152438828-f8a4a47e-9c65-440a-871d-9d0992b99df6.png">

後來雖然有補充說明，不過還是被拒絕了。💔

這邊要特別提醒的是回應後按下 SAVE 他會變成草稿的狀態，還要按第二次才會送出。
我就是按下 SAVE 之後等了一個月想說怎麼沒有下問，才發現根本沒送出的人。



# 其他語言的做法

既然被因為可讀性的原因拒絕了，那其他語言的做法是如何呢？

## Rust

首先就先看到 Rust

在 Rust 裡面是用 `"127.0.0.1".parse()` 的方式判斷的，他會去呼叫實作該 `FromStr` 樣版的 `from_str` ，接著呼叫 `read_ip_addr`

<img width="667" alt="截圖 2022-02-04 上午4 41 40" src="https://user-images.githubusercontent.com/600238/152425288-efe41e6d-27f8-4966-b662-a6a43b8f18bb.png">
<img width="648" alt="截圖 2022-02-04 上午4 54 54" src="https://user-images.githubusercontent.com/600238/152427086-26e44afd-368c-44a7-997b-ce993524f3b2.png">
<img width="1045" alt="截圖 2022-02-04 上午4 55 19" src="https://user-images.githubusercontent.com/600238/152427162-100a7e26-ba6b-4d89-8562-b4fb602bfe81.png">

在這邊可以看到他會先完整判斷這個字串能不能被解析成 IPv4 位址，如果不行的話，那就嘗試解析成 IPv6 位址，因此要分析他從什麼時候確定是 IPv6 位置就要繼續從 `read_ipv4_addr` 繼續追下去。

<img width="759" alt="截圖 2022-02-04 上午5 09 37" src="https://user-images.githubusercontent.com/600238/152429185-d729f84a-2edc-4129-84ea-5592faac14a6.png">
<img width="937" alt="截圖 2022-02-04 上午5 14 22" src="https://user-images.githubusercontent.com/600238/152429830-c44b7278-4b3d-4e45-8742-83ddfbbdb05a.png">

這邊可以看到他會完全的讀入到第一個 `.` 前面的字串，然後把它轉成十進位數，也就是如果輸入一個 IPv6 字串的狀況，他會跑到出現冒號或是 A ~ F 之後只回傳之前出現的字，
接著在後面三段的 `read_number` 失敗導致回傳 `None`，然後改嘗試以 IPv6 進行解析。

這邊在看程式碼時要特別注意那個 `read_given_char` 其實是 skip sep 的意思。

因此整體而言， Rust 在這邊的解析是沒有比較好的，在整個追蹤程式碼的過程又十分麻煩（這段大概花了我幾個小時）。

## C 

其實 ANSI C 並沒有標準的網路函式庫，所以這邊其實是呼叫 POSIX 的函式庫。

在 C 裡面我們可以透過 `getaddrinfo` 指定 `hints` 的 `ai_family` 為 `PF_UNSPEC` 後回傳的 `res` 中的 `ai_family` 來判斷是 `PF_INET` (v4) 或是 `PF_INET6` (v6) 來判斷。

<img width="637" alt="截圖 2022-02-04 上午6 30 57" src="https://user-images.githubusercontent.com/600238/152440108-032c06d2-cf19-4946-a383-943edd652570.png">

不過實際上 POSIX 要怎麼實作還是會回到各個 OS, 因此在不同的手機上面、在 OSX 上面，在 Linux 上面，在不同版本的 Linux 上面都有可能不同。

為了簡化問題，我就隨便找一個網路上 [Google 的到的版本](https://code.woboq.org/userspace/glibc/sysdeps/posix/getaddrinfo.c.html)來追蹤。

首先很快地就能找到他呼叫了 `gaih_inet` (我猜他是 get ai_hints inet) 的意思

<img width="998" alt="截圖 2022-02-04 上午6 54 58" src="https://user-images.githubusercontent.com/600238/152442940-3514398e-d72b-42bf-86fd-cfebb89b3241.png">

接著在 `gaih_inet` 找到他呼叫了 `__inet_aton_exact` 這個 wrapper, 再呼叫 `inet_aton_end`

 <img width="711" alt="截圖 2022-02-04 上午6 58 56" src="https://user-images.githubusercontent.com/600238/152443393-e592f727-e69e-46b7-a5df-2930ed3dae6b.png">
<img width="637" alt="截圖 2022-02-04 上午7 05 24" src="https://user-images.githubusercontent.com/600238/152444098-58683263-7ff0-47bb-98fb-16026ecbc899.png">

看起來是要找到答案了...

<img width="644" alt="截圖 2022-02-04 上午7 09 09" src="https://user-images.githubusercontent.com/600238/152444462-e7cc55fa-4e53-4161-85df-5ae164b34f41.png">

答案揭曉，C其實也是從 pos 0 來判斷是 IPv4 或是 IPv6 的喔。

不過不一樣的地方是這個版本的效率其實非常的高，他一但判斷到不是數字或小數點時就會停止解析，直接回傳錯誤，
所以當 IPv6 的第一個冒號（或是第一個十六進位字元）出現時就會停止解析，然後開始改成解析 IPv6 。

# 結語
Golang 的 `ParseIP` 的設計看起來和 Rust 以及 glibc 是不一樣的設計思維，它分成偵測和解析兩個階段，而 Rust 和 glibc 都是一開始就直接解析，不行才換 IPv6 ，
這樣的話在 IPv6 應用漸增的今日看起來不是最佳解。

而判斷階段的程式碼其實十分清晰、一目瞭然，只用十行程式碼就表達出來是用什麼東西來作為 IPv4 以及 IPv6 的判斷依據。
在效能部分，實際上目前的 CPU 通常都會有[分支預測](https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html#speculative_execution)的能力，
因為在 CPU 執行機器語言時會分成擷取、解碼、執行和儲存的四個階段，這四個階段是同步進行的，然而在不知道下一個字元是 . 還是 : 時，
CPU 就不知道應該要跳進 `ParseIPv4` 還是 `ParseIPv6` 還是該繼續在迴圈裡抓下一個字元。

因此搞不好直接手動把 位置 1 ~ 位置 3 的字元進行位元處理，然後一次判斷後得到的結果會比迴圈從位置 1 來得快速且穩定。

# 補充

這是位置 0 版本的 `ParseIP` 第十行的部分使用的是 x86 指令的 XOR 來清零
<img width="551" alt="截圖 2022-02-04 上午7 55 14" src="https://user-images.githubusercontent.com/600238/152448907-826a3379-3ba7-4769-b858-62c558689cd4.png">

這是位置 1 版本的 `ParseIP` 第十行的部分使用的是 x86 指令的 MOVL 來賦值 1
<img width="665" alt="截圖 2022-02-04 上午7 56 44" src="https://user-images.githubusercontent.com/600238/152449039-d33f6066-3820-411f-9ea6-849437577644.png">
