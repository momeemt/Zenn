---
title: "依存関係を解決する"
---

## 準備

この章の実装を始める前に、以下のコマンドで初期状態に移動してください。

```bash
$ git checkout chapter4-begin
```

---

前章では、ターゲットを手動で正しい順序で指定する必要がありました。`python minimake.py hello.o hello` のように、依存先を先に書かなければなりませんでした。

ファイルが2つならまだよいですが、100個あった場合、正しい順序を覚えておくのは不可能です。この章では、ビルド定義に依存関係を記述し、自動的に正しい順序でビルドする機能を実装します。

## 依存関係の定義

まず、ビルド定義に `deps`（依存関係）フィールドを追加します。

```json
{
  "targets": {
    "hello.o": {
      "deps": [],
      "command": "gcc -c -o hello.o hello.c"
    },
    "hello": {
      "deps": ["hello.o"],
      "command": "gcc -o hello hello.o"
    }
  }
}
```

`hello` は `hello.o` に依存しているので、`deps` に `["hello.o"]` を指定します。これにより、`hello` をビルドする前に `hello.o` が自動的にビルドされるようになります。

## 依存グラフとトポロジカルソート

依存関係はグラフとして表現できます。

```
hello.o ← hello
```

矢印は「A をビルドするには B が必要」という関係を表します。このグラフを使って、正しいビルド順序を決定する必要があります。

これを解決するのがトポロジカルソートです。トポロジカルソートは、有向非巡回グラフ（DAG: Directed Acyclic Graph）の頂点を、すべての辺が「前から後ろ」に向くように並べるアルゴリズムです[^dag]。

[^dag]: 有向非巡回グラフとは、矢印に向きがあり、かつ循環（A→B→C→A のようなループ）がないグラフのことです。依存関係に循環があるとビルドできないため、ビルドシステムでは DAG であることを前提としています。Git のコミット履歴も DAG になっています。

### より複雑な例

もう少し複雑な依存関係を見てみましょう。

```
        ┌─ a.o ─┐
        │       │
main ───┼─ b.o ─┼─── app
        │       │
        └─ c.o ─┘
```

この依存グラフをトポロジカルソートすると、以下のような順序が得られます。

- `a.o` → `b.o` → `c.o` → `app`
- `b.o` → `a.o` → `c.o` → `app`
- `c.o` → `b.o` → `a.o` → `app`

いずれも正しいビルド順序です。重要なのは、`app` が必ず最後に来ることです。`a.o`、`b.o`、`c.o` の順番は入れ替わっても問題ありません。

:::message
トポロジカルソートの結果は一意ではありません。複数の正しい順序が存在し得ます。Make は通常、Makefile に記述された順序を優先しますが、並列ビルド（`-j` オプション）では順序が変わることもあります。
:::

## アルゴリズム

深さ優先探索（DFS: Depth-First Search）を使ったトポロジカルソートを実装します[^dfs]。

[^dfs]: 深さ優先探索は、グラフを探索するための基本的なアルゴリズムです。「行けるところまで進んで、行き止まりになったら戻る」という戦略で、迷路を解くときの方法に似ています。

アルゴリズムの流れは以下の通りです。

1. 指定されたターゲットから開始
2. そのターゲットの依存先を再帰的に処理
3. すべての依存先を処理したら、そのターゲットを結果リストに追加
4. 結果リストはそのままビルド順序になる

ポイントは「依存先を先に処理する」ところです。これにより、自然と正しい順序が得られます。

## 課題: トポロジカルソートの実装

依存関係を解決し、ビルド順序を返す関数を実装してください。

```python
def resolve_build_order(config: dict, target: str) -> list[str]:
    # 依存関係を解決し、ビルド順序を返す
    #
    # config: ビルド定義
    # target: ビルドするターゲット
    # 戻り値: ビルドすべきターゲットのリスト（ビルド順）
    #
    # 例: resolve_build_order(config, "hello") -> ["hello.o", "hello"]

    targets = config.get("targets", {})

    visited = set()      # 処理済みのターゲット
    visiting = set()     # 現在処理中のターゲット（循環検出用）
    order = []           # ビルド順序

    def visit(t: str):
        if t in visited:
            return
        if t in visiting:
            raise ValueError(f"Circular dependency detected: {t}")

        if t not in targets:
            raise ValueError(f"Unknown target: {t}")

        visiting.add(t)

        # TODO: 依存先を再帰的に処理してください
        # ヒント: targets[t].get("deps", []) で依存先のリストを取得できます

        visiting.remove(t)
        visited.add(t)
        order.append(t)

    visit(target)
    return order
```

:::details 解答例
```python
def resolve_build_order(config: dict, target: str) -> list[str]:
    targets = config.get("targets", {})

    visited = set()
    visiting = set()
    order = []

    def visit(t: str):
        if t in visited:
            return
        if t in visiting:
            raise ValueError(f"Circular dependency detected: {t}")

        if t not in targets:
            raise ValueError(f"Unknown target: {t}")

        visiting.add(t)

        for dep in targets[t].get("deps", []):
            visit(dep)

        visiting.remove(t)
        visited.add(t)
        order.append(t)

    visit(target)
    return order
```
:::

