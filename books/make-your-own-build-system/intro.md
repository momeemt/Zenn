---
title: "はじめに"
---

:::message
この書籍は、[セキュリティ・キャンプミニ2026（大阪開催）](https://www.security-camp.or.jp/minicamp/osaka2026.html)の専門講座で行われる『ビルドシステムを自作しよう』という講座のために執筆されました。
:::

こんにちは。この講座の講師を担当します、[浅田睦葉](https://github.com/momeemt)です。
[セキュリティ・キャンプミニ2026（大阪開催）](https://www.security-camp.or.jp/minicamp/osaka2026.html)への参加が決定した皆さま、よろしくお願いします。

## 講義の進め方

この講義は、次のような形式で進めます。

| 時間の目安 | 講義内容 |
| --- | --- |
| 10:30〜11:00 | [2. ビルドシステムとは](./build-system) の解説|
| 11:00〜11:10 | [3. シンプルなビルドの実行](./simple-build) の解説 |
| 11:10〜11:45 | [3. シンプルなビルドの実行](./simple-build) の実装 |
| 11:45〜11:55 | [4. 依存関係を解決する](./resolve-deps) の解説 |
| 11:55〜12:35 | [4. 依存関係を解決する](./resolve-deps) の実装 |
| 12:35〜12:50 | [5. インクリメンタルビルド](./incremental-build) または発展課題 |
| 12:50〜13:00 | 発表 |

事前に資料を読む時間がある方は、この章と『2. ビルドシステムとは』を読んでおくと、当日は第3章の実装から入ることができます。30分ほどあれば読み進められる内容です。
時間が取れない方も、当日に同様の説明をしますので問題ありません。

また、この講義で使うソースコードは、各章における実装が完了した時点でGitのタグを付けています。
基本的なビルドシステムの実装の説明について問題がない場合は、解説を聞きながらより高度なチャプターに挑戦しても構いません。
第5章以降の実装はそれぞれ独立に進めることができます。

- [6. 自動依存解決](./auto-resolve-deps)
- [7. ロックファイル](./lockfiles)
- [8. ビルドキャッシュ](./cache)
- [9. 並列ビルド](./distributed-build)
- [10. 再現可能なビルド](./reproducible-build)

## 質問

当日まであまり時間がありませんが、資料を読んで質問があれば、kintoneの[『ビルドシステムを自作しよう』](https://security-camp.cybozu.com/k/guest/496/#/space/496/thread/4383)という名前の掲示板に投稿してください。講師やチューターが回答します。

## 環境構築

この講義では、以下のツールを使用します。事前にインストールしておいてください。

### 必須

- [Git](https://git-scm.com/)
- [GitHub](https://github.com/) アカウント
- [Python 3.10以上](https://www.python.org/downloads/)
- [GCC](https://gcc.gnu.org/)（Cコンパイラ）

### リポジトリのフォークとクローン

この講義では、講師が用意したリポジトリをフォークして使用します。

1. [https://github.com/momeemt/minimake](https://github.com/momeemt/minimake) にアクセスします
2. 右上の Fork ボタンをクリックして、自分のアカウントにフォークします
3. フォークしたリポジトリをローカルにクローンします

```bash
# 自分のユーザー名に置き換えてください
$ git clone https://github.com/<YOUR_USERNAME>/minimake.git
$ cd minimake
```

4. タグを取得します

```bash
$ git fetch --tags
```

各章の実装を始める前に、対応するタグから新しいブランチを作成して作業することをお勧めします。

```bash
# 例: 第3章の初期状態からブランチを作成
$ git switch -c my-chapter3 chapter3-begin
```

実装が完了したら、フォークしたリポジトリにプッシュしてください。GitHub Actions が自動的にテストを実行し、実装が正しいかどうかを確認できます。

```bash
$ git push -u origin my-chapter3
```

プッシュ後、GitHub のリポジトリページで Actions タブを開くと、テストの実行結果を確認できます。

:::message
フォークしたリポジトリでは、GitHub Actions がデフォルトで無効になっています。初めて Actions タブを開いたときに表示される "I understand my workflows, go ahead and enable them" ボタンをクリックして有効化してください。
:::

また、各章の模範解答を確認したい場合は、以下のようにします。

```bash
# 例: 第3章の模範解答を確認
$ git switch --detach chapter3-end
```

### GCC のインストール

ビルドシステムの動作確認にはCコンパイラが必要です。

macOS の場合:

```bash
# Xcode Command Line Tools に含まれています
xcode-select --install
```

Ubuntu/Debian の場合:

```bash
sudo apt update && sudo apt install build-essential
```

Windows の場合:

WSL2（Windows Subsystem for Linux）を使用するか、[MinGW-w64](https://www.mingw-w64.org/) をインストールしてください。WSL2 の使用を推奨します。

### 動作確認

以下のコマンドが実行できることを確認してください。

```bash
$ python3 --version
Python 3.11.0  # 3.10以上であればOK

$ gcc --version
gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0  # バージョンは異なっていてOK
```

実装したビルドシステムが正しく動いているかどうかは、GitHub Actions およびローカルでテストを実行することで確認できます。

## この講義で得られるもの

この講義を通じて、以下のことを学ぶことができます。

- ビルドシステムの内部構造と設計思想
- 依存関係の解決（トポロジカルソート）
- インクリメンタルビルドの仕組み
- 実用的な Python プログラミング

普段何気なく使っている `make`、`cargo build`、`npm run build` などのコマンドの裏側で何が起きているのか、その本質を理解することで、ビルドに関するトラブルシューティング能力も向上します。
