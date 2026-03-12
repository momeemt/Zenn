---
title: "ロックファイル"
---

## 準備

この章の実装を始める前に、以下のコマンドで初期状態に移動してください。

```bash
$ git checkout chapter7-begin
```

---

:::message
この章は発展課題です。第5章までの内容を理解していれば、独立して取り組むことができます。
:::

チーム開発やCI環境でビルドを行っていると、「自分のマシンでは動くのに、他の環境では動かない」という問題に遭遇することがあります。

原因の多くは、依存するライブラリやツールのバージョンが環境によって異なることです。この章では、依存関係のバージョンを固定するためのロックファイルを実装します。

## ロックファイルとは

ロックファイルは、ビルドに使用した依存関係の正確な情報を記録したファイルです[^lockfile-history]。

[^lockfile-history]: ロックファイルの概念は、Ruby の Bundler（2010年）で広まりました。`Gemfile.lock` という名前で、gem のバージョンを固定します。その後、npm、Cargo、Poetry など多くのパッケージマネージャが同様の仕組みを採用しています。

主に以下の情報を記録します。

- バージョン: どのバージョンを使ったか
- 取得元: どこからダウンロードしたか
- チェックサム: ファイルの内容が正しいか検証するためのハッシュ値

代表的なロックファイルには以下のようなものがあります。

| ツール | ロックファイル |
| --- | --- |
| npm | `package-lock.json` |
| Cargo | `Cargo.lock` |
| Poetry | `poetry.lock` |
| Go | `go.sum` |

:::message
ロックファイルはバージョン管理システム（Git など）にコミットすべきです。これにより、チームの全員が同じバージョンの依存関係を使うことが保証されます。ライブラリの場合は議論がありますが[^lib-lockfile]、アプリケーションの場合はロックファイルをコミットするのが一般的です。
:::

[^lib-lockfile]: ライブラリの場合、ロックファイルをコミットするかどうかは意見が分かれます。ライブラリの利用者は自分の環境でインストールするため、ライブラリ側のロックファイルは意味がないという考え方があります。一方で、ライブラリの開発・テスト環境を再現するために必要という意見もあります。

## minimake にロックファイルを追加する

minimake では、外部の依存関係として「ツールのバージョン」を管理することを考えます。たとえば、GCC のバージョンが異なると、生成されるバイナリが変わる可能性があります[^abi]。

[^abi]: コンパイラのバージョンが変わると、ABI（Application Binary Interface）が変わることがあります。特に C++ では、名前マングリングや例外処理の実装がコンパイラによって異なるため、異なるバージョンでコンパイルされたライブラリをリンクすると問題が発生することがあります。

### ビルド定義の拡張

`build.json` に `tools` セクションを追加します。

```json
{
  "tools": {
    "gcc": {
      "version": ">=11.0.0"
    }
  },
  "targets": {
    "hello.o": {
      "deps": [],
      "inputs": ["hello.c"],
      "command": "gcc -c -o hello.o hello.c"
    }
  }
}
```

`">=11.0.0"` のような形式で、必要なバージョンの制約を指定できるようにします。

### ロックファイルの形式

`build.lock` として、実際に使用したバージョンを記録します。

```json
{
  "tools": {
    "gcc": {
      "version": "11.4.0",
      "path": "/usr/bin/gcc",
      "hash": "sha256:abc123..."
    }
  },
  "generated_at": "2026-01-15T10:30:00Z"
}
```

## 課題: ツールのバージョン検出

システムにインストールされているツールのバージョンを検出する関数を実装してください。

```python
import subprocess
import re


def get_tool_version(tool: str) -> str | None:
    # ツールのバージョンを取得する
    #
    # 以下のツールに対応してください:
    # - gcc: `gcc --version` の出力から抽出
    # - python: `python3 --version` の出力から抽出
    #
    # ヒント: 正規表現 r'(\d+\.\d+\.\d+)' でバージョン番号を抽出できます

    # TODO: ここを実装してください
    pass


def get_tool_path(tool: str) -> str | None:
    result = subprocess.run(
        ["which", tool],
        capture_output=True,
        text=True
    )
    if result.returncode == 0:
        return result.stdout.strip()
    return None
```

