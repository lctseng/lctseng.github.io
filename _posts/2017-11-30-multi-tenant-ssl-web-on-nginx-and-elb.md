---
layout: post
date:   2017-11-30 22:30:00 +0800
category: programing
tags: [web-develop, devops]
title: 在 AWS 上搭建以 Rails/Nginx 為基礎，具有自動部署 SSL 之 multi-tenancy 應用程式
description: 這篇文章要和大家分享我在過去工作中，如何在 Nginx 與 AWS ELB 之上搭建一個具有 SSL 自動簽署的 multi-tenancy 應用程式的經驗。文章中雖不會提及如何撰寫網頁程式本身，但會分享在部署此種應用程式到 Nginx 及 AWS 上會遇到的問題，以及對應的解決方法。
image: https://i.imgur.com/E0Zr6O1.png
---

在這篇文章中，將會分享我過去在商業專案上的經驗：如何在 AWS 上搭建以 Rails/Nginx 為基礎，具有自動部署 SSL 之 multi-tenancy 應用程式。雖然是主題是講「如何搭建」，但這篇文章主要以設定檔與伺服器環境配置為主。畢竟網站程式的功能百百種，沒有辦法在這裡列舉每一種應用程式的情況。因此，筆者會假設讀者已經擁有開發好的 multi-tenancy 專案，但尚未部署至有 SSL 環境。

在開始之前，先定義一下「在 AWS 上搭建以 Rails/Nginx 為基礎，具有自動部署 SSL 之 multi-tenancy 應用程式」這一句話所要描述的程式概要，讓讀者確認一下是不是滿足自己的需求。

- 以 Ruby on Rails 作為開發框架的網站應用程式
- 網站伺服器基於 Nginx
- 網站連線透過 HTTPS，需要簽署並安裝 SSL Certificate
- 網站服務提供可區分網域的 multi-tenancy 功能，且每一個 tenant 可以自由綁定自己想要的網域名稱如 `www.myshop.com`，而非統一的子網域 如 `myshop.example.com`
- 架設在 Amazon Web Service (AWS) 上，使用 EC2 虛擬機運行網站，並以 Elastic Load Balancer (ELB) 進行負載分流 

一個滿足以上條件的具體例子：電子商務服務供應商。
類似於 Shopmatic, Shopify 或 Shopline 這樣子的服務，在這個專案中，網站供應商（也就是架設這個網站的各位讀者）提供一個平台讓各地的商家可以利用他們的平台開設屬於自己的網路商店。在這個網路商店中，商家可以自行購買自己想要使用的網域名稱，並綁定至電商網站中。綁定後，網站會自動驗證該網域的真偽，確認一切正常後，會進行 SSL 憑證的簽發，讓客戶可以透過商家綁定的網域名稱以 HTTPS 的方式造訪商店。

