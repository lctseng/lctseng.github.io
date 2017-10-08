---
layout: post
date:   2016-05-23 13:47:00 +0800
category: experience
tags: [5xruby, web-develop, course]
title: Ruby on Rails 從零開始
description: 這是我在五倍紅寶石實習時參與上課的心得，在這一系列的課程的最後也是最長的主題：Ruby on Rails。除了課程心得，這篇文章中還附上了上課時的筆記內容。
---


### 心得

五倍紅寶石最後一階段也是最精彩的實習課程 "RUBY ON RAILS 從零開始" 也已經告一段落了，
雖然時間短暫，但我從Eddie老師那裏學到非常多我過去所不了解的Ruby的細節，
以及Rails中許多好用的套件與技巧。

Eddie老師一開始就帶大家從建立一個簡單的Rails應用程式開始，
讓大大看看Rails的強大以及如何快速的建立一個簡單的網頁應用程式。
之後帶領大家接觸Ruby程式語言，並著重在與Rails關聯性高的Ruby語法。

在介紹完Ruby語法之後，接著就是從頭開始手刻Rails應用程式的每一部分，
包括在Rails中使用MVC架構、使用Mailer寄信、背景工作等等。
在課程的最後，也帶領大家實作一個具有FB登入以及刷卡付款的購物車功能。
很謝謝Eddie老師帶來的精彩課程！


### 課程筆記

- ```scaffold```
  - 快速產生MVC架構
- ```collection_select```
  - 可以產生具有關聯性的選擇欄位
- N+1 Query
    - 有工具可檢驗
    - 假設post belongs to user (via ```user_id```)
    - ```Post.all.includes(:user)```
        - 連user欄位一起拿過來
- 資料驗證寫在model
- Symbol GC
    - Ruby 2.2之前，產生出來的symbol無法被GC回收，會memory leak
- Range展開成陣列
    - ```[*1..100]```
    - ```*```稱為splat operator
- ```each.with_index```
    - 類似```each_with_index```
    - 好像也可以套在hash
- ```reduce(:+)```
   - ```redice(0) {|x,y| x+y}```
- local variable與method的優先？
    - local variable優先
    
    ```ruby
age = 18
def age
    return 20
end
puts age #=> 18
puts age() #=> 20
    ```

- 寫block時，```{}```的優先順序比```do-end```還要高
    - function call時，```{}```優先比較高
    - ```p [1,2,3,4,5].map {|x| x*2}```
        - 相當於 ```p ([1,2,3,4,5].map {|x| x*2})```
    - ```p [1,2,3,4,5].map do |x| x*2 end```
        - 相當於 ```(p [1,2,3,4,5].map ) do |x| x*2 end```
- receiver的概念
  - 範例

  ```ruby
5.times do ...
end
  ```

    - 5是receiver，對該物件送```times```這個訊息，看它是否回應
- 為什麼類別方法用```self.xxx```來定義？
    - 可以想像為：在Class本身中定義單體方法(singleton method)
- 使用```class << self```來省略定義類別方法時的self
- ```include```的問題
    - 模組```include```時，會類似變成該class的super class
    - 因此
    
    ```ruby
module A
    def foo
	puts "module"
    end
end
class B
    include A
    def foo
	puts "class"
    end
    # include A # => 放這裡也是一樣的結果!
end
B.new.foo #=> "class"
    ```

- superclass並不會列出所```include```的module
- 大部分的Ruby method都是定義在```Kernel```，而非```Object```或```BasicObject```
- ```ClassName.class #=> Class```
    - ```ClassName = Class.new```
    - ```another_name = Class.new #=> 建立新類別，名稱也可以是中文```
    - ```小貓 = Class.new```
- ```Class.superclass #=> Module```
    - ```Module.superclass #=> Object```
    - 但```Module.class #=> Class```
        - 所有class的```class```方法都是```Class```，包括```Class```與```Module```
    - 詳細reference：Ruby Object Model
-動態的繼承
    - 根據運算式結果決定superclass
    
    ```ruby
class Cat < (rand > 0.2) ? Array : Hash
end
    ```
    
- ```extend```
    - 用來擴充呼叫者，類別或者物件實體會有不同結果
- ```extend``` 放在類別
    
    ```ruby 
module Flyable
    def fly
    end
end
class Cat
    extend Flyable
end
Cat.fly # fly成為類別方法
    ```

- ```extend``` 使用在物件上

    ```ruby
kitten = Cat.new
kitten.extend Flyable
kitten.fly #=> 成為實例方法
    ``` 

- 事實上rails下app資料夾內的所有東西都會被載入
    - 分資料夾只是識別
- Ruby的單複數轉換
    - ```"person".pluralize```
    - ```"mice".singularize```
    - 是透過語尾等結構來判斷，即使是亂打的單字也可以轉換
    - 自定義單複數轉換
        - 修改```initializers/inflections.rb```
