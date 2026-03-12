---
title: "インクリメンタルビルド"
---

## 準備

この章の実装を始める前に、以下のコマンドで初期状態に移動してください。

```bash
$ git checkout chapter5-begin
```

---

前章までで、依存関係を解決して正しい順序でビルドできるようになりました。しかし、現在の実装には大きな問題があります。毎回すべてのターゲットを再ビルドしていることです。

たとえば、100個のソースファイルがあるプロジェクトで1つだけ変更した場合を考えます。現在の実装では、変更されていない99個のファイルも含めて、すべてを再コンパイルします。これは非常に非効率です。大規模プロジェクトでは、フルビルドに数十分かかることもあります[^chromium]。

[^chromium]: たとえば Chromium のフルビルドは、高性能なマシンでも30分以上かかることがあります。インクリメンタルビルドがなければ、コードを1行変えるたびに30分待つことになります。

この章では、変更があった部分だけを再ビルドするインクリメンタルビルドを実装します。

## インクリメンタルビルドの考え方

インクリメンタルビルドの基本的な考え方はシンプルです。

1. ターゲット（成果物）が存在しない → ビルドが必要
2. ターゲットよりも依存ファイルが新しい → ビルドが必要
3. それ以外 → スキップ

これを判定するために、ファイルの更新日時（mtime）を使います[^mtime]。

[^mtime]: mtime は modification time の略で、ファイルの内容が最後に変更された日時を表します。UNIX 系のファイルシステムでは、atime（アクセス日時）、ctime（メタデータ変更日時）、mtime の3種類のタイムスタンプが記録されています。`ls -l` で表示されるのは mtime です。

ソースファイルの mtime がオブジェクトファイルの mtime より新しければ再コンパイルが必要、という判定を行います。Make もこの方式を採用しています。

## ビルド定義の拡張

インクリメンタルビルドを実現するには、各ターゲットがどのファイルに依存しているかを知る必要があります。現在の `deps` は他のターゲットへの依存を表していますが、ソースファイルへの依存は記述していません。

`inputs` フィールドを追加して、ソースファイルを明示的に指定できるようにします。

```json
{
  "targets": {
    "hello.o": {
      "deps": [],
      "inputs": ["hello.c"],
      "command": "gcc -c -o hello.o hello.c"
    },
    "hello": {
      "deps": ["hello.o"],
      "inputs": ["hello.o"],
      "command": "gcc -o hello hello.o"
    }
  }
}
```

:::message
`deps` と `inputs` の違いを整理します。`deps` は「他のターゲットへの依存」を表し、ビルド順序の決定に使われます。`inputs` は「ソースファイルへの依存」を表し、再ビルドの判定に使われます。実際のビルドシステム（Make など）では、この区別がより曖昧になっていることもあります。Make ではターゲットと入力ファイルを同じように扱えるため、この区別が不要です。
:::

## 課題: 再ビルドが必要かどうかの判定

ターゲットの再ビルドが必要かどうかを判定する関数を実装してください。

```python
import os
from pathlib import Path


def needs_rebuild(target: str, inputs: list[str]) -> bool:
    # ターゲットの再ビルドが必要かどうかを判定する
    #
    # 以下のロジックを実装してください:
    # 1. ターゲットファイルが存在しない場合は True を返す
    # 2. いずれかの入力ファイルがターゲットより新しい場合は True を返す
    # 3. それ以外は False を返す
    #
    # ヒント: Path.stat().st_mtime でファイルの更新日時を取得できます
    target_path = Path(target)

    # TODO: ここを実装してください
    pass
```

:::details 解答例
```python
def needs_rebuild(target: str, inputs: list[str]) -> bool:
    target_path = Path(target)

    # ターゲットが存在しない場合は必ずビルドが必要
    if not target_path.exists():
        return True

    target_mtime = target_path.stat().st_mtime

    # いずれかの入力ファイルがターゲットより新しければ再ビルドが必要
    for input_file in inputs:
        input_path = Path(input_file)
        if not input_path.exists():
            continue
        if input_path.stat().st_mtime > target_mtime:
            return True

    return False
```
:::

コードはシンプルです。ターゲットが存在しなければビルドが必要、存在していても入力ファイルのほうが新しければビルドが必要、という判定を行っています。

## build_target の更新

`build_target` 関数を更新して、再ビルドが不要な場合はスキップするようにします。

```python
def build_target(config: dict, target: str) -> bool:
    targets = config.get("targets", {})
    target_config = targets[target]

    inputs = target_config.get("inputs", [])
    command = target_config.get("command")

    # 再ビルドが必要かどうかを判定
    if not needs_rebuild(target, inputs):
        print(f"Skipping {target} (up to date)")
        return True

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

## 動かしてみる

更新した `build.json` で試してみましょう。

```bash
# 最初のビルド - すべてのターゲットがビルドされる
$ python minimake.py hello
Build order: hello.o -> hello
Building hello.o...
  $ gcc -c -o hello.o hello.c
Building hello...
  $ gcc -o hello hello.o

# 2回目のビルド - 何も変更していないのでスキップされる
$ python minimake.py hello
Build order: hello.o -> hello
Skipping hello.o (up to date)
Skipping hello (up to date)

# hello.c を変更する
$ touch hello.c

# 3回目のビルド - hello.c が変更されたので hello.o が再ビルドされる
$ python minimake.py hello
Build order: hello.o -> hello
Building hello.o...
  $ gcc -c -o hello.o hello.c
