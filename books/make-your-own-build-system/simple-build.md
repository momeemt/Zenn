---
title: "シンプルなビルドの実行"
---

## 準備

この章の実装を始める前に、以下のコマンドで初期状態からブランチを作成してください。

```bash
$ git switch -c my-chapter3 chapter3-begin
```

リポジトリのディレクトリ構造は以下のようになっています。

```
minimake/
├── src/
│   └── minimake.py    # ビルドシステムのソースコード
├── project/
│   ├── build.json     # ビルド定義ファイル
│   └── hello.c        # ビルド対象のCプログラム
└── tests/             # テストファイル
```

実装は `src/minimake.py` を編集し、動作確認は `project/` ディレクトリで行います。

---

この章では、minimake の最初のバージョンを作ります。まずは JSON で定義されたターゲットをビルドするというシンプルな機能から始めます。

## ビルド定義ファイルの設計

ビルドシステムには、何をどうビルドするかを記述した定義ファイルが必要です。本書では JSON 形式を採用します[^why-json]。

[^why-json]: JSON を選んだ理由は、Python の標準ライブラリだけでパースできること、多くの人が読み書きに慣れていることです。実際のビルドシステムでは YAML、TOML、独自DSL など様々な形式が使われています。Make は独自の文法を持っていますし、Bazel は Starlark（Python風の言語）を使っています。

```json
{
  "targets": {
    "hello.o": {
      "command": "gcc -c -o hello.o hello.c"
    },
    "hello": {
      "command": "gcc -o hello hello.o"
    }
  }
}
```

各ターゲットには `command` を指定します。ターゲット名は生成される成果物のファイル名にするのが慣例です。

:::message
Make の用語では、生成されるファイルを「ターゲット」、ターゲットを生成するために必要なファイルを「依存」と呼びます。この講義でも同じ用語を使います。
:::

## 課題: ビルド定義ファイルの読み込み

まずは、ビルド定義ファイルを読み込む関数を実装してください。

```python
import json


def load_build_file(path: str) -> dict:
    # ビルド定義ファイル（JSON）を読み込んで辞書として返す
    #
    # path: ファイルパス（例: "build.json"）
    # 戻り値: パースされた辞書

    # TODO: ここを実装してください
    pass
```

:::details 解答例
```python
import json


def load_build_file(path: str) -> dict:
    with open(path) as f:
        return json.load(f)
```
:::

JSON の読み込みは `json.load()` を使えば1行で書けます。Python ではファイルを開くときに `with` 文を使うのが定石です[^with-statement]。これにより、例外が発生してもファイルが確実に閉じられます。

[^with-statement]: `with` 文は Python のコンテキストマネージャを利用した構文です。ファイルを開いた後に `close()` を呼び忘れるバグを防げます。

## 課題: ターゲットのビルド

次に、指定されたターゲットのコマンドを実行する関数を実装してください。

```python
import subprocess
import sys


def build_target(config: dict, target: str) -> bool:
    # 指定されたターゲットをビルドする
    #
    # config: load_build_file で読み込んだ設定
    # target: ビルドするターゲット名（例: "hello.o"）
    # 戻り値: ビルド成功なら True、失敗なら False

    targets = config.get("targets", {})

    # ターゲットが存在するか確認
    if target not in targets:
        print(f"Error: Unknown target '{target}'", file=sys.stderr)
        return False

    target_config = targets[target]
    command = target_config.get("command")

    # コマンドが指定されているか確認
    if not command:
        print(f"Error: No command for target '{target}'", file=sys.stderr)
        return False

    print(f"Building {target}...")
    print(f"  $ {command}")

    # TODO: ここでコマンドを実行してください
    # ヒント: subprocess.run() を使います
    # shell=True を指定すると、シェルコマンドとして実行できます
    pass
```

:::details 解答例
```python
import subprocess
import sys


def build_target(config: dict, target: str) -> bool:
    targets = config.get("targets", {})

    if target not in targets:
        print(f"Error: Unknown target '{target}'", file=sys.stderr)
        return False

    target_config = targets[target]
    command = target_config.get("command")

    if not command:
        print(f"Error: No command for target '{target}'", file=sys.stderr)
        return False

    print(f"Building {target}...")
    print(f"  $ {command}")

    result = subprocess.run(command, shell=True)

    if result.returncode != 0:
        print(f"Error: Build failed for '{target}'", file=sys.stderr)
        return False

    return True
```
:::

### subprocess.run について

`subprocess.run()` は外部コマンドを実行するための関数です[^subprocess]。

[^subprocess]: Python 3.5 で導入された関数で、それ以前の `os.system()` や `subprocess.call()` よりも使いやすく安全です。戻り値の `CompletedProcess` オブジェクトから終了コードや出力を取得できます。

```python
result = subprocess.run(command, shell=True)
```

