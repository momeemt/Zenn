---
title: "並列ビルド"
---

## 準備

この章の実装を始める前に、以下のコマンドで初期状態からブランチを作成してください。

```bash
$ git switch -c my-chapter9 chapter9-begin
```

---

:::message
この章は発展課題です。第5章までの内容を理解していれば取り組むことができます。
:::

これまでの実装では、すべてのビルドを逐次的に実行していました。しかし、依存関係のないターゲットは同時にビルドできます。

この章では、複数のCPUコアを使って並列にビルドする機能を実装します。

## 並列ビルドの基本

依存関係のないターゲットは、同時にビルドできます[^parallelism]。たとえば、以下のような依存グラフを考えます。

[^parallelism]: 並列ビルドの効果は、プロジェクトの依存関係構造に大きく依存します。すべてのターゲットが直列に依存している場合は並列化の効果がありませんが、独立したモジュールが多い場合は大きな効果が得られます。

```
        ┌─ hello.o ─┐
        │           │
main ───┼─ greet.o ─┼─── app
        │           │
        └─ utils.o ─┘
```

この例では、`hello.o`、`greet.o`、`utils.o` は互いに依存していないので、並列にビルドできます。しかし、`app` は3つすべてに依存しているので、すべてが完了するまで待つ必要があります。

:::message
Make の `-j` オプションは、まさにこの並列ビルドを実現するものです。`make -j4` とすると、最大4つのジョブを同時に実行します。Ninja はデフォルトで利用可能なCPUコア数に基づいて並列化を行います。
:::

## ビルドレベルの計算

並列実行可能な部分を見つけるには、各ターゲットのレベル（依存の深さ）を計算します。同じレベルのターゲットは並列にビルドできます[^critical-path]。

[^critical-path]: ビルドシステムの最適化では、クリティカルパス（最も時間のかかる依存の連鎖）を見つけることが重要です。クリティカルパス上のターゲットは並列化できないため、全体のビルド時間の下限を決定します。

```python
def compute_build_levels(config: dict, target: str) -> dict[str, int]:
    targets = config.get("targets", {})
    levels = {}

    def get_level(t: str) -> int:
        if t in levels:
            return levels[t]

        deps = targets[t].get("deps", [])
        if not deps:
            levels[t] = 0
        else:
            levels[t] = max(get_level(dep) for dep in deps) + 1

        return levels[t]

    # 指定されたターゲットまでの全ターゲットのレベルを計算
    order = resolve_build_order(config, target)
    for t in order:
        get_level(t)

    return levels
```

レベル0は依存がないターゲット、レベル1はレベル0に依存するターゲット...という形になります。

## 課題: レベルごとのグループ化

計算したレベルを使って、ターゲットをグループ化する関数を実装してください。

```python
def group_by_level(levels: dict[str, int]) -> list[list[str]]:
    # レベルごとにターゲットをグループ化する
    #
    # 例:
    # 入力: {"a": 0, "b": 0, "c": 1, "d": 2}
    # 出力: [["a", "b"], ["c"], ["d"]]
    #
    # レベル0のターゲットから順に、リストのリストとして返してください
    if not levels:
        return []

    # TODO: ここを実装してください
    pass
```

:::details 解答例
```python
def group_by_level(levels: dict[str, int]) -> list[list[str]]:
    if not levels:
        return []

    max_level = max(levels.values())
    groups = [[] for _ in range(max_level + 1)]

    for target, level in levels.items():
        groups[level].append(target)

    return groups
```
:::

## 並列ビルドの実装

Python の `concurrent.futures` を使って、同じレベルのターゲットを並列にビルドします[^gil]。

[^gil]: Python には GIL（Global Interpreter Lock）があるため、CPU バウンドな処理では `ThreadPoolExecutor` よりも `ProcessPoolExecutor` の方が効果的です。しかし、ビルドコマンドの実行は外部プロセスで行われるため、`ThreadPoolExecutor` でも十分な並列化が得られます。

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import os