:::details 解答例
```python
def get_tool_version(tool: str) -> str | None:
    try:
        if tool == "gcc":
            result = subprocess.run(
                ["gcc", "--version"],
                capture_output=True,
                text=True
            )
            match = re.search(r'(\d+\.\d+\.\d+)', result.stdout)
            if match:
                return match.group(1)

        elif tool == "python":
            result = subprocess.run(
                ["python3", "--version"],
                capture_output=True,
                text=True
            )
            match = re.search(r'(\d+\.\d+\.\d+)', result.stdout)
            if match:
                return match.group(1)

    except FileNotFoundError:
        return None

    return None
```
:::

## 課題: バージョン制約のチェック

`>=11.0.0` のようなバージョン制約をチェックする関数を実装してください。

```python
def check_version_constraint(actual: str, constraint: str) -> bool:
    # バージョン制約を満たしているかチェックする
    #
    # 対応する制約:
    # - ">=X.Y.Z": actual が X.Y.Z 以上
    # - "==X.Y.Z": actual が X.Y.Z と一致
    #
    # ヒント: バージョン文字列を tuple(map(int, v.split('.'))) で
    # タプルに変換すると比較しやすくなります

    # TODO: ここを実装してください
    pass
```

:::details 解答例
```python
def parse_version(v: str) -> tuple:
    return tuple(map(int, v.split('.')))


def check_version_constraint(actual: str, constraint: str) -> bool:
    actual_tuple = parse_version(actual)

    if constraint.startswith(">="):
        required = parse_version(constraint[2:])
        return actual_tuple >= required
    elif constraint.startswith("<="):
        required = parse_version(constraint[2:])
        return actual_tuple <= required
    elif constraint.startswith("=="):
        required = parse_version(constraint[2:])
        return actual_tuple == required
    else:
        return actual == constraint
```
:::

:::message
実際のバージョン比較は、セマンティックバージョニング（SemVer）の仕様に従うとより正確です[^semver]。プレリリースバージョン（`1.0.0-alpha`）やビルドメタデータ（`1.0.0+build123`）の扱いも考慮する必要があります。Python では `packaging` ライブラリがこれを実装しています。
:::

[^semver]: セマンティックバージョニング（https://semver.org/ ）は、バージョン番号を `MAJOR.MINOR.PATCH` の形式で表し、それぞれの数字が持つ意味を定義しています。MAJOR は後方互換性のない変更、MINOR は後方互換性のある機能追加、PATCH はバグ修正を表します。

## ロックファイルの生成

ビルド定義から現在の環境の情報を収集し、ロックファイルを生成します。

```python
import json
import hashlib
from datetime import datetime
from pathlib import Path


def compute_file_hash(path: str) -> str:
    sha256 = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            sha256.update(chunk)
    return f"sha256:{sha256.hexdigest()}"


def generate_lockfile(config: dict) -> dict:
    tools_config = config.get("tools", {})
    locked_tools = {}

    for tool, spec in tools_config.items():
        version = get_tool_version(tool)
        path = get_tool_path(tool)

        if version is None:
            raise ValueError(f"Tool not found: {tool}")

        constraint = spec.get("version", "")
        if constraint and not check_version_constraint(version, constraint):
            raise ValueError(
                f"Tool {tool} version {version} does not satisfy {constraint}"
            )

        locked_tools[tool] = {
            "version": version,
            "path": path,
            "hash": compute_file_hash(path) if path else None
        }

    return {
        "tools": locked_tools,
        "generated_at": datetime.now().isoformat()
    }


def save_lockfile(lockfile: dict, path: str = "build.lock"):
    with open(path, "w") as f:
        json.dump(lockfile, f, indent=2)
```

