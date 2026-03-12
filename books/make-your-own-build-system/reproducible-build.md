---
title: "再現可能なビルド"
---

## 準備

この章の実装を始める前に、以下のコマンドで初期状態に移動してください。

```bash
$ git checkout chapter10-begin
```

---

:::message
この章は発展課題です。第5章までの内容を理解していれば取り組むことができます。
:::

同じソースコードをビルドしたら、同じバイナリが生成される、という性質は当たり前のことのように思えますが、実際には多くのビルドシステムでこれが保証されていません。今日ビルドしたバイナリと明日ビルドしたバイナリが違う、ということが普通に起こります。

この章では、再現可能なビルド（Reproducible Builds）を実現するための技術を学びます[^reproducible-builds]。

[^reproducible-builds]: 再現可能なビルドは、セキュリティの観点から特に重要視されています。Debian、Arch Linux、NixOS などのディストリビューションは、パッケージの再現可能なビルドを推進しています。詳しくは https://reproducible-builds.org/ を参照してください。

## なぜ再現性が重要なのか

再現可能なビルドは、以下の点で重要です。

1. **セキュリティ**: ソースコードからバイナリが正しく生成されたことを第三者が検証できる[^supply-chain]
2. **デバッグ**: 過去の特定のバージョンを正確に再現してデバッグできる
3. **信頼性**: 「自分の環境では動く」問題を排除できる
4. **監査**: ビルド成果物の出所を追跡できる

[^supply-chain]: サプライチェーン攻撃では、ビルドプロセスにマルウェアを仕込むことで、ソースコードは正常なのに悪意のあるバイナリが配布されることがあります。再現可能なビルドにより、誰でもソースからバイナリを再構築して、公式のバイナリと比較することで改ざんを検出できます。

## 再現性を阻害する要因

同じソースコードから異なるバイナリが生成される原因はいくつかあります[^non-determinism]。

[^non-determinism]: 再現性を阻害する要因は、「意図的に埋め込まれたもの」（ビルド日時など）と「意図しないもの」（ファイル順序など）に分けられます。前者は設計の問題、後者は実装の問題です。

### 1. タイムスタンプ

多くのコンパイラは、ビルド日時をバイナリに埋め込みます。

```c
// 実行時に展開されるマクロ
printf("Built at: %s %s\n", __DATE__, __TIME__);
```

### 2. ファイルの処理順序

ファイルシステムからファイルを列挙する順序は、環境によって異なる場合があります[^readdir]。

[^readdir]: POSIX の `readdir()` はディレクトリエントリの順序を保証しません。ファイルシステムの種類（ext4、APFS、NTFS など）や、ファイルの作成順序、ファイルシステムの内部状態によって順序が変わります。

```python
# 順序が不定
for file in os.listdir("."):
    process(file)
```

### 3. 環境変数

`PATH`、`HOME`、`USER` などの環境変数がビルド結果に影響することがあります。

### 4. ビルドパス

ソースコードの絶対パスがバイナリに埋め込まれることがあります。

```
/home/alice/project/main.c:10: error
/home/bob/work/main.c:10: error  # 同じエラーでもパスが異なる
```

## 課題: クリーンな環境変数の作成

ビルド時の環境変数を最小限に制限する関数を実装してください。

```python
import os


def create_clean_env() -> dict[str, str]:
    # クリーンな環境変数を作成する
    #
    # 以下の環境変数を設定してください:
    # - PATH: 最小限のパス（/usr/bin:/bin）
    # - HOME: 存在しないパス（/nonexistent）
    # - LANG, LC_ALL: C.UTF-8
    # - TZ: UTC
    # - SOURCE_DATE_EPOCH: 0（タイムスタンプを固定）

    # TODO: ここを実装してください
    pass
```

:::details 解答例
```python
def create_clean_env() -> dict[str, str]:
    return {
        "PATH": "/usr/bin:/bin",
        "HOME": "/nonexistent",
        "USER": "nobody",
        "LANG": "C.UTF-8",
        "LC_ALL": "C.UTF-8",
        "TZ": "UTC",
        "SOURCE_DATE_EPOCH": "0",
    }
```
:::

