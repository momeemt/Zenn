---
title: "2025年のdotfiles"
emoji: "🌳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dotfiles", "Nix", "Kubernetes", "Terraform"]
published: true
---

# はじめに

こんにちは。ソフトウェアエンジニアの生産効率は、高性能なマシンや質の良いキーボード、さらには日々の運動や栄養バランスの取れた食事などに支えられていますが、中でもポータブルな設定ファイル集である **dotfiles** にはそれぞれのエンジニアが持つ計算機環境へのこだわりが凝縮されているのではないでしょうか。
この記事では、まず dotfiles とそれを取り巻くシステムの変遷について説明して、その後に今年2025年に筆者が整備した config（dotfiles、Terraform、Kubernetes 設定のモノレポ）について紹介します。

https://github.com/momeemt/config

:::message
これは[Nix Advent Calendar 2025](https://adventar.org/calendars/11657)の25日目の記事です。遅刻しました、ごめんなさい！遅ればせながら、メリークリスマス！🎄
:::

# dotfiles

:::message
すでに dotfiles を管理・運用されていて、私の dotfiles の内容にのみ興味がある方はスキップして [momeemt/config](#momeemt%2Fconfig) から読んでください。
:::

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

[chezmoi](https://github.com/twpayne/chezmoi) は、複数マシンにまたがって dotfiles を管理するためのツールです。単なるシンボリックリンクを張るスクリプトを運用するのに比べて、テンプレートによる環境差分の吸収や、パスワードマネージャとの連携、ファイル全体の暗号化、スクリプトの実行などを標準機能として持っています。

https://github.com/twpayne/chezmoi

基本的にはリポジトリを初期化して差分を確認し、dotfiles を環境に適用する、という[流れ](https://www.chezmoi.io/user-guide/command-overview/)になります。

```sh
chezmoi init <repo>
chezmoi diff
chezmoi apply -v
```

chezmoi は、内部的には dotfiles を管理用ディレクトリ（source state）に保持し、そこから実際にホームディレクトリに配置される成果物（target state）を計算して反映します。
また、[テンプレート](https://www.chezmoi.io/user-guide/templating/)によって環境の差分を吸収することができます。拡張子が `.tmpl` であるファイルは、Go 言語の `text/template` 形式として扱われ、`.chezmoi.os` のような変数を使って OS ごとに内容を切り替えられます。テンプレート変数は、`chezmoi data` コマンドを実行することで確認できます。
シークレット管理はファイル全体の暗号化を行ってリポジトリに乗せるか、[1Password CLI などを利用して](https://www.chezmoi.io/user-guide/password-managers/1password)パスワードマネージャと連携することで、テンプレート内でパスワードマネージャの項目を読み出す関数を使って値を埋め込むことによって実現します。

chezmoi の概要を掴むために以下の記事が参考になりました。

https://zenn.dev/ryo_kawamata/articles/introduce-chezmoi

### GNU Stow

[GNU Stow](https://www.gnu.org/software/stow/) は、別々のディレクトリに置かれたパッケージを1つのディレクトリツリーにインストールされたように見せるためにシンボリックリンクを張るツールです。もともとはソースからビルドした複数のソフトウェアを衝突させずに管理するために開発されましたが、ホームディレクトリの設定ファイル管理にも用いられています。

https://www.gnu.org/software/stow/

dotfiles の管理ツールとして使う場合には、リポジトリ直下にツールごとにディレクトリを作成し、その中にホームディレクトリからの相対パスでファイルを作成します。その後、以下のコマンドを実行することでシンボリックリンクを作成できます。

```sh
cd dotfiles
stow -t ~ vim git zsh
```

Stow は内部状態を保持せず、今あるディレクトリ構造からシンボリックリンクを計算するだけのシンプルなツールです。ドライラン、削除、再作成などの機能が提供されています。また、dotfiles 向けの機能として `--dotfiles` オプションが提供されています。これは、ドットファイル（たとえば `.vimrc`）をそのまま置く代わりに、`dot-` 接頭辞で（たとえば `dot-vimrc`）管理して、適用時に `.` に自動で変換してリンクを張ることができます。
既存のファイルが存在する場合には衝突として終了しますが、`--adopt` オプションによって既存ファイルをリポジトリ側に取り込むことができます。

### dotfiles 管理ツールの限界

他にも、[yadm](https://yadm.io)、[dotdrop](https://github.com/deadc0de6/dotdrop)、[Dotbot](https://github.com/anishathalye/dotbot)、[rcm](https://github.com/thoughtbot/rcm)、[homeshick](https://github.com/andsens/homeshick)、[Ellipsis](https://github.com/ellipsis/ellipsis)などの同様のツールが存在します。自分が扱う dotfiles の規模や要求に応じて、適切なツールを選択していくと良いでしょう。
一方で、これらの管理ツールではカバーできないことも存在します。

- 設定を管理することはできますが、コマンドやプラグイン自体は別の手段で管理する必要がある
  - バージョンの差やそのソフトウェアが他のツールがインストールされていることを前提にしていることにより必ずしも同じ環境を再現することは難しい
- 適用が Atomic ではなく、事前確認はできるがトランザクションとして巻き戻せる設計ではない

このような要求に対しては、設定ファイルの管理ツールとパッケージマネージャが同一か、あるいは協調して動作できるソフトウェアが必要です。

## Home Manager を利用する

dotfiles 管理ツールは設定ファイルを所定の場所に置くことを中心に責任を持っていますが、ソフトウェアそのもの（バイナリ、プラグイン、依存関係のバージョン）をインストールすることは難しいです。これも含めて管理したい場合には [Home Manager](https://github.com/nix-community/home-manager) が便利です。Home Manager は、ビルドシステムの Nix を用いたホーム構成の適用ツールです。

https://github.com/nix-community/home-manager

Home Manager を始めとした Nix のエコシステムを利用して dotfiles を管理すれば、以下の要求をすべて満たすことができます。

- ビルドマニフェストを記述するドメイン固有言語である Nix を使って、環境差を吸収する記述を行えます
- 適用が Atomic で、ロールバックや冪等性、ファイル退避の処理が考慮されています
- [sops-nix](https://github.com/Mic92/sops-nix) など、シークレット管理をするためのモジュールが提供されています
- ビルドシステムおよびパッケージマネージャとして働く Nix と協業して、ソフトウェアそのものを密結合に管理できます
  - Nix を用いてパッケージを `/nix/store` 配下にインストールできます
  - Neovim の設定だけでなく、Neovim が環境で利用可能であるかについても概ね保証できます

インストール方法については、以下のマニュアルを参照してください。

https://nix-community.github.io/home-manager/

### Nix

Home Manager は Nix を利用して作られたソフトウェアであり、扱うには Nix への理解が必要です。Nix は近年少しずつ注目されている、ビルドシステム（兼パッケージマネージャ、兼ドメイン固有言語）です。

https://nixos.org/

Nix はそれぞれの機能をまとめて提供するソフトウェアで、以下のような3つの側面を持っています。

- ドメイン固有言語（DSL）
  - ビルドを記述するためのドメイン固有言語で、動的型付き純粋関数型言語です
    - 遅延評価モデルを採用しています
  - Nix 式を評価すると、入出力やビルドの実行方法が記述された derivation という中間表現が得られます
- ビルドシステム
  - DSL によって記述された入出力、ビルドの実行方法から、成果物を得るためのプロセスを実行します
  - 事前に明示されないネットワーク[^fixed-output-derivation]や環境変数などへのアクセスを禁止したサンドボックス内でビルドを行います
  - 入力として指定されたオブジェクトの読み取りと、指定された出力への書き込みのみがデフォルトで許される純粋関数的なビルドを提供します
    - 実際にはパッケージ固有の性質やビルド過程の要因が絡むために完全な純粋関数を実現することは難しいですが、このようなサンドボックスを提供しないビルドシステムと比べて高い再現性を実現します
    - [Reproducible builds](https://reproducible-builds.org/) （どんな環境であれビットレベルで同一の成果物を得るためのビルド）というより強い考え方も存在します
- パッケージマネージャ
  - ビルドシステムによってビルドされた成果物を取得するための機能を提供します
  - プログラミング言語に付属するビルドシステム（[Cargo](https://github.com/rust-lang/cargo) など）はパッケージマネージャと一体化していることは珍しくありません
  - Nix の公式が提供する [nixpkgs](https://github.com/NixOS/nixpkgs) は現在12万件以上のパッケージ定義を提供する最大級のパッケージリポジトリですが、これに限らずサードパーティのパッケージリポジトリからもパッケージを入手できます
  - Arch Linux のユーザリポジトリ [AUR](https://aur.archlinux.org/) に近い仕組みである、[NUR](https://github.com/nix-community/NUR) が存在します

[^fixed-output-derivation]: 現実的にネットワークアクセスを完全に禁止してビルドを行うことはできません。そこで、[ネットワークを介して入手するリソースのハッシュ値を事前に与えることで再現性を担保することを許可する仕組み](https://nix.dev/manual/nix/2.32/store/derivation/outputs/content-address.html)が存在します

https://github.com/nix-community/NUR

プログラミング言語に付属するビルドシステムは特定の言語や環境を想定しています。たとえば Cargo は Rust で書かれたプログラムをビルドしますし、Nimble は Nim で書かれたプログラムをビルドします。しかし、Nix は汎用的なビルドシステムであり、「事前に与えられた入力から情報を読み取り、指定された出力への書き込みを行う」というルールさえ守ればどんなものでも対象として扱うことができます。
ビルドシステムやパッケージマネージャと dotfiles には一見関係があるようには見えないので、抵抗感を持つ方も少なからずいらっしゃるかと思いますが、Home Manager は、事前に与えられた入力から情報を読み取り、ホーム構成を出力することに特化した、Nix のルールの上に成り立つソフトウェアであると考えられます。

この記事では Nix の文法については扱いません。Nix について学びたい場合は、[asa1984](https://zenn.dev/asa1984) さんの [Nix入門](https://zenn.dev/asa1984/books/nix-introduction) や [Nix入門: ハンズオン編](https://zenn.dev/asa1984/books/nix-hands-on) がおすすめです。

https://zenn.dev/asa1984/books/nix-introduction

https://zenn.dev/asa1984/books/nix-hands-on

### NixOS

ホーム構成が Nix で扱えるならシステムサービスや OS の設定なども Nix で行えた方が便利です。これを実現したのが、Nix をベースにした Linux ディストリビューションである [NixOS](https://nixos.org/manual/nixos/stable/) です。NixOS は Nix によるパッケージ管理を前提に、nixpkgs に含まれるモジュールとパッケージを組み合わせてシステム全体を構成します。

https://nixos.org/manual/nixos/stable/

NixOS の設定は基本的に `/etc/nixos/configuration.nix` を編集して、`nixos-rebuild switch` コマンドを使って反映する、という流れになります。1回の反映で generation が生成され、それを切り替える形で運用を行うため、設定を変更した履歴が残ります。`nixos-rebuild switch --rollback` コマンドによって、直前の generation に戻すこともできます。ブートローダから過去の generation を選んで起動できるため、もし設定を誤って起動できなくなったとしても、過去の環境を復旧させるのがとても簡単です。
現実的には NixOS にもデフォルトでは状態を持つ領域があり、宣言的に管理する範囲をどこまで広げるかを設計や運用によって決めていくことができます。

NixOS を実際に導入して各項目を設定していくなら以下の記事がおすすめです。Nix の性質や Nix Flakes、NixOS のインストール、IME・フォントの設定、ドライバ、Home Manager の導入、デメリットに至るまで細かく説明されています。この記事では Nix、NixOS 自体の詳細な説明は割愛していますので、こちらを参照してください。

https://zenn.dev/asa1984/articles/nixos-is-the-best

# momeemt/config

ここからは2025年までに筆者が整備した dotfiles について紹介していきます。

## 目標と選定基準

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
  - また、現状の抽象化の結果を他者から見える形で残すことは、誰かがより良い抽象化を発見することへ繋がると思っています
- プログラマの責任を委譲できるシステムを選ぶ
  - 書いたのが自分であれ、時間が経てば他人が書いたコードのようにその意図を読み取るのが難しくなります
  - それは手順も同じであり、質の良い手順書を維持しなければ環境を再現できなくなる可能性がある状態は望ましくありません。できるだけプログラマの責任を減らしてツールに責任を持ってもらい、記憶に頼らなくて良いようにします

## ディレクトリ構成

momeemt/config[^config-naming] は、以下のようなディレクトリに分けています[^dir-ommision]。

[^dir-ommision]: 説明が不要な一部のファイルやディレクトリは省略しています。また、これ以降に示すファイル、ディレクトリ一覧についても同様です
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

このリポジトリは、いわゆる dotfiles に加えて、ホスト構成や自宅インフラをまとめて管理するためのモノレポです。この運用は、以下のような利点があります。

- 整合性のある変更単位を作成できる
  - ホスト構成とインフラは実運用上で相互依存しやすく、1つのコミットで同時に変更できます
  - `flake.lock` により依存（nixpkgs など）も固定できるため、どの組み合わせによって正しく動作するのかをコミット単位で追跡できます
- 開発環境を固定できる
  - `flake.nix` を基に、Terraform / kubectl / sops などのツールチェーンを同じバージョンで揃えられます
  - ディレクトリごとに別々の手順・別々のバージョン管理を持ち込まずに済みます
- シークレットの参照点を一元化できる
  - Nix / Terraform / Kubernetes のいずれからもシークレット（鍵・トークン・証明書など）を参照したくなりますが、モノレポにしておけば参照箇所と更新履歴を追いやすく、ローテーションもしやすくなります
- 探索性が上がる
  - ホスト名、ドメイン、ポート、証明書、クラスタ名などをリポジトリ内検索で辿れるため、設定の所在が分かりやすくなります
- 段階的に Nix に寄せていきやすい
  - 現時点で Nix 管理に乗っていない領域（たとえば `nas/`）も同じリポジトリ内に置いておけるため、移行の途中経過を含めて管理できます

## `nix/`の構成

`nix/`ディレクトリ内では、以下のようなディレクトリに分けています。

```sh
.
├── flakes       # flake.nix の定義ファイルをモジュール化したファイル
├── home         # 各ホストのユーザ空間の設定（Home Manager）
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
先述した Home Manager はホーム構成を適用するツールであり、この節で扱うツールはシステム構成を適用するツールです。システム構成とユーザ環境には以下のような違いがあります。

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
dotfiles で記述したいソフトウェアの設定などは基本的に Home Manager から設定を行う場合が多いため、システム構成の設定量よりも多くなる傾向にあると思います。
一方で、Home Manager が OS ごとの差異を吸収してくれているため、システム構成よりは環境差を意識することが少ないように感じます。

### flake modules (flakes)

Nix Flake は基本的に `flake.nix` の `outputs` にすべてを書けますが、`packages` や `devShells` などの定義する属性が増えてくると、ファイルが肥大化しやすく、見通しも落ちます。[flake-parts](https://github.com/hercules-ci/flake-parts) を使うと、Flake の `outputs` を NixOS の module system に似た形でモジュールとして分割、合成することができます。flake-parts 自体は Flake のスキーマを最小限に記述することを目標にしており、標準的な flake 属性に対応する option とシステムを扱うための枠組みを提供します。

https://github.com/hercules-ci/flake-parts

同様に Flake の記述を楽にする道具として、[flake-utils](https://github.com/numtide/flake-utils) もあります。flake-utils はユーティリティ関数によってシステムごとの出力を生成することが目的です。たとえば `eachDefaultSystem` は、`defaultSystems`[^default-systems]に対してシステムごとの属性セットを構築します。筆者が flake-parts を選んでいる理由を以下に示します。

[^default-systems]: `x86_64-linux`、`aarch64-linux`、`x86_64-darwin`、`aarch64-darwin` の4つのシステム

- `outputs` をモジュールシステムとして分割できる
- `system` の扱いが明示的になる
- option とデバッグ機構を提供する
- flake-parts に対応した flake が多くあり、記述が全体的に楽になる

次のファイルでは、NixOS、並びに macOS のホスト設定を宣言しています。追加で読み込むモジュールやオーバレイなどが共通していることが多いため、`nixosSystem` や `darwinSystem` をラップする関数を定義して、それを呼び出しています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/flakes/hosts.nix

ラップした関数はこちらで定義しています。flake の入力やシステム、後述する config 用のライブラリを渡して、プロファイルやモジュールから参照できるようにしています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/flakes/lib/hosts.nix

また、`nix flake check` によって、コードの品質を担保するための `treefmt` や `pre-commit` などを実行しています。これらは [treefmt-nix](https://github.com/numtide/treefmt-nix) や [git-hooks.nix](https://github.com/cachix/git-hooks.nix) などの flake がそれらのプロジェクトルートにある `flake.nix` から `flakeModule` を提供していることによって、flake-parts からモジュールとして扱えるようになっています。

https://github.com/numtide/treefmt-nix

https://github.com/cachix/git-hooks.nix

### ライブラリ（lib）

Nix の設定が大きくなるにつれて、config 内で共通化できる部分が増えていきます。たとえばいくつかのシステム構成から nixpkgs のバージョンを参照する必要がありますが、それをライブラリから提供しています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/lib/default.nix

### オーバーレイ（overlays）

パッケージ集合などの属性を評価する時に差し替えたり追加する仕組みのことを[オーバレイ](https://wiki.nixos.org/wiki/Overlays)と言います。nixpkgs に対して、自分で作成した、あるいはサードパーティが提供するオーバレイを適用できます。ただしネストした属性は再帰的にマージされない点には注意が必要です。
私は小さなスクリプトなど、`nix/packages/` 以下に定義したパッケージのオーバレイを用意したり、後述する brew-nix にパッチを当てるためのオーバレイを用意しています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/overlays/packages.nix

また、現状ではほとんど手を付けられていませんが、自分の NUR からもパッケージを読み込んでいます。

https://github.com/momeemt/nur-packages

### プロファイル（profiles）

複数のホストやホームから再利用したい設定を、`nix/profiles/` 以下にまとめています。`nix profile` サブコマンドや、`.nix-profile` とは関係がありません。各ホスト、ないしはホームの `imports` にパスを渡すと設定が適用できます。

```sh
├── hm                 # Home Module
│   ├── accounts
│   ├── editorconfig
│   ├── files
│   ├── homebrew
│   ├── nix
│   ├── programs
│   ├── services
│   ├── sops
│   ├── wakatime
│   └── wayland
└── hosts              # NixOS / nix-darwin
    ├── default.nix
    ├── environment
    ├── fonts
    ├── networking
    ├── security
    ├── services
    ├── system
    └── time
```

ディレクトリ構成はまだ曖昧な部分もありますが、`hm` と `hosts` についてはオプション名に合わせてディレクトリを切っています。たとえば、Home Manager には Linux デスクトップを sway によって設定するオプションとして `wayland.windowManager.sway` がありますが、この設定は `hm/wayland/windowManager/sway/default.nix` に配置しています。こうすることで、設定ファイルを新しく作成する際にパスを迷う必要がなく、かつ `imports` にパスを書く際に間違えにくいというメリットもあります。
また、`hosts` については `nixosSystem` と `darwinSystem` の設定が混在しています。これらが提供するオプションは共通したものもありますが、片方にしか存在しないものもあります。そこで、`imports` の内容を OS によって切り替えて、`default.nix` を読み込めば OS の差異を意識せずに設定を反映できるようにしています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hosts/environment/default.nix

### モジュール（modules）

[NixOS module を定義する](#nixos-module-を定義する) で後述しますが、Home Manager などが提供するモジュールが対応していないソフトウェアや設定のオプションを自前で NixOS module を作成して対応しています。`/nix/modules/` 以下に定義ファイルがまとめられており、[ホーム構成](https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/flakes/lib/shared-modules.nix#L5)や[システム構成](https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/flakes/lib/hosts.nix#L27)からモジュールとして読み込んでいます。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/modules/home/default.nix

### テンプレート（templates）

Nix Flakes は、テンプレートから flake の雛形を生成できる `nix flake init` というコマンドを提供しています。これはテンプレートのファイル群をカレントディレクトリにコピーして初期化するもので、既存のファイルは上書きしません。

https://nix.dev/manual/nix/2.18/command-ref/new-cli/nix3-flake-init


テンプレートは、`outputs.templates` で設定しておくことでリポジトリ内の対象ディレクトリがコピーされます。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/flakes/templates.nix

筆者の場合には、`nix flake init -t github:momeemt/config#rust` を実行すると、以下のファイルを含めたディレクトリがコピーされます。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/templates/rust/flake.nix

テンプレートはまだ整備が行き届いていませんが、ユーザ入力によって変数を渡して内容を切り替える機構が含まれておらず、個人的にはインタラクティブに、あるいはオプションによって条件を切り替えてファイルを動的に生成したいと思っているので、今後もあまり真面目に整備しないかもしれません。そのような目的の場合には Nix Flake を作成するか、あるいは [devenv](https://devenv.sh/) を使うか、Nix 式を動的に生成するようなプログラムを実装するのが良いのかと思います。どれも試したことがないので、来年度は時間を見つけて整えていきたいと思っています。

https://devenv.sh/

## シェルの設定（zsh）

シェルは macOS のデフォルトシェルである [zsh](https://zsh.sourceforge.io/) を利用しています。他の [bash](https://www.gnu.org/software/bash/) や [fish](https://fishshell.com/)、[nushell](https://www.nushell.sh/) などと比べて以下のような理由で、バランスが良く使いやすい点を気に入っています。

- bash と比べて補完や履歴などの機能が充実している
- fish や nushell と比べて POSIX 系のシェル文法と親和性が高い[^why-posix]

[^why-posix]: 私が POSIX で定義された範囲のシェルですら満足に使いこなせていないためにこのような選択をしているだけで、大きな疑問がなくなったら nushell を使ってみたいと思っています

### 基本的な設定

zsh の基本的な設定は Home Manager の `programs.zsh` から行っています。zsh についてはオプションが充実しているのと、私が zsh をきちんと勉強できていないため、ほとんど Home Manager に閉じています。

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

補完やシンタックスハイライトは有効にしています。このような定型的だがいくらかスクリプトを書かなければいけない機能やその有効化を簡単に行うことができるように抽象化レイヤを提供してくれている点は、Home Manager の良いところの1つです。

#### キーマップ

メインエディタとして Neovim を利用しているのでデフォルトキーマップは Emacs を選んでいます。一方で同じエディタを使っている友人はシェルのキーマップに Vimcmd [^vimcmd]を利用していました。[toggleterm.nvim](https://github.com/akinsho/toggleterm.nvim) と競合すると思っていたのですがどうやらさほど問題ないようで、来年はそちらに移っても良いかなと思いました。一方で、macOS のアプリケーションは Emacs キーバインドが効くことが多く[^macos-cocoa]、練習した方が他のソフトウェアの操作効率も上がるかなとも思うので、悩みどころです。

#### XDG Base Directory に準拠する

[XDG Base Directory](https://specifications.freedesktop.org/basedir/latest/) とは、設定やキャッシュ、データなどの保存先を標準化することでファイルを整理するための仕様です。
zsh はデフォルトでは各設定ファイルをホームディレクトリ配下に配置しますが、環境変数 `ZDOTDIR` を設定することで設定ファイルをそのディレクトリ配下に収めることができます。
Home Manager では `programs.zsh.dotDir` を適切に設定することで XDG Base Directory に準拠できます。`config.xdg.configHome`  `$XDG_CONFIG_HOME` の値を持つ Nix 式です。

#### ディレクトリ候補の探索パスの追加

`programs.zsh.cdpath` にパスを追加しておくと、`cd` の補完候補として探索されるディレクトリが増えます。
私はリモートから入手した Git リポジトリを管理するツールとして [ghq](https://github.com/x-motemen/ghq) を使っています[^ghq-naming]。このツールは環境変数 `GHQ_ROOT` を設定することで[リポジトリを配置するパスを変更でき](https://github.com/x-motemen/ghq#environment-variables)、私は `$XDG_DATA_HOME/ghq` を指定しています。そのせいで新しく配置したリポジトリに移動するのが非常に面倒であり、楽にタイプするために `cdpath` に GitHub リポジトリ配下と、その中でも私のユーザ名のディレクトリ配下をしています。
そうすることで、以下のように補完が効くようになり、インタラクティブシェル内での体験を改善できます。

[^ghq-naming]: 断りなく会話に出すと知らない人からはギョッとされることが多いが、連合国軍最高司令官総司令部のことではない

一方で、類似した体験を提供するコマンドとして最近は [zoxide](https://github.com/ajeetdsouza/zoxide) が話題になりました。これは過去に移動したディレクトリを記憶しておき、絶対パスや相対パスだけではなくディレクトリ名のみで移動できるようにするコマンドです。
私は `cd` コマンドを `z` にエイリアスした上で、次のように使い分けています。

- 過去に移動したディレクトリは zoxide でジャンプする
- 過去に移動していないディレクトリが多く含まれているディレクトリは `cdpath` に追加する

GitHub のタイムラインで見かけた知らないソフトウェアの Git リポジトリを手元に落として試したり、調べたりすることは多くあるので、`cdpath` の対象にしておくと便利です。

#### 名前付きディレクトリを定義する

`programs.zsh.dirHashes` から zsh で名前付きディレクトリを設定できます。`dirHashes` を設定しておくと、`~github` や `~momeemt` のように名前付きディレクトリとして参照できるようになります。

[^vimcmd]: zsh の vi キーバインドは `viins`（挿入モード）と `vicmd`（コマンドモード）の2つのキーマップを行き来します。Home Manager の `defaultKeymap` は、`emacs` か、先ほどの2つのモードのいずれかです。
[^macos-cocoa]: macOS 向けアプリケーションフレームワークの [Cocoa](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CocoaFundamentals/WhatIsCocoa/WhatIsCocoa.html) が Emacs のキーバインドをデフォルトで採用していたため

### 設定ファイルに直接追記する

基本的にはオプションに割り当てた Nix の値をもとに設定ファイルを生成することが基本ですが、シェルスクリプトや設定のために利用するその他のプログラミング言語[^prog-lang-for-config]は OS やソフトウェアの状態を基にした複雑な条件分岐を表現できますから、あらゆる状態を静的なオプションに落とすのは原理的に難しいです。
そこで、多くのソフトウェアに対するオプションには設定ファイルを文字列で追記する形で自由に記述できるオプションが提供されています。zsh の場合は、`profileExtra` が `.zprofile` に対する追記に、`initContent` が `.zshrc` に対する追記に対応します。

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

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/programs/shells/zsh/history/default.nix

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
`flake.nix` の `outputs` には、`packages.<package>` という derivation を取るフィールドが存在しており、`nix run ".#<package>"` を実行することでパッケージをビルドできます。デフォルトでは補完が提供されていませんが、`nix flake show --json` によって `flake.nix` の情報を JSON によって取得できます。以下のようなコマンドを補完スクリプトとして読み込むことで、`flake.nix` を参照していちいちどのようなパッケージが定義されているか確認しなくても良くなります。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/programs/shells/zsh/completions/_nr

### nixpkgs 由来の zsh を利用する

macOS はデフォルトシェルとして zsh を採用しているため、初期状態から `/bin/zsh` に zsh を配置していますが、バージョンが同じでも OS のアップデートによりビルドオプションや依存ライブラリ、パッチなどが変更される可能性があるため、macOS 標準の zsh ではなく nixpkgs からインストールした zsh を利用したいです。
macOS ではログインシェルがユーザアカウントの属性として管理されており、単に `PATH` の先頭に nixpkgs の zsh を置くだけではデフォルトシェルを変更することはできません。nix-darwin では `users.users.<name>.shell` にパッケージを指定することで、ユーザのログインシェルを宣言的に設定できます。
また、macOS はログインシェルとして許可する実行ファイルの一覧を `/etc/shells` に持っているため、nixpkgs 由来の zsh をログインシェルにする場合は `environment.shells` に `pkgs.zsh` を追加しておく必要もあります。

### zsh についての資料

基本的には Home Manager マニュアルの `programs.zsh.*` オプションの説明やソースコードを参照してください。

https://nix-community.github.io/home-manager/options.xhtml#opt-programs.zsh.enable

最後まで読めていませんが、zsh の技術書や zsh マニュアルに目を通すこともあります。

https://gihyo.jp/dp/ebook/2022/978-4-297-12730-5

https://zsh.sourceforge.io/Doc/Release/zsh_toc.html

## エディタの設定 (Nixvim, VSCode)

エディタには Neovim と VSCode を利用しています。主に Markdown や Typst など、プレビューを見ながら文章を書きたい場合には VSCode を利用していて、それ以外は Neovim を利用しています。Home Manager では、[`programs.neovim`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.neovim.enable) と [`programs.vscode`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.vscode.enable) オプションによって設定を適用できます。

### Nixvim とは

Home Manager を使い始めた頃は `programs.neovim` を使って Neovim を設定していましたが、現在は Nixvim を使っています。

https://github.com/nix-community/nixvim

Nixvim は Nix のモジュールとして Neovim の設定を記述するための仕組みで、flake として配布されています。設定は Nix で行いながら、必要に応じてプラグインを追加したり Lua で直接スクリプトを書き足すことができます。Home Manager の場合はプラグインは nixpkgs から取得しますが設定は Vim script ないしは Lua で書く必要があります。一方、Nixvim の場合にはプラグインの設定まで宣言的に行うことができます。たとえば、カラースキーマとステータスラインを追加するような設定を以下の Nix 式によって記述できます。

```nix
{
  programs.nixvim = {
    enable = true;
    colorschemes.catppuccin.enable = true;
    plugins.lualine.enable = true;
  };
}
```

これにより [lualine](https://github.com/nvim-lualine/lualine.nvim) は[それなりに妥当なデフォルト設定](https://github.com/nix-community/nixvim/blob/66a5dc70e2d8433034bccdbb9c3c7bcecd86f9a6/plugins/by-name/lualine/default.nix)で初期化され、カラースキーマとの連携も含めて追加の Lua 設定なしに動作させることができます。
Nixvim は Home Manager のモジュールをはじめ、nix-darwin、NixOS のモジュールが提供されており、スタンドアロンにビルドして利用することもできます。筆者は Home Manager のモジュールとして利用しています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/nixvim/default.nix

Nixvim はデフォルトでは nixpkgs の Neovim をパッケージとして利用していますが、筆者は copilot-language-server を使用するために [neovim-nightly-overlay](https://github.com/nix-community/neovim-nightly-overlay) が提供している開発版の Neovim 0.12 を使っています。

https://github.com/nix-community/neovim-nightly-overlay

これは Neovim 0.12 に [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) の [`textDocument/inlineCompletion`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.18/specification/#textDocument_inlineCompletion) が入ったことによるもので、以下の記事を参考に設定しました。

https://zenn.dev/vim_jp/articles/a6839f7204a611

### Nixvim に入れているプラグイン

Neovim については学習中で、設定しているキーマップも[矢印を無効にした](https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/nixvim/keymaps/disable-arrow/default.nix)り、[`jj`をESCに割り当てた](https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/nixvim/keymaps/exit/default.nix)りしている程度です。この節では、Nixvim に入れているプラグインについて、Nix で設定することで大きな恩恵があったものを中心にいくつか紹介します。入れているプラグインは以下の通りです。

- [nvim-cmp](https://github.com/hrsh7th/nvim-cmp)（補完）
- [hop.nvim](https://github.com/smoka7/hop.nvim/)（Easymotion系）
- [indent-blankline.nvim](https://github.com/lukas-reineke/indent-blankline.nvim)（インデントラインの表示）
- [lualine.nvim](https://github.com/nvim-lualine/lualine.nvim)（ステータスラインの表示）
- [nvim-autopairs](https://github.com/windwp/nvim-autopairs)（括弧やクォーテーションを自動で閉じる）
- [nvim-surround](https://github.com/kylechui/nvim-surround)（囲み文字の置換）
- [telescope.nvim](https://github.com/nvim-telescope/telescope.nvim)（fuzzy finder）
- [telescope-file-browser.nvim](https://github.com/nvim-telescope/telescope-file-browser.nvim)（Telescopeのファイルブラウザ拡張）
- [toggleterm.nvim](https://github.com/akinsho/toggleterm.nvim)（ターミナルの起動、開閉）
- [nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter)（シンタックスハイライト）
- [vim-wakatime](https://github.com/wakatime/vim-wakatime)（[Wakatime](https://wakatime.com/)）
- [nvim-web-devicons](https://github.com/nvim-tree/nvim-web-devicons)（アイコンの表示）

#### lsp

Nixvim は LSP サーバ の Nix を利用したインストールから設定の適用までをカバーできます。これは非常に便利で、[mason.nvim](https://github.com/mason-org/mason.nvim) のような Nix の管理外でサーバをインストール、管理するプラグインを利用すると、先述の通り設定の適用やサーバの更新が Nix で書かれた設定の適用タイミングから外れるので管理が難しくなります。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/nixvim/plugins/lsp/default.nix

キーマップについても同様に設定できます。個人的には `extra` に追記しているようなファイルに切り出すまでもない短い Lua スクリプトが補完やシンタックスハイライトなどの Lua の LSP の恩恵を受けられないことを不満に感じていますが、解決策を与えられていません。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/nixvim/plugins/lsp/keymaps/default.nix

##### nixd の設定

多くの LSP サーバは規定の設定をそのまま適用していますが、Nix の LSP サーバである [nixd](https://github.com/nix-community/nixd) に対しては工夫して設定を与えているので紹介します。同じ Nix の LSP サーバとしては [nil](https://github.com/oxalica/nil) がありますが、nixd は NixOS や Home Manager などの options の補完が提供されている他、`pkgs.*`、`lib.*` のような Nix 式を評価して初めて得られる情報を部分的に補完する機能を提供しています。これは静的解析のみで実装することは難しく、`libnix*` を利用している恩恵であると言えます。

https://github.com/nix-community/nixd

nixd を最大限活かすには、「どの nixpkgs を使って評価するか」と「どの options を補完対象にするか」の情報を nixd 側に渡す必要があります。筆者の設定は大きく分けて次の3点から成ります。

1. フォーマッタを [alejandra](https://github.com/kamadorueda/alejandra) に固定する
1. nixpkgs の参照先を `stateVersion` に基づいて固定し、`pkgs` を得る式を nixd に渡す
1. Home Manager の構成を評価して `.options` を取り出し、それを nixd に渡す

nixd には `nixpkgs.expr` という設定があり、ここに `pkgs` を返す Nix 式を文字列として渡せます。nixd はこの式を評価し、その結果を基準に `pkgs.*` や `lib.*` などの補完候補を作ります。公式ドキュメントでは、プロジェクトルートの flake を `builtins.getFlake` で読み、そこから nixpkgs を参照する方法が紹介されています。

```nix
{
  nixpkgs.expr = "import (builtins.getFlake (builtins.toString ./.)).inputs.nixpkgs { }";
  options.nixos.expr = "(builtins.getFlake (builtins.toString ./.)).nixosConfigurations.<name>.options";
}
```

この方式の利点は明確で、そのリポジトリの `flake.lock` に基づく nixpkgs やモジュールを前提に補完できることです。リポジトリに `nixosConfigurations` や `homeConfigurations` が揃っていれば、nixd に渡す式も単純になります。一方で、flake の outputs はホスト名やユーザ名などのローカル情報に強く依存します。この構造のままプロジェクトルートの flake を評価して options を取る方式を取ると、nixd の設定側でホスト名やホーム構成を決め打つ必要があります。
nixd の設定を Nixvim の設定として共通化しているため、ここをホストごとに分岐させると設定の責務が混ざりやすくなります。そこで筆者は補完のための評価は、dotfiles の flake.nix とは切り離す方針にしました。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/nixvim/plugins/lsp/servers/nixd/default.nix

筆者は `stateVersion` を基準にして upstream の nixpkgs や Home Manager、Nixvim を参照し、nixd に渡す評価コンテキストを組み立てています。

- `nixpkgs.expr` では `pkgs` を得る式を渡す
- `options.home-manager.expr` では Home Manager を評価して `.options` を返す

この手法により、dotfiles の flake.nix とは別に、nixd が補完に使う `pkgs` や options を渡して補完の動作を安定させることができ、他のユーザの dotfiles や自分の config 以外のプロジェクト[^otherwise-project]にも補完が効くようになりました。
そして、以下のように動作するようになりました。

[^otherwise-project]: シングルファイルで実験用の Nix 式を書く場合など

![nixdで補完が動いている](https://storage.googleapis.com/zenn-user-upload/7025bcefa2d4-20251226.gif)
*nixd で補完が動いている*

#### indent-blankline

![自分のindent-blankline](https://storage.googleapis.com/zenn-user-upload/90c6578fe6c9-20251226.png)

indent-blankline は、インデントラインを表示するプラグインです。少し設定を加えており、異なるインデントには異なる色が割り当てられるような Lua スクリプトを、Nix 式で書かれた色の属性セットから組み立てています。この辺の色合いやカラースキームは完全にその場その場で割り当てているので、Nixvim に限らずツール全体のカラーテーマを揃えても良いなと思っています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/nixvim/plugins/indent-blankline/default.nix

#### Telescope

Telescope は fuzzy finder 系のプラグインで、曖昧検索を行うことができます。筆者は本体と、ファイルブラウザ用の拡張機能を入れています。ファイル名検索、ファイル内文字列検索をよく利用しています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/nixvim/plugins/telescope/default.nix

できればプロジェクトルートからすべてのファイルをファイルブラウザの検索対象にしたいのですが、動作が非常に重くなってしまうので階層を制限しています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/nixvim/plugins/telescope/extensions/file_browser/default.nix

#### hop

![hopを使用している写真](https://user-images.githubusercontent.com/506592/176885253-5f618593-77c5-4843-9101-a9de30f0a022.png)
*[smoka7/hop.nvim](https://github.com/smoka7/hop.nvim) の README から引用*

hop は Easymotion 系のプラグインで、全単語や開始文字パターンにマッチする単語に対して割り当てられたキーによって、その単語の位置にジャンプできる機能を提供します。これは非常に便利で、Neovim を開くたびに必ず使っています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/nixvim/plugins/hop/default.nix

### VSCode の設定

VSCode の設定は、`programs.vscode.*` から行っています。拡張機能をいくつか設定しているだけで、まだ本格的な設定は書いていません。`vscodevim.vim` に普段使いのキーマップやプラグインなどを読み込ませることと、Nix で管理する前に使っていたプラグインを入れること、その設定を書くことが目下の目標です。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/programs/vscode/default.nix

NixOS サーバ側には [nixos-vscode-server](https://github.com/nix-community/nixos-vscode-server) を入れています。これは VSCode Server が同梱する Node.js が NixOS ではそのまま動かないため、Node.js へのシンボリックリンクに書き換えるための flake で、VSCode から NixOS サーバに SSH 接続して作業するために入れています。最近ではもっぱら Neovim から作業を行っているので、あまり意識していませんでした。

https://github.com/nix-community/nixos-vscode-server

## ターミナルマルチプレクサの設定（tmux）

ターミナルマルチプレクサは1つのターミナルエミュレータの中で、複数のターミナルを作成して切り替えたり、画面を分割して並べて表示したりするためのソフトウェアです。作業状態をセッションとして保存できるので、SSH接続が切れてもセッション内で動作しているプロセスを停止することなく、後から同じ状態に復帰することができます。私は [tmux](https://github.com/tmux/tmux/wiki) というターミナルマルチプレクサを使っています。

https://github.com/tmux/tmux/wiki

最も素朴なターミナルマルチプレクサとして [GNU Screen](https://www.gnu.org/software/screen/) がある他、最近では [Zellij](https://github.com/zellij-org/zellij) も注目されています。Zellij は現代的なターミナルワークスペースで、自由なレイアウトを組むことができ、プラグインシステムとして [WebAssembly](https://webassembly.org/) を採用しています。3年前に初めて SSH 接続を切らさないために Screen を使って便利さを実感してから、tmux を使い始めました。Zellij も気になっていますが、現状では tmux の操作に不満を感じていないことと、WebAssembly のプラグインシステムが満足に動作するのか疑問を持っていることもあり移行していません。来年はいくつかプラグインを試したり作ってみたりして、tmux よりも良いと確信を持てたら移行してみたいなとも思っています。

Home Manager が提供する tmux の設定は非常にシンプルで、他のツールと同様に [`programs.tmux.*`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.tmux.enable) を設定しますが、主な設定は [`programs.tmux.extraConfig`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.tmux.extraConfig) に tmux の設定ファイルである `tmux.conf` に記述する内容を書くことになります。

### 基本的な設定

`programs.tmux` では履歴やシェル、プラグインなど tmux の基本的な設定を行っています。
tmux のプラグイン管理としては [tpm](https://github.com/tmux-plugins/tpm) が有名ですが、プラグイン管理も Nix 側に寄せています。nixpkgs 25.11 は2025年12月時点で[59個のプラグインが公開しています](https://search.nixos.org/packages?channel=25.11&buckets=%7B%22package_attr_set%22%3A%5B%22tmuxPlugins%22%5D%2C%22package_license_set%22%3A%5B%5D%2C%22package_maintainers_set%22%3A%5B%5D%2C%22package_teams_set%22%3A%5B%5D%2C%22package_platforms%22%3A%5B%5D%7D&query=tmuxPlugins) ので、基本的には nixpkgs からパッケージを選んで [`programs.tmux.plugins`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.tmux.plugins) に渡せば良いです[^self-tmux-plugins-load-reason]。

[^self-tmux-plugins-load-reason]: 筆者がなぜ自前でプラグインをロードしているのかは[2023年11月の筆者](https://github.com/momeemt/config/commit/efee7efbdef0f76e1bc69c68c6566813f0213119)に聞いてください。2025年12月の筆者には分かりかねます。申し訳ない

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/programs/tmux/default.nix

### `extraConfig` の内容

`programs.tmux.extraConfig` は、Home Manager が生成する tmux 設定に対して文字列を追記するためのオプションです。
Home Manager の `programs.tmux.*` は、履歴サイズやインデックス、デフォルトシェルなどの基本設定を Nix のオプションとして提供します。一方で、キーバインドや copy-mode、ステータスラインのような項目は `tmux.conf` として書かなければいけません。そこで、`extraConfig` に `tmux.conf` の内容を追記して適用しています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/programs/tmux/tmux.conf

いくつかよく使っている設定項目について紹介します。

- `set-option -g prefix C-a`
  - tmux のプレフィックスキーを `C-b` から `C-a` に変更しています
  - Ctrl キーから近いキーを選んで入力しやすくしていますが、シェルの Emacs キーバインドと競合するの今後考え直そうかなと思っています
- `bind-key | split-window -h`
  - ペインの左右分割をより直感的なキーに割り当てています
  - 同様に理由で、上下分割を `-` に割り当てています
- `set-window-option -g mode-keys vi`
  - copy-mode の操作を vi に寄せています
- `set-option -g status-right '...'`
  - ステータスラインに表示する情報を設定しています

### tmux-nix の実装

`extraConfig` を見れば分かる通り、Home Manager が提供する `programs.tmux` のオプションだけで tmux の設定を満足に行うのは難しく、結局 `extraConfig` が中心になってしまいます。そこで、[tmux-nix](https://github.com/momeemt/tmux-nix) というモジュールを開発しています。[Nix meetup #3](https://nix-ja.connpass.com/event/353532/) というイベントで発表した資料で説明していますので、興味がある方はそちらを参照してください。なお、この [Nix meetup](https://nix-ja.connpass.com/) というイベントは 2024年10月 から定期的に開催されており、毎回さまざまな発表が行われ、日本語話者の Nix ユーザの交流の場にもなっています。

https://speakerdeck.com/momeemt/tmux-nixnoshi-zhuang-wotong-sitexue-bunixosmoziyuru

https://github.com/momeemt/tmux-nix

## ターミナルエミュレータの設定（Alacritty）

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

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/programs/alacritty/settings/terminal.nix

Alacritty を立ち上げた時に、tmux のセッションに自動で入れるようにしています。まずセッションが存在する場合にはアタッチを行い、存在しない場合には新しく作成してセッションに入ります。
Nix で dotfiles を管理することが他のツールと比べて優位である理由の1つとして、このような他のバイナリに依存するような設定でも移植性を維持できることが挙げられます。たとえば素の Alacritty の設定として以下のように書いたとします。

```toml :Alacritty.toml
[terminal.shell]
program = "zsh"
args = [
  "-l",
  "-c",
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

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/programs/alacritty/settings/hints.nix

### ウィンドウの設定

この他にもカーソル、フォント、スクロール等の細々とした設定を書いていますが、ウィンドウの設定は気に入っているので紹介します。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/programs/alacritty/settings/window.nix

Alacritty は[背景画像を設定することができません](https://github.com/alacritty/alacritty/issues/565#issuecomment-312459718)が、透明率（`opacity`）を設定することはできます。後述しますが5分ごとに壁紙が切り替わるようにしているので、気分を変えながらコーディングできてとても好きです。また、12月21日には [M-1 グランプリ](https://www.m-1gp.com/)の[決勝](https://tver.jp/series/sryg1lm5fy)が放送されましたが[^m-12025-tanoshimi]、お笑い好きの我々としては 8 月から[^m-1-1kaisen] YouTube に続々と公開されている予選動画を見ることに追われています。ブラウザを後ろに置いて、漫才を流しながら作業に取り組めるという点でも必須の設定です。

[^m-1-1kaisen]: 夏になると今年もM-1の季節かと思うようになりますが、よく考えれば執筆現在もまだM-1の季節です。今こそ、と言った方が良いかもしれない
[^m-12025-tanoshimi]: この節は12月19日に執筆しており、まだ決勝動画を見ていません。楽しみすぎる！個人的には[たくろう](https://profile.yoshimoto.co.jp/talent/detail?id=6202)が決勝行ったのがアツすぎて落ち着くことができていない、状況です

## ウィンドウマネージャの設定（AeroSpace）

![AeroSpaceが動作する様子](https://storage.googleapis.com/zenn-user-upload/00471f459770-20251220.gif)
*複数のウィンドウを切り替えたり入れ替えたりできる*

ウィンドウマネージャ（WM）は、GUIアプリケーションのウィンドウの配置やサイズ、フォーカス管理、仮想デスクトップやマルチモニタ間での移動、操作方法、ルールなどをどのように扱うかを決めて実行するソフトウェアです。ウィンドウ同士が重なることを許して自由に動かす floating 系のウィンドウマネージャと、画面を分割して自動整列する tiling 系のウィンドウマネージャがあり、私は後者の [AeroSpace](https://github.com/nikitabobko/AeroSpace) というソフトウェアを使っています。

https://github.com/nikitabobko/AeroSpace

WM は macOS と Linux デスクトップで扱いが異なります。まず、macOS の場合には OS 標準の WM が提供されており、外部ツールが配置やサイズ変更、ワークスペースの機能を追加します。一方で Linux デスクトップの場合には WM そのものを置き換える形となります。私は Linux デスクトップ環境には [Wayland](https://wayland.freedesktop.org/) を、compositor には [sway](https://swaywm.org/) を利用していますが、そもそも Linux デスクトップ環境をほぼ整備していないので割愛します。
また、以前は macOS の WM として [yabai](https://github.com/asmvik/yabai) と [skhd](https://github.com/asmvik/skhd) を組み合わせて使っていました。前者は tiling 系のウィンドウマネージャ本体ですがキーバインドが割り当てられておらず全ての操作にコマンドが割り当てられています。そこで、ホットキーを実現するデーモンである後者を組み合わせて、特定のキーを入力したらウィンドウが分割されたりワークスペースが操作されるような構成にしていました。しかし、yabai は macOS のアップデートのタイミングで挙動が不安定になることが多く、設定も yabai と skhd でそれぞれ別々に行う必要があります。AeroSpace はそれが統合されており、ワークスペース切り替えの挙動も高速で安定しているところを気に入っています。

### nix-darwin vs Home Manager

AeroSpace の設定は [nix-darwin](https://nix-darwin.github.io/nix-darwin/manual/#opt-services.aerospace.enable) と [Home Manager](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.aerospace.enable) の両方にあります。
システム構成として与えるか、ホーム構成として与えるかのメリット・デメリットは既に整理した通りですが、以下の理由でシステム構成として与えています。

1. 実家のパソコンも Nix で管理する予定があるが、AeroSpace の存在によって情報を専門としていない一般的なユーザの操作を阻害することは基本的にないと考えられる
1. WM はデスクトップ環境の基本ソフトウェアであり、キーバインドやワークスペース設定をホスト単位で固定したい

なお、nix-darwin と Home Manager の両方で AeroSpace を有効化すると二重起動や設定が競合する原因となるため、どちらか一方に統一する必要があります。

### キーバインドの設定

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hosts/services/aerospace/main.nix

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
AeroSpace の設定ではキーバインドをキー名に取り、操作を値に取るので、ワークスペース番号と数字キーの値が一致している時にはこのように記述することで簡潔に設定を表現できてミスも減らせます。
また、Nix は for 文で手続き的に属性セットを構成するのではなく、このようにデータの構造を上手く変換する関数型言語的なアプローチで構成する[^like-ocaml]ところも面白い点の1つです。面白いだけではなく、羅列になりがちな設定の性質を扱いやすい点でも手続き的な言語で記述するよりも向いているとも思います。

[^first-prior-listToAttrs]: なお、同じ`name`が複数回出た場合には[最初のものが優先されます](https://nix.dev/manual/nix/2.30/language/builtins#builtins-listToAttrs)
[^like-ocaml]: 節々から [OCaml](https://ocaml.org/) の影響を受けていることを感じさせられます

### サービスモードの定義

AeroSpace では**モード**という概念があり、キーバインドを切り替えることができます。モードは、`services.aerospace.settings.mode.<mode>.binding` に設定を書くことで作成できますが、デフォルトモード（`main`）から新しく作成したモードに入るためのキーバインドは別途設定しておく必要があります。私の設定では、`alt-shift-semicolon = "mode service";` がそれに該当しています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hosts/services/aerospace/service.nix

基本的には[公式ドキュメント](https://nikitabobko.github.io/AeroSpace/guide)が紹介しているデフォルトの設定をそのまま書いています。サービスモードの存在を思い出した時に調べて使ってみる程度に留まっており、たとえば音量の調節は macOS のデフォルトのホットキーを利用しているので使いません。
時折、タイリングされるとかえって作業がし辛い場合があり、`layout floating tiling`（`floating` 、`tiling` 型を切り替える）は便利そうです。この辺りはまだ未整備なので目下の課題としています。

## バージョン管理システムの設定（Git）

バージョン管理システム（VCS）とは、ソフトウェア開発におけるプログラムやリソース等のファイルの変更履歴を管理するソフトウェアです。多くの方がそうであるように、私も [Git](https://git-scm.com/) を使っています。

https://git-scm.com/

[Subversion](https://subversion.apache.org/)[^subversion] や [Mercurial](https://www.mercurial-scm.org/)、最近では [Jujutsu](https://github.com/jj-vcs/jj) なども話題になりました。VCS については他のツールを試してもいないというのが正直なところですが、自分が Git を使い熟せているとは思えないので保留しています。最近は興味が移ってあまり取り組めていませんが、2025年度前半に [nixpkgs](https://github.com/NixOS/nixpkgs) への Pull Request を作っていた際には、さまざまなソフトウェアのビルドを並行に試していくので stash だけでは追いきれなくなり `git worktree` を使ったり、PR を出す際の要件として必要なコミットにまとめなければいけないため `git rebase` を使ったりしていました。個人開発では扱わない巨大な Git リポジトリに対して貢献を行うことで解像度が深まるという側面はあるかもしれません。

[^subversion]: Gitを始めて学んだ時に、VCSの**歴史**としてSubversionを知りました。もうこんなことを言っても鼻で笑われることはないほどにGitが覇権を握っていますね

### 基本的な設定

Home Manager では [`programs.git.*`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.git.enable) を設定することで、`.gitconfig` 相当の設定を行えます。名前とメールアドレスの設定を行っているのと、デフォルトブランチを GitHub に合わせて `main` にしています。
エイリアスはいくつか設定していて、`st`（`status`）、`aa`（`add --all`）[^git-aa]、`cm`（`commit -m`）、`poh`（`push origin head`）、`d`（`diff`）などはかなり使っています。また、`ignores` にはデフォルトで無視するべきファイルやディレクトリを設定していますが、`.momeemt` というユーザ名の隠しファイルを指定しています。OSS など他のユーザがオーナーシップを持っているリポジトリで Git 管理下に載せたくないが作業ファイルを置いておきたい場合に、プロジェクトルートで `mkdir -p .momeemt` を作るだけで良いので便利です。

[^git-aa]: 楽なのでやってしまいますが、本来はきちんとステージングするファイルや編集範囲を決定するべきだと思います。来年の目標の1つ

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/programs/git/default.nix

### diff を見やすくする

Git のデフォルトの diff は削除行の先頭に `-` が、追加行の先頭に `+` が付いてハイライトされますが、変更前と変更後を左右に分割して表示してくれるツール、[difftastic](https://difftastic.wilfred.me.uk/) を導入しています。

https://difftastic.wilfred.me.uk/

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/programs/difftastic/default.nix

### GPG 鍵によるコミット署名

Git ではコミットやタグに対して署名を付けることができます。署名付きコミットはそのコミットが、対応する秘密鍵を持っている人物によって作成されたことを検証でき、履歴の改ざん検知を行うことができます。
筆者は Home Manager の [`programs.git.signing.*`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.git.signing.key) を使って、コミットに GPG 署名を付けています。

```nix :nix/home/emu/default.nix
programs.git = {
  signing = {
    # `gpg --list-secret-keys --keyid-format LONG`
    key = "86F8F50B69A94DE2";
    signByDefault = true;
  };
}
```

`key` には、`gpg --list-secret-keys --keyid-format LONG` を実行することで得られる LONG 形式の Key ID を指定しています。署名を有効化した後は、デフォルトで署名が行われるようにしているため、明示的に `-S` を付けなくても署名付きコミットになります。

## Homebrew を共存させる（brew-nix）

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

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/modules/home/home/casks.nix

### パッチを当てる

brew-nix は Homebrew Casks の情報を元に機械的に derivation を生成するツールであるため、これが提供する Nix 式そのままではビルドに失敗するアプリケーションも数多く存在しています。最も多く遭遇するのはハッシュ値が生成された derivation に記載されていない場合で、公式ドキュメントでも[約700パッケージが該当する](https://github.com/BatteredBunny/brew-nix#overriding-casks-derivation-attributes-for-casks-with-no-hash:~:text=There%27re%20about%20700%20casks%20without%20hashes%2C%20so%20you%20have%20to%20override%20the%20derivation%20attributes%20%26%20provide%20a%20hash%20explicitly.)と言及されています。
私は `nix/overlays` 内に `pkgs.brewCasks` を上書きするオーバレイを作成し、`pkgs` を作成するときに適用しています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/overlays/brew-casks.nix

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

brew-nix を使ったインストールは1つ目の問題点であった、*GUI アプリケーションのインストール*を解決しますが、2つ目の問題点であった *別のアプリケーションをインストールするようなハブとなるアプリケーションのインストール* の解決策としては微妙です。というのも、brew-nix が生成するのはあくまで derivation なので、アプリケーションの内容（すなわちビット列）が少しでも変わるたびに異なる derivation として扱われます。アプリケーションは往々にしてファイルサイズが大きいためディスクを圧迫しますし[^brew-nix-size]、認証機能や別のアプリケーションをインストールするような機能を持っている場合、derivation が変わるたびにそれらの操作がリセットされてしまうのです。たとえば、私はスクリーンショットを撮影するために [CleanShot X](https://cleanshot.com/) を使っていますが、これは [Setapp](https://setapp.com) というサブスクリプション型のアプリケーションコレクションからインストールしています。`brew-api` を更新するたびに Setapp を介してインストールしたアプリケーションが見えなくなり、スクリーンショットが使えなくなるのは非常に不便です。また、個人で扱いたいアプリケーションでも nix-darwin を使うとグローバルにインストールされてしまうという問題もあります。そこでシステム構成ではなくホーム構成で、かつ素の Homebrew を使う方法について考えます。

[^brew-nix-size]: この問題は brew-nix のドキュメントでも触れられています。普段の作業に影響を与えるほどディスクサイズを圧迫するアプリケーションについてもやはり、brew-nix を使わない方が得策です

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/homebrew/default.nix

Homebrew 自体のインストール、および `Brewfile` を適用するために、Home Manager の `home.activation.<name>` を使っています。これは、Home Manager の設定を適用して世代を切り替える際に実行されるアクティベーションスクリプトのフックです。Nix の宣言だけでは表現しづらい、外部ツールの初期化や状態を持つツールのセットアップなどの手続きを世代切り替えの際に実行できます。
Home Manager は世代ごとに何度も同じ処理を実行する可能性があるため、書き込みをしない検証段階と、書き込みをして良い段階に分けています。今回はファイルの書き込みを行う副作用を持っているため、`writeBoundary`（書き込み境界）の後で実行させる必要があります。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/homebrew/install-homebrew.sh

呼び出しているシェルスクリプトでは、Homebrew が存在しない場合にはインストールを行い、する場合にはスキップして `brew bundle` により `Brewfile` で指定されたアプリケーションを Homebrew にインストールさせています。こうすることで、個々のアプリケーションは Nix から切り離し、アプリケーションのインストール、更新が発生するトリガーは `darwin-rebuild` に揃えることができました。

## シークレット管理（sops-nix）

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

シークレットを使う際には、以下のようにシークレット名と暗号化されたファイルやフィールドの対象、パーミッションなどを指定して `sops.secrets` で宣言します。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/sops/default.nix

これは NixOS や nix-darwin などシステム構成側のモジュールとして読み込んで利用しています。以下のようなシークレットを扱っていますが、これらはシステムサービスが読む、またはユーザ作成など早い段階で必要になります。

- `momeemt-password`
- `k8s-bootstrap-token`
- `"cloudflared/emu.json"` / `"cloudflared/emu-desktop.json"`
- `"k8s/ca.key"`

一方で、ユーザ権限のプログラムが読むシークレットは管理者ユーザが所有するファイルになると困ることもあります。たとえば、`wakatime_api_key` をシステム側で復号してしまうと、ユーザが読めるようにするには権限を緩める必要が出てきます。そこで、ホーム構成側でも別にシークレットを管理しています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/sops/default.nix

## Secure Shell の設定（OpenSSH）

Secure Shell とは暗号化と改竄検知、サーバ認証により安全に遠隔ログインやコマンド実行、ポートフォワーディングなどを行うためのプロトコルで、多くの場合 SSH と呼ばれます。SSH を実装したソフトウェアとしては [OpenSSH](https://www.openssh.org/) が広く使われており、クライアントの `ssh` や、サーバの `sshd` のほか、鍵生成のための `ssh-keygen` 、認証エージェントの `ssh-agent` 、ファイル転送のための `scp` や `sftp` などを提供しています。

https://www.openssh.org/

クライアント設定は Home Manager の [`programs.ssh.*`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.ssh.enable) から、サーバ設定は `services.openssh.*` から行えます。基本的にノートパソコンに対してリモート接続する運用を行っていないので、後者は NixOS に対してのみ設定しています。

### クライアント設定

`ssh` コマンドは、`~/.ssh/config` にホスト設定を書いておくと、ホスト名だけで接続を行うことができます。Home Manager では、[`programs.ssh.matchBlocks.<name>`](https://nix-community.github.io/home-manager/options.xhtml#opt-programs.ssh.matchBlocks) に与えた設定から config ファイルが生成されます。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hm/programs/ssh/default.nix

基本的に IP アドレス等が公開情報のサーバは VCS に載せて管理していますが、そうでないサーバは sops-nix によって暗号化したファイルを適用時に復号化し、`includes` により読み込ませています。現在は秘密鍵が指定したパス（`identityFile`）に存在することを要求していますが、できれば秘密鍵も暗号化して VCS に載せたいと思っています。しかし、作業ミスによって生まれる被害が自分が管理するサーバ以外にも及ぶため怖くてそれは実現できていません。sops の代わりに 1Password からクレデンシャルを利用する [opnix](https://github.com/brizzbuzz/opnix) というツールもあり、将来的にはそちらに移ることも考えています[^or-bitwarden-nix]。

https://github.com/brizzbuzz/opnix

[^or-bitwarden-nix]: パスワードマネージャには [Bitwarden](https://bitwarden.com/ja-jp/) を利用しているのでできればシークレット管理にもそれを使いたいのですが、調べた限りでは [sopswarden](https://github.com/pfassina/sopswarden) しか見つからず、これが信頼できる NixOS モジュールかどうかはまだ分かりません

### サーバ設定

基本的にはできるだけ厳しく設定を書くようにしています。パスワードやキーボードインタラクティブ系の認証方式をすべて切っていて、公開鍵認証に寄せています。また、許可ユーザを `momeemt` に絞っています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/profiles/hosts/services/openssh/default.nix

OpenSSH については勉強中なので [`sshd_config` のマニュアル](https://man.openbsd.org/sshd_config.5)を見ながらおおよその設定をしましたが、来年はもう少し解像度を高められると良いなと思っています。

https://man.openbsd.org/sshd_config.5

## Cloudflare Tunnel を通す

自宅の NixOS サーバには、外出先から SSH 接続したり、リモートデスクトップで操作したくなる場面があります。一方で、集合住宅の備え付け回線やキャリア側の NAT などの事情により、こちらから自由にポート解放やルータ設定ができず、サーバに対する inbound な接続を受けられないことがあります。
この問題に対して、筆者は [Cloudflare](https://www.cloudflare.com/) が提供する [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/) を使っています。Cloudflare Tunnel はサーバ側で `cloudflared` を動かし、Cloudflare のネットワークへ outbound な接続を張っておくことで、外部から Cloudflare 経由でサーバへ到達できるようにする仕組みです。外部へ公開するためにサーバのグローバル IP に直接アクセスさせる形を取らず、Cloudflare 側の DNS とトンネルの対応付けによってトラフィックを中継してくれます。

https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/

Cloudflare Tunnel は HTTP だけでなく、SSH やリモートデスクトップなどのプロトコルも対象にできます。NixOS では `services.cloudflared` モジュールが用意されているので宣言的に `cloudflared` の常駐とトンネルの設定を記述できます。

https://wiki.nixos.org/wiki/Cloudflared

トンネルを UUID ごとに定義しています。また、`ingress` オプションによってどのホスト名へのアクセスをローカルのどのサービスへ転送するかを定義します。認証情報が含まれたファイルについては、先述のシークレット管理によって暗号化したファイルを VCS に載せておき、復号化済みのパスを渡しています。

```nix :nix/hosts/emu/configuration.nix
services.cloudflared = {
  enable = true;
  tunnels = {
    "053a4ebe-82c3-485a-bc8b-f0768cbe1d11" = {
      credentialsFile = "${config.sops.secrets."cloudflared/emu.json".path}";
      ingress = {
        "emu.momee.mt" = {
          service = "ssh://localhost:22";
        };
      };
      default = "http_status:404";
    };
    "b2b83e61-577c-45e7-b1c0-0961503e8d8d" = {
      credentialsFile = "${config.sops.secrets."cloudflared/emu-desktop.json".path}";
      ingress = {
        "emu-desktop.momee.mt" = {
          service = "rdp://localhost:3389";
        };
      };
      default = "http_status:404";
    };
  };
};
```

この設定により、グローバル IP に直接アクセスすることなく外部から自宅のサーバにアクセスすることができます。

## NixOS module を定義する

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

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/modules/home/programs/wget/default.nix

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

## 壁紙を定期的に貼り替えるサービス（set-wallpapers）

前節で NixOS module を定義する方法について見てきました。本節ではその続きとして、自作スクリプトを定期実行サービスとして登録し、 NixOS module によって運用する例を紹介します。対象は macOS + nix-darwin です。後述の通り、AppleScript を利用するため Linux では動作しません。
私はジブリ映画が好きなのですが[^ghibli]、デスクトップ環境の壁紙を STUDIO GHIBLI が配布している画像に設定しています。

[^ghibli]: 好きではあるのですが本格的にジブリファンと言えるほど詳しいわけでもありません。すみません

https://www.ghibli.jp/info/013251/

macOS は OS のバージョン更新などで壁紙設定がリセットされることがあります。そこで、壁紙の設定も Nix 側の構成に含めて再現できるようにしたいと考え、壁紙を定期的に切り替える仕組みを作りました。このサービスがやっていることは単純で、次の2ステップです。

1. 画像一覧から壁紙候補を1枚選ぶ
1. その画像を macOS の壁紙として設定する

### 壁紙を貼り替えるスクリプト

壁紙の設定は [AppleScript](https://developer.apple.com/library/archive/documentation/AppleScript/Conceptual/AppleScriptX/AppleScriptX.html)、壁紙の選択はシェルスクリプトで行い、組み合わせています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/modules/host/services/set-wallpapers/set.applescript

AppleScript を使っている理由は、依存ライブラリが不要で最も簡単に壁紙を変更できるためです。実に7年ぶりです[^seven-years-ago]。

[^seven-years-ago]: 当時は英単語帳を [Numbers](https://www.apple.com/jp/numbers/) で管理していて、特定の条件を満たすテキストの色を変えるのに使っていた記憶があります

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/modules/host/services/set-wallpapers/main.sh

Nix 側から環境変数 `WALLPAPERS` を介して壁紙候補の画像パス一覧を `main.sh` に渡します。`main.sh` はその中からランダムに画像を1枚選び、 `set.applescript` を実行して壁紙を変更します。また、連続で同じ画像が選択されるのは避けたいので、`~/Library/Application Support/set-wallpapers` 配下に前回選んだ壁紙を状態ファイルとして保存し、次回実行時に参照するようにしています。

### パッケージの定義とサービスへの登録

上のスクリプト群は、`pkgs.writeShellScript`によって Nix の derivation として扱えるようにしています。そうすることで、スクリプトが `/nix/store` 配下にコピーされるため dotfiles リポジトリ内のファイルパスや作業ツリーの状態に依存しなくなります。したがって、適用後にリポジトリの内容が変わっても、サービスが参照する実体は変わりません。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/modules/host/services/set-wallpapers/default.nix

壁紙の貼り替えを AppleScript で行っているため、この NixOS module も nix-darwin からのみ呼ばれることを前提としています。nix-darwin は、macOS 標準のサービス管理ソフトウェアである [launchd](https://github.com/apple-oss-distributions/launchd/tree/launchd-842.92.1) へユーザサービスを登録するための [`launchd.user.agents`](https://nix-darwin.github.io/nix-darwin/manual/#opt-launchd.user.agents) が用意されています。この設定で [`launchd.user.agents.<name>.serviceConfig.StartInterval`](https://nix-darwin.github.io/nix-darwin/manual/#opt-launchd.user.agents._name_.serviceConfig.StartInterval) を `300`（秒）に指定すると、ユーザ領域の launchd が5分ごとにスクリプトを実行します。結果として、ログイン後は壁紙が定期的に切り替わるようになります。

### 適用

ここまで作成したモジュールを、次のように適用しています。画像を置かずに [`pkgs.fetchurl`](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-fetchers-fetchurl) で入手しているので、不必要にリポジトリサイズが大きくならない点を気に入っています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/hosts/uguisu/wallpapers.nix

## Docker イメージのビルド

GitHub Actions でホーム構成を適用した Docker イメージをビルドしています。dotfiles を公開しているユーザは、自分の dotfiles をインストールするための方法も併せて整備している場合があります。たとえば、[mathiasbynens/dotfiles](https://github.com/mathiasbynens/dotfiles) は、Git を使ったインストール方法や、Git を使わずにインストールするためのスクリプトを紹介しています。

https://github.com/mathiasbynens/dotfiles

```sh :mathiasbynens/dotfiles を Git を使わずにインストールする
cd; curl -#L https://github.com/mathiasbynens/dotfiles/tarball/main | tar -xzv --strip-components 1 --exclude={README.md,bootstrap.sh,.osx,LICENSE-MIT.txt}
```

しかし、Nix でシステム構成やホーム構成を管理する場合には、ユーザ名やシークレットなど、その dotfiles を整備したユーザ固有の情報が含まれることがあるため、そのまま、他のユーザが使い回しづらいことが多いです[^dotfiles-by-others]。また、システム構成はホスト全体に影響するため、ホーム構成よりも試すコストやリスクが大きいです。そこで、そのようなユーザ固有の情報を抜いてホーム構成のみを適用したコンテナイメージを固め、配布することで、部分的にではありますが、自分の環境を汚すことなく誰でも簡単に試せるようにしています。

[^dotfiles-by-others]: そもそも他のユーザの dotfiles をそのまま使いたくなることはあるのでしょうか。私はそう思ったことは正直一度もありませんが、試したくなることもあるのかもしれません

### コンテナの制約

Docker コンテナは仮想マシンではなく、ホスト OS のカーネルを共有してプロセス空間などを隔離する仕組みです。したがって、カーネルやブートローダ、カーネルモジュール、デバイス管理、ファイルシステムのマウント、`sysctl`、ネットワーク設定のうち、ホスト全体に影響するもの、および特権が必要な操作はコンテナ内だけで完結して変更できません。変更には、`--privileged` 等のオプションが必要になり、十分に環境を隔離することができなくなります。
システム構成は、systemd サービス、`/etc` の生成、ユーザ・グループ管理、cgroups、ログインセッション、ネットワーク、デバイスなど OS 全体を対象にして状態を作り替えます。一方で、多くの Docker イメージは systemd を PID 1 として動かさない前提で作られるため、この種のシステム構成の適用はそのまま行うことができません。
一方で、Home Manager のホーム構成は基本的にホームディレクトリ以下のファイル配置やシンボリックリンクの生成が中心で、カーネル機能に依存しません。そのため、コンテナではシステム構成ではなくホーム構成のみを適用するという割り切りが必要です。ただし、Home Manager の設定の中にも systemd のユーザサービスなどを前提にするものがあり、コンテナでは有効化できない項目があることには注意が必要です。

### コンテナイメージの作成

derivation をそのままコンテナに固める際は、[`pkgs.dockerTools.buildImage`](https://ryantm.github.io/nixpkgs/builders/images/dockertools/) が便利です。指定した derivation の依存だけを取り込んだイメージを作れるため、Dockerfile によってイメージを作成した場合よりも構成要素を絞り込みやすく、コンテナイメージを最小化するために非常に便利です。しかし、Home Manager は activation によってホームディレクトリ以下へリンクやファイルを配置するため、`buildImage` でストアのパスをコピーするだけでは適用済みの状態にはなりません。そこで、ホーム構成を適用するような `RUN` 命令を含めた Dockerfile を作成してコンテナイメージを作成します。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/.devcontainer/Dockerfile

Nix のインストールを行い、dotfiles をコピーし、`nix run` からホーム構成を適用しています。1つ注意点として、GitHub Actions 等で docker buildx の QEMU エミュレーション経由で aarch64 をビルドしている場合、seccomp BPF のロードが `Invalid argument` で失敗する例があります。そこでインストールを実行する前に `/etc/nix/nix.conf` に `filter-syscalls = false` を設定することで回避できるという例がありますが、これはセキュリティ上の影響があるため注意が必要です。

https://github.com/NixOS/nix/issues/5258

しかしこれはバージョンや環境の差によって上手くいくかどうかが変わる未解決な問題であり、私は上手くいかなかったため（`filter-syscalls` のオプション追加行を残したままですが）GitHub Actions の `matrix` を使って各アーキテクチャ向けのイメージをネイティブ実行でビルドしています。

また、適用するホーム構成はコンテナイメージ用に別で用意しています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/home/example/default.nix

### コンテナイメージの公開

Docker イメージは、GitHub Container Registry (GHCR) で[公開](https://github.com/momeemt/config/pkgs/container/config)しています。現時点では明確なバージョン管理をしていないので、`develop` ブランチを更新するたびにデバッグ用にイメージをビルドしています。来年は nixpkgs のように、`yy.mm` バージョン（2025年11月のリリースは、`25.11`）で管理していきたいと思っています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/.github/workflows/image.yaml

### コンテナイメージの利用

公開されているコンテナイメージは、以下のコマンドを実行することで入手できます。

```sh
docker pull ghcr.io/momeemt/config:unstable
```

## Terraform

Terraform は、クラウドや SaaS の設定をコードで管理するための Infrastructure as Code（IaC）ツールです。HCL（HashiCorp Configuration Language）でどのリソースをどんな設定で作るかを宣言し、`terraform plan` で変更差分を確認した上で、`terraform apply` で実際に反映します。外部サービスは provider を通して API 操作として適用されます[^exclude-sops]。適用結果は state として保存され、次回の差分計算に使われます。Terraform は provider を通じて外部サービスの現在状態を読み取り、宣言した設定との差分を確認できるため、ローカル環境の構成管理が中心の Nix と役割分担しやすいです。今年利用した provider は以下の通りです。

[^exclude-sops]: ただし、後述する sops provider は SaaS の API ではなく、sops で暗号化したファイルを Terraform から参照するための provider です。そのようなローカル操作を行う provider も存在します

- sops
- Cloudflare
- GitHub

### Cloudflare

Terraform から Cloudflare の各種サービスや設定を適用できます。ドキュメントについても Terraform Registry に付属しているので逐一確認できます。

https://registry.terraform.io/providers/cloudflare/cloudflare

基本的に DNS レコードの管理を移しました。`cloudflare_dns_record` によって、レコードを作成できます。ダッシュボードから過去に追加した、現在は使われていないレコードを整理するきっかけにもなりました。現在はまだ Cloudflare Tunnel / Zero Trust の設定や、Workers & Pages については Terraform に移せていないので、今後の課題としています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/terraform/cloudflare.tf

### GitHub

SSH 公開鍵、および GPG 公開鍵の反映を移しました。マシンが増えてからコミットへの署名を設定済みのマシンとそうでないマシンが混在しておりきちんと把握できていませんでしたが、テキストベースの設定に移行してからはその見通しが良くなりました。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/terraform/github.tf

GitHub provider からは、リポジトリやGitHub Actions、Issues のラベルの設定なども行うことができます。特にブラウザから設定した GitHub Actions のシークレットはどのリポジトリに何のシークレットがあるのか追いづらく、値も後から確認できないため困ることが多いので、移行する価値があると感じています。

https://registry.terraform.io/providers/integrations/github/latest

### sops

Terraform からシークレットを扱う際も、sops を利用しています。[carlpett/terraform-provider-sops](https://github.com/carlpett/terraform-provider-sops) を使っていて、暗号化済みの YAML ファイルを渡すだけで、Terraform 内の変数としてシークレットが扱えるようになるのでとても便利です。

https://github.com/carlpett/terraform-provider-sops

provider を扱う際に必要な API トークンや、その他 VCS でそのまま扱うことが望ましくない情報については sops 経由で扱うようにしています。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/terraform/sops.tf

### ディレクトリごとに `devShell` を切り替える

余談ですが、`.envrc` にスクリプトを書いてディレクトリごとに `devShell` が自動で切り替わるようにしています。

```sh :.envrc
# This file is auto-generated by ./assets/scripts/env.sh
source_url "https://raw.githubusercontent.com/nix-community/nix-direnv/3.0.7/direnvrc" "sha256-bn8WANE5a91RusFmRI7kS751ApelG02nMcwRekC/qzc="
watch_dir nix/flakes

if [ "${CHILD_ENVRC-0}" != "1" ]; then
  use flake
fi
```

まず、デフォルトの `devShell` に入るための `.envrc` を作成します。基本的にはプロジェクトルートの `flake.nix` や `flake.lock` などが変更された場合のみ、direnv 環境の更新が行われるため、モジュールの分割ファイルを入れている `nix/flakes` を監視対象にしています。次に、`CHILD_ENVRC` が `1` でない場合には `use flake` を実行しています。

```sh :terraform/.envrc
# This file is auto-generated by ./assets/scripts/env.sh
export CHILD_ENVRC=1
source_up
unset CHILD_ENVRC

watch_file devShell.nix
use flake ../#terraform
```

異なる `devShell` に入りたいディレクトリでは、`CHILD_ENVRC` を定義してから `source_up` で親の（すなわち先ほどの）`.envrc` を呼び出します。次に、個別の `devShell` を定義しているファイルを監視対象にしてから、目的の `devShell` に入ります。このように `.envrc` を書き分けることで、ディレクトリごとに異なる開発環境に入ることができるため、デフォルトの `devShell` が必要以上に肥大化することを防げます。

## 仮想マシンを作成する

この節では、自宅に Kubernetes クラスタを導入するために仮想マシンを作成する方法について触れます。まず、NixOS で Kubernetes クラスタを構築すること自体は可能ですが、一般的な kubeadm ベースの手順をそのまま適用するのは難しいです。一般的には `kubeadm init` や `kubeadm join` を中心に設計されており、kubelet の設定や証明書などの管理も `kubeadm` が握っています。一方で、NixOS には `services.kubernetes` のように apiserver、controllerManager、scheduler、kubelet、etcd などをモジュールとして直接構成する仕組みがあり、kubeadm の手順や生成物とは異なります。特にクリティカルなのは周辺コンポーネントが特定のパスや配置を要求することがあり、NixOS では追加の設定や activation script が必要になります。
一方で、Kubernetes クラスタを NixOS 環境で実現するためのツールはいくつか提案されています。

- [`services.kubernetes`](https://nixos.wiki/wiki/Kubernetes)
- [`services.k3s`](https://github.com/NixOS/nixpkgs/blob/master/pkgs/applications/networking/cluster/k3s/README.md)
  - 軽量な Kubernetes ディストリビューションである [K3s](https://k3s.io/) を動作させるためのモジュール
- Nix と Kubernetes を組み合わせるコミュニティツール
  - [kubenix](https://kubenix.org/)
  - [kubernix](https://github.com/saschagrunert/kubernix)
  - [nixos-ha-kubernetes](https://github.com/justinas/nixos-ha-kubernetes)

しかし、私は学習目的のために Kubernetes クラスタを立てるため、kubeadm を使った一般的な構築手順にそって触れたいと考えています。NixOS の標準モジュールや関連ツールは便利ですが、kubeadm 前提の手順や構成と一致しない部分があり、学びたい手続きや設定がそのまま扱えない可能性があります。そこで、NixOS マシンに Ubuntu Server がインストールされた仮想マシンをいくつか立て、その上で Kubernetes クラスタを構築するという方針で進めていきます。

### NixVirt

NixOS で仮想マシンを立てるのに便利なのが [NixVirt](https://github.com/AshleyYakeley/NixVirt) です。NixVirt は Nix flake として提供されており、[libvirt](https://libvirt.org/) 上の仮想マシンや関連オブジェクトを宣言的に管理するための NixOS / Home Manager モジュールを含みます。

https://github.com/AshleyYakeley/NixVirt

NixOS は [`virtualisation.libvirtd`](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/virtualisation/libvirtd.nix) で libvirt デーモン自体の設定は提供しています。しかし、こちらの設定は VM やネットワークの定義を NixOS の構成として揃える仕組みは提供されていません。NixVirt は libvirt の接続 URI ごとに domains、networks、pools を列挙してその定義 XML を libvirt に流し込みます。また、NixVirt による変更の適用は冪等性があり、モジュール内部では定義と状態制御を行う `virtdeclare` というツールが使われています。

https://nixos.wiki/wiki/Libvirt

また、NixVirt からはテンプレートが提供されており、メモリサイズや仮想 CPU の割り当て、ディスクの設定などを書くだけで簡単に domain 層の XML を生成できます。ただし、テンプレートは安定した動作が保証されていないため、依存せずに細かく設定する必要がある場合もあります。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/lib/mkK8sWorker/default.nix#L80C1-L175C50

ここで作っている VM は、ホスト側では libvirt の domain 定義として管理し、ゲスト側の OS 初期化と kubeadm 実行は cloud-init とスクリプトに寄せています。NixOS 側でやるのは、起動に必要なファイル（ディスクと ISO）を用意して、VM の定義を libvirt に反映して起動するところまでです。

### ネットワークの準備

VM はホストの bridge に接続し、cloud-init の network-config で静的 IP を設定します。

- ゲスト側では DHCP を使わず、IP アドレスとデフォルトゲートウェイを固定する
- DNS は固定の resolver を指定する
- インタフェース名の違いは cloud-init の match（`en*`）で吸収する

実装上は `pkgs.formats.yaml` を使って cloud-init が読む YAML を Nix から生成しています。この方式にすると、VM ごとの IP などを Nix の引数として与えるだけで network-config を作れるようになります。

### マスターノードを作成する

では、Kubernetes のマスターノードを立ち上げるモジュールを作成します。この構成は、NixOS ホスト上で libvirt を動かし、NixVirt を使って VM 定義を宣言的に管理します。ゲスト OS は Ubuntu Server として、cloud-init を使ってネットワーク、SSH ログイン用ユーザ、Kubernetes control plane の初期化までを初回の起動時に実行します。
先にモジュールの定義とスクリプトを示します。全体としては、以下のような流れになります。

1. ホスト側（NixOS）で VM の起動に必要なファイルを生成する
1. NixVirt で libvirt domain を生成し、ISO とディスクをアタッチして起動する
1. ゲスト側（Ubuntu）で cloud-init が動作して、payload ISO をマウントして `kubeadm init` を実行する

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/lib/mkK8sMaster/default.nix
https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/lib/mkK8sMaster/main.sh

このモジュールは大きく次の3つの要素でできています。

- libvirt の domain 定義
  - NixVirt で XML を生成して `qemu:///system` に反映する
- ディスク作成
  - Ubuntu の base image を backing file にした qcow2 を作る
- cloud-init 用 ISO と payload 用 ISO の生成
  - systemd oneshot で seed ディレクトリにファイルを集めて ISO を生成する

ディスク作成は `vm-disk-<name>` の oneshot で行い、初回だけ qcow2 を作って以降は再作成しないようにしています。これにより VM を何度起動し直してもディスクが消えない運用になります。ディスクは `/var/lib/libvirt/images/<name>.qcow2` に置き、Ubuntu base image のパス（`ubuntuImage`）を backing file として参照します。
cloud-init と payload の準備は `vm-cloudinit-<name>` の oneshot で行います。ここでは seed ディレクトリを作成し、cloud-init が読む user-data / meta-data / network-config を生成して seed ISO を作ります。加えて、別の CD-ROM として payload ISO を作り、ここに Kubernetes 初期化で必要なファイルをまとめて詰めています。payload ISO には次のようなものを入れています。

- Kubernetes の CA 証明書と CA 秘密鍵
- kubeadm の bootstrap token
- ゲスト側で実行する bootstrap スクリプト
- Argo CD の app-of-apps など、クラスタ初期化後に適用したい manifest

このうち CA 秘密鍵や token は sops-nix で管理しています。そのため `vm-cloudinit-<name>` の service は `after = ["sops-nix.service"];` に設定しており、ホスト側で復号が終わった後に ISO を作るようにしています。Nix の build ではなくホストの oneshot で復号済みファイルのパスを参照して seed ディレクトリに配置し、そこから ISO を生成します。
ゲスト側（Ubuntu）では cloud-init の `runcmd` から payload ISO を `/seed` にマウントし、`/seed/k8s-master-bootstrap.sh` を実行します。`k8s-master-bootstrap.sh` は、kubeadm を実行する前提を整えた上で `kubeadm init` を実行し、その後 CNI（[flannel](https://github.com/flannel-io/flannel)）と Argo CD のインストール、app-of-apps の apply までを行います。ここまでを初回の起動で実行することで、VM を立ち上げるだけで control plane の初期化ができるようにしています。

#### cloud-init の設定ファイルを Nix で生成する

`pkgs.formats.yaml` を使って、cloud-init が読む YAML を Nix から生成します。

- network-config は VM の IP や gateway を引数で受け取り、静的 IP 設定の YAML を生成する
- user-data は ubuntu ユーザの作成と SSH 公開鍵の登録、必要なパッケージの導入、起動後のコマンド実行（payload のマウントと bootstrap）を YAML として生成する
- meta-data は instance-id と hostname を固定する

このように Nix から生成すると、VM のパラメータ（IP、UUID、リソース配分など）を引数として扱えるため、複数台の VM を同じ構造で簡単に立ち上げられるようになります。

### ワーカーノードを作成してクラスタに参加させる

ワーカーノードも同様の構造で、ディスク作成と cloud-init / payload の ISO 作成を oneshot で行い、NixVirt で domain 定義を libvirt に反映して起動します。

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/nix/lib/mkK8sWorker/default.nix

ワーカーノード側の cloud-init は、起動後に payload ISO をマウントし、`k8s-worker-bootstrap.sh` を実行して containerd と kubeadm をセットアップした上で `kubeadm join` を行います。`kubeadm join` には `bootstrap token` と `discovery-token-ca-cert-hash` が必要なので、token は sops-nix で復号したものを payload ISO に詰め、CA のハッシュはホスト側から引数として渡します。
マスターとワーカーを分けたことで、ホスト側の責務は VM の起動に必要なファイルを生成して起動すること、ゲスト側の責務は payload を元に kubeadm を実行することに分割できます。クラスタの台数を増やしたい場合は、同じ関数を別の `name`、`uuid`、`ipAddress` で呼び出すことでワーカーノードを増やすことができます。

## Kubernetes

Kubernetes クラスタは立てられましたが、中で動いているサービスはほとんどありません。Kubernetes 自体の学習もまだ始めたばかりなので、来年度以降の目標とします。[Argo CD](https://argo-cd.readthedocs.io/en/stable/) という Pull 型の GitOps ツールを使っていて、ダッシュボードを眺めるのは楽しいです。

![Argo CDが動く様子](https://storage.googleapis.com/zenn-user-upload/bd8b5461a2ad-20251226.png)
*Argo CD が動く様子*

テスト用に [nginx](https://nginx.org/en/) だけ立てています。目下の目標は、[Mastodon](https://joinmastodon.org/) を立てて [X](https://x.com) を普段使いをするのをやめたい[^why-hate-x]です。

[^why-hate-x]: 考えを整理するために作業中や考えたことをマイクロブログに書き出すことがあるのですが、そのデータを自由に扱えないのが本当に嫌です。最近はアーカイブのリクエストすら通りません。本当に良くない態度だと思います

https://github.com/momeemt/config/blob/ef50a784a49fd04cc77a64e2122f7150a6328b56/k8s/nginx/deployment.yaml

YAML マニフェストへのパッチ適用ツールには [Kustomize](https://kustomize.io/) を使っています。

https://kustomize.io/

## 2026年にやりたいこと

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

記事が長くなりすぎてしまったため、投稿前に確認はしていますが誤字や誤りが含まれている可能性があります。ぜひコメント欄でご指摘いただくか、この記事を管理している [GitHub リポジトリ](https://github.com/momeemt/Zenn/blob/main/articles/dotfiles2025.md)の方にご連絡ください。
もし私の dotfiles に興味を持っていただけた方は、ぜひリポジトリにスターを付けたり、コンテナイメージを試したり、コメントで感想を送ったりしていただければ励みになります！

https://github.com/momeemt/config
