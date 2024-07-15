---
title: "denopsで書いたプラグインをjsrに対応させる"
emoji: "🐜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim", "nvim", "denops", "deno", "jsr"]
published: true
---

## TL;DR:

以下を実行する。

```console
$ deno run -A jsr:@deno/x-to-jsr
$ deno run -A jsr:@omochice/importmap-expand --option deno.json ./**/*.ts
```

必要に応じて、以下も実行する。

```console
$ git restore deno.json
$ deno fmt ./**/*.ts
$ deno check ./**/*.ts
$ deno test
```

## 背景

[deno.land/x は今後レジストリとしては開発されなくなる](https://zenn.dev/kt3k/articles/4aa235ff817a6c#%E7%90%86%E7%94%B12.-deno.land%2Fx-%E3%81%AF%E4%BB%8A%E5%BE%8C%E3%83%AC%E3%82%B8%E3%82%B9%E3%83%88%E3%83%AA%E3%81%A8%E3%81%97%E3%81%A6%E3%81%AF%E9%96%8B%E7%99%BA%E3%81%95%E3%82%8C%E3%81%AA%E3%81%8F%E3%81%AA%E3%82%8B)

という状態になっていて、denops/std も`7.0.0`から[jsr.io](https://jsr.io/@denops/std)での公開になるようです。


プラグインが動かなくなることは、(おそらく)ないはずですが、追従が面倒になるのでできれば合わせておきたいところ。


## 何をすればいいのか

"`https://deno.land`から読み込んでいたライブラリを`jsr:`からの読み込みに変える" ことが必要です。

ただ、使っているライブラリのどれが`jsr:`での読み込みに切り替えられるのか、いちいち調べるのは面倒だったりします。

幸いなことに、[denoland/x-to-jsr](https://github.com/denoland/x-to-jsr)という便利なツールがあるので、これを使います。

```console
$ deno run -A jsr:@deno/x-to-jsr
```

これをすると使っているライブラリのうち、jsrでも公開されているものがあれば、それに置き換えた上で`deno.json`にimport mapを作ってくれます。

(`deno.jsonc`が既に存在しているのならそちらへ書き込みがされます。)

:::message
同じリポジトリから複数のライブラリをjsr.ioに公開している場合、意図しないライブラリ側に置き換えられる場合があるようです。

手元だと`https://deno.land/x/valibot`が`jsr:@valibot/i18n`になる事象を確認しています。
(`x-to-jsr`のコードをみる限りでは直すのは難しそう。)
:::


denopsのブラグインは2024-07の時点では、`deno.json`のimport mapの解決はできないので、ソースコード中での`jsr:`の形式に直してあげないといけません。

稚作のツールがあるのでこれを使います。

```console
$ deno run -A jsr:@omochice/importmap-expand --option deno.json ./**/*.ts
```

import mapが`deno.json`ではなく`deno.jsonc`に書かれている場合、`--option`には`deno.jsonc`を渡す必要があります。


このツールで`deno.json`に書かれているimport mapをソースコード中での`jsr:`に置き換えられます。


## Optional

ここから下は必要に応じて別の方法で置き換えられます。


`deno.json`に書かれているimport mapは不要になったので、消します。

```console
$ git restore deno.json
```


`https://...`から`jsr:`へコードが変化し、折り返さないと`deno fmt`が通らないかもしれません。


```console
$ deno fmt ./**/*.ts
```


移行先のバージョンが`jsr.io`に公開されていない場合など、コードが実行できない状態になっている可能性もあります。

`deno check`で型チェック、`deno test`でテストをして確認したほうが安全です。


```console
$ deno check ./**/*.ts
$ deno test
```
