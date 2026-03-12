---
title: "自動依存解決"
---

## 準備

この章の実装を始める前に、以下のコマンドで初期状態に移動してください。

```bash
$ git checkout chapter6-begin
```

---

:::message
この章は発展課題です。第5章までの内容を理解していれば取り組むことができます。
:::

これまでの実装では、依存関係を `build.json` に手動で書いていました。

```json
{
  "targets": {
    "main.o": {
      "deps": [],
      "inputs": ["main.c", "greet.h", "config.h"],
      "command": "gcc -c -o main.o main.c"
    }
  }
}
```

`main.c` が `#include "greet.h"` しているなら、`inputs` に `greet.h` を書く必要があります。ヘッダファイルを追加するたびに `build.json` を更新するのは面倒ですし、書き忘れるとインクリメンタルビルドが正しく動作しません[^forget-deps]。

[^forget-deps]: 依存関係の書き忘れは、ビルドシステムのバグの中でも特に厄介なものです。「なぜか変更が反映されない」「クリーンビルドすると直る」という症状になり、原因の特定に時間がかかります。

この章では、ソースコードを解析して依存関係を自動的に検出する機能を実装します。

## 依存関係の検出方法

C/C++ のソースコードの場合、`#include` ディレクティブを解析することで依存するヘッダファイルを特定できます[^preprocessor]。

[^preprocessor]: `#include` はCプリプロセッサのディレクティブで、指定されたファイルの内容をその場所に挿入します。コンパイル前に処理されるため、依存関係の解析にはソースコードのテキスト解析が必要になります。

```c
#include <stdio.h>      // システムヘッダ
#include "greet.h"      // ユーザーヘッダ
```

一般的には、`<>` で囲まれたシステムヘッダは無視し、`""` で囲まれたユーザーヘッダのみを依存関係として扱います[^include-search]。システムヘッダはコンパイラツールチェーン（GCC、Clang など）が提供するもので、プロジェクトのビルド中に変更されることがないため、追跡する必要がありません。

[^include-search]: `#include <...>` と `#include "..."` の違いは、ヘッダファイルを検索するディレクトリの順序です。`<>` はコンパイラのインクルードパス（`/usr/include` や `/usr/lib/gcc/.../include` など）のみを検索し、`""` はまずカレントディレクトリを検索してからコンパイラのパスを検索します。

## 課題: #include の解析

正規表現を使って `#include` ディレクティブを解析する関数を実装してください。

```python
import re
from pathlib import Path


def parse_includes(file_path: str) -> list[str]:
    # ソースファイルから #include で参照されているヘッダファイルを抽出する
    #
    # #include "..." の形式のみを抽出してください
    # #include <...> の形式（システムヘッダ）は無視してください
    #
    # ヒント: re.findall を使うと簡単に実装できます
    content = Path(file_path).read_text()

    # TODO: ここを実装してください
    pass
```

:::details 解答例
```python
def parse_includes(file_path: str) -> list[str]:
    content = Path(file_path).read_text()

    # #include "..." の形式を抽出（システムヘッダ <...> は除外）
    pattern = r'#include\s+"([^"]+)"'
    matches = re.findall(pattern, content)

    return matches
```
:::

:::message
この正規表現はシンプルですが、コメント内の `#include` や、複数行にまたがるマクロ内の `#include` を誤検出する可能性があります。実用的なビルドシステムでは、より堅牢なパーサーを使うか、コンパイラの機能（`gcc -M`）を利用します。
:::

## 課題: 再帰的な依存解決

ヘッダファイルも他のヘッダファイルを `#include` している可能性があります。たとえば、`greet.h` が `config.h` を include していたら、`main.c` は間接的に `config.h` にも依存しています。

すべての依存関係を収集するには、再帰的に解析する必要があります。

