---
marp: true
title: Ectoでシャーディング
theme: gaia
class: invert
---

<!-- _class: lead -->

# Ectoでシャーディング

DC 開発T @ryo.konishi

---

## Ecto とは

- Elixir製ORM
    - ErlangVM(BEAM上で動く)
- Rubyでいうと Arel + ActiveModel
- データベースっぽいものとのやり取り

---

## Ecto: 例

##### 1件取得
```elixir
User
|> where(id: 10)
|> MyRepo.one()
```

##### 全件取得
```elixir
User
|> where([u], u.name in ["Tom", "Jerry"])
|> MyRepo.all()
```

---

## Repo / Schema / Query
### Repo
- 接続先DBを指定する
    - アダプター (Postgres, MySQL, MongoDB, ...)
    - url, database (`"foo_production"`)

    ```elixir
    use Ecto.Repo,
        otp_app: :myapp,
        adapter: Ecto.Adapters.MyXQL

    config :testplay_datapicker, MyRepo,
        database: "my_database_production"
    ```

---

## Repo / Schema / Query
### Schema
- データ型の定義をする
    ```elixir
    schema "users" do
      field :name, :string
      field :age,  :integer
    end
    ```
- JSONなどの入力を受けて、要素を指定したElixirの型に変換する
- DBテーブルとは疎結合
    - 同じテーブルに対して複数Schemaの設定も可能

---

## Repo / Schema / Query
### Query
- SQLを作成する関数群
    - 叩くのはRepo
    ```elixir
    import Ecto.Query, only: [from: 2]

    query = from u in UserSchema,
            where: u.age > 18,
            select: u.name
    ```

---

<!--- _class: lead -->
## いざシャーディング

---

<!--- _class: lead invert -->
### シャーディングされたDBとのやり取り

→ Repoを複数用意してあげればいい

---

## シャーディングされたDBへの接続
1. Repoを複数用意する
```elixir
defmodule MyRepo.Shard001 do
  use Ecto.Repo,
    otp_app: :myapp,
    adapter: Ecto.Adapters.MyXQL
end

defmodule MyRepo.Shard002 do
  use Ecto.Repo,
    otp_app: :myapp,
    adapter: Ecto.Adapters.MyXQL
end
```

---

## シャーディングされたDBへの接続
2. それぞれの接続を設定する
```elixir
config :testplay_datapicker, MyRepo.Shard001,
  database: "my_database_production_shard_001",
  hostname: "shard_server.foo",
  username: "...",
  ...

config :testplay_datapicker, MyRepo.Shard002,
  database: "my_database_production_shard_002",
  hostname: "shard_server.bar",
  username: "...",
  ...
```

--- 

<!--- _class: lead invert -->

### 完了:tada:

---

## シャーディングされたDBの利用
1. `id: 1` なUserを取得したい

```elixir
User
|> where(id: 1)
|> MyRepo.Shard001.one!()
```

---

## シャーディングされたDBの利用
2. `age: 10` なUserを全件取得したい

```elixir
[MyRepo.Shard001, MyRepo.Shard002]
|> Enum.flat_map(fn repo ->
  User
  |> where(age: 10)
  |> repo.Shard001.all!()
end)
```

`Repo.all/2` はデータの入ったただのListを返すので、単純に`flat_map`してあげても型の整合性が取れる

---

<!--- _class: lead invert -->

### ちょっとイケてない

---

## 共通化でまとめる

- 各Repoに同じ関数を定義する
- 関数名と引数を指定してすべてのRepoに呼び出しをかける


---

## 共通化でまとめる: 全Repoに関数を定義

- Behaviour(コンパイル時にチェックされる振る舞い)を定義
  ```elixir
  @callback list_users(query) :: [User]
  ```
  - 実装は各Repoでする
- `__using__/1` マクロで共通化
  - `include` みたいなもの
  - 実装は一度

---

## 共通化でまとめる: 呼び出し

関数名と引数ですべてのRepoを呼び出して結果をマージするマクロを作成

```elixir
defmacrop to_all_shards(funcname, arguments, sort_by \\ :updated_at) do
  quote do
    @shards
    |> Enum.flat_map(&apply(&1, unquote(funcname), unquote(arguments)))
    |> Enum.sort_by(&(&1 |> Map.get(unquote(sort_by))))
  end
end

# 呼び出し
to_all_shards(:list_users, [], :created_by)
```

