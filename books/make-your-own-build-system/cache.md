---
title: "ビルドキャッシュ"
---

## 準備

この章の実装を始める前に、以下のコマンドで初期状態からブランチを作成してください。

```bash
$ git switch -c my-chapter8 chapter8-begin
```

---

:::message
この章は発展課題です。第5章までの内容を理解していれば取り組むことができます。
:::

第5章で実装したインクリメンタルビルドは、ファイルの更新日時（mtime）を使って再ビルドの要否を判定していました。しかし、この方法にはいくつかの問題があります。

- ブランチを切り替えると、すべてのファイルの mtime が更新される
- `git clone` した直後は、すべてのファイルが新しいとみなされる
- CI 環境では毎回クリーンな状態から始まるため、キャッシュが効かない

この章では、ファイルの内容のハッシュを使ったビルドキャッシュを実装します。入力が同じなら、以前のビルド結果を再利用できるようになります。

## コンテンツアドレッサブルストレージ

ビルドキャッシュの基本的なアイデアは、入力の内容をハッシュ化し、そのハッシュをキーとして成果物を保存することです[^cas]。

[^cas]: コンテンツアドレッサブルストレージ（CAS）は、データの内容そのものをアドレス（識別子）として使うストレージシステムです。Git の内部構造もこの方式を採用しており、各オブジェクト（blob、tree、commit）はその内容の SHA-1 ハッシュで識別されます。

```
入力ファイルのハッシュ → キャッシュキー → 成果物
```

同じ入力からは同じハッシュが生成されるため、入力が同じなら以前の成果物を再利用できます。これをコンテンツアドレッサブルストレージ（CAS）と呼びます。

:::message
Bazel や Nix などのビルドシステムは、この考え方をさらに発展させています。入力だけでなく、ビルドに使用するツールのバージョンや環境変数もハッシュに含めることで、より厳密な再現性を実現しています。
:::

## 課題: キャッシュキーの計算

ビルドの再現性を保証するためには、ビルド結果に影響を与えるすべての要素をキャッシュキーに含める必要があります。

- 入力ファイルの内容
- ビルドコマンド
- 依存ターゲットのキャッシュキー

これらをすべてハッシュに含めることで、同じキャッシュキーなら同じビルド結果という保証ができます。

```python
import hashlib
from pathlib import Path


def compute_file_hash(path: str) -> str:
    sha256 = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            sha256.update(chunk)
    return sha256.hexdigest()


def compute_cache_key(config: dict, target: str, dep_keys: dict[str, str]) -> str:
    # ターゲットのキャッシュキーを計算する
    #
    # 以下の要素をハッシュに含めてください:
    # 1. ビルドコマンド
    # 2. 各入力ファイルの内容（ファイル名とハッシュ）
    # 3. 依存ターゲットのキャッシュキー
    #
    # ヒント: hasher.update(string.encode()) でハッシュに追加できます
    targets = config.get("targets", {})
    target_config = targets[target]

    hasher = hashlib.sha256()

    # TODO: ここを実装してください
    pass
```

:::details 解答例
```python
def compute_cache_key(config: dict, target: str, dep_keys: dict[str, str]) -> str:
    targets = config.get("targets", {})
    target_config = targets[target]

    hasher = hashlib.sha256()

    # 1. ビルドコマンドをハッシュに含める
    command = target_config.get("command", "")
    hasher.update(command.encode())

    # 2. 入力ファイルの内容をハッシュに含める
    inputs = target_config.get("inputs", [])
    for input_file in sorted(inputs):  # 順序を固定
        if Path(input_file).exists():
            file_hash = compute_file_hash(input_file)
            hasher.update(f"{input_file}:{file_hash}".encode())

    # 3. 依存ターゲットのキャッシュキーをハッシュに含める
    deps = target_config.get("deps", [])
    for dep in sorted(deps):
        if dep in dep_keys:
            hasher.update(f"dep:{dep}:{dep_keys[dep]}".encode())

    return hasher.hexdigest()
```
:::

:::message
`sorted()` を使ってファイル名をソートしている点に注目してください。ファイルをハッシュに追加する順序が変わると、同じ入力でも異なるハッシュが生成されてしまいます。決定論的な結果を得るために、順序を固定することが重要です。
:::

## キャッシュディレクトリの構造

キャッシュは `.minimake-cache` ディレクトリに保存します[^cache-dir]。

[^cache-dir]: キャッシュディレクトリの場所は、プロジェクトごと（`.minimake-cache`）、ユーザーごと（`~/.cache/minimake`）、システム全体（`/var/cache/minimake`）など、用途に応じて選択できます。Bazel はユーザーのホームディレクトリにキャッシュを保存し、複数のプロジェクトで共有します。

```
.minimake-cache/
├── abc123.../           # キャッシュキー
│   ├── hello.o          # 成果物
│   └── metadata.json    # メタデータ
├── def456.../
│   └── ...
```

キャッシュキーをディレクトリ名にして、その中に成果物とメタデータを保存するシンプルな構造です。

## キャッシュの保存と復元

```python
import shutil
import json
from datetime import datetime

CACHE_DIR = Path(".minimake-cache")


def get_cache_path(cache_key: str) -> Path:
    return CACHE_DIR / cache_key


def save_to_cache(cache_key: str, target: str):
    cache_path = get_cache_path(cache_key)
    cache_path.mkdir(parents=True, exist_ok=True)

    # 成果物をコピー
    target_path = Path(target)
    if target_path.exists():
        shutil.copy2(target_path, cache_path / target_path.name)

    # メタデータを保存
    metadata = {
        "target": target,
        "created_at": datetime.now().isoformat()
    }
    with open(cache_path / "metadata.json", "w") as f:
        json.dump(metadata, f)


def restore_from_cache(cache_key: str, target: str) -> bool:
    cache_path = get_cache_path(cache_key)

    if not cache_path.exists():
        return False

    cached_file = cache_path / Path(target).name
    if not cached_file.exists():
        return False

    shutil.copy2(cached_file, target)
    return True
```