## ロックファイルの検証

ビルド前に、ロックファイルと現在の環境が一致しているかを検証します。この検証があるからこそ「環境の違い」を早期に検出できます。

```python
def verify_lockfile(lockfile: dict) -> list[str]:
    errors = []
    locked_tools = lockfile.get("tools", {})

    for tool, locked in locked_tools.items():
        current_version = get_tool_version(tool)
        current_path = get_tool_path(tool)

        if current_version is None:
            errors.append(f"{tool}: not installed")
            continue

        if current_version != locked["version"]:
            errors.append(
                f"{tool}: version mismatch "
                f"(locked: {locked['version']}, current: {current_version})"
            )

        if current_path and locked.get("hash"):
            current_hash = compute_file_hash(current_path)
            if current_hash != locked["hash"]:
                errors.append(f"{tool}: binary hash mismatch")

    return errors
```

## main.py への統合

サブコマンドとして `lock` と `verify` を追加します。

```python
def main():
    if len(sys.argv) < 2:
        print("Usage: minimake <command> [args...]", file=sys.stderr)
        print("Commands: build, lock, verify", file=sys.stderr)
        sys.exit(1)

    command = sys.argv[1]

    if command == "lock":
        config = load_build_file("build.json")
        lockfile = generate_lockfile(config)
        save_lockfile(lockfile)
        print("Generated build.lock")

    elif command == "verify":
        if not Path("build.lock").exists():
            print("Error: build.lock not found", file=sys.stderr)
            sys.exit(1)

        with open("build.lock") as f:
            lockfile = json.load(f)

        errors = verify_lockfile(lockfile)
        if errors:
            print("Verification failed:", file=sys.stderr)
            for error in errors:
                print(f"  - {error}", file=sys.stderr)
            sys.exit(1)
        print("Verification passed")

    elif command == "build":
        # 既存のビルドロジック
        pass
```

## 発展: 依存関係のダウンロード

実際のパッケージマネージャでは、ロックファイルに記録された情報を使って依存関係を自動的にダウンロードします[^registry]。

[^registry]: npm は npmjs.com、Cargo は crates.io、pip は PyPI（Python Package Index）からパッケージをダウンロードします。これらはパッケージレジストリと呼ばれ、パッケージのメタデータとファイルを保存しています。

```python
def download_dependency(name: str, version: str, checksum: str) -> Path:
    # ダウンロード先のパス
    cache_dir = Path(".minimake-deps")
    cache_dir.mkdir(exist_ok=True)

    dest = cache_dir / f"{name}-{version}"

    if dest.exists():
        # チェックサムを検証
        if compute_file_hash(str(dest)) == checksum:
            return dest
        dest.unlink()  # ハッシュが一致しない場合は再ダウンロード

    # ダウンロード（実際にはHTTPリクエストを送る）
    url = f"https://registry.example.com/{name}/{version}"
    # ... ダウンロード処理 ...

    # チェックサムを検証
    if compute_file_hash(str(dest)) != checksum:
        raise ValueError(f"Checksum mismatch for {name}")

    return dest
```

チェックサムの検証をすることで、ダウンロード中にファイルが改ざんされていないことを確認できます。これはセキュリティ上重要なポイントです。

## この章のまとめ

- ロックファイルで依存関係のバージョンを固定
- ツールのバージョン、パス、ハッシュを記録
- ビルド前に環境の一貫性を検証
- CI環境で再現性のあるビルドを実現

ロックファイルは「再現可能なビルド」の第一歩です。同じロックファイルを使えば、異なる環境でも同じ条件でビルドできることが保証されます。次の章で扱う「ビルドキャッシュ」や「再現可能なビルド」とも密接に関係している機能です。

## 模範解答

この章の模範解答を確認したい場合は、以下のコマンドを実行してください。

```bash
$ git checkout chapter7-end
```
