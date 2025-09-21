---
title: "monorepo でも pnpm の利用を強制させる"
emoji: "⛓️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["npm", "yarn", "pnpm", "javascript"]
published: true
---

パッケージマネジャーとして pnpm の利用を強制させる方法はいくつか知られていますが、既存の方法では monorepo で管理するパッケージ内での強制は簡単ではありませんでした。本稿では、解決策として、Mise を用いてディレクトリ単位でエイリアスを張り、`npm` を叩くと「pnpm を利用してね」とだけ表示させる仕組みを紹介します。

## 背景

近頃 npm パッケージのサプライチェーン攻撃が猛威を振るっており、対策の一つとして [pnpm](https://pnpm.io/) が注目されつつあります。pnpm が注目されている理由としては、[公開から時間が経ったパッケージのみのインストールを強制できたり](https://pnpm.io/settings#minimumreleaseage)、[依存パッケージの lifecycle scripts をデフォルトでは実行しない](https://socket.dev/blog/pnpm-10-0-0-blocks-lifecycle-scripts-by-default)ところにあるようです。

さて、npm パッケージのパッケージマネジャーとして pnpm を利用する場合、npm や yarn といった他のパッケージマネジャーの利用は禁止したいところです。その方法として、不完全なリストですが以下が知られています:

- 個人単位
  - （1）[`~/.zshrc` に `alias npm='echo "pnpm を利用してね"'` を記述する方法](https://zenn.dev/link/comments/78e6a55c279d8d)
- パッケージ単位

  - （2）[パッケージ `only-allow` を使う方法](https://www.npmjs.com/package/only-allow)
  - （3）[`package.json` の `engines` を使う方法](https://qiita.com/suin/items/a7bf214f48eb9b2d9afc)

これらの方法は概ね有効に働きますが、問題点もいくつかあります。まず、方法 1 は共同開発者に pnpm の利用を強制できず、方法 2 は `only-allow` が侵害されるリスクがあります。方法 3 は、monorepo 構成（今回は pnpm 標準の workspace のみ考えます）の場合、各パッケージのディレクトリ配下における `npm`/`yarn` の利用を禁止するためには、各パッケージに方法 3 を適用する必要があります。つまり、ルートの `package.json` の `engines` のみ設定すればよい訳ではなく、各パッケージの `package.json` の `engines` も同様に設定する必要があります。そのため、パッケージが増えると単純に面倒ですし、新しく作るパッケージで上記の設定を書き損じる可能性もあります。

## 解決策

[Mise](https://mise.jdx.dev/) というバージョン管理ツールを利用すると、コマンドや環境変数をディレクトリ単位で設定できます。これと同じノリで、ディレクトリ単位でエイリアスを貼れば、先述した方法 1 をパッケージ単位で適用可能だと考えました。

という訳で、monorepo のルートで一度だけ設定すればよい、Mise だけを利用して `npm` と `yarn` のエイリアスを張る仕組みを考えました。開発者への負担は Mise の使用だけです。開発者はこれから説明する仕組みを施した monorepo のディレクトリ配下に移動するだけで、`npm` や `yarn` を叩くと「pnpm を利用してね」が表示されるようになります。

### 仕組み

ディレクトリ単位でエイリアスを張る方法は、「[みんなが本当に欲しかったのは Makefile じゃなくてディレクトリレベルで管理できるエイリアスなのでは - Lambda カクテル](https://blog.3qe.us/entry/2025/07/14/000748)」で提案されています。以降はこちらの方法を理解されている前提で話を進めます[^sorry]。

[^sorry]: 本稿を理解するのに必要な部分のみ説明しようとすると全部説明することになるのでごめんね。

先行事例では direnv を利用して `$PATH` を乗っ取っていましたが、Mise でも同様の振る舞いを実現できます。例えば `mise.toml` で以下のように記述すると、`mise.toml` が存在するディレクトリ直下のディレクトリ `bin` を `$PATH` に追加できます。なお、追加するパスは `$PATH` の先頭に追記されるようです。

```toml
[env]
'_'.path = "{{config_root}}/bin"
```

あとは、追加するパス先に `npm` や `yarn` といったシェルスクリプトを置くだけです。記述するスクリプトは `echo "pnpm を利用してね"` でもいいし、`pnpm "$@"` と書いて pnpm として生きてもらうのもよいです。

ここまで説明してきた仕組みの実例は、本稿を管理している GitHub リポジトリにあります:

https://github.com/ajfAfg/zenn-content

## まとめ

本稿では、monorepo（pnpm 標準の workspace）でも pnpm の利用を強制させるため、Mise を用いてパッケージ単位で `npm`/`yarn` のエイリアスを張る仕組みを紹介しました。

Mise はいいぞ。