## キャッシュを使ったビルド

ビルド処理を更新して、キャッシュを活用するようにします。

```python
def build_with_cache(config: dict, target: str, dep_keys: dict[str, str]) -> tuple[bool, str]:
    # キャッシュを使ってビルドする
    # 成功したら (True, cache_key) を返す
    targets = config.get("targets", {})
    target_config = targets[target]

    # キャッシュキーを計算
    cache_key = compute_cache_key(config, target, dep_keys)

    # キャッシュヒットをチェック
    if restore_from_cache(cache_key, target):
        print(f"Cache hit: {target} ({cache_key[:8]}...)")
        return True, cache_key

    # キャッシュミス - ビルドを実行
    command = target_config.get("command")
    if not command:
        print(f"Error: No command for target '{target}'", file=sys.stderr)
        return False, ""

    print(f"Building {target}... ({cache_key[:8]}...)")
    print(f"  $ {command}")

    result = subprocess.run(command, shell=True)

    if result.returncode != 0:
        print(f"Error: Build failed for '{target}'", file=sys.stderr)
        return False, ""

    # 成果物をキャッシュに保存
    save_to_cache(cache_key, target)

    return True, cache_key


def build_all_with_cache(config: dict, target: str) -> bool:
    try:
        order = resolve_build_order(config, target)
    except ValueError as e:
        print(f"Error: {e}", file=sys.stderr)
        return False

    print(f"Build order: {' -> '.join(order)}")

    dep_keys = {}  # 各ターゲットのキャッシュキーを記録

    for t in order:
        success, cache_key = build_with_cache(config, t, dep_keys)
        if not success:
            return False
        dep_keys[t] = cache_key

    return True
```

## キャッシュの管理

キャッシュが肥大化しないように、管理コマンドを追加しておくと便利です。

```python
def cache_stats():
    if not CACHE_DIR.exists():
        print("No cache found")
        return

    total_size = 0
    entry_count = 0

    for entry in CACHE_DIR.iterdir():
        if entry.is_dir():
            entry_count += 1
            for file in entry.iterdir():
                total_size += file.stat().st_size

    print(f"Cache entries: {entry_count}")
    print(f"Total size: {total_size / 1024 / 1024:.2f} MB")


def cache_clean():
    if CACHE_DIR.exists():
        shutil.rmtree(CACHE_DIR)
        print("Cache cleaned")
```

## 発展: リモートキャッシュ

チームで共有できるリモートキャッシュを実装することもできます。基本的な考え方は以下の通りです。

1. ローカルキャッシュを確認
2. ローカルになければリモートキャッシュを確認
3. どちらにもなければビルドを実行
4. ビルド後、ローカルとリモートの両方に保存

実用的なリモートキャッシュの実装には、HTTP サーバーや S3 などのストレージサービスを使用します[^remote-cache]。

[^remote-cache]: Bazel は独自のリモートキャッシュプロトコル（Remote Execution API）を定義しています。このプロトコルは gRPC ベースで、Google Cloud Build や BuildBuddy などのサービスが実装しています。より単純な HTTP ベースのキャッシュも広く使われています。

```python
import urllib.request
import urllib.error

REMOTE_CACHE_URL = "https://cache.example.com"


def fetch_from_remote_cache(cache_key: str, target: str) -> bool:
    url = f"{REMOTE_CACHE_URL}/{cache_key}/{Path(target).name}"

    try:
        with urllib.request.urlopen(url) as response:
            with open(target, "wb") as f:
                f.write(response.read())
        return True
    except urllib.error.HTTPError:
        return False
```

リモートキャッシュがあると、CIでビルドされた成果物をローカル開発でも利用できます。これはチーム開発で非常に有効です。

## Bazel のキャッシュ戦略

Bazel は、ビルドキャッシュの実装において最も洗練されたビルドシステムの1つです[^bazel-cache]。

[^bazel-cache]: Bazel のキャッシュシステムは、Google 社内の Blaze で培われた技術を基にしています。数万人の開発者が数十億行のコードを共有するモノレポで効率的にビルドできるよう設計されています。

Bazel のキャッシュには主に3つの層があります。

- **アクションキャッシュ**: 各ビルドアクション（コンパイル、リンクなど）の入出力をキャッシュ
- **リモート実行**: ビルドを分散して実行し、結果をキャッシュで共有
- **コンテンツアドレッサブルストレージ**: 同じ内容のファイルは1度だけ保存

:::message
Bazel のリモートキャッシュは、CI でビルドされた成果物をローカル開発でも利用できるため、開発者のビルド時間を大幅に短縮できます。たとえば、main ブランチでビルドされた成果物は、ローカルで同じコードをビルドするときにキャッシュヒットします。
:::

## この章のまとめ

- ファイルの内容のハッシュをキャッシュキーとして使用
- 同じ入力からは同じ出力を再利用
- ローカルキャッシュとリモートキャッシュで高速化
- CI環境でも効果的にキャッシュを活用

ビルドキャッシュは、大規模プロジェクトでのビルド時間を劇的に短縮できる機能です。特に、CI環境やチーム開発では、リモートキャッシュによって全員がビルド結果を共有できるため、効果が大きくなります。

## 模範解答

この章の模範解答を確認したい場合は、以下のコマンドを実行してください。

```bash
$ git switch --detach chapter8-end
```
