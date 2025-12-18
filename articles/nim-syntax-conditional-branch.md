---
title: "Nim | 条件分岐の基本"
emoji: "👑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Nim]
published: true
---

# 条件分岐
条件分岐とは制御構文のひとつで、ある処理を条件によって切り替えることができます。
Nimにおける条件分岐はいくつかの選択肢があり、目的に応じて使い分ける必要があります。

## 単純な条件分岐
`if`はNimの最も単純な条件分岐構文です。
`bool`型の値を条件として受け取り、`true`の場合は`if`ブロックのプログラムを、条件が`false`の場合は`else`ブロックのプログラムを実行します。
また、2つ以上の条件を記述するときは`elif`ブロックを記述します。

```nim:if文
let number = 23

if number mod 2 == 0:
  echo "even"
elif number mod 3 == 0:
  echo "multiple of three"
else:
  echo  number
```

```bash:stdout
23
```

また、Nimには値を返す**式**と値を返さない**文**が存在しますが、条件分岐は文としても式としても記述できます。

```nim:if式
let number = 23

echo if number mod 2 == 0:
  number div 2
elif number mod 3 == 0:
  number div 3 + 1
else:
  (number - 1) div 2
```

```bash:stdout
11
```

`if`文は`else`ブロックを書くことは強制されませんが、`if`式では何らかの値を必ず返す必要があるので、書かなければいけません。

```nim:elseブロックを省略する
let number = 23

echo if number mod 23 == 0:
  "lucky number"
```

```bash:コンパイル時エラー
Error: expression '"lucky number"' is of type 'string' and has to be used (or discarded)
```
*`lucky number`に問題があるように見えるが、そもそもパースできていない*

## 値を複数のパターンと比較する
`case`は、値を複数のパターンと比較する強力な機能です。
序数型クラスに属する型、もしくは`string`、`float`のいずれかの値を受け取って、複数のパターンと比較します。
Javaなどの`switch`と異なり、最初にマッチしたパターンのみを実行するため、`break`文などを記述する必要はありません。

```nim:case文
type fruits = enum
  orange, banana, apple, muscat, grape

let recently_eaten_fruit = muscat

case recently_eaten_fruit
of orange, muscat, grape:
  echo "My favorite flutes!"
else:
  echo "Not my favorite, but a tasty fruit."
```

このとき、序数型クラスに属する型の場合は全てのパターンが網羅されている必要があります。

```nim:網羅しない場合
case recently_eaten_fruit
of orange, muscat, grape:
  echo "My favorite flutes!"
```
```bash:コンパイルエラー
Error: not all cases are covered; missing: {banana, apple}
```

`if`と同様に式として記述できます。式として記述する場合には、値が`string`型か`float`型であったとしても全ての場合で何らかの値を返すようにしなければなりません。

```nim:case式
let n = 3.14

echo case n:
  of 3.14: "pi"
  of 2.71: "e"
  else: "unknown"
```

## コンパイル時評価条件分岐
Nimはコンパイル時に評価される条件分岐、`when`を備えています。
`if`や`case`は基本的には実行時に評価される[^1]のですが、`when`はコンパイル時に条件が評価され、**条件を満たすブロックのみがコンパイルされ**ます。
C言語の`#ifdef`に似た概念と言えます。

[^1]: `macro`内に記述された`if`や`case`はもちろんコンパイル時に評価されます

```nim:when文
when defined(windows):
  echo "windows"
elif defined(macos):
  echo "MacOS"
elif defined(linux):
  echo "Linux"
else:
  echo "unknown operation system"
```

OS固有のコードや環境変数に依存するプログラムを生成するために便利です。