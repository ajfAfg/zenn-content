---
title: "car と cdr【エイプリルフールネタ】"
emoji: "🚗"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["joke", "elixir", "lisp"]
published: true
---

LISP の `car` って car だよねって気持ちから生まれたジョークライブラリの紹介です．
ライブラリは [PSIL](https://github.com/ajfAfg/psil) と名付けてて，LISP の回文になってます．実装は Elixir です．

## 例

`car` を呼ぶと車が返ってきます．

```elixir
iex> import Psil
iex> car [1, 2, 3]
"🚗"
```

`cdr` はいつも通りです．

```elixir
iex> import Psil
iex> cdr [1, 2, 3]
[2,3]
```

## まとめ

長年の気持ちを発散できて満足です．