- `command`: 実行するコマンド（文字列）
- `shell=True`: シェルを経由してコマンドを実行する
- `result.returncode`: コマンドの終了コード（0 なら成功）

:::message alert
`shell=True` はシェルインジェクションの脆弱性を生む可能性があります。ユーザー入力を直接コマンドに含める場合は注意が必要です。この講義では簡単のため `shell=True` を使いますが、本番環境では `shlex.split()` でコマンドを分割し、`shell=False`（デフォルト）で実行することを推奨します。
:::

## main 関数の実装

コマンドライン引数を処理する main 関数を実装します。

```python
def main():
    if len(sys.argv) < 2:
        print("Usage: minimake <target> [build_file]", file=sys.stderr)
        sys.exit(1)

    target = sys.argv[1]
    build_file = sys.argv[2] if len(sys.argv) > 2 else "build.json"

    config = load_build_file(build_file)

    if not build_target(config, target):
        sys.exit(1)


if __name__ == "__main__":
    main()
```

`sys.argv` はコマンドライン引数のリストです。`sys.argv[0]` はスクリプト名、`sys.argv[1]` 以降が引数です。

## 動かしてみる

簡単なCプログラムで試してみましょう。

**hello.c**

```c
#include <stdio.h>

int main(void) {
    printf("Hello, minimake!\n");
    return 0;
}
```

**build.json**

```json
{
  "targets": {
    "hello.o": {
      "command": "gcc -c -o hello.o hello.c"
    },
    "hello": {
      "command": "gcc -o hello hello.o"
    }
  }
}
```

ターミナルで実行します。

```bash
$ cd project
$ python ../src/minimake.py hello.o
Building hello.o...
  $ gcc -c -o hello.o hello.c

$ python ../src/minimake.py hello
Building hello...
  $ gcc -o hello hello.o

$ ./hello
Hello, minimake!
```

動作しました。しかし、いくつか問題があります。

## 現状の問題点

### 問題1: ビルド順序を手動で指定する必要がある

`hello` をビルドするには、先に `hello.o` をビルドしておく必要があります。しかし、現在の実装ではこの順序を人間が把握して、正しい順番でコマンドを実行しなければなりません。

```bash
# 正しい順序
$ python ../src/minimake.py hello.o
$ python ../src/minimake.py hello

# 間違った順序 - hello.o がないのでリンクに失敗
$ python ../src/minimake.py hello
Building hello...
  $ gcc -o hello hello.o
/usr/bin/ld: cannot find hello.o: No such file or directory
```

ファイルが2つならまだよいですが、100個あった場合、正しい順序を覚えておくのは不可能です。これは Make が解決した最初の問題です。Make では依存関係を定義することで、自動的に正しい順序でビルドされます。

### 問題2: 複数ターゲットを一度にビルドできない

1つのコマンドで複数のターゲットをビルドしたい場合も、1つずつ実行する必要があります。

## 課題: 複数ターゲットのビルドに対応する

複数のターゲットを順番にビルドできるように、main 関数を修正してください。

```python
def main():
    if len(sys.argv) < 2:
        print("Usage: minimake <target>... [--file build_file]", file=sys.stderr)
        sys.exit(1)

    # TODO: 引数をパースして、複数のターゲットを順番にビルドできるようにしてください
    # --file オプションでビルド定義ファイルを指定できるようにしてください
    pass
```

:::details 解答例
```python
def main():
    if len(sys.argv) < 2:
        print("Usage: minimake <target>... [--file build_file]", file=sys.stderr)
        sys.exit(1)

    targets = []
    build_file = "build.json"

    i = 1
    while i < len(sys.argv):
        if sys.argv[i] == "--file" and i + 1 < len(sys.argv):
            build_file = sys.argv[i + 1]
            i += 2
        else:
            targets.append(sys.argv[i])
            i += 1

    config = load_build_file(build_file)

    for target in targets:
        if not build_target(config, target):
            sys.exit(1)


if __name__ == "__main__":
    main()
```
:::

これで、次のように実行できます。

```bash
$ python ../src/minimake.py hello.o hello
Building hello.o...
  $ gcc -c -o hello.o hello.c
Building hello...
  $ gcc -o hello hello.o
```

しかし、ビルド順序を人間が指定しなければならない問題は解決していません。次の章では、依存関係を定義ファイルに記述し、自動的に正しい順序でビルドする機能を実装します。

## この章のまとめ

- JSON 形式でビルドターゲットを定義
- `subprocess.run()` でコマンドを実行
- 複数ターゲットの逐次ビルドに対応

次の章では、依存関係を定義し、自動的に正しい順序でビルドする機能を追加します。これにより、`python minimake.py hello` だけで必要なすべてのターゲットがビルドされるようになります。

## 模範解答

この章の模範解答を確認したい場合は、以下のコマンドを実行してください。

```bash
$ git switch --detach chapter3-end
```