```python
def collect_all_includes(file_path: str, base_dir: str = ".") -> set[str]:
    # ソースファイルから再帰的にすべての依存ヘッダを収集する
    #
    # 1. file_path から #include を解析
    # 2. 見つかったヘッダファイルに対して再帰的に同じ処理を行う
    # 3. 循環参照を避けるため、訪問済みのファイルは記録しておく
    collected = set()
    visited = set()

    def visit(path: str):
        # TODO: ここを実装してください
        pass

    visit(file_path)
    return collected
```

:::details 解答例
```python
def collect_all_includes(file_path: str, base_dir: str = ".") -> set[str]:
    collected = set()
    visited = set()

    def visit(path: str):
        if path in visited:
            return
        visited.add(path)

        full_path = Path(base_dir) / path
        if not full_path.exists():
            return

        includes = parse_includes(str(full_path))
        for inc in includes:
            inc_path = Path(base_dir) / inc
            if inc_path.exists():
                collected.add(inc)
                visit(inc)

    visit(file_path)
    return collected
```
:::

## ビルド定義の自動生成

収集した依存関係を使って、`inputs` を自動的に設定する機能を追加します。

```python
def auto_resolve_inputs(config: dict, base_dir: str = ".") -> dict:
    targets = config.get("targets", {})

    for target_name, target_config in targets.items():
        # 明示的に inputs が指定されていればスキップ
        if "inputs" in target_config:
            continue

        # command から入力ファイルを推測
        command = target_config.get("command", "")

        # gcc -c -o output.o input.c のようなパターンを想定
        # 簡易的に .c ファイルを抽出
        c_files = re.findall(r'\b(\w+\.c)\b', command)

        all_inputs = set(c_files)

        # 各 .c ファイルから #include を解析
        for c_file in c_files:
            includes = collect_all_includes(c_file, base_dir)
            all_inputs.update(includes)

        target_config["inputs"] = list(all_inputs)

    return config
```

## 使用例

以下のようなプロジェクト構造を考えます。

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

#include "config.h"

void greet(const char *name);

#endif
```

**config.h**
```c
#ifndef CONFIG_H
#define CONFIG_H

#define VERSION "1.0.0"

#endif
```

`#ifndef GREET_H` のようなインクルードガードは、ヘッダファイルの多重インクルードを防ぐための慣習的な手法です[^include-guard]。これにより、同じヘッダが複数回 `#include` されても、内容は1度しか処理されません。

[^include-guard]: インクルードガードの代わりに `#pragma once` を使う方法もありますが、これは標準C/C++ではなく、コンパイラ拡張です。ただし、主要なコンパイラ（GCC、Clang、MSVC）はすべてサポートしているので、実用上は問題ありません。

**build.json**（inputs を省略）
```json
{
  "targets": {
    "main.o": {
      "deps": [],
      "command": "gcc -c -o main.o main.c"
    }
  }
}
```

自動依存解決を実行すると、`main.o` の `inputs` が自動的に `["main.c", "greet.h", "config.h"]` に設定されます。

## main.py への統合

起動時に自動依存解決を実行するようにします。

```python
def main():
    # ... 引数のパース ...

    config = load_build_file(build_file)

    # 自動依存解決
    config = auto_resolve_inputs(config)

    for target in targets:
        if not build_with_deps(config, target):
            sys.exit(1)
```

## gcc -M を使った方法

GCC には依存関係を出力する `-M` オプションがあります[^gcc-m]。自分で正規表現を書かなくても、コンパイラが正確な依存関係を出力します。

[^gcc-m]: `-M` オプションは、Make 形式の依存関係ルールを出力します。`-MM` を使うとシステムヘッダを除外できます。`-MF` でファイルに出力、`-MT` でターゲット名を指定することもできます。

```bash
$ gcc -M main.c
main.o: main.c greet.h config.h
```

これを使えば、より正確な依存関係を取得できます。

```python
def get_gcc_deps(source_file: str) -> list[str]:
    result = subprocess.run(
        ["gcc", "-M", source_file],
        capture_output=True,
        text=True
    )

    if result.returncode != 0:
        return []

    # 出力をパース: "target: dep1 dep2 dep3 ..."
    output = result.stdout.replace("\\\n", " ")  # 行継続を処理
    if ":" not in output:
        return []

    deps_part = output.split(":", 1)[1]
    deps = deps_part.split()

    return deps
```