def build_target_simple(config: dict, target: str) -> tuple[str, bool]:
    targets = config.get("targets", {})
    target_config = targets[target]
    command = target_config.get("command")

    if not command:
        return target, False

    print(f"[Thread] Building {target}...")

    result = subprocess.run(command, shell=True, capture_output=True)

    if result.returncode != 0:
        print(f"Error building {target}: {result.stderr.decode()}")
        return target, False

    print(f"[Thread] Completed {target}")
    return target, True


def parallel_build(config: dict, target: str, max_workers: int = None) -> bool:
    if max_workers is None:
        max_workers = os.cpu_count() or 4

    # レベルごとにグループ化
    levels = compute_build_levels(config, target)
    groups = group_by_level(levels)

    print(f"Build levels: {len(groups)}, Max parallelism: {max_workers}")

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        for level, targets_at_level in enumerate(groups):
            print(f"\n=== Level {level}: {targets_at_level} ===")

            # このレベルのターゲットを並列にビルド
            futures = {
                executor.submit(build_target_simple, config, t): t
                for t in targets_at_level
            }

            # すべてのタスクが完了するまで待機
            for future in as_completed(futures):
                target_name, success = future.result()
                if not success:
                    print(f"Build failed at level {level}")
                    return False

    print("\nBuild completed successfully!")
    return True
```

## 動かしてみる

並列ビルドの効果を確認するために、時間のかかるビルドをシミュレートします。`sleep` を使って各ターゲットに1秒かかるようにします。

```json
{
  "targets": {
    "a.txt": {"deps": [], "command": "sleep 1 && echo 'a' > a.txt"},
    "b.txt": {"deps": [], "command": "sleep 1 && echo 'b' > b.txt"},
    "c.txt": {"deps": [], "command": "sleep 1 && echo 'c' > c.txt"},
    "result.txt": {
      "deps": ["a.txt", "b.txt", "c.txt"],
      "command": "cat a.txt b.txt c.txt > result.txt"
    }
  }
}
```

```bash
# 逐次ビルド（約4秒）
$ time python minimake.py result.txt
Building a.txt...
Building b.txt...
Building c.txt...
Building result.txt...
real    0m4.05s

# 並列ビルド（約2秒）
$ time python minimake.py --parallel result.txt
Build levels: 2, Max parallelism: 4

=== Level 0: ['a.txt', 'b.txt', 'c.txt'] ===
[Thread] Building a.txt...
[Thread] Building b.txt...
[Thread] Building c.txt...
[Thread] Completed a.txt
[Thread] Completed b.txt
[Thread] Completed c.txt

=== Level 1: ['result.txt'] ===
[Thread] Building result.txt...
[Thread] Completed result.txt

Build completed successfully!
real    0m2.03s
```

`a.txt`、`b.txt`、`c.txt` が同時にビルドされるため、時間が短縮されます。4秒が2秒になりました。

## main 関数への統合

`--parallel` オプションを追加します。

```python
def main():
    targets = []
    build_file = "build.json"
    parallel = False
    max_workers = None

    i = 1
    while i < len(sys.argv):
        if sys.argv[i] == "--file" and i + 1 < len(sys.argv):
            build_file = sys.argv[i + 1]
            i += 2
        elif sys.argv[i] == "--parallel":
            parallel = True
            i += 1
        elif sys.argv[i] == "-j" and i + 1 < len(sys.argv):
            parallel = True
            max_workers = int(sys.argv[i + 1])
            i += 2
        else:
            targets.append(sys.argv[i])
            i += 1

    config = load_build_file(build_file)

    for target in targets:
        if parallel:
            if not parallel_build(config, target, max_workers):
                sys.exit(1)
        else:
            if not build_with_deps(config, target):
                sys.exit(1)
```

```bash
# デフォルトのワーカー数で並列ビルド
$ python minimake.py --parallel app

