---
title: "DeeplをCLI上で実行するアプリを作った"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "deepl"]
published: true
---

# DeeplをCLI上で実行するアプリ

@[card](https://github.com/Omochice/deepl-translate-cli)

## 使い方

リリースページからダウンロードするかリポジトリをクローンしてビルドしてください。

ダウンロードした（or ビルドした）ファイルを`PATH`の通っているディレクトリにおいてください。
`$HOME/.local/bin/`以下に置く場合は

```bash
$ cp path/to/deepl-translation $HOME/.local/bin/
```

deeplのAPIを叩く都合上、アクセストークンが必要なのでdeeplの無料プランに登録してください。

アクセストークンは環境変数`DEEPL_TOKEN`に登録してください。

```bash:bash
export DEEPL_TOKEN=<YOUR TOKEN>
```

翻訳元の言語や翻訳先言語は`$HOME/.config/deepl-translation/setting.json`にデフォルト値を設定することができます。

```json:setting.json
{
    "source_lang": "EN",
    "target_lang": "JA"
}
```

このファイルが存在しないときに`deepl-translation`コマンドを実行すると値が空の設定ファイルが生成されます。

コマンドはファイル入力と標準入力を受け付けます。
標準入力を使う場合は`--stdin`オプションをつけてください。

```
$ deepl-translation text.txt
```

翻訳後のデータは標準出力に書き出されるのでリダイレクト等で保存してください。

一時的に翻訳元・翻訳先言語を変えたいときは`-s(--source_lang)`,`-t(--target_lang)`オプションが使用できます。


## 開発を通じて得た知見

- 標準入力でパイプとそうでないときの切り替え
 `terminal.IsTernimal(syscall.Stdin)`で判断できる。
 ```go
 if terminal.IsTerminal(syscall.Stdin) {
     // is not pipe
     fmt.Scan(&rawSentense)
 } else {
     // is pipe
     pipeIn, err := ioutil.ReadAll(os.Stdin)
     if err != nil {
         return err
     }
     rawSentense = string(pipeIn)
 }
 ```