コードは短いですが、再帰を使うことで、依存先を先に処理するという要件を自然に実現できています。

### 循環依存の検出

上のコードでは、`visiting` セットを使って循環依存を検出しています。

```
A → B → C → A （循環！）
```

あるターゲットを処理中（`visiting` に含まれる）に、再びそのターゲットに到達した場合、循環依存が存在します。

循環依存があるとビルドは不可能です。`A` をビルドするには `B` が必要、`B` には `C` が必要、`C` には `A` が必要...という無限ループになってしまいます[^chicken-egg]。

[^chicken-egg]: 循環依存は、ソフトウェア設計においても避けるべきアンチパターンです。モジュール A が B を参照し、B が A を参照しているような状態は、テストやリファクタリングを難しくします。

:::message
循環依存は設計上の問題を示唆していることが多いです。実際のプロジェクトで循環依存が発生した場合、モジュールの分割を見直すべきサインです。共通部分を別のモジュールに切り出すことで、循環を解消できることが多いです。
:::

## 実装を更新する

依存解決を組み込んだビルド処理を実装します。

```python
def build_with_deps(config: dict, target: str) -> bool:
    try:
        order = resolve_build_order(config, target)
    except ValueError as e:
        print(f"Error: {e}", file=sys.stderr)
        return False

    print(f"Build order: {' -> '.join(order)}")

    for t in order:
        if not build_target(config, t):
            return False

    return True
```

そして main 関数を更新します。

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
        if not build_with_deps(config, target):
            sys.exit(1)


if __name__ == "__main__":
    main()
```

## 動かしてみる

更新した `build.json` で試してみましょう。

```json
{
  "targets": {
    "hello.o": {
      "deps": [],
      "command": "gcc -c -o hello.o hello.c"
    },
    "hello": {
      "deps": ["hello.o"],
      "command": "gcc -o hello hello.o"
    }
  }
}
```

```bash
$ python minimake.py hello
Build order: hello.o -> hello
Building hello.o...
  $ gcc -c -o hello.o hello.c
Building hello...
  $ gcc -o hello hello.o
```

`hello` だけを指定しても、依存関係を解決して `hello.o` → `hello` の順でビルドされます。これで、正しい順序を覚えておく必要がなくなりました。

### 循環依存のテスト

循環依存がある場合のエラーも確認します。

```json
{
  "targets": {
    "a": {"deps": ["b"], "command": "echo a"},
    "b": {"deps": ["c"], "command": "echo b"},
    "c": {"deps": ["a"], "command": "echo c"}
  }
}
```

```bash
$ python minimake.py a
Error: Circular dependency detected: a
```

循環依存を検出できています。

## より複雑な例

複数のソースファイルを持つプロジェクトで試してみましょう。これは、実際の C プロジェクトに近い構成です。

**main.c**

```c
#include <stdio.h>
#include "greet.h"

int main(void) {
    greet("minimake");
    return 0;
}
```

**greet.h**

```c
#ifndef GREET_H
#define GREET_H

void greet(const char *name);

#endif
```

**greet.c**

```c
#include <stdio.h>
#include "greet.h"

void greet(const char *name) {
    printf("Hello, %s!\n", name);
}
```

**build.json**

```json
{
  "targets": {
    "main.o": {
      "deps": [],
      "command": "gcc -c -o main.o main.c"
    },
    "greet.o": {
      "deps": [],
      "command": "gcc -c -o greet.o greet.c"
    },
    "app": {
      "deps": ["main.o", "greet.o"],
      "command": "gcc -o app main.o greet.o"
    }
  }
}
```

```bash
$ python minimake.py app
Build order: main.o -> greet.o -> app
Building main.o...
  $ gcc -c -o main.o main.c
Building greet.o...
  $ gcc -c -o greet.o greet.c
Building app...
  $ gcc -o app main.o greet.o

$ ./app
Hello, minimake!
```

複数のファイルがあっても、依存関係を正しく解決してビルドできています。

## この章のまとめ

- `deps` フィールドで依存関係を定義
- トポロジカルソートで正しいビルド順序を決定
- 深さ優先探索（DFS）で依存グラフを探索
- `visiting` セットで循環依存を検出

これで、依存関係を考慮した基本的なビルドシステムが完成しました。`python minimake.py hello` と打つだけで、必要なターゲットがすべて正しい順序でビルドされます。

しかし、まだ問題が残っています。ファイルを変更していなくても、毎回すべてのターゲットを再ビルドしてしまいます。100個のファイルがあるプロジェクトで、1つだけ変更したのに全部ビルドし直すのは非効率です。

次の章では、インクリメンタルビルドを実装して、変更があった部分だけを再ビルドするようにします。

## 模範解答

この章の模範解答を確認したい場合は、以下のコマンドを実行してください。

```bash
$ git checkout chapter4-end
```
