---
layout: post
date:   2018-03-30 16:45:00 +0800
category: programing
tags: [web-develop]
title: Multi-tenant 應用程式：讓人又愛又恨的 Apartment
description: 隨著現在個人化網站的興起，許多網路服務都提供使用者自訂網站的功能，如 Weebly 與 Shopmatic 等。使用者可以在某些限度下自定網站的內容，建立屬於自己的資料，不會與其他使用這個服務的使用者衝突。此篇文章中會大致介紹此概念並帶讀者用 Rails 建立一個示範網站，並提出可能會遇到的問題。
image: https://i.imgur.com/dFAhCxU.jpg
---

隨著現在個人化網站的興起，許多網路服務都提供使用者自訂網站的功能。這些功能涵蓋許多面向，包含論壇、部落格與網路個人商店等。典型的自訂網站例子如 Wix 與 Weebly，個人商店則如 Shopmatic 與 Shopify 等等。在這些服務中，使用者可以在某些限度下自定網站的內容，建立屬於自己的資料，而不會與其他使用這個服務的使用者衝突。換句話說，不同的使用者使用同一個服務，但他們的資料卻是各自分離的，這便是最基本的 multi-tenant 的精神。由於以 multi-tenant 風格的網站建立越來越熱門，因此在各大網站開發框架上都有提供相關的開發工具可用。今天要跟各位讀者分享的，便是最常被用在 Rails 應用程式中的 multi-tenant 工具：`apartment`。

此篇文章中會大致介紹 `apartment` 這個套件如何使用，以及帶讀者建立一個簡單的 multi-tenant 部落格示範網站，並且提出一些使用 `apartment` 可能會遇到的問題。

## `apartment` 是什麼