# ワーカー数を指定（Make と同じ構文）
$ python minimake.py -j 4 app
```

## スレッドセーフな出力

並列実行時に出力が混ざらないようにするには、ロックを使います[^thread-safe]。

[^thread-safe]: Python の `print` 関数は、内部的に `sys.stdout` にロックをかけているため、単一の `print` 呼び出しは途中で分断されることはありません。しかし、複数の `print` 呼び出しが混在する可能性があるため、ビルドの進行状況を分かりやすく表示するには明示的なロックが有効です。

```python
import threading

print_lock = threading.Lock()


def safe_print(message: str):
    with print_lock:
        print(message)


def build_target_simple(config: dict, target: str) -> tuple[str, bool]:
    targets = config.get("targets", {})
    target_config = targets[target]
    command = target_config.get("command")

    if not command:
        return target, False

    safe_print(f"[{threading.current_thread().name}] Building {target}...")

    result = subprocess.run(command, shell=True, capture_output=True)

    if result.returncode != 0:
        safe_print(f"Error building {target}: {result.stderr.decode()}")
        return target, False

    safe_print(f"[{threading.current_thread().name}] Completed {target}")
    return target, True
```

## 発展: 分散ビルド

並列ビルドをさらに発展させると、複数のマシンにビルドを分散させる分散ビルドになります[^distcc]。

[^distcc]: distcc は、C/C++ のコンパイルを複数のマシンに分散するツールです。プリプロセス（`#include` の展開）はローカルで行い、コンパイル本体だけをリモートに送信することで、ネットワークの帯域幅を節約しています。

実用的な分散ビルドシステム（distcc、Bazel Remote Execution など）では、以下のような機能が必要になります。

1. **ファイルの転送**: ソースファイルをワーカーに送信し、成果物を受け取る
2. **障害対応**: ワーカーがダウンした場合のリトライ
3. **負荷分散**: ワーカーの負荷に応じたタスクの割り当て

C/C++ のビルドであれば、distcc を使った分散コンパイルが手軽です。

```json
{
  "targets": {
    "hello.o": {
      "deps": [],
      "command": "distcc gcc -c -o hello.o hello.c"
    }
  }
}
```

distcc は、プリプロセス済みのソースコードをリモートに送信し、コンパイルだけを分散実行します。minimake の command を変えるだけで使えます。

:::message
Bazel や Buck などのビルドシステムは、リモート実行（Remote Execution）という機能を提供しています。これは、ビルドコマンドをクラウド上のワーカーで実行し、結果をキャッシュと組み合わせて効率的にビルドする仕組みです。Google Cloud Build や BuildBuddy などのサービスが Bazel のリモート実行をサポートしています。
:::

## アムダールの法則

並列化による高速化には理論的な限界があります。アムダールの法則によると、プログラムの並列化可能な部分の割合を $P$、プロセッサ数を $N$ とすると、高速化の上限 $U$ は次の式で表されます[^amdahl]。

[^amdahl]: アムダールの法則は1967年に Gene Amdahl によって提唱されました。この法則は、並列化の効果を考える上で重要な指針を与えます。たとえば、プログラムの50%が並列化できない場合、どれだけプロセッサを増やしても2倍以上の高速化は得られません。

$$
U = \frac{1}{1-P + \frac{P}{N}}
$$

ビルドシステムでは、依存関係による制約が並列化できない部分に相当します。クリティカルパス上のビルドは並列化できないため、全体の高速化に限界があります。

つまり、どれだけCPUコアを増やしても、依存関係が直列になっている部分の時間は短縮できません。並列ビルドを効果的に活用するには、プロジェクトの構造自体を見直すことも重要です。

## この章のまとめ

- 依存関係のないターゲットは並列にビルド可能
- ビルドレベルを計算してグループ化
- `concurrent.futures.ThreadPoolExecutor` でローカル並列化
- `-j` オプションでワーカー数を制御

並列ビルドは、マルチコアCPUの性能を活かしてビルド時間を短縮できます。特に、独立したターゲットが多いプロジェクトで効果を発揮します。

## 模範解答

この章の模範解答を確認したい場合は、以下のコマンドを実行してください。

```bash
$ git switch --detach chapter9-end
```
