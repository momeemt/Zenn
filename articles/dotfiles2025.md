---
title: "2025年のdotfiles"
emoji: "🌳"
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

## ✅ 目標と選定基準

dotfiles を構成する上での目標と選定基準について予め明らかにしておきます。

### 目標

計算機を入れ替えたり、突然初期化したとしても設定を適用すると以前と全く同様の環境を復元できるような dotfiles を構築することを目標にしています。
静的に定まるソフトウェアの設定については比較的に管理しやすい一方で、ファイルシステムやプロセスの管理、自宅インフラの環境などの動的な状態を完全に再現することは難しいです。2025年の dotfiles の改善のほとんどはソフトウェア設定の整備に留まっていたので、長い目で理想の状態を目指す必要があるのだと思います。
以下の記事には状態を再現するための重要な示唆を与えられました。

https://masawada.hatenablog.jp/entry/2022/09/09/234159

### 選定基準

- 手続き的なシステムよりも宣言的なシステムを選ぶ
  - 記述言語がチューリング完全のプログラミング言語であれば、設定の意味を静的に把握して、すべてをカバーできるような宣言に落とし込むことは非常に難しくなります
  - したがって宣言的なシステムを扱っていても**手続き的な操作を記述することからは避けられません**
  - それでも抽象化の試行錯誤が積み重なることで宣言として扱える範囲が増えていきます
  - また、現状の抽象化の結果を他者から見える形で残すことは、誰かがより良い抽象化を発見することへ繋がると思っています[^not-ip]
- プログラマの責任を委譲できるシステムを選ぶ
  - 書いたのが自分であれ、時間が経てば他人が書いたコードのようにその意図を読み取るのが難しくなります
  - それは手順も同じであり、質の良い手順書を維持しなければ環境を再現できなくなる可能性がある状態は望ましくありません。できるだけプログラマの責任を減らしてツールに責任を持ってもらい、記憶に頼るのをやめる必要があります

[^not-ip]: 完全に余談ですが、私は知的財産という考え方があまり好きではありません。すべての人間は自分が（偶然）得た環境によって能力を獲得して価値ある物を作っていくのであって、その結果の大半はその人の努力ではなく単なる運によって支えられていると考えています。発明であれ、研究であれ、ソフトウェアでも映像でも絵でもなんでもそうですが、個人の承認欲求や自社の利益のためにそれらの権利を独占するのではなく、社会全体の利益に使われるべきです。しかし社会の基本的な構造を変えるのは難しいので、自分ができる範囲で公開する（つまり、何らかの契約で縛られていない限り、自分が作ったソフトウェアやソースコードはすべて公開して自由に使えるようにする）ようにしています

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

## エディタの設定 (Neovim, nixvim, VSCode)

エディタには Neovim と VSCode を利用しています。主に Markdown や Typst など、プレビューを見ながら文章を書きたい場合には VSCode を利用していて、それ以外は Neovim を利用しています。

## 🚧 ターミナルマルチプレクサの設定（tmux）