### 自前解析 vs gcc -M

それぞれメリット・デメリットがあります。

| 方法 | メリット | デメリット |
| --- | --- | --- |
| 自前解析 | GCC 不要、言語に依存しない | 不完全な場合がある |
| gcc -M | 正確、コンパイラと同じ結果 | GCC が必要、C/C++ 限定 |

:::message
実際の Make では、`-M` オプションで生成された依存関係ファイル（`.d` ファイル）を自動的にインクルードする仕組みがよく使われます。これにより、ヘッダファイルの変更を正確に追跡できます。
:::

## 発展: 他の言語への対応

同様のアプローチで、他の言語の依存関係も解析できます。C/C++ の `gcc -M` と同様に、各言語にも専用のツールがあります。

### Python

Python には標準ライブラリの `ast` モジュールを使った解析方法があります。

```python
import ast
from pathlib import Path


def get_python_imports(file_path: str) -> list[str]:
    content = Path(file_path).read_text()
    tree = ast.parse(content)

    imports = []
    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            for alias in node.names:
                imports.append(alias.name)
        elif isinstance(node, ast.ImportFrom):
            if node.module:
                imports.append(node.module)

    return imports
```

正規表現よりも `ast` モジュールを使う方が正確です。コメント内の import 文を誤検出することがなく、構文的に正しい import のみを抽出できます。

また、外部ツールとして `pipdeptree` や `pydeps` があります。

```bash
# パッケージの依存関係ツリーを表示
$ pipdeptree

# モジュールの依存関係グラフを生成
$ pydeps mymodule.py --show-deps
```

### JavaScript/TypeScript

JavaScript/TypeScript では、`madge` や `dependency-cruiser` といったツールが依存関係の解析に使われます。

```bash
# madge: 依存関係グラフを生成
$ npx madge --json src/index.ts

# dependency-cruiser: 詳細な依存関係レポート
$ npx depcruise --output-type json src/
```

自前で解析する場合は、TypeScript コンパイラの API を使う方法もあります。

```typescript
import * as ts from "typescript";

function getImports(fileName: string): string[] {
  const sourceFile = ts.createSourceFile(
    fileName,
    fs.readFileSync(fileName, "utf8"),
    ts.ScriptTarget.Latest
  );

  const imports: string[] = [];
  ts.forEachChild(sourceFile, (node) => {
    if (ts.isImportDeclaration(node)) {
      const specifier = node.moduleSpecifier;
      if (ts.isStringLiteral(specifier)) {
        imports.push(specifier.text);
      }
    }
  });

  return imports;
}
```

### 各言語のツール比較

| 言語 | 自前解析 | 専用ツール |
| --- | --- | --- |
| C/C++ | `#include` の正規表現 | `gcc -M`, `clang -M` |
| Python | `ast` モジュール | `pipdeptree`, `pydeps` |
| JavaScript/TypeScript | 正規表現 | `madge`, `dependency-cruiser` |

専用ツールを使う方が正確で、エッジケース（動的 import、条件付き import など）にも対応できます。

## この章のまとめ

- `#include` ディレクティブを解析して依存関係を自動検出
- 再帰的に解析することで、間接的な依存関係も収集
- `gcc -M` を使えばより正確な依存関係を取得可能
- Python では `ast` モジュール、JS/TS では TypeScript コンパイラ API が利用可能
- 各言語に専用の依存関係解析ツール（`pipdeptree`、`madge` など）がある

手動で依存関係を管理する手間が省け、書き忘れによるインクリメンタルビルドの不具合も防げます。地味ですが、実用的なビルドシステムには欠かせない機能です。

## 模範解答

この章の模範解答を確認したい場合は、以下のコマンドを実行してください。

```bash
$ git checkout chapter6-end
```
