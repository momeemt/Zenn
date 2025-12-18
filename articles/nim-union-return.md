---
title: "Nim | 異なる構造体型を返すプロシージャを実装したい"
emoji: "🧙‍♀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Nim]
published: true
---

# はじめに
Nimはconcreteではない型[^1]を型定義やプロシージャ/関数の引数以外で扱うことはできません。
つまり異なる構造体型`A`、`B`が存在した時に`A | B`型を返り値の型として定義することはできません。
一方であるJSONから複数の構造体をdeserializeしたい場合には、それらの構造体を返すイテレータがあると便利です。
本記事では、**継承**と**参照**を利用して異なる構造体型を返す黒魔術を紹介します[^2]。

# 継承と参照を利用して異なる構造体型を返す
返したい構造体型は、`RootObj`を継承する`ref object`として定義します[^3]。

```nim:main.nim
type
  A = ref object of RootObj
    a_field1: int

  B = ref object of RootObj
    b_field1: string

proc getAorB (cond: bool): ref RootObj =
  if cond:
    result = A(a_field1: 10)
  else:
    result = B(b_field1: "Hello")
```

`RootObj`を継承する`object`（構造体）である場合、`getAorB`プロシージャを呼んだ時点でランタイムエラーが発生します[^4]が、参照の場合は代入できます[^5]。
ここで注意することは、`getAorB`によって得られる値の型は`ref RootObj`なので雑に参照をはがすと`RootObj`が得られて`A`、`B`のプロパティは消滅することで、キャストしてあげることでフィールドを読み出します。

~~以下は挙動です。不正な操作に対して何が起こっているのかはよく分からないです。そもそも`cast[T]()`は安全な操作ではないので仕方ないです、やめましょう。~~

https://twitter.com/demotomohiro/status/1552287944203182082?s=20&t=eLdGbjEwZzO79g2rr2XqlA

[@demotomohiro](https://twitter.com/demotomohiro)さんにご指摘をいただきました。強制的なキャスト（`cast[T]`）ではなく安全なキャストを利用することで`ObjectConversionDefect`例外を捕捉できます。
また、`of`を利用することで参照型の判別を行うことができます。

```nim:main.nim
echo getAorB(true)[]
echo A(getAorB(true))[]
echo B(getAorB(false))[]

# ofによる判別
let AorB = getAorB(true)
if AorB of A:
  echo A(AorB).a_field1
else:
  echo B(AorB).b_field1

# ObjectConversionDefect例外の発生
echo A(getAorB(false))[].a_field1
echo B(getAorB(true))[].b_field1
```
```bash:zsh
()
(a_field1: 10)
(b_field1: "Hello")
10
/usercode/in.nim(26)     in
/playground/nim/lib/system/fatal.nim(53) sysFatal
Error: unhandled exception: invalid object conversion [ObjectConversionDefect]
```

[^1]: コンパイル時に型が一意に定まらない型のことです。非型クラスと呼んでも良い気がしますがコンパイラメッセージに従っています。
[^2]: Nimでconcreteではない型を返したいほとんどの場合では設計が誤っていると個人的には思います。deserializerに関しても本当はもっと良い実装方法があるのではないかと疑っているので、あれば教えていただけると助かります。
[^3]: Nimでは`RootObj`を継承する構造体型のみが継承することが可能です。
[^4]: `RootObj`に対して、`A`、`B`の異なる構造体を割り当てることに対してのエラー。
[^5]: 理由は全然分かっていませんが、`ref T`は`pointer`型くらい緩く扱われているのではないかなと思います。この挙動自体はコントリビューターは認知していそうなのでバグとかではないと思うけど...

# おまけ
たくさんフィールドを生やすことでも実装することができます。
こちらは簡単で、返す可能性のある構造体型を全てフィールドに持つ構造体を定義します。
実装としては次のようなものが考えられます。

```nim:anyKind.nim
type
  AnyKind = enum
    akA, akB, akC
  
  A = object
    value: int
  B = object
    value: int
  C = object
    value: int

  AnyObj = object
    case kind: AnyKind
    of akA:
      valueA: A
    of akB:
      valueB: B
    of akC:
      valueC: C

proc getAnyObj (num: int): AnyObj =
  if num > 0:
    result = AnyObj(kind: akA, valueA: A(value: num))
  elif num == 0:
    result = AnyObj(kind: akB, valueB: B(value: num))
  else:
    result = AnyObj(kind: akC, valueC: C(value: num))

echo getAnyObj(3).valueA.value
```

構造体は`case`/`when`文を利用したメンバーの条件分岐が可能です。
`getAnyObj(3)`は`valueB`、`valueC`を持たないのでアクセスするとランタイムで落ちる点は注意が必要です。
特に拘りがなければこれでも良いですが、返す可能性のある構造体型を全てフィールドに持つ構造体をコンパイル時に準備する必要があります。ユーザーによる拡張を含むdeserializerを考えていたためそれを構築することは難しく、黒魔術で実装しました。
