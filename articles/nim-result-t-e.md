---
title: "Nim | Result[T, E]型を定義する"
emoji: "👑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Nim, Result]
published: true
---

# NimでResult[T, E]型を定義する
こんにちは。今回はNimで`Result[T, E]`型の実装を試みます。
この記事は、[Nim Advent Calendar 2021](https://qiita.com/advent-calendar/2021/nim)の5日目です。

`Result[T, E]`型は、`T`型の正常値か`E`型の異常値を持つ型です。エラーハンドリングや非同期処理に用いられます[^0]。
これを持つ多くの言語[^1]では、列挙型を用いて実装されています。しかし、Nimの列挙型`enum`は非定数のデータを持つことはできません[^2]ので、構造体`object`を用いて実装したいと思います。

```nim:results.nim
type
  ResultKind {.pure.} = enum
    kOk, kErr

  Result* [T, E] = object
    case kind: ResultKind
    of kOk:
      ok: T
    of kErr:
      err: E
```

これは簡単な構造で、`kind`メンバーが`kOk`をとるとき`T`型の`ok`メンバーを、`kErr`をとるとき`E`型の`err`メンバーを持ちます[^3]。とはいえ、`kind`メンバーが定まった時点で`ok`をとるか`err`をとるかが定まるため、実行時に`kind`の値を変更することはできません[^4]。

## `Ok`・`Err`を実装する
正常値・異常値をそれぞれ代入するような`Ok`・`Err`を実装します。次のような素直なプロシージャによる実装がまず考えられます。

```nim:results.nim
proc Ok[T, E] (value: T): Result[T, E] =
  result = Result[T, E](kind: kOk, ok: value)
  
proc Err[T, E] (err: E): Result[T, E] =
  result = Result[T, E](kind: kErr, err: err)
```

これはこれで良いのですが、次のようなコードをコンパイルできないことに注意しなければなりません。

```nim:over100.nim
proc over100 (num: int): Result[int, string] =
  if num >= 100:
    Ok(num)
  else:
    Err("too small")
```

プロシージャの戻り値は`Result[int, string]`であることは明白ですが、Nimの型推論器では`Ok`の型変数`E`が`string`であるべきと推論することができないからです。

```nim:over100.nim
proc over100 (num: int): Result[int, string] =
  if num >= 100:
    Ok[int, string](num)
  else:
    Err[int, string]("too small")
```

このように明示的に型変数を与えることでコンパイルを通せます。
しかし、これは少し煩雑です。

```rust:sq.rs
fn sq(x: u32) -> Result<u32, u32> { Ok(x * x) }
```

Rustでは上のようなコード[^5]が許されていますから、Nimにおける`Result[T, E]`でも上手く表現したいところです。
そこで、`macro`を用いてプロシージャの`result`変数から型情報を入手します。

```nim:results.nim
macro Ok* (value: untyped): untyped =
  result = quote do:
    proc Ok2 [T, E] (res: Result[T, E], value: T): Result[T, E] =
      result = Result[T, E](kind: ResultKind.kOk, ok: value)
    result = result.Ok2(`value`)

macro Err* (err: untyped): untyped =
  result = quote do:
    proc Err2 [T, E] (res: Result[T, E], err: E): Result[T, E] =
      result = Result[T, E](kind: ResultKind.kErr, err: err)
    result = result.Err2(`err`)
```

`Ok`マクロは`Ok2`プロシージャと`result`変数への代入に展開されます。`Ok2`は、最初に考えた素直な実装と同じです。次に、展開先のプロシージャにおける`result`変数に対して`Ok2`を適用します。`result`はプロシージャ内で暗黙に定義される変数で、戻り値の型から`Result[T, E]`の型変数は定まっていますから、型推論器の代わりに型情報を手に入れることができます。

## `match`を実装する
Nimにはパターンマッチがありません[^6]ので、`Result[T, E]`を処理する`match`を実装します。

```nim:match.nim
macro match* (value, body: untyped): untyped =
  expectLen(body, 2)
  let branches = [body[0][0], body[1][0]]
  var (okExists, errExists) = (0, 0)
  expectKind(branches[0], {nnkIdent, nnkOpenSymChoice})
  expectKind(branches[1], {nnkIdent, nnkOpenSymChoice})
  for branch in branches:
    if branch.repr == "Ok": okExists += 1
    elif branch.repr == "Err": errExists += 1
    else: error("Unexpect Identify " & branch.repr, branch)
  if okExists == 2: error("You describes two Ok clauses.", branches[1])
  elif errExists == 2:  error("You describes two Err clauses.", branches[1])
  result = quote:
    block:
      var res = `value`
      template Ok (okName, okBody: untyped): untyped =
        proc okProc (okName: auto) =
          okBody
        if res.kind == ResultKind.kOk:
          okProc(res.ok)
      template Err (errName, errBody: untyped): untyped =
        proc errProc (errName: auto) =
          errBody
        if res.kind == ResultKind.kErr:
          errProc(res.err)
      `body`
```

前半はvalidな抽象構文木であるかのチェックです。後半の`result`変数に代入されたASTが`match`の展開結果ですが、主に`Ok`・`Err`の定義と呼び出しです。

```nim:over100.nim
200.over100.match:
  Ok value:
    echo value == 200
  Err err:
    fail()
```

`match`は上のように利用されます。`Ok`と`Err`の2パターンを両方記述する必要があり、ここでは`200.over100`の値が`(kind: kOk, ok: 100)`であるため、`Ok`テンプレートが展開されたコードで`if res.kind == ResultKind.kOk: OkProc(res.ok)`が呼び出され、`echo 200 == 200`が実行されます。一方、`Err`テンプレートの方では`errProc`は定義されますが呼び出されません。

## エラー伝搬
`?`・`?.`演算子を用いると、値が`kErr`の時に処理を中断して呼び出し元に値を返します。

```nim:results.nim
template `?`* [T, E] (value: Result[T, E]): T =
  case value.kind:
  of ResultKind.kOk:
    value.ok
  of ResultKind.kErr:
    return

template `?.`* [T] (left: T, right: proc): untyped =
  right(?(left))
```

Nimはユーザー定義演算子において後置演算子を認めないため、`?.`中置演算子を用いてコードの順番を入れ替えます。

## 感想
利用価値の高い型だと思いますが、実現にはかなり黒魔術めいたことをしなければなりません[^7]。堅牢なテストによって管理したいところです。
Araqをはじめとしてコアメンバーは例外についてどのように考えているかはかなり気になっています。標準ライブラリからは関数型言語的な機能を好んで提供している雰囲気を感じ取っていますし、そもそも`options`を提供しています。
組み込みなどメモリパフォーマンスが求められる用途についてはさておき、Nimの実行時エラーはあまり情報量がなくて辛いことがかなりあるので、実行時エラーを出さない方向に進んでもいいんじゃないかなと思いました。

## Advent Calender 2021
- 昨日(12/4): [Nimでアプリケーション開発をするための設計のベストプラクティス](https://qiita.com/dumblepy/items/8d13592d6760d0155d89)
- 明日(12/6): [Nim で TwitterAPIv2 (OAuth1) 叩こうと思ってミスった話](https://qiita.com/imtanization/items/ba6378dbcb23b45d5711)

[^0]: `Result[T, E]`をどういう用途に使うべきかは各言語ごとに立場が異なる。Nimでは例外やEffect Systemへのサポートが手厚いのでこちらが主流になることはあまりないように思える。
[^1]: Rust、Swiftなど
[^2]: 困った。`enum`は列挙型であるが序数型クラス（`Ordinal`）に属しているため、非定数のデータを持つことは`inc`・`dec`など`Ordinal`な型全般が満たすべき操作を壊してしまう。とはいえ、disjointな値を持たせることができてしまう。Cとの互換性を持たせる以外の理由は恐らくないが、それでもあまりここら辺の感覚が一貫しているようには思えない。
[^3]: 受け取った型変数を必ずしも全て使う必要はない。
[^4]: Error: unhandled exception: assignment to discriminant changes object branch; compile with -d:nimOldCaseObjects for a transition period [FieldDefect]
[^5]: [Rust Docs - Enum std::result::Result](https://doc.rust-lang.org/std/result/enum.Result.html#examples-17)から引用
[^6]: Fusionにはある。`case`はパターンマッチではない。
[^7]: とはいえ、かなりライトなマクロの使い方だと思う。これくらいならやっても怒らないでほしい。