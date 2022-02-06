## 升版動機

想要 [`graphql-ruby 1.13.0`](https://github.com/rmosolgo/graphql-ruby/blob/master/CHANGELOG.md#1130-24-november-2021) 的這個功能：

> Visibility: A schema may contain multiple members with the same name. For each name, GraphQL-Ruby will use the one that returns true for .visible?(context) for each query (and raise an error if multiple objects with the same name are visible). rmosolgo/graphql-ruby#3651 , rmosolgo/graphql-ruby#3716 , rmosolgo/graphql-ruby#3725

閱讀 [#3651][1]，作者原先想像的應用場景應該是可以不必 API versioning 就對 legacy client 和新 client 提供不同的 schema，例如想把 id: Int! 改成 id: ID! 未來只需要提供兩種不同的 Field 實作，在 runtime 被判斷為 visible? 的那一個（也必須是唯一一個）Field 實作才會被拿來用

筆者專案中的應用場景比較不一樣，因為 Hahow for Business 是一個有相對複雜的角色權限規則的 SaaS 產品，在 GraphQL schema 上同一個 field 對於不同權限的 user 會有大大小小不同的行為差異。 像這次遇到的具體例子是「課程管理後台」相關的 API 對平台的講師、企業內部的講師、企業的 HR 分別共有三種不同的 authorization 規則和 query filtering 規則，雖然這兩件事都可以（也確實有）被抽象到不是 API 層的地方去所以都是 one liner，但 N 種角色就會有 N 個 one liner，還是覺得不太好：

  case current_account
  when 平台講師
    # authorize! 平台講師 管理 我開的平台課程
  when 企業講師
    # authorize! 企業講師 管理 我開的企業課程
  when 企業HR
    # authorize! HR 管理 本公司的所有企業課程
  end

而且只要有 M 種行為要支援，就會有 M 個重複的 switch statement:

  courses = case current_account
            when 平台講師
              # 撈出 平台講師 他本人開的平台課程
            when 企業講師
              # 撈出 企業講師 他本人開的企業課程
            when 企業HR
              # 撈出 企業 HR 他公司的所有企業課程
            end

如果要找個名詞來佐證對這種重複 switch statement 的害怕，或許可以說它不符合 OCP 說的 open for extension, close to modification：如果要支援第四種角色，就必須 open 這個 clase 去所有地方修改 switch statement 才能 modification，我們都清楚加新功能如果要修改到舊 code 通常都會比較辛苦。

這種重複的 switch statement 幾乎就是教科書範例 [Refacoring 的 Replace Type Code with Subclasses][3]，之前也在幾個不同場景嘗試應用過這個技巧，效果都不錯，雖然拆出來的 subclasses 每個都短短的，但一方面符合 OCP 了，而且各自的 unit test 就可以分開成不同檔案各別撰寫，對於我這種生產力跟 file line count 成平方反比的人十分有幫助。

然而，這次想再 graphql-ruby 的 resolver 應用 Replace Type Code with Subclasses 發現沒有之前的情況那麼容易，因為 resolver 具體要拿哪個 class 來用並不是我們的 application code 能決定的，而是 graphql-ruby 的實作在 runtime 選擇的。

根據我實測的經驗在 1.30.0 以前的行為就只是簡單地選最後一個定義的 Field 來用，例如：

  class Types::ArticleType < Types::BaseObject
    field :id, Int
    field :id, ID
    field :title, String
  end

上面的 code 在 1.30.0 以前並不會有錯誤，似乎只會簡單地以最後一個定義的 id: ID 來用，id: Int 就直接被忽略了

那麼 1.30.0 做了什麼改變呢？ 同樣的 code 現在會發生類似下面的 crash:

  GraphQL::Schema::DuplicateNamesError:
    Found two visible definitions for `Article.id`: #<Types::BaseField Article.id: ID!>, #<Types::BaseField Article.id: ID!>

如果重複定同名 field，代表你想要在 runtime 時透過 visible? 來選其中一個 Field 拿來用，如果選出了超過一個就會 DuplicatedNamesError

回到原本的例子，我們正是想利用這個新行為讓同名 field 有不同的實作，雖然跟 #3601 提出的場景（同名 field 的 type 不同、實作不同）不一樣（同名 field 的 type 相同、實作不同），但這 1.30.0 的新行為似乎是我目前找到唯一可以實現動態選擇不同 resolver 實作的唯一方法



## graphql-ruby 的版號慣例與建議的升版策略

觀察 [graphql-ruby] 的 [CHANGELOG.md]，patch 版號（x.y.x 的 y）更新時通常就會有 breaking change，因此筆者這次 1.26.x -> 1.30.6 就需要在 1.30.0 處理幾個 breaking change

我這次採取的策略是分成 1.26.2 -> 1.26.6、1.26.6 -> 1.30.0、1.30.6 (latest) 共三個 commit 分別跑 CI，心得是或許再分得更細會比較好，也許每個 patch 版號都跑一個 CI 會是更好的策略，因為後來也有發現 patch 版號升級導致的 bug，不只是預期的 1.30.0 才有 breaking change。 因為我們的專案有在這部分有蠻完整的 test coverage，這種一次升級好幾的版本的時候就能發會不少價值，多跑幾次 CI 的成本肯定是比人力時間成本便宜的

## 1. [Improve default complexity calculation for connection fields #3609][2]

這是我認為最難處理的一個 breaking change，簡言之 #3609 幫 connection 也加上了預設的 complexity ，簡單瀏覽 [File Changed]



[3]: https://refactoring.com/catalog/replaceTypeCodeWithSubclasses.html
