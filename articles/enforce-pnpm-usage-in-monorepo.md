---
title: "monorepo でも pnpm の利用を強制させる"
emoji: "⛓️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["npm", "yarn", "pnpm", "javascript"]
published: true
---

パッケージマネジャーとして pnpm の利用を強制させる（i.e. `npm` や `yarn` を叩けないようにする）方法はいくつか知られていますが、既存の方法では monorepo（pnpm workspace）で管理するパッケージ内での強制は簡単ではありませんでした。本稿では、解決策として、Mise を用いてディレクトリ単位でエイリアスを張り、`npm` や `yarn` を叩くと「pnpm を利用してね」とだけ表示させる仕組みを紹介します。

## 背景

近頃 npm パッケージのサプライチェーン攻撃が猛威を振るっており、対策の一つとして [pnpm](https://pnpm.io/) が注目されつつあります。pnpm が注目されている理由としては、[公開から時間が経ったパッケージのみのインストールを強制できたり](https://pnpm.io/settings#minimumreleaseage)、[依存パッケージの lifecycle scripts をデフォルトでは実行しない](https://socket.dev/blog/pnpm-10-0-0-blocks-lifecycle-scripts-by-default)ところにあるようです。

さて、npm パッケージのパッケージマネジャーとして pnpm を利用する場合、npm や yarn といった他のパッケージマネジャーの利用は禁止したいところです。その方法として、不完全なリストですが以下が知られています:

- 個人単位で実施する方法
  - （1）[`~/.zshrc` に `alias npm='echo "pnpm を利用してね"'` を記述する方法](https://zenn.dev/link/comments/78e6a55c279d8d)
- パッケージ単位で実施する方法
  - （2）[パッケージ `only-allow` を使う方法](https://www.npmjs.com/package/only-allow)
  - （3）[`package.json` の `.engines` を使う方法](https://qiita.com/suin/items/a7bf214f48eb9b2d9afc)

これらの方法は概ね有効に働きますが、問題点もいくつかあります。まず、方法 1 は共同開発者に pnpm の利用を強制できず、方法 2 は `only-allow` が侵害されるリスクがあります。方法 3 は、monorepo 構成（今回は pnpm の workspace のみ考えます）の場合、各パッケージのディレクトリ配下における npm/yarn の利用を禁止するためには、各パッケージに方法 3 を適用する必要があります。つまり、ルートの `package.json` の `.engines` のみ設定すればよい訳ではなく、各パッケージの `package.json` の `.engines` も同様に設定する必要があります。そのため、パッケージが増えると単純に面倒ですし、新しく作るパッケージで上記の設定を書き損じる可能性もあります[^about-usage-of-packagemanager]。

[^about-usage-of-packagemanager]: Discussion で議論していますが、`package.json` の `.devEngines.packageManager` を使う方法もまた、方法 3 と同様の問題点があります。

## 解決策

[Mise](https://mise.jdx.dev/) というバージョン管理ツールを利用すると、コマンドや環境変数をディレクトリ単位[^about-directory-level]で設定できます。これと同じノリで、ディレクトリ単位でエイリアスを貼れば、先述した方法 1 をパッケージ単位で適用可能だと考えました。

[^about-directory-level]: Mise で設定した環境変数は子ディレクトリでも有効です。これは後述する direnv でも同様です。

という訳で、monorepo のルートで一度だけ設定すればよい、Mise だけを利用して `npm` と `yarn` のエイリアスを張る仕組みを考えました。開発者への負担は Mise の使用だけです。開発者はこれから説明する仕組みを施した monorepo のディレクトリ配下に移動するだけで、`npm` や `yarn` を叩くと「pnpm を利用してね」が表示されるようになります。

### 仕組み

ディレクトリ単位でエイリアスを張る方法は、「[みんなが本当に欲しかったのは Makefile じゃなくてディレクトリレベルで管理できるエイリアスなのでは - Lambda カクテル](https://blog.3qe.us/entry/2025/07/14/000748)」で提案されています。超絶ざっくり要約すると次の通りです。[`direnv`](https://direnv.net/) というコマンドを利用するとディレクトリ単位で環境変数を設定できます。これを利用して `$PATH` を拡張し、`$PATH` に追記したディレクトリ（e.g. `<project-root>/.aliases`）にシェルスクリプトを置くと、ディレクトリ単位でコマンドを定義できます。これをエイリアスと呼んでいます。以下では `l` というエイリアスの定義例を示しています（先行事例ではもっと賢く定義されています）:

```sh:<project-root>/.aliases/l
#!/bin/bash

ls -lah
```

さて、先行事例では direnv を利用して `$PATH` に追記していましたが、今回は Mise で同様の振る舞いを実現します[^why-mise]。例えば `mise.toml` へ以下のように記述すると、`mise.toml` が存在するディレクトリ直下のディレクトリ `.aliases` を `$PATH` に追記できます。なお、ディレクトリは `$PATH` の先頭に追記されるようです。

[^why-mise]: 今回 direnv ではなく Mise を使っているのは、ぶっちゃけ好みです。今回紹介する手法は direnv でも実現できます。ただし、コマンドのバージョン管理に Mise を使っている場合は、環境変数の管理にも Mise を使うのが素直かなとは思います。

```toml:mise.toml
[env]
'_'.path = "{{config_root}}/.aliases"
```

あとは、パスを通したディレクトリに `npm` や `yarn` といったシェルスクリプトを置くだけです。スクリプトの内容は `echo "pnpm を利用してね"` です[^as-pnpm]。

[^as-pnpm]: `pnpm "$@"` と書いて pnpm として生きてもらうのもよいと思います。

ここまでの説明をまとめると、`mise.toml` やエイリアス `npm`/`yarn` のディレクトリ構成は以下の通りとなります:

```
.
├── .aliases
│   ├── npm
│   └── yarn
└── mise.toml
```

実例は本稿を管理している GitHub リポジトリにあります:

https://github.com/ajfAfg/zenn-content/tree/7dcdd84bf6cb8ce8de4adaf4338a14c177fd013f

## まとめ

本稿では、monorepo（pnpm workspace）でも pnpm の利用を強制させるため、Mise を用いてパッケージ単位で `npm`/`yarn` のエイリアスを張る仕組みを紹介しました。

Mise はいいぞ。
