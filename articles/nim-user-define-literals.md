---
title: "Nim 1.6 でユーザー定義リテラルが使えるようになった"
emoji: "👑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Nim]
published: true
---

# ユーザー定義リテラルが追加された
こんにちは。先日、Nimが1.6.0にアップデートされました。
相変わらずWindowsでは未だにセキュリティソフトにトロイの木馬と検出されてしまうようで何より[^0]です。お家芸[^5]ですね。

[^0]: https://twitter.com/takscape/status/1450607389820915719?s=20
[^5]: 長らくセキュリティソフトウェアやWindows Defenderにトロイの木馬扱いされています。実の所これは[新興マルウェアの開発の一部にNimが利用された](https://thehackernews.com/2021/03/researchers-spotted-malware-written-in.html)ことが原因の1つ。

@[tweet](https://twitter.com/nim_lang/status/1376571789086765057?s=20)

さて、本記事ではようやく利用できるようになった**ユーザー定義リテラル**について解説していきます。
2021年3月時点では公式TwitterでTipsとして投稿され、[Nim Manual](https://nim-lang.github.io/Nim/manual.html#numeric-literals-custom-numeric-literals)にも記載されていた本機能ですが、Nim 1.4.8までは解釈できずにコンパイルエラー[^1]を検出していました。
なぜあたかも利用可能な機能のように説明されていたのかは不明です。

[^1]: `Error: missing closing ' for character literal`

---

ユーザー定義リテラルにより、開発者が独自の数値リテラルを追加できるようになります。
多くのプラットフォームでは、`23`のようなシンプルな整数は`int`型に、`56.7`のような浮動小数点数は`float`型に型付けされます[^2]。しかし`system`モジュールでは`int16`や`float32`など特定のメモリサイズを確保する型が存在します[^3]。
明示的な型変換の代替手段として、Nimではシングルクォーテーション `'` と接尾辞を用いて数値リテラル[^4]がどの型に型付けするかを指定することができます。

```nim:リテラル接尾辞
echo typeof 32
echo typeof 32'i8
echo typeof 32'i16
echo typeof 32'i32
echo typeof 32'i64
```
```txt:stdout
int
int8
int16
int32
int64
```

[^2]: 多くのプラットフォームでは、`int32`型の範囲の数値の場合は`int`型に型付けされ、それに留まらず`int64`型の範囲が必要な場合は`int64`型に型付けされます。しかし、`int`型は対応する数値範囲に反して8バイトのメモリを確保します。また、この仕様は環境依存であり、必ず`int`型が上のような振る舞いをするとは限りません。
[^3]: これは単にメモリの削減に役立つだけではなく、CやC++などポインタを直接操作する言語とのFFIにおいて、メモリ領域を厳密に管理したい場合に有用です。
[^4]: 本記事では数値リテラルの詳細な説明は省略しますが、興味があれば[Nim Manual - Numeric Literals](https://nim-lang.github.io/Nim/manual.html#lexical-analysis-numeric-literals)に規則表が載せられていますのでそちらを参照してください。

ユーザー定義リテラルは、少なくとも1つの文字列を含むパラメータを受け取る、シングルクォーテーションから開始する演算子と見做せます[^6]。コンパイラは`125'cm`を`125.'cm`と変換することで、呼び出し可能な識別子として呼び出せるようになります。
`distinct float`型を2つ定義し、それらの簡単な演算を実装したサンプルを示します。

[^6]: `proc`、`iterator`、`template`、`macro`など呼び出し可能な識別子として定義します。

```nim:単位リテラルを実装する
import strutils, typetraits

type
  unit = m | cm
  m = distinct float
  cm = distinct float

proc `'m` (num: string): m = num.parseFloat.m
proc `'cm` (num: string): cm = num.parseFloat.cm
  
proc `$`[T: unit] (num: T): string = $num.float & T.name
proc `+`[T: unit] (left, right: T): T = (left.float + right.float).T
proc `*`[T: unit] (left: T, right: float): T = (left.float * right).T

proc `+` (left: m, right: cm): m = left + right.m * 0.01
proc `+` (left: cm, right: m): cm = left + right.cm * 100

echo 2'm + 50'cm
echo 125'cm + 3'm
```

```txt:stdout
2.5m
425.0cm
```

---

数値を文字列として受け取るのは、数値リテラルがアルファベットを含む16進数を許容しているからだと考えられます。
数値リテラルを対象にしていますが、返り値の型に関する制約がないため文字列を受け取り文字列を返すようなユーザー定義リテラルが実現できます。

```nim:ユーザー定義リテラルの悪用
import unicode
proc `'str` (num: string): string = num.reversed
proc `'phone` (num: string): string = num[0..1] & "-" & num[2..5] & "-" & num[6..9]

echo 12345'str
echo 0312345678'phone
```

また、複数のパラメータを取ることが可能です。

```nim:複数のパラメータを取るユーザー定義リテラル
import strutils

type myFloat = distinct float

proc `'m` (num: string): myFloat = num.parseFloat.myFloat

template `'m` (num: string, op: untyped, value: float): myFloat =
  var res = num.parseFloat
  res = op(res, value)
  res.myFloat
  
proc `$` (num: myFloat): string = $num.float

echo 10'm
echo 10'm(`+`, 10)
echo 10'm(`*`, 200)
```

## 感想
地味ですが、通貨や単位など`distinct`型クラスで扱いたい場合にこれまでは愚直に型変換のプロシージャを通す必要があったので、非常に便利な機能追加と言えると思います。
内部的には演算子と同様に扱われるため自由度が制限されていない点は微妙ですね。実質的には数値リテラルに対する呼び出し可能な識別子を呼び出す演算子が一つ増えたようなものです。好き好んで使う開発者は多くはないと思うのですが[^8]。

Nim 1.6で追加/変更/削除された機能は他にもたくさんあるので、時間を作りながらぼちぼちと投稿していこうと思います。それでは。

[^8]: ちなみに多くのシンタックスハイライトではシングルクォーテーションを文字リテラルとして解釈しているため、Nim Playgroundすら解釈が間に合っていない状況であり、自動インデントがかなり崩れます。