`apartment`（[Github repo](https://github.com/influitive/apartment)）是 Ruby/Rails 圈最常用來製作 multi-tenant 的 gem。如同其英文「公寓」，`apartment` 可以讓我們蓋出公寓般的資料庫結構，每一戶的格局相同但卻都是獨立運行，互不影響。我們通常會把一戶單獨的使用者稱為一個「tenant」，也就是（公寓）的承租人。`apartment` 最大的特色是與 Rails 的 ActiveRecord 做了相當緊密的結合，使用者可以在指定了特定的 tenant 後，直接使用自己所熟悉的 ActiveRecord ORM（如部落格網站中取得所有文章的 ORM: `Article.all`，來取得只屬於自己 tenant 的資料），而不需要明確的加上 `where` 篩選條件（也就是不必用 `Article.where(tenant_id: 1)` 明確篩選自己的 `tenant_id`）。因為有著如此便利的特性，使得 `apartment` 經常被用來快速開發 multi-tenant 網站的原型。

底層資料庫的支援部分，`apartment` 最早先是以 PostgreSQL 為主要支援的資料庫，主要原因是因為 PostgreSQL 有「schema」的特性可以使用。`apartment` 會替每一個不同 tenant 各自開一個 schema，已達到資料隔離的功能。

後來版本的 `apartmemt` 也逐漸支援如 MySQL 等資料庫系統。不過基於穩定性與功能性因素，仍然建議大家使用 PostgreSQL 作為 `apartment` 底層的資料庫系統。因此，以下的情境都會假設使用者使用的資料庫是 PostgreSQL。

## `apartment` 的基本應用與例子

為了讓大家了解更近一步了解 multi-tenant 的應用程式運作，在這個章節中會帶讀者一步一步建立一個 multi-tenant 的部落格網站。以下環境是基於 `ruby 2.4.1` 與 `rails 5.1.4`

> **附註**：這個範例因為夠簡單，即使不需要使用 `apartment`，也可以輕易透過傳統的 `where` 條件來實作。`apartment` 帶來的好處在當網站中的 model 數量夠多時才會比較明顯（也比較讓人願意使用 `apartment`），但這裡基於示範的目的而刻意在這個迷你專案中使用 `apartment`。

首先我們先來定義一下這個迷你專案的功能。我們會製作一個具有登入功能的部落格網站，每一個使用者登入之後都可以對屬於自己的文章進行 CRUD （Create, Read, Update, Delete）操作，且無法干涉其他使用者的資料。

資料庫結構的部分如下圖，我們只會有兩個 model，分別是 `User` 與 `Article`。`User` 是存放登入用的使用者，同時也是文章的擁有者。使用者登入系統的部分我們會直接使用 `devise`（[Github repo](https://github.com/plataformatec/devise)）這一個套件來建立。至於 `Article` 則是每個使用者自己的文章，這個 model 會直接透過 Rails 的 Scaffold 功能直接建立。

在這個架構中，我們希望 `Article` 是每個使用者各自分開的，讓他們不能互相干擾。而 `User` 則是不根據 tenant 分開，因為這個 model 是給 `devise` 登入系統使用，因此是所有 tenant 之間共用的（稱為 `public` schema）。

大部分的欄位相信讀者都不陌生，幾個值得注意的是：`User` 有個特殊的欄位稱為 `tenant_name`，這是用來區別這一個使用者的 tenant，讓 `apartment` 知道該去哪一個 PostgreSQL schema 存取資料。至於存放在各 schema 中的表得（如 `Article`）則不需要傳統的 `user_id` 之類的欄位來記錄該文章屬於哪一個使用者。

![資料庫結構圖](https://i.imgur.com/1P7WJYc.png)

### 第一階段：建立沒有 multi-tenant 功能的一般專案

我們先由建立一個沒有 multi-tenant 功能的專案開始，先確保我們需要的每個零件都有正常運作。之後再來加入 `apartment` 來實現 multi-tenant 功能。

- 建立新專案

```text
rails new apartment-demo
```

- 修改 `Gemfile`，加入要用到的 gem，並且將資料庫 gem 改為 `pg`，並執行 `bundle`

```ruby
gem 'devise'
gem 'apartment'
gem 'pg'
# gem 'sqlite3' # 註解掉原本的 sqlite3
```

- 修改 `database.yml`，改為使用 PostgreSQL

```yml
default: &default
  adapter: postgresql
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: apartment-demo-dev

test:
  <<: *default
  database: apartment-demo-test

production:
  <<: *default
  database: apartment-demo-production
```

- 建立資料庫

```text
rake db:create
```

- 初始化 `devise`，並產生 devise views

```text
rails g devise:install
rails g devise:views
```

- 修改 `application.html.erb`，新增登入登出的連結

```erb
<% if user_signed_in? %>
  <h2> Hi, <%= current_user.email %></h2>
  <li>
  <%= link_to('Logout', destroy_user_session_path, method: :delete) %>
  </li>
<% else %>
  <li>
  <%= link_to('Login', new_user_session_path)  %>
  </li>
<% end %>
```

- 使用 `devise` 建立 `User` model，並加入 tenant_name 的欄位，同時也產生 controllers 供未來修改使用

```text
rails g devise user tenant_name:string
rails g devise:controllers users -c=registrations
```

- 修改產生的 migration file，記得將 `tenant_name` 設定為 `null: false` 並加入 unique index

```ruby
class DeviseCreateUsers < ActiveRecord::Migration[5.1]
  def change
    create_table :users do |t|
      ## Database authenticatable
      t.string :email,              null: false, default: ""
      t.string :encrypted_password, null: false, default: ""

      ## Recoverable
      t.string   :reset_password_token
      t.datetime :reset_password_sent_at

      ## Rememberable
      t.datetime :remember_created_at

      ## Trackable
      t.integer  :sign_in_count, default: 0, null: false
      t.datetime :current_sign_in_at
      t.datetime :last_sign_in_at
      t.inet     :current_sign_in_ip
      t.inet     :last_sign_in_ip

      t.string :tenant_name, null: false

      t.timestamps null: false
    end

    add_index :users, :email,                unique: true
    add_index :users, :reset_password_token, unique: true

    add_index :users, :tenant_name, unique: true
  end
end
```

- 修改 `app/views/devise/registrations/new.html.erb`，新增欄位讓我們可以填寫 `tenant_name`

```erb
<div class="field">
  <%= f.label :tenant_name %><br />
  <%= f.text_field :tenant_name %>
</div>
```

- 修改 `routes.rb`，讓 `devise` 使用我們自訂的 controllers

```ruby
devise_for :users, controllers: {
  registrations: 'users/registrations'
}
```

- 修改 `app/controllers/users/registrations_controller.rb`，使得我們可以更新 `tenant_name` 這個欄位

```ruby
class Users::RegistrationsController < Devise::RegistrationsController
  before_action :configure_sign_up_params, only: [:create]
  def configure_sign_up_params
    devise_parameter_sanitizer.permit(:sign_up, keys: [:tenant_name])
  end
end
```

- 建立 `Article` 的 Scaffold。注意我們並不需要 `user_id` 欄位。

```text
rails g scaffold article title:string content:text
```

- 在產生的 `app/controllers/articles_controller.rb` 上方加入權限控制

```ruby
before_action :authenticate_user!
```

- 執行 DB migrate，此時可能會看到 `apartment` 噴出的一些警告，目前可以暫時忽略

```text
rake db:migrate
```

- 設定 `routes.rb`，將網頁的首頁指向文章列表

```ruby
root to: 'articles#index'
```

到目前為止，讀者可以啟動 `rails server`，檢視 `devise` 與 Scaffold 是否可以正常運作。這邊我們先註冊兩個使用者 Alice 與 Bob，其 `tenant_name` 分別設置為 `alice` 與 `bob`。此時雖然有使用者驗證，但你會發現此時的 `Article` 並沒有根據使用者做區別，所有人的文章都混在一起。

### 第二階段：使用 `apartment` 將不同 tenant 的資料區隔開

有了基礎專案之後，接下來就是進行 `apartment` 相關的修改，使得不同使用者之間的資料完全區隔。

- 建立 `apartment` 設定檔：`config/initializers/apartment.rb`，並修改內容：

```ruby
Apartment.configure do |config|
  # 因為 Devise 登入只能放在同一個 schema，因此將之從 apartment 排除
  config.excluded_models = ["User"]
  # 告知 apartment 如何找出所有的 tenant_name 供 migration 使用，這裡我們使用 User model 同名的欄位 tenant_name
  config.tenant_names = lambda{ User.pluck(:tenant_name) }
end
```

- 手動替 `alice` 與 `bob` 建立各自的資料庫 schema。由於前一階段並沒有替這兩個 tenant 建立 schema，因此這邊透過   `rails console` 手動建立

```ruby
Apartment::Tenant.create('alice')
Apartment::Tenant.create('bob')
```

- 讓未來新註冊的使用者可以自動建立 schema，修改 `app/controllers/users/registrations_controller.rb`

```ruby
class Users::RegistrationsController < Devise::RegistrationsController
  def create
    super do |user|
      # 這個 block 將在使用者被建立後執行，因此我們可以從這裡取出 tenant_name 來建立 schema
      Apartment::Tenant.create(user.tenant_name)
    end
  end
end
```

- 修改`app/controllers/articles_controller.rb`，限制所有的資料都只能在當前登入的 tenant schema 底下操作（也就是資料隔離）

```ruby
class ArticlesController < ApplicationController
  # 注意這幾個 before_action 的順序
  before_action :authenticate_user! # 設定 devise 時加入的
  before_action :switch_tenant # 新加入
  before_action :set_article, only: [:show, :edit, :update, :destroy] # scaffold 產生的

  # 這個 before_action 將會根據當前的使用者切換至該使用者的 schema
  def switch_tenant
    Apartment::Tenant.switch! current_user.tenant_name
  end
end
```

完成了！沒錯，就這麼簡單。重新啟動 rails server 後，你會發現現在 alice 與 bob 所建立的 `Article` 已經完全隔離了。不但不會在列表中看到別人的文章，就算透過 `/articles/1` 這樣的路徑，也只能看到自己名下編號為1的文章而已，沒辦法看到其他使用者的。此時你如果建立第三個使用者 Carol，其 `tenant_name` 為 `carol`，你也會發現這個新建立的使用者完全與其他人資料隔離。

本範例專案的原始碼也可以在 [Github](https://github.com/lctseng/rails-apartment-demo) 上找到。

## 使用 `apartment` 可能會遭遇到的問題

從上面的專案中我們可以看到使用 `apartment` 建立 multi-tenant 應用程式是多麼容易的事情。那為什麼這篇文章的標題卻又說「又愛又恨」呢？原因是 `apartment` 天生設計上就有一個缺陷，那就是當 tenant 數量成長到一定程度時，會出現許多預想不到的副作用。

最簡單的例子就是 Database Migration。因為 `apartment` 把資料完全隔離，所以在進行 migration 時各個 schema 會別處理。因此，當你使用的 schema 數量越來越大時，最首先感覺到的變化就是進行 migration 的時間會拉長。

另一個問題是記憶體消耗與程式啟動時間。由於在 `rails 4.2.2` 之前版本的架構中，會在啟動時根據資料庫 schema 中的表格逐個建立一份對應的資料存在記憶體中。這是個相當消耗時間與記憶體的動作。這個問題在 `apartment` 的 Github repo 中有給予一個 [patch 後的 rails 版本](https://github.com/influitive/apartment#excessive-memory-issues-on-activerecord-4x)，可以稍微舒緩記憶體壓力。

最後一個問題是硬碟空間。在某些檔案系統中，單一磁區所能儲存的檔案數是有上限的。因此如果 schema 總數太高，會造成 PostgreSQL 建立過多的檔案導致碰撞到這個上限。雖然這個問題解法比較多元，且不同檔案系統對檔案數上限也不同，因此也不一定會被讀者的專案所遇到。但如果今天萬一遇到了，不像前面幾個問題可以透過升級記憶體或者等待更長時間來解決，而是必須進行立即的資料搬移。我們都知道資料搬移是一個相當繁重的過程，且過程中可能需要讓服務停機，其損失不可估計。

## 實際案例

筆者近期經手的專案就一次碰到了以上三個問題。曾經有一個 `apartment` 專案使用了超過十萬個 schema，除了 migration 幾乎快要一整天以外，每次啟動 server 或 console 都要花費十幾分鐘的時間，且 server 記憶體動輒就消耗到 10GB。最嚴重的是這個專案還曾經在 production 營運時撞到了檔案系統的檔案數上限，導致服務差點就要凍結。

最後，在筆者與該專案開發團隊的努力下，成功找出一勞永逸的方案，並僅花四小時的停機時間進行資料轉換與搬移。詳細過程與解決方案，筆者將會在今年四月底舉辦的 [Ruby X Elixir Conf Taiwan 2018](https://2018.rubyconf.tw/program#liang-chi-tseng) 中分享這些細節，歡迎各位讀者共襄盛舉！