## SOURCE_DATE_EPOCH

`SOURCE_DATE_EPOCH` は、再現可能なビルドのために広く採用されている環境変数です[^sde]。この環境変数が設定されていると、多くのツールがこの値をタイムスタンプとして使用します。

[^sde]: `SOURCE_DATE_EPOCH` は2015年に Debian プロジェクトで提案され、現在では GCC、Clang、Python、ZIP、tar など多くのツールがサポートしています。値は Unix エポック（1970年1月1日からの秒数）で指定します。

```python
import os
import time

# SOURCE_DATE_EPOCH が設定されていればその値を使う
timestamp = int(os.environ.get("SOURCE_DATE_EPOCH", time.time()))
```

GCC、Clang、Python、ZIP など、多くのツールが対応しています。`SOURCE_DATE_EPOCH=0` を設定すると、すべてのタイムスタンプが1970年1月1日になります。

:::message
Git のコミット日時を `SOURCE_DATE_EPOCH` として使うことで、このコミットからビルドされたことを表現できます。`git log -1 --format=%ct` でコミットの Unix タイムスタンプを取得できます。
:::

## サンドボックス内でのビルド

クリーンな環境でビルドを実行します。

```python
def build_in_sandbox(command: str) -> subprocess.CompletedProcess:
    env = create_clean_env()

    return subprocess.run(
        command,
        shell=True,
        env=env,
        capture_output=True
    )
```

## ビルドパスの正規化

ソースコードのパスがバイナリに含まれないようにします。

### GCC/Clang のオプション

```bash
# デバッグ情報のパスを相対パスに
gcc -fdebug-prefix-map=/home/user/project=.

# マクロのパスも変換
gcc -fmacro-prefix-map=/home/user/project=.

# 両方をまとめて（GCC 8以降）
gcc -ffile-prefix-map=/home/user/project=.
```

これらのオプションを使うと、バイナリに含まれるパス情報が正規化されます。

### minimake での実装

```python
def normalize_build_command(command: str, source_dir: str) -> str:
    if "gcc" in command or "clang" in command:
        prefix_map = f"-ffile-prefix-map={source_dir}=."
        command = command.replace("gcc ", f"gcc {prefix_map} ")
        command = command.replace("clang ", f"clang {prefix_map} ")

    return command
```

## ファイル順序の固定

ファイルの処理順序を固定して、決定論的な結果を得ます。

```python
def list_files_sorted(directory: str) -> list[str]:
    files = os.listdir(directory)
    return sorted(files)
```

`sorted()` を使うだけですが、これを忘れると再現性が壊れることがあります。

## 再現性の検証

ビルドが本当に再現可能かどうかを検証するには、同じソースを2回ビルドして、結果を比較します[^verification]。

[^verification]: 理想的には、異なるマシン、異なる時刻でビルドしても同じ結果が得られることを確認すべきです。Reproducible Builds プロジェクトでは、複数の独立したビルドサーバーで同じパッケージをビルドし、結果を比較しています。

```python
import hashlib


def compute_file_hash(path: str) -> str:
    sha256 = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            sha256.update(chunk)
    return sha256.hexdigest()


def verify_reproducibility(build_func, target: str) -> bool:
    # 1回目のビルド
    build_func()
    first_hash = compute_file_hash(target)

    # 成果物を削除
    os.remove(target)

    # 2回目のビルド
    build_func()
    second_hash = compute_file_hash(target)

    # ハッシュを比較
    if first_hash == second_hash:
        print(f"Reproducible! Hash: {first_hash[:16]}...")
        return True
    else:
        print(f"NOT reproducible!")
        print(f"  First:  {first_hash[:16]}...")
        print(f"  Second: {second_hash[:16]}...")
        return False
```

## diffoscope による差分分析

