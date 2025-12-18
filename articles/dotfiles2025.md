---
title: "2025年のdotfiles"
emoji: "🏠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dotfiles", "Nix", "Kubernetes", "Terraform"]
published: false
---

こんにちは。ソフトウェアエンジニアの生産効率は、高性能なマシンや質の良いキーボード、さらには日々の運動や栄養バランスの取れた食事などに支えられていますが、中でもポータブルな設定ファイル集である **dotfiles** にはそれぞれのエンジニアが持つ計算機環境へのこだわりが凝縮されているのではないでしょうか。
この記事では、まず dotfiles とそれを取り巻くシステムの変遷について説明して、その後に今年2025年に筆者が整備した dotfiles について紹介します。

https://github.com/momeemt/config

:::message
これは[Nix Advent Calendar 2025](https://adventar.org/calendars/11657)の25日目の記事です。メリークリスマス！🎄
:::

# dotfiles

最初に dotfiles についての説明をします。すでに dotfiles を管理・運用されていて内容にのみ興味がある方はスキップして [momeemt/config](#momeemt%2Fconfig) から読んでください。

## dotfiles とは

ドットファイルとは、ファイル名が`.`で始まるファイルのことを指します。
多くのシステムでは隠しファイルに設定されていることが多く、ソフトウェアやOS等の設定ファイルは一般的なユーザから目につかないようにドットファイルとして作成されます。
たとえば、以下のようなファイルがドットファイルの例として挙げられます。

- `~/.bashrc`
  - bash をインタラクティブシェルとして起動する際に実行されるスクリプトの1つ
- `~/.vimrc`
  - テキストエディタの vim の設定ファイル

このような設定ファイルには拡張機能やエイリアス、キーバインドの設定などが集約されており、各エンジニアの生産性を大きく左右する要素の1つと言っても良いでしょう。
エンジニアが扱う計算機環境が複数ある場合、会社用のマシンと自宅用のマシンで設定を共通させたい、SSH先で作業する際にも普段と同じ体系で操作を行いたいという欲求は自然と生まれますが、ソフトウェアのインストールや設定の反映を場当たり的に行っている場合、手順や環境の状態が明文化されないため、新しい環境に移行する際の環境構築や設定の反映には長い時間が掛かります。
そこで、複数あるドットファイルを1つのリポジトリに集約すれば、新しい環境に移った際にも以前の環境を再現する作業を定型化することができそうです。このリポジトリを **dotfiles** と呼びます。

## 素朴な dotfiles

まずは最もシンプルな dotfiles について考えます。最低限必要なのは以下の2つです。

- 設定ファイル
- 適切なパスにファイルを配置するインストールスクリプト

以下にインストールスクリプトを示します。

```sh:install.sh
#!/bin/sh
set -eu

repo_dir=$(cd "$(dirname "$0")" && pwd)
rm -f "$HOME/.vimrc"
ln -s "$repo_dir/vim/.vimrc" "$HOME/.vimrc"
```

これを実行すれば、リポジトリ内の`.vimrc`からホームディレクトリ配下にシンボリックリンクを張ることができます。ファイルをコピーするのではなく、シンボリックリンクを張っておくことで、仮に設定を書き換えたくなった場合にもリポジトリ内で設定を書き換えればそのままvimは設定ファイルを読み出せるため便利です。

このような素朴な dotfiles を運用することには次のようなメリットがあります。

- リンクを行うだけなので実行時間が短く高速
- 外部依存が少ない
- 挙動がシンプルで、デバッグが容易
- シェルスクリプトさえ書ければ良いので学習コストが低い

一方で、次のようなデメリット、辛さも考えられます。

- OSやパッケージマネージャ、シェル、ターミナルなどの環境差によりスクリプトが複雑になる
- 途中で失敗した場合のロールバックや冪等性の保証、上書き/マージ/ファイル退避などの実装を自分で行う必要がある
- シークレット管理を行うことが難しく、環境の再現が難しくなる
- 既存設定と競合した際の扱いが難しい

こうしたファイル配置と環境の差分吸収の問題は、chezmoi や GNU Stow などのツールを導入することで解決できます。

## dotfiles 管理ツールを利用する

複雑な dotfiles を扱う場合、chezmoi や GNU Stow が提供する機能を利用することで処理を共通化して簡潔に記述することができます。それぞれについて見てみましょう。

### chezmoi

https://github.com/twpayne/chezmoi

chezmoi の概要を掴むために以下の記事が参考になりました。

https://zenn.dev/ryo_kawamata/articles/introduce-chezmoi

### GNU Stow

https://www.gnu.org/software/stow/

他にも、[yadm](https://yadm.io)、[dotdrop](https://github.com/deadc0de6/dotdrop)、[Dotbot](https://github.com/anishathalye/dotbot)、[rcm](https://github.com/thoughtbot/rcm)、[homeshick](https://github.com/andsens/homeshick)、[Ellipsis](https://github.com/ellipsis/ellipsis)などの同様のツールが存在します。自分が扱う dotfiles の規模や要求に応じて、適切なツールを選択していくと良いでしょう。
一方で、これらの管理ツールではカバーできないことも存在します。

- 設定を管理することはできますが、コマンドやプラグイン自体は別の手段で管理する必要がある
  - バージョンの差やそのソフトウェアが他のツールがインストールされていることを前提にしていることにより必ずしも同じ環境を再現することは難しい
- 適用が Atomic ではなく、ロールバックすることが難しい

このような要求に対しては、設定ファイルの管理ツールとパッケージマネージャが同一か、あるいは協調して動作できるソフトウェアが必要です。
そこで登場するのがビルドシステムの Nix と、Nix を用いたユーザ空間の設定ツールである home-manager です。

## Nix / NixOS

# momeemt/config

ここからは2025年までに筆者が整備した dotfiles について紹介していきます。

momeemt/config は、以下のようなディレクトリに分けています[^dir-ommision]。元々は `dotfiles` や `nixos-configuration` というリポジトリ名を使用していたのですが、扱うスコープが広くなったため `config` というアバウトな名前を使っています。このような設定ファイル集に充てる適切な名前をご存知の方がいらっしゃいましたら、ぜひ教えてください。

[^dir-omission]: 説明が不要な一部のファイルやディレクトリは省略しています。また、これ以降に示すファイル、ディレクトリ一覧についても同様です。

```sh
.
├── assets       # 公開鍵、スクリプト、画像ファイルなどのアセットを格納する
├── docs         # ドキュメント（未整備）
├── flake.lock   # Nix flakes のロックファイル
├── flake.nix    # Nix flakes のルートファイル
├── k8s          # Kubernetes クラスタのマニフェストを格納する
├── nas          # Nix で管理していない NAS の設定ファイルを格納する
├── nix          # Nix で管理するホスト、ユーザ空間の設定ファイルを格納する
├── secrets      # sops で暗号化されたシークレットを格納する
└── terraform    # インフラのプロビジョニング定義ファイルを格納する
```

## `nix/`の構成

`nix/`ディレクトリ内では、以下のようなディレクトリに分けています。

```sh
.
├── flakes       # flake.nix の定義ファイルをモジュール化したファイル
├── home         # 各ホストのユーザ空間の設定（home-manager）
├── hosts        # 各ホストのシステム空間の設定（NixOS / nix-darwin）
├── lib          # momeemt/config で利用する共通化した関数など
├── modules      # momeemt/config で利用する NixOS module の定義
├── overlays     # パッケージセットに対するオーバーレイ
├── packages     # momeemt/config で利用するパッケージ集
├── profiles     # home, hosts の共通化した設定
└── templates    # `nix flake init -t` で利用するテンプレートファイル
```

### システム空間 (hosts)

### ユーザ空間 (home)

### flake modules (flakes)

## エディタの設定 (Neovim, VSCode)

エディタには Neovim と VSCode を利用しています。主に Markdown や Typst など、プレビューを見ながら文章を書きたい場合には VSCode を利用していて、それ以外は Neovim を利用しています。

# 終わりに

もし私のdotfilesに興味があれば、リポジトリにスターをいただけたり、コンテナイメージを試していただければ励みになります。

https://github.com/momeemt/config
