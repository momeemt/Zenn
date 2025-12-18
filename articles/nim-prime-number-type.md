---
title: "Nim | 素数型を定義する"
emoji: "🔢"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Nim, メタプログラミング]
published: true
---

# 素数型を定義する
こんにちは、Nimで素数のみを許容する型について考えます。

:::message alert
結論から言えば型クラスを**変数の型**に当てることで素数型を実装しますが、型クラスは**ジェネリクスの制約として**働く言語仕様であって、このような使い方は開発者の意図から外れるものです。プロシージャの引数型・戻り値型・変数の型の明示的な記述など、プログラムにおける型の記述はすべて同様に解釈されるために起こるため可能なHackで、今後修正される可能性もあります。
:::

## 実装
基本的な方針として、コンパイル時に素数を列挙して型クラスを定義することを考えます。
しかしNimはTypeScriptにあるようなリテラル型[^2]が存在しないので素朴な方法では実装することはできません。

```nim:コンパイルできない
type PrimeNumber = 2 | 3 | 5 | 7
```

そこで`range`型を使います。
`range[n..m]`型は序数型クラスに属する型の値`n`から`m`までの範囲内の値を許容する型で、`range[n..n]`型は値`n`のみを許容する型となるため擬似的にある値に依存した型を表現することができます。

```nim:range型を利用した列挙
type PrimeNumber = range[2..2] | range[3..3] | range[5..5] | range[7..7]
```

ここで問題になるのは`n`をリテラルで与えると`range[n..n]`型だと見做されますが、Union型では`n`や`m`をリテラルで与えても`range[n..n] | range[m ..m]`型と見做されないことです。確認してみましょう。

```nim:range型のUnionの挙動を確認
type PrimeNumber = range[2..2] | range[3..3] | range[5..5] | range[7..7]

const
  num1: range[2..2] = 2
  num2: PrimeNumber = 2
```

```text:stdout
/usercode/in.nim(5, 23) Error: type mismatch: got 'int literal(2)' for '2' but expected 'PrimeNumber = range 2..2(int) or range 3..3(int) or range 5..5(int) or range 7..7(int)'
```

この挙動は明らかにバグですが、元々型クラスは変数の型になることが想定されていませんので仕方ありません。
一旦は型変換によって値を受け入れてもらうことにします。
`range`型は型引数を2つ記述する必要がありますが、今回は単一の値に依存した型としてしか利用しないので`R`型というヘルパー型と、その値をより簡単に生成できる`r`というヘルパープロシージャを実装します。

```nim:rangeのヘルパー
type R[I: static int] = range[I..I]

proc r (I: static int): R[I] =
  result = R[I](I)
```

これにより、たとえば`r(2)`と呼び出すことで`range[2..2]`型に型変換できるようになりました。
次に静的な整数型の値を`R[I]`型に暗黙に変換し自然な表記を実現するためにコンバータを定義します。

```nim:R[I]へのコンバータ
converter toR (I: static int): R[I] =
  result = r(I)
```

すると、最初に定義した`PrimeNumber`型に対して`2`、`3`、`5`、`7`の数値リテラルが受け入れられるようになりました。

```nim:ok
const
  num1: range[2..2] = 2
  num2: PrimeNumber = 2
```

では、次に`|`演算子で各`range`型のUnionを作成します。
真心のこもったハードコーティング...でも私は一向に構いませんが、ある上限値を設けてそれ以下の素数をすべて`|`演算子でつないだ型クラスを自動生成したくなります。
Nimにはマクロと呼ばれる抽象構文木を操作できる便利な言語機能があり、これで実装します。

```nim:PrimeNumberの構造を観察
type PrimeNumber = range[2..2] | range[3..3] | range[5..5] | range[7..7]
```

まず`range`型は中括弧（`nnkBracketExpr`）と識別子`range`（`nnkIdent`）、リテラル（`newLit`）から成ります。
単一値の`range[I, J]`型、もとい`R[I]`型を生成するために、以下のような`NimNode`を返すプロシージャを定義します。

```nim:range型のASTを返す
proc getRTypeNimNode (n: int): NimNode =
  result = nnkBracketExpr.newTree(
    newIdentNode("range"),
    nnkInfix.newTree(
      newIdentNode(".."),
      newLit(n),
      newLit(n)
    )
  )
```

次にそれらは`|`という**中置演算子**（`nnkInfix`）から成ります。
2つの`R[I]`型をつなぐとき、`range[2..2] | range[3..3]`のように両方とも`range`型である場合と、`(range[2..2] | range[3..3]) | range[5..5]`のように片方が既に中置演算子を含む抽象構文木である場合があります。
そこで、整数を2つ受け取って中置演算子を含むASTを返すプロシージャと、ASTと整数を1つ受け取ってASTを返すプロシージャを実装します。

```nim:中置演算子を含むASTを返す
proc infixUnionTypeNimNode (n, m: int): NimNode =
  result = nnkInfix.newTree(
    newIdentNode("|"),
    getRTypeNimNode(n),
    getRTypeNimNode(m)
  )

proc infixUnionTypeNimNode (ast: NimNode, n: int): NimNode =
  result = nnkInfix.newTree(
    newIdentNode("|"),
    ast,
    getRTypeNimNode(n)
  )
```

これらを統合して実装したのが以下になります。やや長いので折りたたみます。

:::details 実装
@[gist](https://gist.github.com/momeemt/790ec8dd89a4b163f2ac9ec0311a1562)
:::

```nim:静的に素数判定が可能になった
type PrimeNumber {.pn(200).}

const
  pn1: PrimeNumber = 181

  # type mismatch: got 'int literal(122)' for '122' but expected 'PrimeNumber = ran...
  pn2: PrimeNumber = 122
```

# 終わりに
マクロにより型の抽象構文木を自由に操作できるため、Nimの型システムの表現力では人為的に厳しいものも作成できます。
CTFE（コンパイル時関数実行）が効くのも良い点で、たとえば`PrimeNumber`型はある数までの素数のリストアップをやっていて、他のほとんどの計算も可能です。
実用的なトピックではありませんが、マクロや`static[T]`型を使えば楽しく遊ぶには十分です。
もし関心があればやってみてください。

[^2]: https://typescriptbook.jp/reference/values-types-variables/literal-types