---

## 共通化でまとめる: 結果

```elixir
# 具象Repo
defmodule User.Repo.Shard001 do
  @behaviour User.Repo

  def list_users(args), do: ...
end

defmodule User.Repo do
  @shards [User.Repo.Shard001, ...]

  def list_users(args), do: to_all_shards(:list_users, args, :id)
end
```

---

<!--- _class: lead invert -->

### まだイケてない…

---

## イケてないポイント
- シャーディングするレイヤーが高すぎる
- 関数定義のたびに振る舞いを更新しなくてはならない

### ⇒ Ecto.Repoの振る舞いを模倣したい

---

## Ecto.Repoの振る舞いを模倣する

### Ecto.Repo
実は関数定義と同時に __Behaviourの定義もある__
→ Ecto.Repoのように振る舞うものを作ることが想定されている？

- 必要な情報を渡してよしなにシャードされたクエリを叩きたい
  - 対象のRepo群
  - シャードキー
  - 対象Repoを選ぶためのシャーディング関数
  - クエリ自体

---

## Ecto.Repoの振る舞いを模倣する

### シャードキー
- Userのデータだったら `:id`、UserPostだったら `:user_id`
- Schemaに定義するのはイケてない
  - Schemaは使い回す可能性がある
  - あまりDBと密結合にしたくない

---

## Ecto.Repoの振る舞いを模倣する
### シャーディング関数
- シャード対象の値から対象のRepoを導出する関数
  - 「Repoの数で割った余りで決める」みたいなやつ
    - `fn i -> rem(i, repo_count) end`
  - 剰余法以外にもやり方はある
- 抽象Repoあたり１つあればOKなはず

---

## 模倣したRepoの使い方想定: 導入

```elixir
defmodule AllUserRepo do
  @repos [Repo.User.Shard001, Repo.User.Shard002]

  def shard_fun(key), do: rem(key, length(@shards))

  # 対象のRepo群と評価関数を渡す
  use ShardRepo,
    shard_repos: @repos,
    shard_function: &__MODULE__.shard_fun/1
end
```

---

## 模倣したRepoの使い方想定: 利用

```elixir
UserPost
|> where(user_id: 1)
|> limit(10)
|> AllUserRepo.all(shard_key: :user_id)
# => [%UserPost{...}, %UserPost{...}]
```

- クエリを呼ぶときにそのクエリで使うシャードキーを宣言する
  - `shard_key: :user_id`

→ `:shard_key` に対応する値をクエリ自体から取ってきてほしい

---

## Ecto.Queryable

`Repo.all()`をする前に呼び出しを止めると…？

```elixir
UserPost
|> where(user_id: 1)
|> limit(10)
# => #Ecto.Query<from u0 in UserPost, where: u0.user_id == 1, limit: 1>
```

- Ecto.Query型の構造体 = ただの連想配列
  - クエリに関するデータはすべてここに入っている
  - 読める…読めるぞ…！

---

### `user_id`を読む

```elixir
sharding_key = :user_id

# パターンマッチ
%{expr: {_, _, [_, %{type: {_, ^sharding_key}, value: val}]}} = query

IO.puts val
# => 1
```

---

## 結果

- `all/2` にシャードキーが渡されている場合、クエリが効率化！
  - データの存在し得ないデータベース → アクセスなし
- 渡されていなければ全検索できる

- 同じことを書き込みでも行える！

---

## 実装してます:grey_exclamation:

### EctoShardRepo
- https://github.com/ndac-todoroki/ecto_shard_repo
- とりあえず動くくらいです
- 実装しているもの:  
  `all/2` `delete/2` `delete_all/2` `exists?/2` `get/2` などなど
  - おおよそ使うもの

---

## 勉強会してます

- 「Elixir勉強会しませんか」
  - 同期と数人でElixir勉強する会
  - 最近の内容
    - DDD
    - `Module.__info__/1` の定義追跡

いまのところ非公開ですが、成果をDocbaseなどで公開しようと思っています

---

<!--- _class: lead -->

## ありがとうございました！