![Overview](https://imgur.com/RbhK0st.jpg)

看到這裡，讀者可能會覺得這樣子的網站似乎沒什麼，以 Rails 作為開發框架的話，只要使用基本的 MVC 搭配 [apartment](https://github.com/influitive/apartment) 這個支援 multi-tenancy 的 gem，似乎就完成了。至於 AWS 的 EC2/ELB 也只需要照著 AWS 官方說明，也可以一步步搭建起來。網域的綁定則是請商家在申請完網域後，設定合適的 `CNAME` record 指向供應商的網域，這部分並非供應商需要處理的事情。 最後是 SSL 憑證的部分，現在已經有如 [certbot](https://certbot.eff.org/) 這種基於 [Let's Encrypt](https://letsencrypt.org/) 的全自動的免費 SSL 憑證取得工具，只要寫一些背景程式在使用者綁定網域後自動執行憑證取得就好了，到底有什麼難的？

的確，如果今天只需要做到「透過網域區分不同商家的資料」與「在商家綁定網域後自動取得 SSL 憑證」這兩個功能的話，活用上述幾個套件應該就可以滿足需求了。但如果今天讀者對於 Nginx 架設 SSL 網站有所涉略的話，應該會發現問題點在於「如何把這些 SSL 憑證部署到伺服器上」。以 Nginx 為例，假設供應商的網域為 `example.com`，使用的 SSL 憑證網域是 `*.example.com`，憑證檔案路徑是 `/etc/ssl/example.com/fullchain.pem`，SSL Private Key 檔案路徑是 `/etc/ssl/example.com/privkey.pem`，那麼可能會在 Nginx 的設定檔中出現以下片段：

```nginx
server {
  listen 443 ssl http2;

  server_name ~^(.+\.)?example.com;
  
  ssl_certificate /etc/ssl/example.com/fullchain.pem 
  ssl_certificate_key /etc/ssl/example.com/privkey.pem;

  # Other configurations...
}
```

乍看之下沒有什麼問題，就只是常見的 SSL 網站伺服器的基礎設置。但現在問題來了：在這個設置之下的 Nginx，只會將來自 `*.example.com` 的請求交給這個 `server` block 來處理。因此如果使用商家自行綁定的網域名稱（例如 `www.myshop.com`）來瀏覽的話，會發現 Nginx 不會將瀏覽器的請求交給這一個 `server` block 處理。為了能夠支援商家自行綁定的網域，我們必須要移除 `server_name` 這一個指令。省略 `server_name` 後，Nginx 便會允許客戶瀏覽器以 `www.myshop.com` 的網域來連到商家的網站。
問題就這樣解決了嗎？當然不是！當客戶以 `www.myshop.com` 連接網站時，會發現雖然可以連上，但卻會收到瀏覽器發出的「SSL domain mismatch」之類的警告。原因很簡單：因為 `www.myshop.com` 很明顯就不屬於 `*.example.com` 這一張憑證的涵蓋範圍內。

咦？之前不是有替商家自己綁定的網域使用 `certbot` 取得憑證嗎？如果要讓 SSL 憑證正確，不就只要新建一個 `server` block，並且填入該網域的憑證與 Private Key 不就得了？例如下面這樣的設定檔片段：

```nginx
server {
  listen 443 ssl http2;

  server_name www.myshop.com;
  
  ssl_certificate /etc/ssl/www.myshop.com/fullchain.pem;
  ssl_certificate_key /etc/ssl/www.myshop.com/privkey.pem;

  # Other configurations...
}
```

沒錯，如果新增網域是極少數發生的情況，說不定可以透過手動設定不同的 `server` block 來解決。但通常一個電商網站動輒有上千上萬個自訂網域，這種每次新增網域都要更新 Nginx 設定檔的方式肯定不行。此外，將 `certbot` 取得的憑證檔案放在硬碟上也不是個好的管理辦法。
因此，有沒有某種機制，可以讓我們在新增網域時不需要更動 Nginx 的設定檔呢？有的，這也是今天要講的主題之一：如何讓 Nginx 根據不同網址選擇不同的 SSL 憑證來建立連線，並且使用 Redis 來管理憑證。

![Nginx with static certificates](https://imgur.com/NaVo9t6.jpg)

## 第一階段：如何設定 Nginx 來支援動態切換 SSL 憑證

由於根據網址切換不同 SSL 憑證這件事情需要特殊的邏輯，目前版本的 Nginx (1.11.x版本)沒辦法直接支援這樣的功能。因此我們需要一個特製的 Nginx：[ngx_mruby](https://github.com/matsumotory/ngx_mruby)。這個 Nginx 外掛模組中允許使用者使用 MRuby (Ruby 程式語言的一種高性能的變形) 來撰寫 Nginx 的邏輯。我們會將切換 SSL 憑證的功能寫成 Ruby 程式，並利用這個外掛模組將把程式碼加入到 Nginx 中。至於這個外掛模組具體的安裝與設定方式都已經在專案 repo 中有詳細的教學了，這裡不再贅述。

安裝完 `ngx_mruby` 外掛模組後，下一個任務就是將自動簽署得到的 SSL 憑證存入到如 Redis 這樣的記憶體式資料庫。之後我們便可以在 Nginx 中利用 MRuby 程式碼從 Redis 中取出合適的憑證。要完成這一步，我們可以使用另一個 Rubygem: [rails-letsencrypt](https://github.com/elct9620/rails-letsencrypt) 來幫助我們完成。這個 gem 可以讓我們輕鬆的在 Rails 環境中進行 SSL 憑證的自動簽署，並將之存到 Redis 中。

詳細的安裝與操作方式都可以從該 repo 的 README 中找到，這裡僅舉出與 Nginx 叫相關的部分。
當正確安裝好這個 gem 之後，我們便可以在 Rails 環境中以背景工作的方式進行憑證簽署，簽署成功的憑證會自動存到 Redis 中。接下來，為了要讓 Nginx 可以存取到 Redis，我們要在 Nginx 啟動時，建立連線到 Redis：

```nginx
http {
  # ...
  mruby_init_worker_code '
    userdata = Userdata.new
    userdata.redis = Redis.new "127.0.0.1", 6379 # 這裡填入 Redis server 的網址與 port
    userdata.redis.select 1 # 填入 Redis database 的編號
  ';
}
```

接下來，我們要在 `server` block 中加入根據網域替換 SSL 憑證的功能：

```nginx
server {
  listen 443 ssl;

  mruby_ssl_handshake_handler_code '
    ssl = Nginx::SSL.new
    domain = ssl.servername

    redis = Userdata.new.redis
    unless redis["#{domain}.crt"].nil? and redis["#{domain}.key"].nil?
      ssl.certificate_data = redis["#{domain}.crt"]
      ssl.certificate_key_data = redis["#{domain}.key"]
    end
  ';
}
```

完成到這一步之後，我們的 Nginx 就可以根據網域名稱自 Redis 中取得合適的 SSL 憑證了！

![Nginx with dynamic certificates](https://imgur.com/YMBRU53.jpg)

## 第二階段：如何在 AWS ELB 之上運行動態替換 SSL 憑證的服務

如果這個服務僅在單台主機上運行，那麼是不需要做到這一階段的。不過通常一個擁有數千數萬電商的專案，不會只有一台 web server。因此很多情況我們會在 AWS 上開許多的 EC2 作為網站伺服器，並使用 ELB 作為負載平衡，將來自世界各地的瀏覽器請求分散至數台伺服器中。在筆者的專案中，是使用 Classic ELB，進行傳統的 HTTP/HTTPS 負載平衡。如果是一般不使用 `ngx_mruby` 的專案，那麼我們通常會設定兩個 HTTP 的 listener，分別指定 80 與 443 作為對外的 port，藉此將瀏覽器請求導向內部的伺服器。這樣的做法我們會需要在 ELB 上綁定網域的 SSL 憑證（如上述例子的 `*.example.com` 或 `www.myshop.com` 的憑證），好讓客戶的瀏覽器可以與 ELB 建立正常的 SSL 連線，如下圖：

![ELB with single certs](https://imgur.com/xJn0fIR.jpg)

但如果要把這種 ELB 的設定方式套用在動態改變 SSL 憑證的專案上，恐怕就行不通了。就好比一開始提到一般 Nginx 無法在同一個 `server` block 內綁定多個 SSL 憑證一樣，在 ELB 上我們也無法在同一個 listener 上綁定多個 SSL 憑證。因此若強行瀏覽網站，便會出現 SSL 憑證不一致的問題。

![ELB with single certs, not match](https://imgur.com/Fll9ETA.jpg)

但這次的問題比之前更棘手，因為我們沒辦法透過重新編譯的方式來改變 ELB 的行為，因此像之前那樣「在 Nginx 上動手腳」的方式沒辦法適用在 ELB 上。因此這次我們只能透過使用 ELB 原本就有的功能來達成這個任務。

所幸 Classic ELB 除了提供 HTTP/HTTPS 的 listener 之外，還提供直接使用 TCP socket 的 listener。既然我們無法在 ELB 上進行多張憑證的綁定，那麼就把所有工作轉交到 Nginx 上吧！ 我們可以如下圖這樣設定，將80與443的 TCP 連線直接交給 ELB 後方的 EC2 instance：

調整完 ELB 設定後，就可以將替換 SSL 憑證的任務交給 Nginx 了！而 ELB 在此處變成了單純的 Network Load Balancer。

![ELB with TCP Proxy](https://imgur.com/x0LRE6W.jpg)

## 第三階段：設定 TCP Proxy 以保留客戶的來源 IP

如果讀者的 Rails 應用程式中需要根據客戶的來源 IP (如 Rails 中的 `remote_ip`)來進行某些判斷的話（例如提供不同地區不同的內容），會發現將 listener 改成 TCP 後，Rails 內收到的所有的 `remote_ip` 都會變成 `10.x.x.x` 這樣子的內部 IP，使得我們的網站伺服器無法判斷消費者連線時所使用的 IP。

![ELB with internal IP](https://imgur.com/NnlCY2x.jpg)

這其實是因為 ELB 變成只負責 TCP 連線後，會失去將來源 IP 製作成 `$proxy_add_x_forwarded_for` 的能力。而一般透過 Nginx 作為 Reverse Proxy 的 Rails 應用程式，應該會有以下這個設定檔片段：

```nginx
location / { 
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_pass http://localhost;
}  
``` 

在失去 ELB 提供傳統 HTTP `$proxy_add_x_forwarded_for`後，我們必須要使用別的方式讓 ELB 把客戶端 IP 的資訊轉交到 Nginx 手上。幸好對於純 TCP 的 listener，我們可以透過使用「TCP Proxy」這個技術，來讓純 TCP 的 Load Balancer 可以把 IP 資訊記錄在連線中。可惜在筆者轉寫此文章時，AWS Web Console 仍未支援 TCP Proxy 的設定，因此要使用這項功能必須透過 AWS 的 Command Line Tool: [aws-cli](https://aws.amazon.com/tw/cli/) 來達成。 

如果讀者已經安裝並設置好`aws-cli`，再來就是透過它設定我們的 ELB `my-load-balancer` 來啟用 TCP Proxy。啟用的指令如下：

```txt
aws elb create-load-balancer-policy --load-balancer-name my-load-balancer --policy-name http-ProxyProtocol-policy --policy-type-name ProxyProtocolPolicyType --policy-attributes AttributeName=ProxyProtocol,AttributeValue=true
aws elb set-load-balancer-policies-for-backend-server --load-balancer-name my-load-balancer --instance-port 80 --policy-names http-ProxyProtocol-policy
aws elb set-load-balancer-policies-for-backend-server --load-balancer-name my-load-balancer --instance-port 443 --policy-names http-ProxyProtocol-policy
```

完成 ELB 的設定後，我們也需要更動 Nginx 的設定檔。具體的設定可以參考這篇官方文章：[Configuring NGINX to Accept the PROXY protocol](https://www.nginx.com/resources/admin-guide/proxy-protocol/)。這裡提出幾個重要的步驟：
1. 設定 Nginx 接受 proxy_protocol 的連線

```nginx
server {
    listen 80   proxy_protocol;
    listen 443  ssl proxy_protocol;
    ...
}
```

2. 設置`set_real_ip_from`，這個欄位填寫 ELB 所在的內部 IP 位址。

```nginx    
server {
    ...
    set_real_ip_from 10.0.0.0/8;
    ...
}
```

3. 設置`real_ip_header`：

```nginx
server {
    ...
    real_ip_header proxy_protocol;
}
```

4. 調整`proxy_set_header`的設定，將 TCP Proxy Protocol 的位址交給上游的 web server：

```nginx
location / { 
        proxy_set_header X-Real-IP       $proxy_protocol_addr;
        proxy_set_header X-Forwarded-For $proxy_protocol_addr;
        proxy_set_header Host $http_host;
        proxy_pass http://localhost;
}  

```

完成設定並重新啟動 Nginx 後，便可以看到內部的 web server 正確收到客戶端的 IP 了！

![ELB with TCP PROXY](https://imgur.com/ygY8jQN.jpg)


> 注意：啟用 proxy protocol 後，那一個 server block 將不可以再接受沒有使用 Proxy Protocol 的連線。也就是說，不可以直接使用 80 或 443 port 去連接內部的 server。


## 總結

在這篇文章中，我們探討如何在 AWS ELB 搭配 Nginx 的環境上部署自動驗證與發給 SSL 憑證的 web server。雖然這篇文章沒有教讀者如何撰寫那一個 multi-tenancy 的應用程式本體，而僅是著重在如何部署，因此沒有辦法幫助到那些正在開發這類應用程式的朋友們。但如果讀者已經完成應用程式並打算上線運行，這篇文章或許可以參考一下。畢竟這方面的資源並不多，筆者當初也是到處碰壁才整理出這些心得的。至於可信度，筆者接手的電商平台專案中，有高達十萬個 tenant，其中活躍使用中的 SSL 憑證也有接近一千筆，因此可以證實這個做法在這樣的流量下仍可正常運作。當然，不同的應用程式情境不同，這個流量僅供各位參考。

那麼，這次的分享就到這裡告一段落，謝謝閱讀到此處的各位讀者！