Building hello...
  $ gcc -o hello hello.o
```

2回目のビルドでは何も変更していないので、すべてのターゲットがスキップされています。

`touch` コマンドはファイルの更新日時を現在時刻に変更します。これにより、`hello.c` が変更されたとみなされ、再ビルドが走ります[^touch]。

[^touch]: `touch` コマンドは本来、ファイルが存在しなければ作成し、存在すれば更新日時を更新するコマンドです。ビルドシステムのテストでは、ファイルを変更したことをシミュレートするためによく使われます。`touch -t 197001010000 file` のように、特定の日時を指定することもできます。

## 課題: 依存ターゲットの考慮

現在の実装では `inputs` に指定されたファイルのみをチェックしていますが、依存ターゲット（`deps`）も考慮する必要があります。

たとえば、`hello` は `hello.o` に依存しています。`hello.o` が再ビルドされた場合、`hello` も再ビルドする必要があります。現在の実装では、`hello.o` が再ビルドされても `hello` がスキップされてしまう可能性があります。

`needs_rebuild` 関数を拡張して、`deps` もチェックするようにしてください。

```python
def needs_rebuild(config: dict, target: str) -> bool:
    # inputs に加えて deps もチェックするように拡張してください
    targets = config.get("targets", {})
    target_config = targets[target]

    target_path = Path(target)

    if not target_path.exists():
        return True

    target_mtime = target_path.stat().st_mtime

    # TODO: inputs と deps の両方をチェックしてください
    pass
```

:::details 解答例
```python
def needs_rebuild(config: dict, target: str) -> bool:
    targets = config.get("targets", {})
    target_config = targets[target]

    target_path = Path(target)

    if not target_path.exists():
        return True

    target_mtime = target_path.stat().st_mtime

    # inputs のチェック
    inputs = target_config.get("inputs", [])
    for input_file in inputs:
        input_path = Path(input_file)
        if input_path.exists() and input_path.stat().st_mtime > target_mtime:
            return True

    # deps（依存ターゲット）のチェック
    deps = target_config.get("deps", [])
    for dep in deps:
        dep_path = Path(dep)
        if dep_path.exists() and dep_path.stat().st_mtime > target_mtime:
            return True

    return False
```
:::

これで、依存ターゲットが更新された場合も正しく再ビルドされるようになりました。

## mtime ベースの限界

ファイルの更新日時を使った方法はシンプルで高速ですが、いくつかの限界があります。

1. 時刻の巻き戻し
    - システム時刻が変更されると、正しく動作しない可能性がある
2. 内容が同じでも再ビルド
    - ファイルを開いて保存しただけで（内容が変わっていなくても）再ビルドされる
3. ネットワークファイルシステム
    - NFS などでは時刻の同期が問題になることがある[^nfs]

[^nfs]: NFS（Network File System）では、サーバーとクライアントの時刻がずれていると、ファイルの更新日時を正しく比較できません。これを解決するために、`make` には `-t`（touch）オプションがあり、タイムスタンプを強制的に更新できます。

これらの問題を解決するには、ファイルの内容のハッシュを使う方法があります。ビルドキャッシュの章で詳しく扱いますが、ファイルの SHA-256 ハッシュを計算して比較することで、内容が本当に変わったかどうかを正確に判定できます。

:::message
Make では `--touch`（`-t`）オプションでターゲットのタイムスタンプを更新したり、`--assume-old`（`-o`）オプションで特定のファイルを古いとみなしたりできます。これらはデバッグや特殊な状況で役立ちます。たとえば、ヘッダファイルを変更したけど影響がないことがわかっている場合に、`-o` を使って再ビルドを抑制できます。
:::

## 実際のビルドシステムの実装

実際のビルドシステムでは、もう少し洗練された方法が使われています。

**Make** は基本的に mtime を使いますが、`-B`（always-make）オプションで強制的に全ビルドしたり、`-n`（dry-run）で実際には実行せずに何がビルドされるかを確認したりできます。

**Ninja** も mtime を使いますが、ビルドログ（`.ninja_log`）を保持して、前回のビルドで使われたコマンドを記録しています。コマンドが変わった場合も再ビルドの対象になります[^ninja-log]。

[^ninja-log]: たとえば、`gcc -O2` から `gcc -O3` に最適化レベルを変更した場合、ソースファイルの mtime は変わりませんが、コマンドが変わったので再ビルドが必要です。Ninja はこれを検出できます。

**Bazel** や **Buck** はファイルの内容のハッシュを使います。これにより、mtime の問題を回避しつつ、より正確なインクリメンタルビルドを実現しています。

## この章のまとめ

- `inputs` フィールドでソースファイルへの依存を明示
- ファイルの更新日時（mtime）を比較して再ビルドの要否を判定
- 変更がなければビルドをスキップして高速化
- `deps` の依存ターゲットも mtime をチェック

これで、基本的なビルドシステムとしての機能が揃いました。

- 第3章: JSON で定義されたターゲットをビルド
- 第4章: 依存関係を解決して正しい順序でビルド
- 第5章: 変更があった部分だけを再ビルド

次の章からは、より高度な機能を独立して実装していきます。興味のあるトピックを選んで、挑戦してみてください。

## 模範解答

この章の模範解答を確認したい場合は、以下のコマンドを実行してください。

```bash
$ git checkout chapter5-end
```