- ```camelcase```與```underscore```
    - ```"PostsController".underscore #=> "posts_controller"```
    - 反向為```camelcase```
- SCSS
    - 模組使用
        - ```@import 'reset'```
- Coffeescript
    - 可使用class (轉成JS的prototype)
- 分頁
    - ```kaminari```
    - 美化：```bootstrap-kaminari-views```
- Simple form
    - ```f.input```
    - ```f.assocication```
- ```carrierwave```：檔案上傳
- migration
    - ```rails g migration add_aaa_to_bbb aaa:string```
    - 自動解析，知道要新增到bbb這個table
- 縮圖
    - RMagick
    - 可保留原圖並製造縮圖
- 讓Heroke做migrate等指令
    - ```heroku run ...```
- ```link_to```、```redirect_to```可以猜測使用者的目的
    - ```link_to "Title", article_path(article)```
    - ```link_to "Title", article```
    - ```redirect_to @user```
- ```redirect_to```可以帶入flash message
    - ```redirect_to xxx_path, :notice: "Error Message" ```
- ```render :partial => "form"```
    - 可簡寫為：```render "form"```
- 當```object.save```因為```validates```而回傳```false```時，可以用```object.errors.full_messages```看錯誤訊息
    - ```object.errors.any?```：查看是否有任何錯誤訊息
- ```create```中，為何要使用實例變數？
    - 因為可以```render```到```new``` page中，成為既有欄位
    - 與```redirect_to```不同，因為```redirect_to```是轉址，```render```只是產生別的view
    
    ```ruby
def create
    @obj = ObjectClass.new(obj_params)
    if @obj.save
	# ...
    else
	render action: :new
    end 
end
    ```

- ```form_for @object```的方式會檢查該物件是不是過去```save```失敗，而多增加警示框等CSS或其他HTML
- 自己顯示錯誤：用```@object.errors.full_messages.each {...}``` 的方式條列出來
- 現在大部分的瀏覽器都支援檔案的解壓縮，所以可以讓asset用gz等壓縮傳給瀏覽器
- Asset中js和css的檔案名稱有接一串隨機碼，用意是在更新CSS/JS時讓web server被迫放掉cache重抓
- 用CDN的缺點：如果自己的asset有depend其他的CDN，萬一CDN速度比較慢，導致讀取順序是自己的code先到時，就可能壞掉
    - 而使用asset pipeline會整包asset一起到，就沒這問題
- Ruby的Private, Public, Public
    - 與子類別一點關係也沒有，子類別總是可以拿到所有父類別的方法，無論```private```、```protected```
    - Private method：不能有明確receiver
        - ```self.private_method``` 這樣是不行的，因為```self```是receiver
        - 並非像其他的程式語言中，```private```只能由內部class呼叫
        - example：```puts```其實是一個private method，因為沒有receiver
        - 甚至```public```, ```protected```, ```private```這三個本身也是private method
        - 強制呼叫Ruby的private method：可用```object.send(:private_method)```
        - ```object.send(:send, :send, :send, :send, .... , :private_method) # 同單一次呼叫```
    - Protected method：限自己與sub-class可使用receiver
- 為何需要使用```binstub```？
    - 因為系統版本與專案內的binary的版本可能不同
    - 如果直接下指令(```rails```, ```rake```等)，系統會自動去找最新版本
        - 但現今版本的```rails```, ```bundle```等程式會根據是否在專案目錄，自動挑選```rails```的版本
        - 其實就是優先使用bin資料夾內的特製版本
    - 過去用```bundle exec```，會去檢查```Gemfile.lock```的內容，挑選系統中合適的版本
        - ```Gemfile.lock是```由```bundle install```根據```Gemfile```安裝後產生的套件相依性紀錄檔
    - 使用binstub：會把該專案內需要的特殊版本產生在bin內(特殊的script)
        - 使得```rails, bundle```等指令不用再加```bundle exec````
        - 能夠自動判斷
- Rails中讓controller不要產生一堆有的沒的CSS和JS
    - in ```config/application.rb````
    
    ```ruby
class Application < Rails::Application
    config.generators do |g|
	g.stylesheets       false
	g.javascripts       false
	g.test_framework    false
	g.helper            false
    end
end
    ```

- 避免```/articles/4/comments/3```這種串很長的網址
    - 因為ID已經有專一性了，不用在comment前串一個article
- Session用的名稱```session["name"]```可以抽離到```Settings.card_name```，方便管理
    - 建議也使用比較複雜的名字，避免被撞到或攔截
- ```settingslogic```
    - 用yml來管理設定檔
- ```aasm```或```state_mechine````
    - 提供有限狀態機的功能
- 刷卡機制
    - BrainTree