ターミナルマルチプレクサは1つのターミナルエミュレータの中で、複数のターミナルを作成して切り替えたり、画面を分割して並べて表示したりするためのソフトウェアです。作業状態をセッションとして保存できるので、SSH接続が切れてもセッション内で動作しているプロセスを停止することなく、後から同じ状態に復帰することができます。私は [tmux](https://github.com/tmux/tmux/wiki) というターミナルマルチプレクサを使っています。

https://github.com/tmux/tmux/wiki

最も素朴なターミナルマルチプレクサとして [GNU Screen]() がある他、最近では [Zellij](https://github.com/zellij-org/zellij) も注目されています。Zellij は現代的なターミナルワークスペースで、自由なレイアウトを組むことができ、プラグインシステムとして [WebAssembly]() を採用しています。3年前に初めて SSH 接続を切らさないために Screen を使って便利さを実感してから、tmux を使い始めました。Zellij も気になっていますが、現状では tmux の操作に不満を感じていないことと、個人的な経験として WebAssembly のプラグインシステムが満足に動作するのか疑問を持っていることもあり移行していません。来年はいくつかプラグインを試したり作ってみたりして、tmux よりも良いと確信を持てたら移行してみたいなとも思っています。

Home Manager が提供する tmux の設定は非常にシンプルで、他のツールと同様に [`programs.tmux.*`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.tmux.enable) を設定しますが、主な設定は [`programs.tmux.extraConfig`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.tmux.extraConfig) に tmux の設定ファイルである `tmux.conf` に記述する内容を書くことになります。

### 基本的な設定

https://github.com/momeemt/config/blob/develop/nix/profiles/hm/programs/tmux/default.nix

### `extraConfig` の内容

https://github.com/momeemt/config/blob/develop/nix/profiles/hm/programs/tmux/tmux.conf

## ✅ ターミナルエミュレータの設定（Alacritty）

![背景透過が設定されたAlacritty](https://storage.googleapis.com/zenn-user-upload/eb040bd817e4-20251219.png)
*背景が透過したシンプルなターミナルエミュレータ*

ターミナルエミュレータはシェルやCLIプログラムが出力する文字列を表示し、キーボード入力を渡すためのアプリケーションです。
私は [Alacritty](https://github.com/alacritty/alacritty) を使っています。GPU アクセラレーションにより高速に動作するのが強みで、使い始めてから速度面で不満を感じたことはありません。一方で、シンプルな作りであるためタブやワークスペース、SSH 連携などの機能は付属していません。筆者は、ターミナルエミュレータは入出力を担当し、タブやペイン分割などの多重化はターミナルマルチプレクサで補う構成にしています。

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

すると、Alacritty が開いているシェルから `zsh` や `tmux` が見えていることを暗黙に要求することになります。別途、それらのツールをインストールするようなスクリプトを記述して同時に実行するようにすれば良いと思うかもしれませんが、それでもそのツールをインストールする責任は dotfiles を管理するプログラマにあります。しかし、Home Manager を使えばそのバイナリを Nix store にインストールした上で、`pkgs.zsh` や `pkgs.tmux` から評価されるバイナリの絶対パスを設定ファイルに埋め込むため、依存関係を解決する責任を委譲することができ、プログラマの認知負荷を1つ取り除くことができるのです。

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

Alacritty は[背景画像を設定することができません](https://github.com/alacritty/alacritty/issues/565#issuecomment-312459718)が、透明率（`opacity`）を設定することはできます。後述しますが5分ごとに壁紙が切り替わるようにしているので、気分を変えながらコーディングできてとても好きです。また、12月21日には [M-1 グランプリ](https://www.m-1gp.com/)の[決勝](https://tver.jp/series/sryg1lm5fy)が放送されましたが[^m-12025-tanoshimi]、お笑い好きの我々としては 8 月から[^m-1-1kaisen] YouTube に続々と公開されている予選動画を見ることに追われています[^m-1-half-year]。ブラウザを後ろに置いて、漫才を流しながら作業に取り組めるという点でも必須の設定です。

[^m-1-1kaisen]: 夏になると今年もM-1の季節かと思うようになりますが、よく考えれば執筆現在もまだM-1の季節です。今こそ、と言った方が良いかもしれない
[^m-12025-tanoshimi]: この節は12月19日に執筆しており、まだ決勝動画を見ていません。楽しみすぎる！個人的には[たくろう](https://profile.yoshimoto.co.jp/talent/detail?id=6202)が決勝行ったのがアツすぎて落ち着くことができていない、状況です
[^m-1-half-year]: 1年間の半分は予選がどう、誰が通った、誰々が行かなくて悔しい、などと思い続けており、そもそもお笑いの賞レースはM-1以外にもたくさんあるんです。明らかに笑いが過剰に供給され続けていて、でも、それを、追いたいんです泣

## ✅ ウィンドウマネージャの設定（AeroSpace）

![AeroSpaceが動作する様子](https://storage.googleapis.com/zenn-user-upload/00471f459770-20251220.gif)
*複数のウィンドウを切り替えたり入れ替えたりできる*

ウィンドウマネージャ（WM）は、GUIアプリケーションのウィンドウの配置やサイズ、フォーカス管理、仮想デスクトップやマルチモニタ間での移動、操作方法、ルールなどをどのように扱うかを決めて実行するソフトウェアです。ウィンドウ同士が重なることを許して自由に動かす floating 系のウィンドウマネージャと、画面を分割して自動整列する tiling 系のウィンドウマネージャがあり、私は後者の [AeroSpace](https://github.com/nikitabobko/AeroSpace) というソフトウェアを使っています。

https://github.com/nikitabobko/AeroSpace

WM は macOS と Linux デスクトップで扱いが異なります。まず、macOS の場合には OS 標準の WM が提供されており、外部ツールが配置やサイズ変更、ワークスペースの機能を追加します。一方で Linux デスクトップの場合には WM そのものを置き換える形となります。私は Linux デスクトップサ環境には [Wayland](https://wayland.freedesktop.org/) を、compositor には [sway](https://swaywm.org/) を利用していますが、そもそも Linux デスクトップ環境をほぼ整備していないので割愛します。
また、以前は macOS の WM として [yabai](https://github.com/asmvik/yabai) と [skhd](https://github.com/asmvik/skhd) を組み合わせて使っていました。前者は tiling 系のウィンドウマネージャ本体ですがキーバインドが割り当てられておらず全ての操作にコマンドが割り当てられています。そこで、ホットキーを実現するデーモンである後者を組み合わせて、特定のキーを入力したらウィンドウが分割されたりワークスペースが操作されるような構成にしていました。しかし、yabai は macOS のアップデートのタイミングで挙動が不安定になることが多く、設定も yabai と skhd でそれぞれ別々に行う必要があります。AeroSpace はそれが統合されており、ワークスペース切り替えの挙動も高速で安定しているところを気に入っています。

### nix-darwin vs Home Manager

Aerospace の設定は [nix-darwin](https://nix-darwin.github.io/nix-darwin/manual/#opt-services.aerospace.enable) と [Home Manager](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.aerospace.enable) の両方にあります。
システム構成として与えるか、ホーム構成として与えるかのメリット・デメリットは既に整理した通りですが、以下の理由でシステム構成として与えています。

1. 実家のパソコンも Nix で管理する予定があるが、AeroSpace の存在によって情報を専門としていない一般的なユーザの操作を阻害することは基本的にないと考えられる
1. WM はデスクトップ環境の基本ソフトウェアであり、キーバインドやワークスペース設定をホスト単位で固定したい

なお、nix-darwin と Home Manager の両方で AeroSpace を有効化すると二重起動や設定が競合する原因となるため、どちらか一方に統一する必要があります。

### キーバインドの設定

https://github.com/momeemt/config/blob/develop/nix/profiles/hosts/services/aerospace/main.nix

フォーカスは `Alt + hjkl`、ウィンドウの移動は `Alt + Shift + hjkl` に割り当てています。サイズ変更はよく使いますが、他の操作（分割、マルチモニタの移動）などはあまり使えていません。
ワークスペースの移動は `Alt + <workspace-n>`、ウィンドウを別のワークスペースへ飛ばす際には `Alt + Shift + <workspace-n>`ですが、これは少し楽に書けるように工夫しています。

#### リストの生成、属性セットへの変換

`builtins.genList fn length` は `fn` に `0` から `length-1` までの整数を与えて得られる要素から成るリストを生成する関数です。Python の[リストの内包表記](https://docs.python.org/ja/3/tutorial/datastructures.html#list-comprehensions)と同じような処理を行います。
`builtins.listToAttrs` は、キー名を表現する `name` と値を表現する `value` の属性を要素に取るリストを、属性セットに変換する関数です。すなわち、

```nix
[
  {
    name = "key1";
    value = "value1";
  }
  {
    name = "key2";
    value = "value2";
  }
]
```

を、以下の属性セット

```nix
{
  key1 = "value1";
  key2 = "value2";
}
```

に変換します[^first-prior-listToAttrs]。このように柔軟に属性セットを構成できるのは便利です。
Aerospace の設定ではキーバインドをキー名に取り、操作を値に取るので、ワークスペース番号と数字キーの値が一致している時にはこのように記述することで簡潔に設定を表現できてミスも減らせます。
また、Nix は for 文で手続き的に属性セットを構成するのではなく、このようにデータの構造を上手く変換する関数型言語的なアプローチで構成する[^like-ocaml]ところも面白い点の1つです。面白いだけではなく、羅列になりがちな設定の性質を扱いやすい点でも手続き的な言語で記述するよりも向いているとも思います。

[^first-prior-listToAttrs]: なお、同じ`name`が複数回出た場合には[最初のものが優先されます](https://nix.dev/manual/nix/2.30/language/builtins?utm_source=chatgpt.com#builtins-listToAttrs)
[^like-ocaml]: 節々から [OCaml](https://ocaml.org/) の影響を受けていることを感じさせられます

### サービスモードの定義

AeroSpace では**モード**という概念があり、キーバインドを切り替えることができます。モードは、`services.aerospace.settings.mode.<mode>.binding` に設定を書くことで作成できますが、デフォルトモード（`main`）から新しく作成したモードに入るためのキーバインドは別途設定しておく必要があります。私の設定では、`alt-shift-semicolon = "mode service";` がそれに該当しています。

https://github.com/momeemt/config/blob/develop/nix/profiles/hosts/services/aerospace/service.nix

基本的には[公式ドキュメント](https://nikitabobko.github.io/AeroSpace/guide)が紹介しているデフォルトの設定をそのまま書いています。サービスモードの存在を思い出した時に調べて使ってみる程度に留まっており、たとえば音量の調節は macOS のデフォルトのホットキーを利用しているので使いません。
時折、タイリングされるとかえって作業がし辛い場合があり、`layout floating tiling`（`floating` 、`tiling` 型を切り替える）は便利そうです。この辺りはまだ未整備なので目下の課題としています。

## 🚧 バージョン管理システムの設定（Git）

バージョン管理システム（VCS）とは、ソフトウェア開発におけるプログラムやリソース等のファイルの変更履歴を管理するソフトウェアです。多くの方がそうであるように、私も [Git](https://git-scm.com/) を使っています。

https://git-scm.com/

[Subversion](https://subversion.apache.org/)[^subversion] や [Mercurial](https://www.mercurial-scm.org/)、最近では [Jujutsu](https://github.com/jj-vcs/jj) なども話題になりました。VCS については他のツールを試してもいないというのが正直なところですが、自分が Git を使い熟せているとは思えないので保留しています。最近は興味が移ってあまり取り組めていませんが、2025年度前半に [nixpkgs](https://github.com/NixOS/nixpkgs) への Pull Request を作っていた際には、さまざまなソフトウェアのビルドを並行に試していくので stash だけでは追いきれなくなり `git worktree` を使ったり、PR を出す際の要件として必要なコミットにまとめなければいけないため `git rebase` を使ったりしていました。個人開発では扱わない巨大な Git リポジトリに対して貢献を行うことで解像度が深まるという側面はあるかもしれません。

[^subversion]: Gitを始めて学んだ時に、VCSの**歴史**としてSubversionを知りました。もうこんなことを言っても鼻で笑われることはないほどにGitが覇権を握っていますね

### 基本的な設定

Home Manager では [`programs.git.*`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.git.enable) を設定することで、`.gitconfig` 相当の設定を行えます。名前とメールアドレスの設定を行っているのと、デフォルトブランチを GitHub に合わせて `main` にしています。
エイリアスはいくつか設定していて、`st`（`status`）、`aa`（`add --all`）[^git-aa]、`cm`（`commit -m`）、`poh`（`push origin head`）、`d`（`diff`）などはかなり使っています。また、`ignores` にはデフォルトで無視するべきファイルやディレクトリを設定していますが、`.momeemt` というユーザ名の隠しファイルを指定しています。OSS など他のユーザがオーナーシップを持っているリポジトリで Git 管理下に載せたくないが作業ファイルを置いておきたい場合に、プロジェクトルートで `mkdir -p .momeemt` を作るだけで良いので便利です。

[^git-aa]: 楽なのでやってしまいますが、本来はきちんとステージングするファイルや編集範囲を決定するべきだと思います。来年の目標の1つ

https://github.com/momeemt/config/blob/develop/nix/profiles/hm/programs/git/default.nix

### diff を見やすくする

Git のデフォルトの diff は削除行の先頭に `-` が、追加行の先頭に `+` が付いてハイライトされますが、変更前と変更後を左右に分割して表示してくれるツール、[difftastic](https://difftastic.wilfred.me.uk/) を導入しています。

https://difftastic.wilfred.me.uk/

https://github.com/momeemt/config/blob/develop/nix/profiles/hm/programs/difftastic/default.nix

## ✅ Homebrew を共存させる（brew-nix）

macOS の著名なパッケージマネージャといえば [Homebrew](https://brew.sh/) ですが、ホーム構成を Nix エコシステムに寄せたとしても Homebrew が必要になるケースがあります。

https://brew.sh/

1. GUI アプリケーションのインストール
1. 別のアプリケーションをインストールするようなハブとなるアプリケーションのインストール

まず、GUI アプリケーションですが、nixpkgs は基本的に macOS と比べて Linux を優先してパッケージングしています。ですから macOS のみをターゲットにしたアプリケーションや、そうでなくても Linux 版しか配布されていないアプリケーションは存在します。そこで、GUI アプリケーションをすでに多くのユーザに利用されておりいて知見が溜まっている [Homebrew Cask](https://github.com/Homebrew/homebrew-cask) によってインストールすることが考えられます。
しかし、[シェルの節](#プラグインマネージャ)で言及したように、別のパッケージマネージャを併用すると構成の差分や更新のタイミングが Nix の管理からは外れます。この問題を解決しつつ、Homebrew Cask を使用するために、[brew-nix](https://github.com/BatteredBunny/brew-nix) というツールが作られています。

https://github.com/BatteredBunny/brew-nix

### brew-nix

brew-nix は、Homebrew Cask の情報をもとに、macOS のアプリケーション（`.app`）を Nix の derivation として自動生成するツールです。使うには、まず `flake.nix` の `inputs` に以下の入力を追加します。

```nix :flake.nix
inputs = {
  brew-nix = {
    url = "github:BatteredBunny/brew-nix";
    inputs.brew-api.follows = "brew-api";
  };
  brew-api = {
    url = "github:BatteredBunny/brew-api";
    flake = false;
  };
};
```

brew-nix の作者が、Homebrew Casks からパッケージ情報を抽出した JSON を [brew-api](https://github.com/BatteredBunny/brew-api) の `cask.json` として配布しています。このリポジトリは定期的に更新されています。

https://github.com/BatteredBunny/brew-api

このツールが提供するオーバレイを適用することで、Homebrew をインストールして `brew install --cask <package>` を実行する代わりに、`pkgs.brewCasks.<package>` のような形で Nix を使ってビルド・インストールできます。

```nix
home.packages = with pkgs; [ brewCasks.visual-studio-code ];
```

筆者は [Discord](https://discord.com/) や [ChatGPT](https://chatgpt.com/)、[Google Chrome](https://www.google.com/intl/ja_jp/chrome/)、Microsoft Office製品など、さまざまなアプリケーションを `pkgs.brewCasks` から入れています。

https://github.com/momeemt/config/blob/develop/nix/modules/home/home/casks.nix

### パッチを当てる

brew-nix は Homebrew Casks の情報を元に機械的に derivation を生成するツールであるため、これが提供する Nix 式そのままではビルドに失敗するアプリケーションも数多く存在しています。最も多く遭遇するのはハッシュ値が生成された derivation に記載されていない場合で、公式ドキュメントでも[約700パッケージが該当する](https://github.com/BatteredBunny/brew-nix#overriding-casks-derivation-attributes-for-casks-with-no-hash:~:text=There%27re%20about%20700%20casks%20without%20hashes%2C%20so%20you%20have%20to%20override%20the%20derivation%20attributes%20%26%20provide%20a%20hash%20explicitly.)と言及されています。
私は `nix/overlays` 内に `pkgs.brewCasks` を上書きするオーバレイを作成し、`pkgs` を作成するときに適用しています。

https://github.com/momeemt/config/blob/develop/nix/overlays/brew-casks.nix

最も簡単なのはハッシュ値を与えるだけのパッチです。

```nix :nix/overlays/brew-casks.nix
spotify = prev.brewCasks.spotify.overrideAttrs (oldAttrs: {
  src = prev.fetchurl {
    url = prev.lib.lists.head oldAttrs.src.urls;
    hash = "sha256-gMPLn/QMbbwrlorPRyH9GACqi4jmunfnnMc4AEvyIMU=";
  };
});
```

[nix-prefetch-url](https://nix.dev/manual/nix/2.24/command-ref/nix-prefetch-url) を使うか、まず `hash = lib.fakeHash` としてビルドすることで、エラー報告から正しいハッシュ値を知ることができます。しかし、ハッシュ値を加えるだけではビルドエラーが解消できない場合があります。経験的には主な要因は以下の2つがありますが、実際にはよりビルドエラーにはバリエーションがあると思われます。

1. アプリケーションの配布元が brew-nix の対応していない圧縮形式を用いている（たとえば pbzx）
1. ディスクイメージファイル（`.dmg`）形式で配布している

前者は `microsoft-teams` 、後者は `google-drive` などで発生しました。この場合、`unpackPhase`を上書きして少しずつどのような形式で配布されているのか、何がビルドエラーになっているのかを根気よく突き止めていくしかありません。また、アプリケーションの更新によって以前まで動いていたアプリケーションのビルドが通らなくなることもあります。
以上の理由から、brew-nix を使うには Nix を用いたビルドの経験値が求められるので誰にでも勧められるツールではないように思えます。筆者は使ったことがないですが、nix-darwin には [`homebrew.casks.*`](https://nix-darwin.github.io/nix-darwin/manual/#opt-homebrew.casks) というオプションが用意されており、こちらは Nix 式から `Brewfile` を生成して `brew bundle` を実行することでアプリケーションをインストールします。brew-nix は derivation を生成するのでキャッシュ単位になるという点こそ優位ですが、`homebrew.casks.*` を使っても多くの場合は問題ないのだろうと思います。

### 素の Homebrew を使う

brew-nix を使ったインストールは1つ目の問題点であった、*GUI アプリケーションのインストール*を解決しますが、2つ目の問題点であった *別のアプリケーションをインストールするようなハブとなるアプリケーションのインストール* の解決策としては微妙です。というのも、brew-nix が生成するのはあくまで derivation なので、アプリケーションの内容（すなわちビット列）が少しでも変わるたびに異なる derivation として扱われます。アプリケーションは往々にしてファイルサイズが大きいためディスクを圧迫しますし[^brew-nix-size]、認証機能や別のアプリケーションをインストールするような機能を持っている場合、derivation が変わるたびにそれらの操作がリセットされてしまうのです。たとえば、私はスクリーンショットを撮影するために [CleanShot X](https://cleanshot.com/) を使っていますが、これは [Setapp](https://setapp.com/?srsltid=AfmBOooyxTXDaE8ZmK6Ts1_wn0t4k7nL-BLGP1a82VJ874PiTYR8wuhz) というサブスクリプション型のアプリケーションコレクションからインストールしています。`brew-api` を更新するたびに Setapp を介してインストールしたアプリケーションが見えなくなり、スクリーンショットが使えなくなるのは非常に不便です。また、個人で扱いたいアプリケーションでも nix-darwin を使うとグローバルにインストールされてしまうという問題もあります。そこでシステム構成ではなくホーム構成で、かつ素の Homebrew を使う方法について考えます。

[^brew-nix-size]: この問題は brew-nix のドキュメントでも触れられています。普段の作業に影響を与えるほどディスクサイズを圧迫するアプリケーションについてもやはり、brew-nix を使わない方が得策です

https://github.com/momeemt/config/blob/develop/nix/profiles/hm/homebrew/default.nix

Homebrew 自体のインストール、および `Brewfile` を適用するために、Home Manager の `home.activation.<name>` を使っています。これは、Home Manager の設定を適用して世代を切り替える際に実行されるアクティベーションスクリプトのフックです。Nix の宣言だけでは表現しづらい、外部ツールの初期化や状態を持つツールのセットアップなどの手続きを世代切り替えの際に実行できます。
Home Manager は世代ごとに何度も同じ処理を実行する可能性があるため、書き込みをしない検証段階と、書き込みをして良い段階に分けています。今回はファイルの書き込みを行う副作用を持っているため、`writeBoundary`（書き込み境界）の後で実行させる必要があります。

https://github.com/momeemt/config/blob/develop/nix/profiles/hm/homebrew/install-homebrew.sh

呼び出しているシェルスクリプトでは、Homebrew が存在しない場合にはインストールを行い、する場合にはスキップして `brew bundle` により `Brewfile` で指定されたアプリケーションを Homebrew にインストールさせています。こうすることで、個々のアプリケーションは Nix から切り離し、アプリケーションのインストール、更新が発生するトリガーは `darwin-rebuild` に揃えることができました。

## 🚧 シークレット管理（sops-nix）

dotfiles を運用する上で、シークレットをどのように管理するかは難しい問題です。以下のように3つの方法が選択肢に上がります。

1. そもそもシークレットを VCS で管理しない
1. 1Password や Bitwarden などの外部のパスワードマネージャから適用時に吸い出す
1. [sops](https://github.com/getsops/sops) などのツールを使ってシークレットを暗号化した上で VCS に載せる

上から下に進むほど手作業は減りますが、一方で運用ミスによる危険性が伴います。基本的にはどれも正しく管理されていれば安全ですので、[secretlint](https://github.com/secretlint/secretlint) などの機密情報をコミットできないように制限するツールを組み合わせて運用するのが良いかと思います。
Nix で dotfiles やインフラを管理する際には、[sops-nix](https://github.com/Mic92/sops-nix) というモジュールを使うのが便利です。

https://github.com/Mic92/sops-nix

`/nix/store` はすべてのユーザ、グループが読み出すことができるので、復号化したデータそのものや、それを含むスクリプトなどを derivation として扱うと機密情報の流出に繋がります。sops-nix は、あらかじめ指定した権限設定を基に、アクティベーションの際に復号化できます。使うには、`flake.nix` の `inputs` に以下の入力を追加します。

```nix :flake.nix
inputs = {
  sops-nix = {
    url = "github:Mic92/sops-nix";
    inputs.nixpkgs.follows = "nixpkgs";
  };
};
```

### システム構成で利用する



## ✅ Secure Shell の設定（OpenSSH）

Secure Shell とは暗号化と改竄検知、サーバ認証により安全に遠隔ログインやコマンド実行、ポートフォワーディングなどを行うためのプロトコルで、多くの場合 SSH と呼ばれます。SSH を実装したソフトウェアとしては [OpenSSH](https://www.openssh.org/) が広く使われており、クライアントの `ssh` や、サーバの `sshd` のほか、鍵生成のための `ssh-keygen` 、認証エージェントの `ssh-agent` 、ファイル転送のための `scp` や `sftp` などを提供しています。

https://www.openssh.org/

クライアント設定は Home Manager の [`programs.ssh.*`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.ssh.enable) から、サーバ設定は `services.openssh.*` から行えます。基本的にノートパソコンに対してリモート接続する運用を行っていないので、後者は NixOS に対してのみ設定しています。

### クライアント設定

`ssh` コマンドは、`~/.ssh/config` にホスト設定を書いておくと、ホスト名だけで接続を行うことができます。Home Manager では、[`programs.ssh.matchBlocks.<name>`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.ssh.matchBlocks) に与えた設定から config ファイルが生成されます。

https://github.com/momeemt/config/blob/develop/nix/profiles/hm/programs/ssh/default.nix

基本的に IP アドレス等が公開情報のサーバは VCS に載せて管理していますが、そうでないサーバは sops-nix によって暗号化したファイルを適用時に復号化し、`includes` により読み込ませています。現在は秘密鍵が指定したパス（`identityFile`）に存在することを要求していますが、できれば秘密鍵も暗号化して VCS に載せたいと思っています。しかし、作業ミスによって生まれる被害が自分が管理するサーバ以外にも及ぶため怖くてそれは実現できていません。sops の代わりに 1Password からクレデンシャルを利用する [opnix](https://github.com/brizzbuzz/opnix) というツールもあり、将来的にはそちらに移ることも考えています[^or-bitwarden-nix]。

https://github.com/brizzbuzz/opnix

[^or-bitwarden-nix]: パスワードマネージャには [Bitwarden](https://bitwarden.com/ja-jp/) を利用しているのでできればシークレット管理にもそれを使いたいのですが、調べた限りでは [sopswarden](https://github.com/pfassina/sopswarden) しか見つからず、これが信頼できる NixOS モジュールかどうかはまだ分かりません

### サーバ設定

基本的にはできるだけ厳しく設定を書くようにしています。パスワードやキーボードインタラクティブ系の認証方式をすべて切っていて、公開鍵認証に寄せています。また、許可ユーザを `momeemt` に絞っています。

https://github.com/momeemt/config/blob/develop/nix/profiles/hosts/services/openssh/default.nix

OpenSSH については勉強中なので [`sshd_config` のマニュアル](https://man.openbsd.org/sshd_config.5)を見ながらおおよその設定をしましたが、来年はもう少し解像度を高められると良いなと思っています。

https://man.openbsd.org/sshd_config.5

## LLM の管理と設定（mcp-server-nix）

## ネットワークの設定

## 🚧 Cloudflare Tunnel を通す

自宅の NixOS サーバは外から接続したい場合があります。私はネットワークを提供するタイプの集合住宅に住んでおり、ネットワーク回線を別に引いていないためグローバル IP アドレスからサービスを公開することができません。そこで、[Cloudflare]() が提供する [Cloudflare Tunnel]() を通すことでアクセスできるようにしています。
NixOS は、`services.cloudflared.*` で Cloudflare Tunnel が設定できるオプションを公開しています。

## ✅ NixOS module を定義する

システム構成やホーム構成を単に共通化したいだけの場合はファイルに切り出して `imports` にファイルパスの配列を渡すことで展開されます。一方で、ホストやユーザごとに共通化した内容を切り替えたい場合、具体的に言えば引数を渡したい場合があります。
その場合は `imports` で実現することは難しく、モジュールを作成する必要があります。

https://nixos.wiki/wiki/NixOS_modules

これまでは Home Manager や nix-darwin 等が提供するオプションに対して、何らかの値を割り当ててきました。それに対して、NixOS module は**オプションを定義する手段**を提供します。NixOS module は次の3つのパーツに分類できます。

```nix
{
  imports = []; # import したいモジュールのパスをリストで渡す
  options = {}; # オプションの定義
  config = {};  # オプションの定義を使って具体的に評価する値
}
```

`imports` は、これまで使ってきた `imports` と同様で、評価したい他のモジュールのパスをリストで渡します。基本的には設定の共通化に用いてきました。`options` は、オプションの定義です。すなわち、これまで出てきた `programs.zsh.enable` が `bool` 型の値を受け取る、`programs.zsh.initContent` が `string` 型の値を受け取る、などオプション1つ1つの性質について定義します。`config` は、そのモジュールで定義されたオプションが使用された際に、実際にどのような値に評価されるかを定義します。たとえば、`programs.zsh.enable` が `true` になった場合、Home Manager はユーザ環境で `zsh` が使えるようにインストールしてパスを通します。そのように具体的な評価を定義します。
では、具体例を見てみましょう。私は Home Manager や nix-darwin などの既存のモジュールに欲しい設定がなかった場合には NixOS module を定義するようにしています。以下に、[GNU Wget](https://www.gnu.org/software/wget/) のための NixOS module を示します。

https://github.com/momeemt/config/blob/develop/nix/modules/home/programs/wget/default.nix

これは、GNU Wget を XDG Base Directory に準拠させるために書きました。詳細を見ていきましょう。

### options

まずオプション名を決める必要があります。サービスではないソフトウェアの設定を行うので、Home Manager に倣って `programs.wget` としたいですが、将来的に Home Manager などのより著名なツールと競合する可能性があります。また、オプション名の命名規則を完全に一致させてしまうと、どれが自分で定義したオプションであるか見ただけでは判断できません。
そこで、`site.programs.wget` のように自分専用のモジュールであることが分かるようにしています。命名の仕方にルールはないので、`my.programs.wget` や、ユーザ名を付けて `momeemt.programs.wget` のようにしている方もいます。

オプションの性質は、`options.<module-name>.<field>` に代入する `mk` 系のオプションビルダーによって作成されるオプション属性セットによって規定されます。たとえば、[`lib.mkOption`](https://noogle.dev/f/lib/mkOption) 、[`lib.mkEnableOption`](https://noogle.dev/f/lib/mkEnableOption) 、[`lib.mkPackageOption`](https://noogle.dev/f/lib/mkPackageOption) などが存在します。
`lib.mkOption` は最も基本的なオプションビルダーで、その他のオプションは特定のオプションを簡単に書くためのラッパーです[^not-only-easy-mkHogeOption]。`enable` オプションは大抵の場合、 `bool` 型であり、デフォルト値が `false` に設定されます。そこで `lib.mkEnableOption` を使えば、`description` を渡すだけでオプションを作成できます。

[^not-only-easy-mkHogeOption]: `enable` や `package` など、慣習的にどのような性質を持っているかが期待されるオプション名はいくつか存在します。このようなラッパーは簡単にオプションが適用できるだけでなく、出すべきではない独自性を出さないためにも重要です。オプションを定義する前に、オプションビルダーの種類やどのような性質のオプションに使われるべきなのかを知るためにマニュアルに通した方が良いでしょう。現に、それを怠った筆者は、`lib.mkPackageOption` を使わずに `package` オプションを定義してしまっています

### config

前節で、`site.programs.wget.*` オプションをいくつか定義しました。今度は、これらにセットされる値を使って実際にどのような値を評価するかを決めていきます。まず、[`lib.mkIf`](https://noogle.dev/f/lib/mkIf) を用いて、`site.programs.wget.enable` が `false` である場合には空の属性セットを返すようにします。
次に、`true` である場合には以下の処理が行われるように属性セットを構成します。

- `wget` をユーザから使えるようにする
- 環境変数 `WGETRC` にパスを設定する
- `wgetrc` ファイルに、`hsts-file` のパスと追加設定を含めて規定のパスに保存する

仮にこのモジュールが依存としてどのモジュールも要求しないなら、上のような処理が行われる属性セットを構成するのは少し大変です。しかし、今は Home Manager に存在していないソフトウェアの設定をホーム構成に反映させることを目的にしているので、Home Manager のオプションを気兼ねなく利用できます。これらは [`home.packages`](https://nix-community.github.io/home-manager/options.xhtml#opt-home.packages) 、[`home.sessionVariables`](https://nix-community.github.io/home-manager/options.xhtml#opt-home.sessionVariables)、[`home.file.<name>.text`](https://nix-community.github.io/home-manager/options.xhtml#opt-home.file._name_.text) によって簡単に実現できます。

ここまでで、NixOS module を構成することができました。あとは、自分の dotfiles から自作のモジュールを import しておき、ホーム構成（`nix/home/`）等で設定すれば適用できます。
過去に、tmux の NixOS module を作成した際のスライドがあり、どちらかといえば tmux 側の解説に寄っていますが興味があればご覧ください。

https://speakerdeck.com/momeemt/tmux-nixnoshi-zhuang-wotong-sitexue-bunixosmoziyuru

## ✅ 壁紙を定期的に貼り替えるサービス（set-wallpapers）

前節で NixOS module を定義する方法について見てきました。本節ではその続きとして、自作スクリプトを定期実行サービスとして登録し、 NixOS module によって運用する例を紹介します。対象は macOS + nix-darwin です。後述の通り、AppleScript を利用するため Linux では動作しません。
私はジブリ映画が好きなのですが[^ghibli]、デスクトップ環境の壁紙を STUDIO GHIBLI が配布している画像に設定しています。

[^ghibli]: 好きではあるのですが本格的にジブリファンと言えるほど詳しいわけでもありません。すみません

https://www.ghibli.jp/info/013251/

macOS は OS のバージョン更新などで壁紙設定がリセットされることがあります。そこで、壁紙の設定も Nix 側の構成に含めて再現できるようにしたいと考え、壁紙を定期的に切り替える仕組みを作りました。このサービスがやっていることは単純で、次の2ステップです。

1. 画像一覧から壁紙候補を1枚選ぶ
1. その画像を macOS の壁紙として設定する

### 壁紙を貼り替えるスクリプト

壁紙の設定は [AppleScript](https://developer.apple.com/library/archive/documentation/AppleScript/Conceptual/AppleScriptX/AppleScriptX.html)、壁紙の選択はシェルスクリプトで行い、組み合わせています。

https://github.com/momeemt/config/blob/develop/nix/modules/host/services/set-wallpapers/set.applescript

AppleScript を使っている理由は、依存ライブラリが不要で最も簡単に壁紙を変更できるためです。実に7年ぶりです[^seven-years-ago]。

[^seven-years-ago]: 当時は英単語帳を [Numbers](https://www.apple.com/jp/numbers/) で管理していて、特定の条件を満たすテキストの色を変えるのに使っていた記憶があります

https://github.com/momeemt/config/blob/develop/nix/modules/host/services/set-wallpapers/main.sh

Nix 側から環境変数 `WALLPAPERS` を介して壁紙候補の画像パス一覧を `main.sh` に渡します。`main.sh` はその中からランダムに画像を1枚選び、 `set.applescript` を実行して壁紙を変更します。また、連続で同じ画像が選択されるのは避けたいので、`~/Library/Application Support/set-wallpapers` 配下に前回選んだ壁紙を状態ファイルとして保存し、次回実行時に参照するようにしています。

### パッケージの定義とサービスへの登録

上のスクリプト群は、`pkgs.writeShellScript`によって Nix の derivation として扱えるようにしています。そうすることで、スクリプトが `/nix/store` 配下にコピーされるため dotfiles リポジトリ内のファイルパスや作業ツリーの状態に依存しなくなります。したがって、適用後にリポジトリの内容が変わっても、サービスが参照する実体は変わりません。

https://github.com/momeemt/config/blob/develop/nix/modules/host/services/set-wallpapers/default.nix

壁紙の貼り替えを AppleScript で行っているため、この NixOS module も nix-darwin からのみ呼ばれることを前提としています。nix-darwin は、macOS 標準のサービス管理ソフトウェアである [launchd](https://github.com/apple-oss-distributions/launchd/tree/launchd-842.92.1) へユーザサービスを登録するための [`launchd.user.agents`](https://nix-darwin.github.io/nix-darwin/manual/#opt-launchd.user.agents) が用意されています。この設定で [`launchd.user.agents.<name>.serviceConfig.StartInterval`](https://nix-darwin.github.io/nix-darwin/manual/#opt-launchd.user.agents._name_.serviceConfig.StartInterval) を `300`（秒）に指定すると、ユーザ領域の launchd が5分ごとにスクリプトを実行します。結果として、ログイン後は壁紙が定期的に切り替わるようになります。

## 🚧 Docker イメージのビルド

あまり整備できていないのですが、GitHub Actions でホーム構成を適用した Docker イメージをビルドしています。dotfiles を公開しているユーザは、自分の dotfiles をインストールするための方法も併せて整備している場合があります。たとえば、[mathiasbynens/dotfiles](https://github.com/mathiasbynens/dotfiles) は、Git を使ったインストール方法や、Git を使わずにインストールするためのスクリプトを紹介しています。

https://github.com/mathiasbynens/dotfiles

```sh :mathiasbynens/dotfiles を Git を使わずにインストールする
cd; curl -#L https://github.com/mathiasbynens/dotfiles/tarball/main | tar -xzv --strip-components 1 --exclude={README.md,bootstrap.sh,.osx,LICENSE-MIT.txt}
```

しかし、Nix でシステム構成やホーム構成を管理する場合には、ユーザ名やシークレットなど、その dotfiles を整備したユーザ固有の情報が含まれることがあるため、そのまま他のユーザが使い回すことはできません[^dotfiles-by-others]。また、システム構成の適用を巻き戻すことは、ホーム構成の適用を巻き戻すよりも少し大変です。そこで、そのようなユーザ固有の情報を抜いてホーム構成のみを適用したコンテナイメージを固め、配布することで、部分的にではありますが、自分の環境を汚すことなく誰でも簡単に試せるようにしています。

[^dotfiles-by-others]: そもそも他のユーザの dotfiles をそのまま使いたくなることはあるのでしょうか。私はそう思ったことは正直一度もありませんが、試したくなることもあるのかもしれません

## `terraform/`の構成

## 仮想マシンを作成する

## `k8s/`の構成

## ノードを cloudinit から作成してクラスタに参加させる

## ✅ 2026年にやりたいこと

dotfiles について、来年に取り組みたいことについて説明します。

### ビルドワークフローを追加する

現在は、GitHub Actions でコンテナイメージのビルドや `nix flake check --all-systems` による検査のみを行っていますが、実際に macOS や NixOS でビルドできるかどうかは確認していません。事実、`inputs` の更新によって CI は通るが手元で反映しようとするとエラーが発生することは時折起こります。
そこで、ビルドワークフローを追加して、実際に反映可能な設定であるかどうかもチェックしたいと考えています。ただし、反映には大量のディスク容量を要求しますが、GitHub Actions で割り当てられるディスク容量には制限があり、普通にビルドすると上限に引っかかってしまいます。ワークフローが実行されるサーバの不要なファイル等を削除してギリギリまで切り詰めてビルドする手法について、2025年の3月に開催された [Nix meetup #2](https://nix-ja.connpass.com/event/342908/) にて [Sumi-Sumi](https://github.com/misumisumi) さんが発表されていたので興味があれば以下の資料を参照してください。

https://speakerdeck.com/misumisumi/tuo-xie-sinai-xuan-yan-de-masinhuan-jing-gou-zhu

### karabiner-elements

来年はできるだけマウスとトラックパッドを使うのをやめ、キーボードのみで生活したいと思っています。そのために、karabiner-elements を使ってトラックパッドを無効化したり、特定の操作をキーボードに割り当てようと思ったのですが、nix-darwin で提供されている [`services.karabiner-elements.*`](https://nix-darwin.github.io/nix-darwin/manual/#opt-services.karabiner-elements.enable) は上手く動作しません。

https://karabiner-elements.pqrs.org/

karabiner-elements が macOS の内部 API に依存しており権限を割り当てる必要があるのですが、これが失敗しているのが原因に見えています。少し複雑なので手をつけていなかったのですが、来年はキーボード周辺の設定も整理したいなと思っています。

### Linux デスクトップ環境の整備

今年度は研究室の PC に NixOS を入れたり、眠らせていた計算機に NixOS を入れるなど、NixOS をインストールする機会が多く、その度に GNOME のデスクトップ環境が付属した Graphical ISO image でインストールをしてきました。しかし、GNOME はプリインストールされる不要なソフトウェアが多く、できれば環境を自分でコントロールしたいなと思うようになり、Sway を入れました。これは実際使いやすそうで良かったのですが、普段のメインマシンの OS は依然として macOS であり、整備する機会があまりありませんでした。
[Hyprland](https://hypr.land/) や [niri](https://github.com/YaLTeR/niri) など、他にも興味がある Wayland compositor があるので来年は時間を見つけて整備したいなと思っています。

https://hypr.land/

https://github.com/YaLTeR/niri

### Windows デスクトップ環境の整備

大学生活を通して、Windows に触れる機会がほとんどありませんでした。2年生の春休みに取った特別講義で、[Designspark PCB](https://www.rs-online.com/designspark/pcb-software-jp) というプリント基板 CAD ソフトウェアを使ったことがあり、大学から貸与された Windows マシンに触れましたが慣れず終いでした。WSL2 にせよ、セキュリティキーにせよ、PowerShell にせよ、[Windows Internals](https://www.amazon.co.jp/dp/0735684189/) にせよ、気になる機能や書籍が色々とあるので触ってみたいと最近思うようになりました。
Windows Subsystem for Linux で NixOS を実行するためのモジュールも用意されていますので、こちらについても試してみようと思っています。

https://github.com/nix-community/NixOS-WSL

### k8sクラスタを充実させる

学部生の間に [Kubernetes](https://kubernetes.io/) を触りたいと思っていて、今年ギリギリ滑り込みでクラスタを立てました。ですが、まだクラスタの作成と [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) 程度しか動いておらず、分散ストレージ（[Rook](https://rook.io/)/[Ceph](https://ceph.io/en/)）のデプロイや、[Mastodon](https://github.com/mastodon/mastodon) や [Mattermost](https://mattermost.com/) の SNS アプリケーション、[Bitwarden](https://bitwarden.com/ja-jp/help/self-host-bitwarden/) や [Wakatime](https://github.com/muety/wakapi) のセルフホストなど、色々と運用してみたいシステムやアプリケーションがあります。分散ストレージの理解が進まなくてハングしているのですが、知人曰く「自宅で運用する場合には辛さやパフォーマンスの割に恩恵がない」らしいので、一旦妥協して次に進んでみた方が良いかなとも思っています。
Kubernetes 自体の学習がしたかったので使っていませんが、NixOS は [k3s](https://github.com/NixOS/nixpkgs/blob/master/pkgs/applications/networking/cluster/k3s/README.md) をサポートしていますし、実際に運用されている方がいますので参考になりやすいはずです。

https://www.docswell.com/s/ushitora-anqou/Z4473L-2025-10-11-141944

https://speakerdeck.com/ichi_h3/nixos-plus-kubernetesdegou-zhu-suruzi-zhai-sabanosubete

### config 管理用のツールを実装する

現在はシェルスクリプトを書いて Just で呼び出しているのですが、どの機能を有効化するかインタラクティブに決定しながら新しいホストやユーザを追加するツールを作りたいと考えています。しかし、肥大化したシェルスクリプトを管理していける自信がないので、config を管理するためのツールが欲しいなと思っています。発想としては、新しいパッケージの Nix 式を生成する [nix-init](https://github.com/nix-community/nix-init) に近いです。

https://github.com/nix-community/nix-init

# 終わりに

もし私の dotfiles に興味を持っていただけた方は、ぜひリポジトリにスターを付けたり、コンテナイメージを試したり、コメントで感想を送ったりしていただければ励みになります！

https://github.com/momeemt/config

メリークリスマス、良いお年を。
