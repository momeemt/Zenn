---
title: "Alacritty Hints と tmux でターミナル操作を快適にする"
emoji: "⌨️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Alacritty, tmux, Nix]
published: true
---

# はじめに

こんにちは。標準出力に表示された文字列を元に、何らかのシェル操作をしたくなる場面はよくあると思います。
ターミナルエミュレータに [Alacritty](https://github.com/alacritty/alacritty) を、ターミナルマルチプレクサに [tmux](https://github.com/tmux/tmux) を使っている環境を前提に、先ほどのような操作のうち頻出のものを、これらの機能を使って簡単に実行する方法を説明します。

:::message
この記事は、[coins Advent Calendar 2025](https://adventar.org/calendars/11747) の8日目の記事です。
:::

# モチベーション

コマンドを実行した時にそのコマンドが失敗し、あるファイルにログが書き出され、そのパスが標準出力に吐き出された場合について考えます。
そのような例として、`nix-build`によってパッケージをビルドしようとしたが失敗した際のログを以下に示します[^no-nix-assumed]。

[^no-nix-assumed]: 特に Nix の知識は仮定しません。

```
$ nix-build -A python313Packages.nfcpy
this derivation will be built:
  /nix/store/a2g691rxa9j5k4kdgczzf7safi0i61ms-python3.13-nfcpy-1.0.4.drv
these 9 paths will be fetched (0.80 MiB download, 5.13 MiB unpacked):
  /nix/store/z6z0cr5qc6f2j3dhly45zky8kzniq6v4-python3.13-libusb1-3.3.1
  /nix/store/ycnz7ys5ab5832wjkz4h3n661qiszr2f-python3.13-mock-5.2.0

# 略

       >         threading.Timer(0.03, dlc.close).start()
       > >       with pytest.raises(RuntimeError) as excinfo:
       >              ^^^^^^^^^^^^^^^^^^^^^^^^^^^
       > E       Failed: DID NOT RAISE <class 'RuntimeError'>
       >
       > tests/test_llcp_tco.py:494: Failed
       > =========================== short test summary info ============================
       > FAILED tests/test_llcp_tco.py::TestDataLinkConnection::test_recv_unexpected_pdu - Failed: DID NOT RAISE <class 'RuntimeError'>
       > ================= 1 failed, 2157 passed, 19 skipped in 37.22s ==================
       For full logs, run:
         nix log /nix/store/a2g691rxa9j5k4kdgczzf7safi0i61ms-python3.13-nfcpy-1.0.4.drv
```

aarch64-darwin[^aarch64-darwin]環境で [python3.13Packages.nfcpy](https://github.com/nfcpy/nfcpy) のビルドが失敗しました[^hydra-search][^twice-build]。
Nix ではビルドに失敗した時に `nix log` サブコマンドを利用して、ビルド失敗したパッケージのビルドログを参照することができます。
エラーと共に実行するべきコマンドが出力されます。具体的には `nix log /nix/store/a2g691rxa9j5k4kdgczzf7safi0i61ms-python3.13-nfcpy-1.0.4.drv` です。
これをターミナル上で実行するにはどのような方法が考えられるでしょうか。

1. マウスでコマンドを選択し、コピーし、貼り付ける
1. tmux のコピーモードを利用する

私はこれまでトラックパッドを常用していたので①か、長いログをコピーしたい場合には②を利用していました。
しかし、マウスを利用するにせよ tmux を利用するにせよ、目的のコマンドを実行するのに数秒掛かってしまいます。
そこで便利なのが Alacritty の Hints 機能です。

[^aarch64-darwin]: M1チップ以降を載せた macOS マシン。Apple Silicon は AArch64（ARMアーキテクチャの64ビット命令）を採用しているが、2019年までの macOS マシンは x86_64 を採用していた。
[^hydra-search]: Nix ベースの CI である [Hydra](https://hydra.nixos.org/) がパッケージリポジトリである [nixpkgs](https://github.com/NixOS/nixpkgs) の master ブランチを対象に日夜回り続けているので、[最新のビルド結果](https://hydra.nixos.org/eval/1820823)から適当に落ちているパッケージを選んでビルドした。
[^twice-build]: 余談ですが、2回ビルドすると2回目は成功する変なパッケージを選んでしまったので撮影に苦労しました。

# Alacritty Hints

## Hints とは

Hints とは、ターミナルに表示されているテキストを正規表現で検索し、マッチしたテキストを他のプロセスに引数として渡せる機能です。
具体的には、正規表現によって表示されているテキストを絞り込み、Vim プラグインの [Easymotion](https://github.com/easymotion/vim-easymotion) のように対象のテキストを選択し、あらかじめ設定ファイルで指定したコマンドの引数として渡します。
例として、`Ctrl+Shift+O` によって URL を開くヒントの定義を見てみましょう。

```toml
[[hints.enabled]]
command = "open"
hyperlinks = true
post_processing = true
regex = "(https?://[^\\s]+)"

[hints.enabled.binding]
key = "O"
mods = "Control|Shift"
```

`regex` に指定した正規表現にマッチする文字列が選択されます。
`(https?://[^\\s]+)` は、`http` か `https` から開始して、`://` が続き、空白以外の文字が1文字以上続く文字列にマッチします。
URLの定義とは異なりかなり簡略化されていますが、大抵の場合は上手くいくので問題なさそうです。

実際に、URLがターミナルに表示されている場面で発動させてみます。

![](https://storage.googleapis.com/zenn-user-upload/7f8433482c84-20251208.png =500x)
*3つの URL がマッチして、それぞれ k, f, j とハイライトされている*

Ctrl+Shift+O を入力すると、先述のように Easymotion と同様のハイライトが行われます。
このあと、「k」を入力すれば `echo https://momee.mt` に含まれる URL が、「f」を入力すれば標準出力に出力された URL が、「j」を入力すれば nixpkgs のリモート URL が選択され、設定ファイルの `command` で入力したコマンドの引数として渡されます。
すなわち、`open https://momee.mt` を Alacritty が実行することで、デフォルトブラウザで指定した URL の Web サイトを閲覧することができます。

ちなみに、`post_processing` を有効にすると、マッチしたテキストの末尾に付いている句読点など、ヒントとして不要そうな文字を取り除いてくれます。

![](https://storage.googleapis.com/zenn-user-upload/23ae01fe7d72-20251208.png =500x)
*末尾のピリオドが除かれている*

## `nix log` を実行する

では、同じ要領でモチベーションであった `nix log` を実行してみましょう。
愚直に考えれば、`nix log /nix/store/...` に一致する文字列をそのままシェルの `-c` オプションに渡せば実行できそうです。
すなわち、以下のような設定が考えられます。

```toml
[[hints.enabled]]
command = { program = "bash", args = [ "-c" ] }
post_processing = true
regex = "(nix log /nix/store/[^\\s]+)"

[hints.enabled.binding]
key = "L"
mods = "Control|Shift"
```

次に、この設定を反映させて Ctrl+Shift+L を押下します。

![](https://storage.googleapis.com/zenn-user-upload/68423d2e33f7-20251208.gif =500x)
*何も起こらない*

しかし、この設定では `nix log /nix/store/...` にマッチはしますが、その後選択しても何も起こっていないように見えます。
考えれば当然の話で、ヒントによって起動されるプロセスは、いま入力しているシェルとは別プロセスとして立ち上がります。その標準出力は、現在表示している tmux のペインにはつながっていないため、裏側でログを吐いて正常終了しても、画面上は「何も起きていない」ように見えます。
したがって、何らかの方法でそちらのシェルの標準出力を今使っているシェルに送るか、あるいは別の方法を選択する必要があります。

### 解決策1: Paste を利用する

1つ目の解決策として、Paste が挙げられます。
Paste は、ヒントでマッチしたテキストをターミナルに貼り付けるアクションで、次のように設定できます。

```toml
[[hints.enabled]]
action = "Paste"
post_processing = true
regex = "(nix log /nix/store/[^\\s]+)"

[hints.enabled.binding]
key = "L"
mods = "Control|Shift"
```

同様に、この設定を反映させて Ctrl+Shift+L を押下します。

![](https://storage.googleapis.com/zenn-user-upload/1e72630fe6b0-20251208.gif =500x)
*コマンドが入力された*

これは便利ですね。マウスを使ったり、tmux のコピーモードに入ることなく対象のコマンドを入力欄にペーストすることができました。

### 解決策2: tmux を利用する

もしあなたが tmux ユーザなら、対象のコマンドを入力欄にペーストした上で実行まですることができます。
`tmux` の `send-keys` を使えば、任意のペインに対してキーボード入力を送信することができます。ペインの指定を省略すれば、現在アクティブなペインに入力が送られます。
したがって、以下のようなスクリプトを使えば `nix log /nix/store/...` を丸ごと送信し、`Enter` によって実行まで完了させることができます。

```sh
#!/bin/bash
cmd="$1"
tmux send-keys "$cmd" Enter
```

同様に、この設定を反映させて Ctrl+Shift+L を押下します。

![](https://storage.googleapis.com/zenn-user-upload/31ccaeee78a7-20251208.gif =500x)
*コマンドが実行された*

さらに短縮することができました。
`nix log` は単なる一例で、他にはターミナル上のパスが示すファイルを Neovim で開くヒントも書いています。
ターミナルに表示された文字列をコピーすることはソフトウェア開発では避けられない操作なので、もしマウスやトラックパッドに手が伸びる瞬間があればその度にヒントを追加していくのも良いのではないでしょうか。

# home-manager による設定

:::message
この記事は、[Nix Advent Calendar 2025](https://adventar.org/calendars/11657) の8日目の記事です。
:::

ここまでに述べたような、他のツールやスクリプトに依存する設定を書きたい場合には [Nix](https://github.com/NixOS/nix) が便利です。
Alacritty の設定を素直に TOML で書くと、たとえば tmux がインストールされていない環境だったり、スクリプトの設置を忘れていたりすると、その設定だけ効かずに困ることがあります。
Nix は本来、パッケージビルドのための DSL とパッケージマネージャの仕組みですが、ユーザ空間の設定を行うための [home-manager](https://github.com/nix-community/home-manager) というツールもよく利用されており、公式のパッケージリポジトリである [nixpkgs](https://github.com/NixOS/nixpkgs) からパッケージを入手する形で依存解決を行うことができます。
もし先ほどの設定を Nix で書いて反映させれば、必ず tmux とスクリプトが一緒に配置されるため、無用な手作業が減り、設定が効かずに困ることも激減するでしょう。

## OS による設定の変更

最初に `open` コマンドを用いたアプリケーションの起動を行うヒントをお見せしましたが、Linux ユーザは `xdg-open` を使用する必要があります。
素直に TOML で管理すれば OS ごとに設定ファイルを分けなければいけません。
一方、Nix では nixpkgs がホスト OS を判定するための関数である `pkgs.stdenv.isLinux`、`pkgs.stdenv.isDarwin` を提供しているため、以下のように記述できます。

```nix
{pkgs, ...}: {
  programs.alacritty.settings.hints = {
    alphabet = "jfkdls;ahgurieowpq"; # default
    enabled = [
      {
        regex = "(https?://[^\\\\s]+)";
        post_processing = true;
        hyperlinks = true;
        command =
          if pkgs.stdenv.isDarwin
          then "open"
          else "xdg-open";
        binding = {
          key = "O";
          mods = "Control|Shift";
        };
      }
    ];
  };
}
```

## 依存関係の解決

次に、`nix log` を実行するヒントのように外部プログラム（tmux やスクリプト）に依存する設定を記述します。
まず、`pkgs` はパッケージリポジトリの nixpkgs であり、12万件以上のパッケージを利用できます。
Nix は遅延評価ですので、`pkgs.tmux` が評価されるタイミングでリポジトリに含まれるパッケージ定義により tmux がビルドされ（あるいはバイナリキャッシュから tmux がダウンロードされ）、`/nix/store/`配下に配置されます。
また、`pkgs.writeShellScriptBin` に渡されたスクリプトも同様に `/nix/store/`配下に配置されます。
したがって、この設定が評価されて反映に成功した場合、必ずこれらの外部依存を呼び出せることが保証されます。

```nix
{pkgs, ...}: let
  exec-arg = pkgs.writeShellScriptBin "exec-arg" ''
    cmd="$1"
    ${pkgs.tmux}/bin/tmux send-keys "$cmd" Enter
  '';
in {
  programs.alacritty.settings.hints = {
    alphabet = "jfkdls;ahgurieowpq"; # default
    enabled = [
      {
        regex = "(nix log /nix/store/[^\\\\s]+)";
        post_processing = true;
        command = "${exec-arg}/bin/exec-arg";
        binding = {
          key = "L";
          mods = "Control|Shift";
        };
      }
    ];
  };
}
```

Nix は難しそうな謎のビルドシステムだという印象をお持ちの方も多いと思いますが、大量のパッケージを提供する公式のパッケージリポジトリが利用でき、各種ツールによって異なる設定言語を Nix という DSL に統一して管理できるという点で、非常に分かりやすく便利なツールです。
特に、下のようなコマンドを実行することで、パッケージが利用できるシェル環境に入ることができる `nix-shell` というキラーツールを使ってみるだけでも、Nix を始める価値があるはずです。

```sh
$ go
The program 'go' is currently not installed. It is provided by
several packages. You can install it by typing one of the following:
  nix profile install nixpkgs#go_1_19.out
  nix profile install nixpkgs#go_1_20.out
  nix profile install nixpkgs#go.out

Or run it once with:
  nix shell nixpkgs#go_1_19.out -c go ...
  nix shell nixpkgs#go_1_20.out -c go ...
  nix shell nixpkgs#go.out -c go ...

$ nix-shell -p go
$ go version
go version go1.25.4 darwin/arm64
```

# おわりに

うっかり正規表現を書き間違えてすべてのテキストにマッチさせてしまうと、大量の Hint を処理してメモリを食い潰します。お気を付けください。

![Alacrittyが328.40GBのメモリを確保してすべてのアプリが止まる様子](https://storage.googleapis.com/zenn-user-upload/fb70e6c94c27-20251206.png =500x)
*Alacrittyが328.40GBのメモリを確保してすべてのアプリが止まる様子*

最後に Alacritty やその他のツールの設定集を紹介して終わります。それでは。

https://github.com/momeemt/config
