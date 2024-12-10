---
title: "Typixを使って複数環境でTypstでスライドをコンパイルする"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nix", "typst", "typix", "touying"]
published: true
published_at: 2024-12-11 00:00
---

:::message
この記事は[Typst Advent Calendar 2024](https://qiita.com/advent-calendar/2024/typst)の11日目の記事です。
:::

## TL;DR:

- Typixを使うとNixとTypstが扱いやすい
- Nixでも抽象化しきれない部分でエラーになることがある
  - 動作しているOSの情報を参照している部分とか

## はじめに

[Typst](https://typst.app/)でスライドを作成するツールに[Touying](https://github.com/touying-typ/touying)というものがあります。

これを使うとスライドをTypstのコードから生成できます。

Typstのコードから生成できるということはPlain textから生成できるという訳で、Plain textということはgitで管理してdiffをGitHubなどで見ることができるということです。

プログラマとしてdiffが見ることができないツールをバージョン管理するのはたいへんつらいです。

なので、これができるだけでtypstを採用する理由になるのではないかと個人的には思っています。

さて、筆者は種々の事情からMac, Windows, Linuxを併用しているのですが、これらの環境で同じ見た目のファイルをコンパイルしようとしたとき、いくつか困ったことがあったので、それについての備忘録を残します。

## この記事で解説しないこと

- Typstの書き方
- Touyingの使い方
- Nixの使い方

## この記事で解説すること

- 複数環境でTypst, Touyingのcompileをしたときに困ったこと

## この記事の対象読者

- Plain textからスライドを生成したい人
- 複数の環境で同じスライドを生成したい人
- ~~お絵描きツールをGitで管理したくない人~~

## 再現性のあるTypstのコンパイル

7日目の[@tani_qiita](https://qiita.com/tani_qiita)さんの[Nixによる再現性のあるTypst執筆環境](https://scrapbox.io/tani-note/Nix%E3%81%AB%E3%82%88%E3%82%8B%E5%86%8D%E7%8F%BE%E6%80%A7%E3%81%AE%E3%81%82%E3%82%8BTypst%E5%9F%B7%E7%AD%86%E7%92%B0%E5%A2%83)で述べられているように、システムから独立したビルドの手法として[Nix](https://nixos.org/)というものがあります。

先述の記事ではNixをそのまま使う方法が紹介されていますが、NixからTypstをeasyに扱うためのツールとして[loqusion/typix](https://github.com/loqusion/typix)というものもあります。

fontの管理など生のNixを触ると煩雑になる部分が抽象化されているようです。

> Typix aims to make it easier to use Nix in Typst projects.
>
> - Dependency management: supports arbitrary dependencies including fonts, images, and data
> - Reproducible: via a hermetically sealed build environment
> - Extensible: fully compatible with Typst packages
>
> https://loqusion.github.io/typix/#feature

このツールを使って複数環境でTypstのコンパイルをしようとしたとき、環境ごとにビルドできたりできなかったりなどの事象が起きました。

:::details コード 長いので折り畳み
```typst
#import "@preview/touying:0.5.3": *
#import themes.simple: *

#show: simple-theme.with(aspect-ratio: "16-9")

= Title

== First Slide

Hello, Touying!

#pause

Hello, Typst!

#pause

こんにちは、Typst!
```

```nix
{
  description = "A Typst project";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    typix = {
      url = "github:loqusion/typix";
      inputs.nixpkgs.follows = "nixpkgs";
    };
    flake-utils.url = "github:numtide/flake-utils";
    typst-packages = {
      url = "github:typst/packages";
      flake = false;
    };
  };

  outputs =
    inputs@{
      nixpkgs,
      typix,
      flake-utils,
      ...
    }:
    flake-utils.lib.eachDefaultSystem (
      system:
      let
        pkgs = nixpkgs.legacyPackages.${system};
        typixLib = typix.lib.${system};

        src = typixLib.cleanTypstSource ./.;
        # Watch a project and recompile on changes
        watch-script = typixLib.watchTypstProject commonArgs;

        typstPackagesSrc = pkgs.symlinkJoin {
          name = "typst-packages-src";
          paths = [
            "${inputs.typst-packages}/packages"
          ];
        };

        typstPackagesCache = pkgs.stdenv.mkDerivation {
          name = "typst-packages-cache";
          src = typstPackagesSrc;
          dontBuild = true;
          installPhase = ''
            mkdir -p "$out"
            cp -LR --reflink=auto --no-preserve=mode -t "$out" "$src"/*
          '';
        };

        commonArgs = {
          typstSource = "main.typ";
          fontPaths = [
            "${pkgs.udev-gothic}/share/fonts/udev-gothic"
          ];
          typstOpts = {};
          virtualPaths = [];
        };

        build-drv = typixLib.buildTypstProject (
          commonArgs
          // {
            inherit src;
            XDG_CACHE_HOME = typstPackagesCache;
          }
        );

        # Compile a Typst project, and then copy the result
        # to the current directory
        build-script = typixLib.buildTypstProjectLocal (
          commonArgs
          // {
            inherit src;
            XDG_CACHE_HOME = typstPackagesCache;
          }
        );
      in
      {
        checks = {
          inherit build-drv build-script watch-script;
        };

        packages.default = build-drv;

        apps = rec {
          default = watch;
          build = flake-utils.lib.mkApp {
            drv = build-script;
          };
          watch = flake-utils.lib.mkApp {
            drv = watch-script;
          };
        };

        devShells.default = typixLib.devShell {
          inherit (commonArgs) fontPaths virtualPaths;
          packages = [
            # WARNING: Don't run `typst-build` directly, instead use `nix run .#build`
            # See https://github.com/loqusion/typix/issues/2
            # build-script
            watch-script
            # More packages can be added here, like typstfmt
            # pkgs.typstfmt
          ];
        };
      }
    );
}
```
:::

具体的には下記のようなコンパイルエラーが起きました。

```
> (前略)
> downloading @preview/touying:0.5.3
> 295.3 KiB / 295.3 KiB (100 %) 295.3 KiB/s in 26.27 ms ETA: 0 s
>
> error: failed to decompress package (failed to create `/homeless-shelter/Library/Caches/typst/packages/preview/touying/0.5.3`)
>   ┌─ sample.typ:1:8
>   │
> 1 │ #import "@preview/touying:0.5.3": *
>   │         ^^^^^^^^^^^^^^^^^^^^^^^^
> (後略)
```

このエラーはMacだけで起こるものでした。

Typstのpackageの使用については[workaroundで紹介されているもの](https://loqusion.github.io/typix/recipes/using-typst-packages.html)だったので、typix側の問題かと思いましたが違いました。

どうやらTypstは内部的に[dirs](https://docs.rs/dirs/latest/dirs/)のcrateを使っているようで、動作しているOSによってパッケージのインストール先を変更していました。

ref: https://github.com/typst/typst/issues/3892

なので、mac OSを対象にする場合、Typstの[0.12.0](https://github.com/typst/typst/releases/tag/v0.12.0)で追加された`package-path`のオプションを使用する(あるいは`/homeless-shelter/Library/...`をMacのときだけ用意するようにNixを設定するか)が必要です。

```diff:nix
56c56,58
<           typstOpts = {};
---
>           typstOpts = {
>             package-path = typstPackagesCache;
>           };
64d65
<             XDG_CACHE_HOME = typstPackagesCache;
74d74
<             XDG_CACHE_HOME = typstPackagesCache;
```

これで手元がMacでCIでビルドしたいなど複数OSで再現性を持ってコンパイルできるようになりました。

## おわりに

本記事で使用したスクリプトは[Omochice/toy-touying-typix](https://github.com/Omochice/toy-touying-typix)にあります。

再現性のあるコンパイルができていていればCIでも他の人のPCでも同じ見た目のPDFが生成されるはずです。

Happy Typst Life!
