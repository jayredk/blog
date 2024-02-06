---
author: jayredk
pubDatetime: 2024-01-29T14:22:00Z
title: 在網址列輸入 URL 後到網頁顯示的過程中發生了什麼事？
slug: what-happens-when-you-type-a-url-into-your-browser
featured: true
draft: false
tags:
  - interview-questions
  - network
description:
  以簡單直白的方式分享瀏覽器造訪一個網站到顯示出來的過程
---

## 前言
本文會簡單扼要的介紹在網址列輸入 URL 後到網頁顯示的整個流程

涉及到的範疇僅限於 [OSI 模型（OSI Model）](https://www.cloudflare.com/zh-tw/learning/ddos/glossary/open-systems-interconnection-model-osi/)中的「應用層（Application Layer）」

## 為什麼有這篇文章？

這是一個從國外流傳到國內的經典面試題，

儘管已經有不少人寫過，

終究還是要自己寫一遍才是真的，

作為前端開發者，寫出來的程式會與瀏覽器交互，

勢必得了解一些網路、網頁、瀏覽器等基礎知識、運作原理，

以便在需要的時候能夠對網頁進行性能優化、除錯，

而這一連串的過程，考驗了我們對於這些知識的了解程度。

## Table of contents

## 流程總覽

1. 使用者在瀏覽器的網址列輸入 URL
2. 瀏覽器向 DNS Resolver 詢問 URL 的 IP 位址
（`DNS Lookup`）
3. 瀏覽器透過 IP 位址向主機`建立 TCP 連線`
4. 瀏覽器`發送 HTTP/HTTPS Request` 給主機
5. 瀏覽器`接收到 HTTP/HTTPS Response`
6. 瀏覽器`根據 Response 的檔案渲染網頁`


## 什麼是 URL
URL 是 `Uniform Resource Locater` 的縮寫，也是所謂的「網址」。

它就像網路的門牌一樣，你可以透過它前往其他網站、存取不同資源

以下圖為例，常見的 [URL 組成](https://www.geeksforgeeks.org/components-of-a-url/)分為三塊，分別是：協定（Protocol）、域名（Domain name）以及路徑（Path）

![Structure of URL](@assets/images/url.png)



對於瀏覽器來說，可以直白的理解 URL 代表的意思是使用 HTTPS 協定存取 en.wikipedia.org 下的 wiki/URL


## DNS Lookup

DNS Lookup 主要的目的就是將 URL 轉換成 IP 位址。

原因是因為網路上的門牌是 IP 位址，而 IP 位址是一串數字，人們不擅長記憶一連串的數字，因此出現了 DNS（Domain name system）

人們透過記憶域名來訪問各個網站，相對於 `172.217.163.46` 這類 IP 位址，`google.com` 還是比較好記的

什麼？人很擅長記數字？怕是看到這張圖先投降了，OK 的，投降輸一半

![List of root nameserver](@assets/images/root_nameserver.png)
[Reference](https://www.iana.org/domains/root/servers)

### DNS Lookup 的前置作業

回到正題，

在 DNS Lookup 前，其實瀏覽器會先檢查各個 cache 看看是否已經有記錄對應 URL 的 IP 位址

依序為：
1. 瀏覽器的 cache
2. 作業系統的 cache

瀏覽器的 cache 可以直接透過瀏覽器看到，作業系統的 cache 則還沒找到比較通用的方法。

若是使用 chrome 可以直接在網址列輸入 `chrome://net-internals/#dns`，若是 firefox 則可以輸入 `about:networking#dns`

以 firefox 為例，可以看到 cache 記錄了許多域名對應的 IP 位址
![Browser's cache](@assets/images/browser_dns_cache.png)


### DNS Lookup 的流程
DNS Lookup 的過程中，若 cache 中有找到對應的 IP 位址會提前返回給瀏覽器，忽略剩下的步驟。

假設 cache 沒有對應的 IP，

完整的流程為：
1. 瀏覽器向 DNS resolver 查詢域名對應的 IP 位址
  - 檢查路由器的 cache 是否有對應的 IP 位址
  - DNS resolver 檢查自身 cache 是否有對應的 IP 位址
2. DNS resolver 向 root nameserver 查詢
  - 若無對應 IP 位址，會返回 TLD nameserver 的位址
3. DNS resolver 向 TLD nameserver 查詢
  - 若無對應 IP 位址，會返回 authoritative nameserver 的位址
4. DNS resolver 向 authoritative nameserver 查詢
5. authoritative nameserver 回傳對應到域名的 IP 位址
6. DNS resolver 向瀏覽器回傳 IP 位址

![DNS lookup](@assets/images/dns_lookup.png)

流程中的 DNS resolver 通常指的是 ISP 的 DNS server，也就是說如果你的網路廠商是種花電信，那麼這裡的 ISP 指的就是種花電信。



## 建立 TCP 連線

### 為什麼要建立 TCP 連線？
HTTP 協定最終的目的是希望讓資料在網路傳輸的過程中保有正確性、完整性，

而 TCP 是一個可靠的協定，負責確保資料的完整性，TCP 的機制會將欲傳送的資料在發送端拆分，並在接收端重組，

因此，在發送 HTTP/HTTPS 請求前會需要先建立 TCP 連線，在此基礎上傳送 / 接收資料。

> NOTE: 雖然這裡提到 HTTP/HTTPS 協定會在 TCP 連線的基礎傳送 / 接收資料，
>
> 不過最新的 HTTP/3 已經正式棄用 TCP，改採用以 UDP 為基礎的 QUIC 協定
>
> 而部分的網站也確實已經改採 HTTP/3 進行傳輸了
>
> 日後會再撰寫一篇關於 HTTP 的文章，敬請期待

### 建立 TCP 連線的流程

1. 瀏覽器透過 IP 位址，向主機發送一個 SYN 封包
2. 主機準備好接收連結後，回傳一個 SYN/ACK 封包
3. 瀏覽器向主機發送一個 ACK 封包

這樣的流程又被稱為 TCP 3 way handshake

![TCP 三次握手](@assets/images/tcp_three_way_handshake.png)

當主機收到瀏覽器發送的 ACK 封包，TCP 連線就成功建立了。

## 發送 HTTP/HTTPS Request

當要發送 HTTP Request 時，就是我們傳送 / 請求資料的時候了

HTTP Request 主要分為 headers 和 body，也就是頭跟身體

下列是 header 所帶的一些常見資訊：
1. HTTP method
2. URL
3. HTTP version
4. Cookie
5. User-Agent
6. Accept

包含 HTTP method、URL、HTTP version 的例子
![HTTP request headers](@assets/images/http_request_headers_1.png)

包含 Cookie、User-Agent、Accept 的例子
![HTTP request headers](@assets/images/http_request_headers_2.png)

 

而 body 則取決於 Content-Type 來決定傳送的資料格式，

> Note： Content-Type 會出現在 HTTP request / response headers 中

常見的就是透過 JSON 格式傳遞資料，

例如：
```javascript
{
  "message": "hello"
}
```

若以訪問網站為例，當瀏覽器發送 HTTP Request，主機接收到後，就會返回相應的資源給瀏覽器，像是 html 檔

在返回 html 檔之前，主機會在 HTTP response headers 的 content-type 欄位加上 `text/html; charset=utf-8`，確保瀏覽器收到檔案後會以 html 的形式來處理檔案

![HTTP response headers](@assets/images/http_response_headers.png)


## 根據 Response 渲染網頁

瀏覽器接收到 html 檔案後會根據 html 的內容從上到下進行解析，

渲染的過程大致包含：

- 形成欲顯示的網頁結構
- 下載需要的動 / 靜態資源，如 css、js 檔、圖片、字型、icon 等等
- 分析 CSS，形成樣式結構
- 將 CSS 樣式套用到網頁結構上，形成要渲染的結構
- 根據渲染結構進行排版
- 將排版好的結果呈現在瀏覽器中
- 執行 JS 程式，執行的過程中若遇到有缺少資料可能會透過 AJAX 請求資料


## 參考資料
- [A brief explanation of what happens when you type a URL into a web browser by Justus njogu - linkedin](https://www.linkedin.com/pulse/brief-explanation-what-happens-when-you-type-url-web-browser-njogu)
- [What happens when you type a URL into your browser? by Jenna Pederson - amazon blogs](https://aws.amazon.com/blogs/mobile/what-happens-when-you-type-a-url-into-your-browser/)
- [經典前端面試題：從瀏覽器網址列輸入 URL 按下 enter 發生了什麼？ - Shubo 的程式開發筆記](https://www.shubo.io/what-happens-when-you-type-a-url-in-the-browser-and-press-enter/)
- [What is DNS? | How DNS works - cloudflare](https://www.cloudflare.com/learning/dns/what-is-dns/)
- [DNS lookup | what-happens-when - GiHub](https://github.com/alex/what-happens-when?tab=readme-ov-file#dns-lookup)
- [Populating the page: how browsers work - MDN](https://developer.mozilla.org/en-US/docs/Web/Performance/How_browsers_work)