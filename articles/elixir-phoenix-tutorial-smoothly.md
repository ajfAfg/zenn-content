---
title: "Phoenix のチュートリアルでめっちゃエラー出る"
emoji: "🐣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["elixir", "phoenix"]
published: true
---

タイトルの通り，Phoenix 公式の[チュートリアル](https://hexdocs.pm/phoenix/up_and_running.html)を進めていたらエラーが出て困ったので，その原因と解決方法をまとめます．

この記事が Phoenix のチュートリアルをサクッと試してみる助けとなってくれたら嬉しいです．

# 対象読者

Phoenix 公式のチュートリアルを進めようと思っている方，進めている方．

# 前提条件

[こちらのドキュメント](https://hexdocs.pm/phoenix/installation.html)に従い，チュートリアルを進める上で必要な各ソフトウェアのインストールは完了しているものとします．

# 環境

- OS
  - macOS Big Sur 11.5.2 (Intel Core)
- Elixir
  - 1.12.0
- Erlang/OTP
  - 24
- Phoenix
  - 1.5.12
- Node.js
  - 16.7.0
- npm
  - 7.20.3
- PostgreSQL
  - 13.4

# エラーが出たコマンド

- `npm install`
- `mix ecto.create`

## `npm install`

`mix phx.new hello` を実行した後，「Fetch and install dependencies? [Yn]」 と聞かれます．
これに Y と答えると実行される最後のステップの中に `npm install` があります．

```bash
$ mix phx.new hello
...省略...

Fetch and install dependencies? [Yn] Y
* running mix deps.get
* running mix deps.compile
* running cd assets && npm install && node node_modules/webpack/bin/webpack.js --mode development
```

ここでは特にエラーメッセージが表示されないのですが，続く出力を見てみると......

```bash
We are almost there! The following steps are missing:

    $ cd hello
    $ cd assets && npm install && node node_modules/webpack/bin/webpack.js --mode development

...省略...
```

実行されたはずのステップが不足していると言ってますね．妙だな......

原因の特定は割と簡単で，先ほどのステップをもう一度実行してみると，今度は大量のエラーメッセージが表示されて `npm install` でコケているのがわかります．

```bash
$ cd assets && npm install && node node_modules/webpack/bin/webpack.js --mode development
npm WARN deprecated urix@0.1.0: Please see https://github.com/lydell/urix#deprecated
npm WARN deprecated har-validator@5.1.5: this library is no longer supported
npm WARN deprecated resolve-url@0.2.1: https://github.com/lydell/resolve-url#deprecated
...省略...
```

幸いなことに，この問題は[こちらの issue ](https://github.com/phoenixframework/phoenix/issues/4359#issuecomment-873133536)で議論されていて，解決に至っていました．
どうやらエラーの原因はパッケージ間の依存関係にあるようです．`assets` ディレクトリにある `package.json` 内の `node-sass` を `sass` に書き換え，そのバージョンを `1.35.1` とすると解決できます．
具体的には，以下の部分を

```json
"node-sass": "^4.13.1"
```

次のように書き換えます．

```json
"sass": "^1.35.1"
```

## `mix ecto.create`

データベースを作ろうとこのコマンドを叩いてみると，GenServer が突然死んでしまいます．

```bash
$ mix ecto.create
Compiling 14 files (.ex)
Generated hello app

23:18:44.852 [error] GenServer #PID<0.379.0> terminating
** (DBConnection.ConnectionError) tcp connect (localhost:5432): connection refused - :econnrefused
    (db_connection 2.4.0) lib/db_connection/connection.ex:100: DBConnection.Connection.connect/2
    (connection 1.1.0) lib/connection.ex:622: Connection.enter_connect/5
    (stdlib 3.15) proc_lib.erl:226: :proc_lib.init_p_do_apply/3
Last message: nil
State: Postgrex.Protocol

...省略...
```

このエラーの原因は PostgreSQL を起動していないことでした．~~だって書いてなかったから......~~

Homebrew で PostgreSQL をインストールした私は，[こちらの記事](https://qiita.com/domodomodomo/items/12fe7555513de6b078db)を参考にして以下のように起動しました．

```bash
brew services start postgresql
```

# やったか！？

これらのエラーに対処して，プロジェクトのトップディレクトリから `mix phx.server` 叩いて，ブラウザから http://localhost:4000 を開いたら......

![](/images/elixir-phoenix-tutorial-smoothly/success.png)

やった！！！！！！！！！！！
