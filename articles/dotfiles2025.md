---
title: "2025年のdotfiles"
emoji: "🏠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dotfiles", "Nix", "Kubernetes", "Terraform"]
published: false
---

こんにちは。ソフトウェアエンジニアの生産効率は、高性能なマシンや質の良いキーボード、さらには日々の運動や栄養バランスの取れた食事などに支えられていますが、中でもポータブルな設定ファイル集である **dotfiles** にはそれぞれのエンジニアが持つ計算機環境へのこだわりが凝縮されているのではないでしょうか。
この記事では、まず dotfiles とそれを取り巻くシステムの変遷について説明して[^already-known-dotfiles]、その後に今年2025年に筆者が整備した dotfiles について紹介します。

[^already-known-dotfiles]: すでに dotfiles を管理・運用されていて、私の dotfiles の内容にのみ興味がある方はスキップして [momeemt/config](#momeemt%2Fconfig) から読んでください。

https://github.com/momeemt/config

:::message
これは[Nix Advent Calendar 2025](https://adventar.org/calendars/11657)の25日目の記事です。メリークリスマス！🎄
:::

# dotfiles

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

## ✅ 素朴な dotfiles

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

## 🚧 dotfiles 管理ツールを利用する

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

## 🚧 Nix / NixOS

Nix について学びたい場合は、[asa1984]() さんの [Nix入門](https://zenn.dev/asa1984/books/nix-introduction) や [Nix入門: ハンズオン編](https://zenn.dev/asa1984/books/nix-hands-on) がおすすめです。

https://zenn.dev/asa1984/books/nix-introduction

https://zenn.dev/asa1984/books/nix-hands-on

## home-manager

# momeemt/config

ここからは2025年までに筆者が整備した dotfiles について紹介していきます。

## 目標と選定基準

## 🚧 ディレクトリ構成

momeemt/config[^config-naming] は、以下のようなディレクトリに分けています[^dir-ommision]。

[^dir-ommision]: 説明が不要な一部のファイルやディレクトリは省略しています。また、これ以降に示すファイル、ディレクトリ一覧についても同様です。
[^config-naming]: 元々は `dotfiles` や `nixos-configuration` というリポジトリ名を使用していたのですが、扱うスコープが広くなったため `config` というアバウトな名前を使っています。このような設定ファイル集に充てる適切な名前をご存知の方がいらっしゃいましたら、ぜひ教えてください。

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

## 🚧 `nix/`の構成

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

### システム構成 (hosts)

`nix/hosts/`には、各ホストのシステム構成について記述しています。
先述した home-manager はホーム構成を適用するツールであり、この節で扱うツールはシステム構成を適用するツールです。システム構成とユーザ環境には以下のような違いがあります。

- 影響範囲
  - システム構成は、システムサービスやグローバルなパッケージ、`/etc`、起動設定など、OS 全体に影響します
  - ホーム構成は、ユーザのパッケージやシェル設定、dotfiles、アプリケーションの設定など、特定のユーザの環境に閉じています
- 権限
  - システム構成は多くの場合、管理者権限を要求します
  - ホーム構成は多くの場合、ユーザ権限で適用できます

システム構成を適用するための方法は使用している OS によって異なります。

#### NixOS

NixOS の場合には、flakes を利用していれば `outputs` に `nixosConfigurations.<host>` に対して構成を定義でき、 `nixos-rebuild switch --flake` コマンドを利用して適用できます。
flakes を利用していない場合にも、`/etc/nixos/configuration.nix` や、明示的に与えた Nix ファイルを基にして `nixos-rebuild switch` コマンドを利用して構成を適用できます。

#### macOS

macOS の場合には、[nix-darwin](https://github.com/nix-darwin/nix-darwin) というモジュールを利用します。私はノートパソコンには macbook を使用しているので、システム構成を管理するためにこのモジュールを使っています。

https://github.com/nix-darwin/nix-darwin

`outputs` に `darwinConfigurations.<host>` に対して構成を定義でき、`sudo darwin-rebuild switch --flake ".#<host>"` コマンドを利用して適用できます。

#### NixOS 以外の Linux

NixOS 以外の Linux ディストロを利用している場合には、[system-manager](https://github.com/numtide/system-manager) によってシステム構成を管理できます。

https://github.com/numtide/system-manager

筆者は NixOS 以外の Linux ディストロを利用していないのでこのモジュールを使ったことはありませんが、現在の環境を徐々に Nix に寄せていく場合には便利なモジュールなのではないでしょうか。現時点では Ubuntu と NixOS でのみテストされており、それ以外の OS で扱う際には `allowAnyDistro = true` により明示的に許可を与えて自分で責任を持たなければいけない点は注意が必要です。

### ホーム構成 (home)

`nix/home/`には、各ホストのホーム構成について記述しています。
dotfiles で記述したいソフトウェアの設定などは基本的に home-manager から設定を行う場合が多いため、システム構成の設定量よりも多くなる傾向にあると思います。
一方で、home-manager が OS ごとの差異を吸収してくれているため、システム構成よりは環境差を意識することが少ないように感じます。

### flake modules (flakes)

## 🚧 シェルの設定（zsh）

シェルは macOS のデフォルトシェルである [zsh](https://zsh.sourceforge.io/) を利用しています。他の [bash](https://www.gnu.org/software/bash/) や [fish](https://fishshell.com/)、[nushell](https://www.nushell.sh/) などと比べて以下のような理由で、バランスが良く使いやすい点を気に入っています。

- bash と比べて補完や履歴などの機能が充実している
- fish や nushell と比べて POSIX 系のシェル文法と親和性が高い[^why-posix]

[^why-posix]: 私が POSIX で定義された範囲のシェルですら満足に使いこなせていないためにこのような選択をしているだけで、大きな疑問がなくなったら nushell を使ってみたいと思っています

### 基本的な設定

zsh の基本的な設定は home-manager の `programs.zsh` から行っています。zsh についてはオプションが充実しているのと、私が zsh をきちんと勉強できていないため、ほとんど home-manager に閉じています。

```nix :nix/profiles/hm/programs/shells/zsh/default.nix
programs.zsh = {
  enable = true;
  enableCompletion = true;
  autocd = false;
  defaultKeymap = "emacs";
  dotDir = "${config.xdg.configHome}/zsh";
  syntaxHighlighting.enable = true;

  cdpath = [
    "${config.xdg.dataHome}/ghq/github.com"
    "${config.xdg.dataHome}/ghq/github.com/momeemt"
  ];

  dirHashes = {
    github = "${config.xdg.dataHome}/ghq/github.com";
    momeemt = "${config.xdg.dataHome}/ghq/github.com/momeemt";
  };
}
```

補完やシンタックスハイライトは有効にしています。このような定型的だがいくらかスクリプトを書かなければいけない機能やその有効化を簡単に行うことができるように抽象化レイヤを提供してくれている点は、home-manager の良いところの1つです。

#### キーマップ

メインエディタとして Neovim を利用しているのでデフォルトキーマップは Emacs を選んでいます。一方で同じエディタを使っている友人はシェルのキーマップに Vimcmd [^vimcmd]を利用していました。[toggleterm.nvim](https://github.com/akinsho/toggleterm.nvim) と競合すると思っていたのですがどうやらさほど問題ないようで、来年はそちらに移っても良いかなと思いました。一方で、macOS のアプリケーションは Emacs キーバインドが効くことが多く[^macos-cocoa]、練習した方が他のソフトウェアの操作効率も上がるかなとも思うので、悩みどころです。

#### XDG Base Directory に準拠する

[XDG Base Directory](https://specifications.freedesktop.org/basedir/latest/) とは、設定やキャッシュ、データなどの保存先を標準化することでファイルを整理するための仕様です。
zsh はデフォルトでは各設定ファイルをホームディレクトリ配下に配置しますが、環境変数 `ZDOTDIR` を設定することで設定ファイルをそのディレクトリ配下に収めることができます。
home-manager では `programs.zsh.dotDir` を適切に設定することで XDG Base Directory に準拠できます。`config.xdg.configHome`  `$XDG_CONFIG_HOME` の値を持つ Nix 式です。

#### ディレクトリ候補の探索パスの追加

`programs.zsh.cdpath` にパスを追加しておくと、
このオプションは `AUTO_CD` の設定に変換されます。
私はリモートから入手した Git リポジトリを管理するツールとして [ghq](https://github.com/x-motemen/ghq) を使っています[^ghq-naming]。このツールは環境変数 `GHQ_ROOT` を設定することで[リポジトリを配置するパスを変更でき](https://github.com/x-motemen/ghq#environment-variables)、私は `$XDG_DATA_HOME/ghq` を指定しています。そのせいで新しく配置したリポジトリに移動するのが非常に面倒であり、楽にタイプするために `cdpath` に GitHub リポジトリ配下と、その中でも私のユーザ名のディレクトリ配下をしています。
そうすることで、以下のように補完が効くようになり、インタラクティブシェル内での体験を改善できます。

[^ghq-naming]: 断りなく会話に出すと知らない人からはギョッとされることが多いが、連合国軍最高司令官総司令部のことではない

一方で、類似した体験を提供するコマンドとして最近は [zoxide](https://github.com/ajeetdsouza/zoxide) が話題になりました。これは過去に移動したディレクトリを記憶しておき、絶対パスや相対パスだけではなくディレクトリ名のみで移動できるようにするコマンドです。
私は `z` コマンドを `cd` にエイリアスした上で、次のように使い分けています。

- 過去に移動したディレクトリは zoxide でジャンプする
- 過去に移動していないディレクトリが多く含まれているディレクトリは `cdpath` に追加する

GitHub のタイムラインで見かけた知らないソフトウェアの Git リポジトリを手元に落として試したり、調べたりすることは多くあるので、`cdpath` の対象にしておくと便利です。

#### 名前付きディレクトリを定義する

`programs.zsh.dirHashes` から zsh で名前付きディレクトリを設定できます。

[^vimcmd]: これを説明したい
[^macos-cocoa]: macOS 向けアプリケーションフレームワークの [Cocoa](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CocoaFundamentals/WhatIsCocoa/WhatIsCocoa.html) が Emacs のキーバインドをデフォルトで採用していたため

### 設定ファイルに直接追記する

基本的にはオプションに割り当てた Nix の値をもとに設定ファイルを生成することが基本ですが、シェルスクリプトや設定のために利用するその他のプログラミング言語[^prog-lang-for-config]は OS やソフトウェアの状態を基にした複雑な条件分岐を表現できますから、あらゆる状態を静的なオプションに落とすのは原理的に難しいです。
そこで、多くのソフトウェアに対するオプションには設定ファイルを文字列で追記する形で自由に記述できるオプションが提供されています。zsh の場合は、`profileExtra` が `.zprofile` に対する追記に、`initContent`　が `.zshrc` に対する追記に対応します。

[^prog-lang-for-config]: Lua、Lisp、Pythonなど

```nix :nix/profiles/hm/programs/shells/zsh/default.nix
programs.zsh = {
  profileExtra = ''
    if [ -x /opt/homebrew/bin/brew ]; then
      eval "$(/opt/homebrew/bin/brew shellenv)"
    fi
  '';

  initContent = ''
    typeset -U path
    bindkey -r "^[[A"
    bindkey -r "^[[B"
    bindkey -r "^[[C"
    bindkey -r "^[[D"
  '';
}
```

それぞれシンプルな設定であるため文字列で直接渡していますが、長くなる場合には LSP などのエディタ支援の恩恵を受けるためにファイルを切り出して `builtins.readFile` で読み出すと良いでしょう。パスなど、Nix 式を評価して得られる値に依存したい場合にはシェル引数などを利用して渡してあげると良いと思います。

`profileExtra` には macOS のパッケージマネージャである Homebrew 関係のパスを追加するスクリプトを、`initContent` にはパスの重複を防ぐスクリプト（`typeset -U path`）と、最近はもう慣れましたが過去には Emacs キーバインドではなく矢印キーを使って履歴やカーソルの移動をしていたため、それを防ぐために矢印キーを無効化するためのスクリプトを追加しています。

### プラグインマネージャ

`programs.zsh` は、[antidote](https://antidote.sh/)、[Oh My Zsh](https://ohmyz.sh/)、[prezto](https://github.com/sorin-ionescu/prezto)、[zplug](https://github.com/zplug/zplug) などのプラグインマネージャをサポートしています。しかし、これらはすべて無効化しています。

```nix :nix/profiles/hm/programs/shells/zsh/default.nix
programs.zsh = {
  # plugin managers
  antidote.enable = false;
  oh-my-zsh.enable = false;
  prezto.enable = false;
  zplug.enable = false;
}
```

プラグインマネージャは起動時に外部リポジトリを取得・更新して、ホームディレクトリ配下に状態を持つという運用になり、構成の差分や更新のタイミングが Nix の管理から外れます。何が、何のツールにより取得され、更新やロールバックは何が責任を持つのか分かりにくくなり、必要以上に dotfiles を複雑化します。プラグインを `fetchFromGitHub` などの nixpkgs が提供する fetcher に寄せれば、ビルドキャッシュやロールバックの恩恵を受けることができるようになります。

### 履歴の管理

zsh の履歴は `programs.zsh.history` から設定できます。

https://github.com/momeemt/config/blob/develop/nix/profiles/hm/programs/shells/zsh/history/default.nix

履歴ファイルは `$XDG_STATE_HOME/zsh/history` 配下に格納するように設定しています。また、履歴ファイルのサイズは上限が決まっているので重複を可能な限り排除すると補完の体験が良くなります。
特に、認証情報を含むコマンドや `ls` 、`cd` などの履歴補完の価値があまり大きくないコマンドについては、`ignorePatterns` から履歴ファイルに書き込まれないように設定しています。

### 関数の定義とロード

普段使い用のシェル関数をいくつか定義しています。zsh は環境変数 `$fpath` に存在するファイルのシェル関数を自動で読み込む機能があり、これを利用しています。

```nix :nix/profiles/hm/programs/shells/zsh/default.nix 
programs.zsh = {
  completionInit = ''
    typeset -U fpath
    fpath=(${./completions} ${./functions} $fpath)
    autoload -U compinit
    compinit -i
  '';
  
  initContent = ''
    autoload -Uz nr mkcd
  '';
}
```

`programs.zsh.completionInit` は、補完機能が有効化されている際に与えたスクリプトを `.zshrc` の[割と上の方](https://github.com/nix-community/home-manager/blob/89c9508bbe9b40d36b3dc206c2483ef176f15173/modules/programs/zsh/default.nix#L481-L483) に展開するオプションです。
ここで `fpath` に config 内のシェル関数、および補完を定義したシェルスクリプトを格納したディレクトリを渡しています。このように Nix 式としてパスを渡すと `/nix/store/` 配下にコピーされ、1つの derivation として扱われます。

これまで定義したシェル関数は `mkcd` と `nr` です。前者はその名の通り、ディレクトリを作成した上で移動を行うスクリプトですが、後者は `nix run` を実行するスクリプトです。
`flake.nix` の `outputs` には、`packages.<package>` という derivation を取るフィールドが存在しており、これは `nix run ".#<package>"` を実行することでパッケージをビルドできます。nixpkgs には

### nixpkgs 由来の zsh を利用する

macOS はデフォルトシェルとして zsh を採用しているため、初期状態から `/bin/zsh` に zsh を配置していますが、バージョンが同じでも OS のアップデートによりビルドオプションや依存ライブラリ、パッチなどが変更される可能性があるため、macOS 標準の zsh ではなく nixpkgs からインストールした zsh を利用したいです。

### zsh についての資料

基本的には Home Manager マニュアルの `programs.zsh.*` オプションの説明やソースコードを参照してください。

https://nix-community.github.io/home-manager/options.xhtml#opt-programs.zsh.enable

最後まで読めていませんが、zsh の技術書や zsh マニュアルに目を通すこともあります。

https://gihyo.jp/dp/ebook/2022/978-4-297-12730-5

https://zsh.sourceforge.io/Doc/Release/zsh_toc.html

## エディタの設定 (Neovim, VSCode)

エディタには Neovim と VSCode を利用しています。主に Markdown や Typst など、プレビューを見ながら文章を書きたい場合には VSCode を利用していて、それ以外は Neovim を利用しています。

## ターミナルマルチプレクサの設定（tmux）

## ✅ ターミナルエミュレータの設定（Alacritty）

ターミナルエミュレータには [Alacritty](https://github.com/alacritty/alacritty) を使っています。GPU アクセラレーションにより高速に動作するのが強みで、使い始めてから速度面で不満を感じたことはありません。一方で、シンプルな作りであるためタブやワークスペース、SSH 連携などの機能は付属していません。筆者はターミナルマルチプレクサと組み合わせることを前提に使っています。

https://github.com/alacritty/alacritty

最近では [Ghostty](https://github.com/ghostty-org/ghostty) や [wezterm](https://github.com/wezterm/wezterm) などの高機能ターミナルエミュレータや、[Kitty](https://github.com/kovidgoyal/kitty)、macOS であれば [iTerm2](https://github.com/gnachman/iTerm2) などの選択肢もあります。実際のところ、Alacritty に不満がないため他のターミナルエミュレータを使ったことがなく、来年度はいくつか試してみたいなと思っています。

Alacritty の設定は Home Manager が提供する、`programs.alacritty.*` オプションを設定していきます。Alacritty の設定ファイルは TOML であり、基本的には以下の Web サイトに設定が載っているため、これを参照しながら設定します。

https://alacritty.org/config-alacritty.html

### ターミナルの設定

Alacritty が起動する際に立ち上がるシェルと、OSC52 の設定を書いています。

https://github.com/momeemt/config/blob/develop/nix/profiles/hm/programs/alacritty/settings/terminal.nix

Alacritty を立ち上げた時に、tmux のセッションに自動で入れるようにしています。まずセッションが存在する場合にはアタッチを行い、存在しない場合には新しく作成してセッションに入ります。
Nix で dotfiles を管理することが他のツールと比べて優位である理由の1つとして、このような他のバイナリに依存するような設定でも移植性を維持できることが挙げられます。たとえば素の Alacritty の設定として以下のように書いたとします。

```toml :Alacritty.toml
[terminal.shell]
program = "zsh"
args = [
  "-l"
  "-c"
  "tmux attach -t main || tmux new -s main"
]
```

すると、Alacritty が開いているシェルから `zsh` や `tmux` が見えていることを暗黙に要求することになります。別途、それらのツールをインストールするようなスクリプトを記述して同時に実行するようにすれば良いと思うかもしれませんが、それでもそのツールをインストールする責任は dotfiles を管理するプログラマにあります。しかし、Home Manager を使えば依存関係を解決する責任を委譲することができ、プログラマの認知負荷を1つ取り除くことができるのです。

### ヒントの設定

Alacritty はヒントという、ターミナルに表示されているテキストを正規表現で検索し、マッチしたテキストを他のプロセスに引数として渡せる機能を提供しています。
筆者は以下の3つのヒントを定義して使っています。

- `open`コマンド[^xdg-open]を使って画面に表示されたURLを既定のブラウザで開く
- Nix のビルドが失敗した際に与えられるコマンド `nix log /nix/store/xxx` を実行してログを開く
- 画面に表示されているパスを Neovim で開く

[^xdg-open]: Linux では `xdg-open`

この設定については以下の記事を投稿したので、詳しくはそちらを参照してください。

https://zenn.dev/momeemt/articles/alacritty-hints

https://github.com/momeemt/config/blob/develop/nix/profiles/hm/programs/alacritty/settings/hints.nix

### ウィンドウの設定

この他にもカーソル、フォント、スクロール等の細々とした設定を書いていますが、ウィンドウの設定は気に入っているので紹介します。

https://github.com/momeemt/config/blob/develop/nix/profiles/hm/programs/alacritty/settings/window.nix

Alacritty は背景画像を設定することができませんが、透明率（`opacity`）を設定することはできます。後述しますが5分ごとに壁紙が切り替わるようにしているので、気分を変えながらコーディングできてとても好きです。また、12月21日には [M-1 グランプリ](https://www.m-1gp.com/)の[決勝](https://tver.jp/series/sryg1lm5fy)が放送されましたが[^m-12025-tanoshimi]、お笑い好きの我々としては 8 月から[^m-1-1kaisen] YouTube に続々と公開されている予選動画を見ることに追われています[^m-1-half-year]。ブラウザを後ろに置いて、漫才を流しながら作業に取り組めるという点でも必須の設定です。

[^m-1-1kaisen]: 夏になると今年もM-1の季節かと思うようになりますが、よく考えれば執筆現在もまだM-1の季節です。今こそ、と言った方が良いかもしれない
[^m-12025-tanoshimi]: この節は12月19日に執筆しており、まだ決勝動画を見ていません。楽しみすぎる！個人的には[たくろう](https://profile.yoshimoto.co.jp/talent/detail?id=6202)が決勝行ったのがアツすぎて落ち着くことができていない、状況です
[^m-1-half-year]: 1年間の半分は予選がどう、誰が通った、誰々が行かなくて悔しい、などと思い続けており、そもそもお笑いの賞レースはM-1以外にもたくさんあるんです。明らかに笑いが過剰に供給され続けていて、でも、それを、追いたいんです泣

![背景透過が設定されたAlacritty](https://storage.googleapis.com/zenn-user-upload/eb040bd817e4-20251219.png)
*背景が透過しているので考え事をしながらカルチェルタンを眺められる*

## Window Manager の設定（Aerospace）

## バージョン管理システムの設定（Git）

## Secure Shell の設定（ssh, openssh）

## Homebrew を共存させる（brew-nix）

## シークレット管理（sops-nix）

## LLM の管理と設定（mcp-server-nix）

## ネットワークの設定

## Cloudflared Tunnel を通す

## NixOS module を定義する

## 壁紙を定期的に切り替えるサービス（set-wallpapers）

## Docker イメージのビルド

## `terraform/`の構成

## 仮想マシンを作成する

## `k8s/`の構成

## ノードを cloudinit から作成してクラスタに参加させる

## 2026年にやりたいこと

- CI のビルド
- karabiner-elements
- Linux デスクトップ環境の整備
- Windows デスクトップ環境の整備
- k8sクラスタを充実させる
- config 管理用のツールを実装する

# 終わりに

もし私のdotfilesに興味があれば、リポジトリにスターをいただけたり、コンテナイメージを試していただければ励みになります。

https://github.com/momeemt/config
