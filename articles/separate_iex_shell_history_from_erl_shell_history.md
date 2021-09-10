---
title: "IEx と erl のシェル履歴を分ける"
emoji: "🤼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["elixir", "erlang"]
published: true
---

# 背景

[IEx のドキュメント](https://hexdocs.pm/iex/IEx.html#module-shell-history)にある通り，IEx では以下のように環境変数を設定することでシェル履歴を保存できます．

```bash
export ERL_AFLAGS="-kernel shell_history enabled"
```

しかし，その履歴の保存先は Erlang の erl と共有されているといった問題点があります．すなわち，IEx と erl で履歴が共有されます．
Erlang と Elixir のシンタックスは大きく異なっているので，履歴を共有することによるメリットは特にないかと思われます．

本記事ではこの問題点を解消するための方法を紹介します．

# 解決方法

以下のように環境変数を設定します．

```bash
readonly elixir_history_path=$(elixir --eval ':filename.basedir(:user_cache, "elixir-history") |> IO.write()')
export ELIXIR_ERL_OPTIONS="-kernel shell_history_path '$elixir_history_path'"
```

# 解説

## デフォルトのシェル履歴の保存先

[こちらのドキュメント](https://erlang.org/doc/man/kernel_app.html)によると，デフォルトでは以下の関数呼び出しで調べられるディレクトリに履歴が保存されます．

```elixir
iex(1)> :filename.basedir(:user_cache, "erlang_history")
```

## シェル履歴の保存先を設定する

シェル履歴に関する機能は Erlang の[`kernel` アプリケーション](https://erlang.org/doc/man/kernel_app.html)の一部として実装されています．`kernel` アプリケーションはいくつかの設定パラメタを持っており，その中に履歴の保存先を表すパラメタ `shell_history_path` があります．このパラメタを利用することで，履歴の保存先を設定できます．例えば，erl では以下のように履歴の保存先を設定できます[^1]．

```bash
erl -kernel shell_history_path '"/foo/bar"'
```

IEx は erl と同様に `kernel` アプリケーションを利用しているので，同じ設定が IEx でも可能なら履歴の保存先を設定できそうです．

IEx でアプリケーションの設定パラメタを設定する方法はいくつかあります．今回は永続的にシェル履歴を分けたいので，IEx の環境変数 `ELIXIR_ERL_OPTIONS` を利用することにします．
環境変数 `ELIXIR_ERL_OPTIONS` では erl で利用できるオプションを設定できます[^2]．設定されたオプションは IEx で使用されます．

以上を踏まえると，IEx のシェル履歴の保存先を設定するためには，環境変数 `ELIXIR_ERL_OPTIONS` を以下のように設定すれば良いことがわかります[^3]．

```bash
export ELIXIR_ERL_OPTIONS='-kernel shell_history_path "/foo/bar"'
```

## 設定例

私は Erlang のシェル履歴と近い場所に保存したかったので，以下のように設定しました．

```bash
readonly elixir_history_path=$(elixir --eval ':filename.basedir(:user_cache, "elixir-history") |> IO.write()')
export ELIXIR_ERL_OPTIONS="-kernel shell_history_path '$elixir_history_path'"
```

# 注意点

`shell_history_path` には絶対パスを設定する必要があります．これは，相対パスで設定してもその相対パスが絶対パスに変換されないためです．

また，ある環境では上記のパスの計算に以下で示すほどの時間がかかります．

```bash
$ time elixir --eval ':filename.basedir(:user_cache, "elixir-history") |> IO.write()'
0.53s user 0.11s system 136% cpu 0.467 total
```

そのため，シェル（bash や zsh など）の起動時間が気になる方は，パスの計算結果を直に記述するなどの対策をとっていただけたらと思います．

# 終わりに

本記事では，IEx と erl のシェル履歴を分ける方法を紹介しました．
これで Erlang & Elixir 生活がさらに快適になるはず......！

[^1]: 二重の引用付で囲まれた値を渡す必要があるみたいです．理由がわかっていないので，ご教授いただけると幸いです．
[^2]: IEx のドキュメントにはこの環境変数に関する説明がないのですが，`man` コマンドによる IEx のマニュアルの ENVIRONMENT という項目にしれっと説明が書かれています．
[^3]: 値を囲む引用符は一重で問題ないです．