ビルドが再現可能でない場合、[diffoscope](https://diffoscope.org/) を使って差分を分析できます[^diffoscope]。

[^diffoscope]: diffoscope は Debian プロジェクトで開発されたツールで、2つのファイルやディレクトリの意味的な差分を表示します。アーカイブを展開し、バイナリを逆アセンブルするなど、深い解析を行います。

```bash
$ diffoscope build1/hello build2/hello
--- build1/hello
+++ build2/hello
├── readelf --wide --notes {}
│ @@ -1,3 +1,3 @@
│  ...
│ -    Build ID: 0x1234...
│ +    Build ID: 0x5678...
```

diffoscope は、バイナリの内部構造を解析して、どの部分が異なるかを詳細に表示します。再現性を阻害している原因を特定するのに非常に役立ちます。

## minimake での再現可能ビルドモード

これまでの要素を組み合わせて、再現可能ビルドモードを実装します。

```python
def reproducible_build(config: dict, target: str) -> bool:
    targets = config.get("targets", {})

    try:
        order = resolve_build_order(config, target)
    except ValueError as e:
        print(f"Error: {e}", file=sys.stderr)
        return False

    # クリーンな環境を作成
    env = create_clean_env()

    for t in order:
        t_config = targets[t]
        command = t_config.get("command", "")

        # パスを正規化
        command = normalize_build_command(command, os.getcwd())

        print(f"Building {t} (reproducible mode)...")

        result = subprocess.run(
            command,
            shell=True,
            env=env,
            capture_output=True
        )

        if result.returncode != 0:
            print(f"Error: {result.stderr.decode()}")
            return False

    return True
```

## 発展: ファイルシステムの隔離

環境変数のクリーンアップだけでは、ビルドの再現性を完全には保証できません。ホストシステムのファイル（`/usr/lib` のライブラリ、`/etc` の設定ファイルなど）にアクセスできてしまうと、環境ごとに異なる結果になる可能性があります。

より厳密な再現性を実現するには、ファイルシステムレベルでビルド環境を隔離する必要があります。

### chroot によるルートディレクトリの変更

Unix 系システムでは、`chroot` を使ってプロセスのルートディレクトリを変更できます[^chroot]。

[^chroot]: `chroot` は1979年の Unix V7 で導入された古い機能です。セキュリティ目的では不十分ですが（root 権限があれば脱出可能）、ビルド環境の隔離には有効です。

```python
import os
import subprocess


def build_in_chroot(command: str, chroot_dir: str) -> subprocess.CompletedProcess:
    # chroot 環境でビルドを実行
    # 注意: root 権限が必要
    return subprocess.run(
        ["chroot", chroot_dir, "/bin/sh", "-c", command],
        capture_output=True
    )
```

ただし、`chroot` には root 権限が必要で、ネットワークや他のリソースは隔離されません。

### Linux namespaces による隔離

Linux の namespace 機能を使うと、より細かい隔離が可能です[^namespaces]。

[^namespaces]: Linux namespaces は、プロセスごとに異なるシステムリソースのビューを提供する機能です。mount namespace（ファイルシステム）、network namespace（ネットワーク）、PID namespace（プロセス ID）などがあり、コンテナ技術の基盤となっています。

```bash
# unshare コマンドで新しい namespace を作成
$ unshare --mount --net --pid --fork /bin/bash

# この中ではホストと異なるファイルシステムビューを持てる
```

`bubblewrap` は、namespace を使った軽量なサンドボックスツールです。Flatpak でも使われています。

```bash
# bubblewrap でビルドを隔離
$ bwrap \
    --ro-bind /usr /usr \
    --ro-bind /lib /lib \
    --bind ./src /build/src \
    --bind ./out /build/out \
    --unshare-net \
    --chdir /build \
    gcc -o out/hello src/hello.c
```

この例では:
- `/usr` と `/lib` を読み取り専用でマウント
- `./src` と `./out` だけを書き込み可能でマウント
- ネットワークを無効化（`--unshare-net`）

### ネットワーク隔離の仕組み

ビルド中にネットワークアクセスがあると、再現性が損なわれます。外部サーバーからダウンロードする内容は時間とともに変化する可能性があるためです[^network-nondeterminism]。

[^network-nondeterminism]: 例えば、ビルドスクリプト内で `curl https://example.com/latest.tar.gz` を実行すると、ダウンロード時期によって異なるファイルが取得されます。また、CDN のキャッシュや DNS の応答も環境によって異なる可能性があります。

Linux でネットワークを隔離する方法は主に2つあります。

**1. Network namespace**

`unshare --net` や Docker の `--network=none` は、network namespace を使用しています。新しい network namespace を作成すると、そのプロセスは空のネットワークスタック（ループバックインターフェースのみ）を持ちます。

```bash
# 新しい network namespace でコマンドを実行
$ unshare --net ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

# ホストのネットワークインターフェースは見えない
```

ホストの `eth0` などのインターフェースは見えなくなり、外部への通信ができなくなります。

**2. seccomp によるシステムコールの制限**

seccomp（secure computing mode）は、プロセスが呼び出せるシステムコールを制限する Linux カーネルの機能です[^seccomp]。ネットワーク関連のシステムコール（`socket`、`connect`、`sendto` など）をブロックすることで、ネットワークアクセスを禁止できます。

[^seccomp]: seccomp は Linux 2.6.12 で導入され、seccomp-bpf（BPF フィルタによる柔軟な制御）は Linux 3.5 で追加されました。Chrome、Firefox、Docker など多くのソフトウェアがセキュリティ強化に使用しています。

```c
// seccomp でネットワーク系 syscall をブロックする例（概念的なコード）
scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_ALLOW);
seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(socket), 0);
seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(connect), 0);
seccomp_load(ctx);
```

Nix のサンドボックスは、network namespace と seccomp を組み合わせて、より堅牢なネットワーク隔離を実現しています。

**各ツールのネットワーク隔離方法**

| ツール | 方法 | 備考 |
| --- | --- | --- |
| Docker | network namespace | `--network=none` で無効化 |
| bubblewrap | network namespace | `--unshare-net` で無効化 |
| Nix | namespace + seccomp | デフォルトで無効、FOD は例外 |
| Firejail | namespace + seccomp | セキュリティサンドボックス |

:::message
Nix では「固定出力ハッシュの derivation（FOD: Fixed-Output Derivation）」だけがネットワークアクセスを許可されます。FOD は出力のハッシュを事前に指定するため、ダウンロード内容が変わると検証に失敗します。これにより、ネットワークアクセスを許可しつつ再現性を保証しています。
:::

### Docker によるコンテナビルド

より手軽に隔離環境を構築するには、Docker を使う方法があります[^docker-reproducible]。

[^docker-reproducible]: Docker 自体は再現性を保証しませんが、ベースイメージとビルド手順を固定することで、ある程度の再現性を実現できます。ただし、`apt-get update` のようなコマンドは実行時期によって結果が変わるため注意が必要です。

```dockerfile
# Dockerfile
FROM ubuntu:22.04

# パッケージのバージョンを固定
RUN apt-get update && apt-get install -y \
    gcc=4:11.2.0-1ubuntu1 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build
COPY . .

# SOURCE_DATE_EPOCH を設定
ENV SOURCE_DATE_EPOCH=0

RUN gcc -o hello hello.c
```

```bash
# イメージをビルド
$ docker build -t myapp:reproducible .

# コンテナ内でビルドを実行
$ docker run --rm -v $(pwd):/build myapp:reproducible make
```

Docker の利点は、環境構築が宣言的で、チーム全体で同じ環境を共有しやすいことです。

### Nix のサンドボックス

Nix のサンドボックスは、これらの技術を組み合わせた高度な隔離を提供します[^nix-sandbox]。

[^nix-sandbox]: Nix のサンドボックスは Linux では namespaces と seccomp を、macOS では `sandbox-exec` を使用しています。ビルドに必要な依存関係だけを `/nix/store` から bind mount し、それ以外のファイルシステムへのアクセスを禁止します。

```bash
# Nix のサンドボックス設定（/etc/nix/nix.conf）
sandbox = true
```

Nix のサンドボックスでは:
- ネットワークアクセスを禁止（固定出力ハッシュの derivation を除く）
- `/nix/store` の依存パッケージのみを読み取り可能でマウント
- ビルドディレクトリは一時的で、ビルド後に成果物だけが `/nix/store` に保存される
- ホストの `/usr`、`/lib` などは見えない

この厳格な隔離により、依存関係として宣言していないものに依存するというバグを防げます。

## Nix のアプローチ

再現可能なビルドを追求しているビルドシステムとして、Nix があります[^nix]。Nix は以下のようなアプローチを採用しています。

[^nix]: Nix は2003年に Eelco Dolstra の博士論文として発表されました。「純粋関数型パッケージマネージャ」というコンセプトで、ビルドを副作用のない関数として扱います。同じ入力（ソース、依存関係、ビルドスクリプト）からは常に同じ出力が得られることを目指しています。

1. **すべての依存関係をハッシュで管理**: ソースコード、コンパイラ、ライブラリすべてのハッシュを計算
2. **サンドボックス**: ネットワークアクセスを禁止、最小限のファイルシステム
3. **固定されたビルドパス**: `/nix/store/<hash>-<name>` という形式
4. **宣言的な設定**: ビルド手順を関数として記述

```nix
# Nix での再現可能なビルドの例
{ stdenv }:
stdenv.mkDerivation {
  name = "hello";
  src = ./src;
  buildPhase = ''
    gcc -o hello hello.c
  '';
}
```

:::message
Nix のストアパス（`/nix/store/abc123...-hello`）は、パッケージの全入力のハッシュから決定されます。これにより、依存関係の異なるバージョンを同時にインストールでき、ビルドの再現性も保証されます。
:::

### 隔離レベルの比較

| 方法 | ファイルシステム | ネットワーク | 依存管理 | 導入の容易さ |
| --- | --- | --- | --- | --- |
| 環境変数のみ | × | × | × | ◎ |
| chroot | ○ | × | △ | △ |
| Docker | ○ | ○ | △ | ○ |
| bubblewrap | ○ | ○ | △ | △ |
| Nix サンドボックス | ◎ | ◎ | ◎ | △ |

Nix は最も厳格な隔離を提供しますが、学習コストが高いです。プロジェクトの要件に応じて適切な方法を選択してください。

興味がある方は、[Reproducible Builds](https://reproducible-builds.org/) プロジェクトのドキュメントを参照してください。

## 再現可能なビルドの課題

完全に再現可能なビルドを実現するのは、実際にはなかなか難しいです[^challenges]。

[^challenges]: Debian プロジェクトの統計によると、2024年時点で約95%のパッケージが再現可能なビルドに対応しています。残りの5%は、タイムスタンプやランダムな要素を含むため、まだ対応が進んでいません。

以下のような要因は、サンドボックスだけでは完全に制御できません。

- **カーネルバージョン**: `/proc/version` を読み込むプログラム
- **CPU機能**: CPUID に基づく最適化
- **メモリレイアウト**: ASLR（Address Space Layout Randomization）
- **署名**: ビルドごとに異なる署名

Nix や Guix のような高度なビルドシステムでも、100%の再現性を保証することは困難です。しかし、95%でも十分に価値があります。少なくとも「いつの間にかビルド結果が変わっていた」という問題の多くは解決できます。

## この章のまとめ

- 再現可能なビルドは、同じ入力から常に同じ出力を生成する
- タイムスタンプ、ファイル順序、環境変数が再現性を阻害する
- `SOURCE_DATE_EPOCH` でタイムスタンプを固定
- パス正規化オプションでビルドパスの影響を排除
- diffoscope で差分を分析
- ファイルシステムの隔離（chroot、namespace、Docker、Nix）でより厳密な再現性を実現

再現可能なビルドは、ソフトウェアの信頼性とセキュリティを高める重要な技術です。環境変数の制御から始めて、必要に応じてコンテナや Nix のような高度な隔離技術を導入することで、段階的に再現性を向上させることができます。

## 模範解答

この章の模範解答を確認したい場合は、以下のコマンドを実行してください。

```bash
$ git checkout chapter10-end
```
