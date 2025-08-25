---
title: "GitHub Actions の定期実行ワークフローを「時間通り」に実行する"
emoji: "🕰️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["githubactions", "github", "cron", "構文木"]
published: false
publication_name: "cybozu_ept"
---

:::message
この記事は、[CYBOZU SUMMER BLOG FES '25](https://cybozu.github.io/summer-blog-fes-2025/)の記事です。
:::

`schedule` イベントを用いた、GitHub Actions のワークフローの定期実行は時間通りに実行されないがちです。この問題を解決するため、ワークフローファイルにアノテーションという形で cron 式を記述すると、自動的にその cron 式が示す時間に `workflow_dispatch` する社内システム兼 GitHub App を作りました。**cronium**（/krɒnɪəm/）と呼んでいます。Chromium じゃないよ。

本稿の構成は次の通りです。まず、cronium を作るに至った背景を説明し、問題を解決するアイディアを紹介します。次に、ユーザーはどのような体験を得られるのかを示すため、cronium の使い方を簡単に説明します。そして、cronium がどのように作られているのか、どうしてそのように作ったのかを解説し、最後に今後の展望を述べて本稿を閉じます。

## 背景

GitHub Actions のワークフローは、[`schedule` というイベントを利用して定期実行できます](https://docs.github.com/ja/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule)。例えば以下で示すワークフローは、毎朝 8 時に「Hello, World!」と標準出力します。なお、UTC です。

```yaml
name: デモ

on:
  schedule:
    - cron: "0 8 * * *"

jobs:
  demo:
    runs-on: ubuntu-latest
    steps:
      - run: echo 'Hello, World!'
```

とても便利な GitHub Actions の定期実行ワークフローですが、時間通りに実行されないことが多々あります。[この挙動は公式ドキュメントにも記述されています](https://docs.github.com/ja/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule)が、遅延時間が 1 時間を超える場合もあり、たびたび困ってしまいます。特に、定期実行ワークフローの実行を監視している場合は、誤ってアラートが鳴ることもあります。もちろん、time-sensitive な処理は GitHub Actions で動かさない手もありますが、GitHub Actions に関する資産を使いたい場面もままあります。

この問題を解決するため、「[Github Actions を定刻に実行する方法](https://zenn.dev/no4_dev/articles/14b295b8dafbfd)」という記事では、[cron-job](https://cron-job.org/en/) というサービスを利用し、ワークフローを定期的に [`repository_dispatch`](https://docs.github.com/ja/actions/reference/workflows-and-actions/events-that-trigger-workflows#repository_dispatch) する方法が紹介されています。しかし、この方法では定期実行する時間を cron-job に登録するため、ワークフローファイルに cron 式を記述する GitHub Actions 標準の方法と大きく開発者体験が異なります。また、cron-job を業務で利用する場合はチームでの利用が大半だと思われるので、退職者管理や属人性の低い運用整備の必要性も出てきます。

## アイディア

`schedule` イベントを用いる方法と似た体験を得るためには、ワークフローファイル中に cron 式を記述できるとよさそうです。ただし、ナイーブに `on.schedule[].cron` で cron 式を書くと `schedule` イベントが発火してしまうので、別の書き方が望ましいです。そこで、[ESLint ではコメントを用いて部分的にルールを制御できる](https://eslint.org/docs/latest/use/configure/rules#using-configuration-comments)のと同じノリで、あるルールに従うコメントをアノテーションとして利用し、そこに cron 式を記述するとよいのではないかと考えました。

あとは、GitHub App がインストールされているリポジトリからワークフローファイルを拾って、その具象構文木をガチャガチャして cron 式を抽出し、`repository_dispatch` なり [`workflow_dispatch`](https://docs.github.com/ja/actions/reference/workflows-and-actions/events-that-trigger-workflows#workflow_dispatch) なりでワークフローを実行すればよさそうです。

## cronium とは

先述のアイディアを実現した社内システム兼 GitHub App が cronium です。本節では、cronium の使い方の説明を通して、ユーザーはどのような開発者体験が得られるのか示します。また、cronium のシステム概要を述べます。

### cronium の使い方

#### 1. リポジトリに cronium をインストール

cronium の GitHub App をリポジトリにインストールします（c.f. [独自の GitHub App のインストール - GitHub Docs](https://docs.github.com/ja/apps/using-github-apps/installing-your-own-github-app)）。なお、この操作は各リポジトリに対して一度だけ行えばよいです。

#### 2. ワークフローファイルにアノテーションを記述

cronium で定期実行したいワークフローファイルにおける `on.workflow_dispatch` の上部に、`# cronium: '* * * * *'` という形式のコメントを記述します。以下に示す例の場合、cronium は毎朝 8 時と 10 分おきにワークフローを実行します:

```yaml
name: demo
on:
  # cronium: "0 8 0 0 *"
  # cronium: '*/10 * * * *'
  workflow_dispatch:
```

これだけです。`schedule` イベントを用いる方法と遜色ないのではないでしょうか。

### cronium のシステム概要

cronium は単一のサーバーと GitHub App から成ります。サーバーは GitHub App がインストールされているリポジトリを 3 分おきにポーリングし、ワークフローファイルを収集します。そして、各ワークフローファイルからアノテーションを抽出し、時間が来たら `workflow_dispatch` する cron をサーバー上に作ります[^reason-for-using-workflow-dispatch]。なお、すでに cron がサーバー上に存在する場合は何もせず、アノテーションが消された場合は対応する cron を削除します。Kubernetes における Reconciliation Loop と同じメンタルモデルです。GitHub App の利用目的は、[cronium がサポートすべきリポジトリ一覧の取得](https://docs.github.com/ja/rest/apps/installations?apiVersion=2022-11-28#list-repositories-accessible-to-the-app-installation)と、GitHub API を叩くためのトークン発行です。以下にシステム構成図を示します:

[^reason-for-using-workflow-dispatch]: `repository_dispatch` ではなく `workflow_dispatch` を使う理由はぶっちゃけありません。（バイブス採用でした。）ただし、`workflow_dispatch` はワークフローファイルに記述されていがちなので、イベントを使い回せる嬉しさはあるのかなとは思います。

![cronium のシステム概要図](/images/run-github-actions-scheduled-workflows-on-time/system-structure.drawio.png)

<!-- TODO: 図を書く。Reconcile まで含めて書く。 -->

cronium は社内 Kubernetes クラスタである [Neco](https://blog.cybozu.io/entry/2025/03/19/112000) にデプロイしています[^reason-for-using-neco]。詳細は後述しますが、cronium は外部ストレージを用いず単一のサーバーで cron を管理する設計なので、Pod の数は 1 つだけです。

[^reason-for-using-neco]: 生産性向上チームではインフラとして AWS を利用することが多いですが、今回は学習目的で Neco を使いました。

## cronium の作り方

cronium の主な仕事は以下の 2 つです。本節では、これらの仕事がどのように実装されているのか深掘りします。

- ワークフローファイルからのアノテーション抽出
- cron の Reconcile

### ワークフローファイルからのアノテーション抽出

アノテーションとは、ワークフローファイルにおける `on.workflow_dispatch` の上部に記述された `# cronium: '* * * * *'` という形式のコメントでした。以下に例を再掲します:

```yaml
name: demo
on:
  # cronium: "0 8 0 0 *"
  # cronium: '*/10 * * * *'
  workflow_dispatch:
```

アノテーションの詳しい仕様は以下の通りです:

- `# cronium: "<cron_expression>"` という形式。
- `on.workflow_dispatch` の上部に位置する。
- 0 個以上記述可能。
- cron 式を囲む文字は単一引用符（`'`）か二重引用符（`"`）のいずれか。

アノテーションはコメントで表現されるので、ワークフローファイルの具象構文木[^cst]を用いてアノテーションを抽出します。cronium は Deno 製なので、今回は [`eemeli/yaml`](https://github.com/eemeli/yaml) というライブラリを用いてワークフローファイルを具象構文木に変換しています[^reason-for-using-npm-yaml]。

[^cst]: 具象構文木（concrete syntax tree）とは、ソースコードと一対一に対応する木です。具象構文木は元のソースコードに変換可能なので、もちろんコメントも木の中に存在します。なお、抽象構文木（abstract syntax tree）の中には一般にコメントが存在しません。
[^reason-for-using-npm-yaml]: [Deno は標準の YAML パーサーを備えています](https://docs.deno.com/examples/parsing_serializing_yaml/)が、これは抽象構文木のみ扱うので採用していません。

あとは具象構文木を気合いで再帰的に走査して、`on.workflow_dispatch` の上部にあるコメントを取り出し、各コメントに対して正規表現などで cron 式を抽出すればよいです。仕様に従わないコメントは単に無視します。
ハマりポイントとしては、`eemeli/yaml` における YAML の具象構文木では、アノテーションが `on.workflow_dispatch` の上部に来たり `on` の下部に来たりする点です。例えば、以下のワークフローファイルの場合、アノテーションは `on.workflow_dispatch` の上部に来ます:

```yaml
on:
  push:
    branches:
      - main
  # cronium: '* * * * *'
  workflow_dispatch:
```

一方で、以下の場合、アノテーションは `on` の下部に来ます:

```yaml
on:
  # cronium: '* * * * *'
  workflow_dispatch:
```

このような仕様になっているため、アノテーションを抽出する際は、`on.workflow_dispatch` の上部と `on` の下部の両方を見る必要があります。

### cron の Reconcile

cron の Reconcile のロジックを考える前に、cron の作り方と管理方法を考える必要があります。cron の作り方に関して、cronium では [`Hexagon/croner`](https://github.com/Hexagon/croner) というライブラリを利用して、Deno のランタイム上に cron を作っています[^reson-for-using-croner]。このライブラリで作る cron は停止可能です。cron の管理方法に関して、キーとしてオーナー名・リポジトリ名・ワークフロー名・cron 式の 4 つ組を持ち、バリューとして cron を持つ、グローバルなマップで管理します。外部ストレージを使っていない理由は、Reconcile するのは単一のサーバーであり、かつ、全ての情報は GitHub リポジトリに存在するので、データは複数の Pod で共有しなくていいし揮発してもいいと考えたためです。

[^reson-for-using-croner]: [Deno は標準の cron ライブラリを備えています](https://docs.deno.com/deploy/kv/manual/cron/)が、これは cron をプログラム実行中に停止できないので採用していません。

以上の準備を踏まえて、Reconcile のロジックはキーに基づく集合演算によって差分を計算して cron を作成したり停止したりするだけです。具体的には、Actual State のキーの集合を $S$ とし、Desired State のキーの集合を $S'$ とすると、作成すべき cron のキーは $S' \setminus S$ で求められ、停止すべき cron のキーは $S \setminus S'$ で求められます。
注意点としては、JavaScript のオブジェクトに対する比較演算子は物理的等価性[^what-is-physical-equality]を見るので、[JavaScript 標準の集合ライブラリ](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Set)ではオブジェクトの集合演算が期待通りに動かないがちな点があります。この問題の対処方法はいくつか考えられますが、今回は [`immutable-js/immutable-js`](https://github.com/immutable-js/immutable-js) というライブラリを用いて作ったオブジェクトで集合演算しています。

[^what-is-physical-equality]: 物理的等価性（physical equality）とは、データが存在するアドレスに基づく等しさです。例えば、`const foo = {x:1,y:1}; const bar = {x:1,y:1};` というプログラムがあったとして、`foo === foo` と `bar === bar` は成り立ちますが、`foo === bar` は成り立ちません。これはそれぞれのデータが存在するアドレスが異なるためです。

## cronium の設計意図、あるいは想定質問

ここまで述べてきた通り、cronium は単一のサーバーにおいて、GitHub リポジトリをポーリングしたり、サーバー上に作った cron から `workflow_dispatch` する設計にしています。本節では、どうしてこのように作ったのか、質問に答える形で解説します。

### Kubernetes の CronJob を動的に作るのはダメ？

つまり、自作 Operator で CronJob を Reconcile すればいいんじゃ？というアイディアです。

この方法は、CronJob は cron 式が示す時間になるとジョブを実行する Pod を作成する仕様なので、Neco における Pod 作成の rate limit に引っかかる可能性があったので断念しました。また、Operator を「正しく」実装するのは大変がちなので、今回のケースで持ち出すのはやりすぎなんじゃないかとも考えました。

### なぜ Webhook を使わずポーリングしてる？

主に 2 つ理由があります。1 つ目は、GitHub から Neco に通信するためには Neco に穴を開ける必要があるので、上長の承認を受ける必要があって大変だからです。2 つ目は、cron を作成・停止すべき一部のイベントが Webhook では検知できないからです。具体的には、cron は以下で列挙するタイミングで作成・停止すべきですが、ワークフローの enable/disable を Webhook で検知する方法は見つけられませんでした。

- GitHub App がインストール・アンインストールされたとき
- ワークフローファイルを追加・編集・削除するコミットが push されたとき
- ワークフローが enable/disable になったとき
- リポジトリが削除されたとき
- リポジトリがアーカイブ・アンアーカイブされたとき

### ポーリングの際、そのときの時間が cron 式の示す時間を超えてたら `workflow_dispatch` すればよくない？

この手法だとサーバー上に cron を作らなくてよくて嬉しいですが、GitHub App トークンの rate limit に引っかかる可能性があったので断念しました。
弊社のリポジトリは GitHub Enterprise Cloud organization に存在する[^ghes-exists]ので、[GitHub App トークンの rate limit は 15,000/h です](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api?apiVersion=2022-11-28#primary-rate-limit-for-github-app-installations)。cronium のウリは「時間通り」にワークフローを実行することなので、そうなるとポーリングの間隔は最も短い 1 分としたくなります。そのため、1 回のポーリングで叩けるリクエスト数は 250（= 15,000/60）回となります。1 回のポーリングで発生するリクエスト数は、サポートすべき全てのリポジトリの数を $x$ とし、サポートすべき全てのワークフローの数を $y$ とすると、$1 + x + y$ で表せます。なお、最初の 1 回はリポジトリ一覧を取得している分です。弊社には複数の organization が存在しますが、最も活発に利用されている organization のリポジトリ数は 745 個（2025/08/25 現在、アーカイブを除く）なので、 cronium の利用を広げるなら 250 回の limit はかなり厳しいです。

[^ghes-exists]: 歴史的経緯により、GitHub Enterprise Server に存在するリポジトリもあります。

と考えていたのですが、GraphQL API を駆使すればリクエスト数を削減できるかもな〜と本稿を書いていて気づきました。伸び代ですね。

### cronium を利用するリポジトリ、ワークフローの増加にシステムは耐えうる？

GitHub の rate limit の問題が解決したとして、cronium をスケールさせるのは簡単ではないと考えています。もちろんスケールアップは容易ですが、スケールアウトはサーバーが状態を持っているので容易ではないです。
ということで、今回は問題を先送りして解決しました。言い訳をすると、cronium が今後どれだけ使われるのかわからないので、問題が発生する前から着手するのはコスパが悪いと判断しました。また、cronium はモジュラーで簡単な作り方をしているので、作り直しの判断も取りやすいと考えています。

### ワークフローが消された直後に `workflow_dispatch` したらエラーになるといった、並行・並列処理にまつわる問題はどう対処してる？

そもそもこういった問題が発生しない作り方はできないかという問いに対しては No と言えて、GitHub API ではワークフロー取得と `workflow_dispatch` をアトミックに処理できないので対処は必要になります。
この問題に対しては、GitHub API のエラーを分類し、既知のエラーであれば許容し、そうでなければ CRITICAL でログを出す方針で対処しています。具体的には、「正しく」ポーリング・Reconcile できているとの前提のもとで、`workflow_dispatch` したワークフローが存在しなかったり disable になっていたりした場合は、直前に変更があったと判断して許容しています。なお、既知のエラーと表現しているのは、GitHub API には Undocumented なエラーがたくさんあるからです。

### なぜ cronium って名前？

バイブス。

## 今後の展望

主に以下の 2 つです:

1. Enterprise 向け GitHub App の利用
2. 監視の充実

1 つ目に関して、これまで弊社の誰でも cronium を使えそうな説明をしてきましたが、実はまだ生産性向上チームのメンバーしか使えないです（より正確には、生産性向上チームが持っている GitHub organization に属するリポジトリでのみ使えます）。[2025/03/10 にエンタープライズ向け GitHub App が GA となり](https://github.blog/changelog/2025-03-10-enterprise-owned-github-apps-are-now-generally-available/)、GitHub App を Enterprise アカウントの下に作れるようになったので、今後はこれを利用して cronium を弊社全体に広げていきたいです。

2 つ目に関して、現在は最低限の監視しかできていないので、弊社全体に利用を広げるためにも監視を充実させたいです。ワークフローが実行されるべき時間と実際に実行された時間の差に関する SLO を設定するのもいいなと考えています。

## まとめ

本稿では、`schedule` イベントを用いた、GitHub Actions のワークフローの定期実行は時間通りに実行されないがち問題を解決した事例を紹介しました。解決方法は、ワークフローファイルにアノテーションという形で cron 式を記述すると、自動的にその cron 式が示す時間に `workflow_dispatch` する社内システム兼 GitHub App を作る、という手段でした。

cronium は草の根的に僕と [@defaultcf](https://zenn.dev/defaultcf) さんの二人で作りました。僕が主に全体設計とプログラミングを行い、@defaultcf さんには主にインフラを作っていただきました。最初からこの役割分担でやろうとはしていなくて、自作 Operator による CronJob の Reconcile 案は @defaultcf さんの提案だったのですが頓挫したのでこうなりました。僕は Kubernetes も Neco もまだまだ初心者なのでとても心強かったです。この場を借りて感謝を申し上げます。

## おまけ

Reconcile のロジックも自動テストしたいのが人情というものです。今回は cron を Reconcile するので、その結果として期待通りに cron が作成されてジョブが実行されることを保証したいです。そこで、今回は cron のジョブ（実体はコールバック関数）からチャネルを通してメッセージを送信し、そのメッセージを受信できたら Reconcile が期待通りに実装できているであろう的な自動テストを書きました。[Deno は KV というキーバリューデータベースを備えており](https://docs.deno.com/deploy/kv/manual/)、これを利用すると簡単にチャネルを用いたメッセージパッシングを実装できます。以下に具体的な実装を示します:

```ts
const send = async <T>(kv: Deno.Kv, value: T): Promise<T> => {
  await kv.enqueue(value);
  return value;
};

const receive = <T>(kv: Deno.Kv): Promise<T> => {
  return new Promise((resolve) => {
    kv.listenQueue((msg: T) => {
      resolve(msg);
    });
  });
};

// 利用例
const channel = await Deno.openKv();
await send(channel, "Hello, World!");
console.log(await receive(channel)); // "Hello, World!" が出力される。
```

メッセージパッシング × 静的型付けといえば [session types](https://en.wikipedia.org/wiki/Session_type) ですが、TypeScript で実現する方法はまだよく分かってなくてそこまでできてないです。今後はそこに挑戦しても面白そうです。

以上、おまけでした。
