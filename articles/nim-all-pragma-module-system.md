---
title: "Nim 1.6 で追加されたモジュールのプライベートアクセスについて"
emoji: "👑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Nim]
published: true
---

# プライベートアクセスが追加された
こんにちは、[前回](https://zenn.dev/momeemt/articles/nim-user-define-literals)に引き続きNim 1.6 Major Updateで追加された機能について紹介します。
この記事は、[Nim Advent Calendar 2021](https://qiita.com/advent-calendar/2021/nim)の12日目です。

実験的機能で試されていたモジュール上でプライベートシンボル・メンバーへアクセスを行う`{.all.}`プラグマが追加されました。
本機能は[timotheecour](https://github.com/timotheecour)氏によって2020年12月9日に[RFC](https://github.com/nim-lang/RFCs/issues/299)で提案されたものであり、氏とコアメンバーが実装に取り組んできました。
次のコード[^1]で示されているように、`import`文で読み込むモジュールオブジェクトに対してプラグマを付与することで、公開されたシンボルだけでなく非公開のシンボルも含めて全てのシンボルにアクセス可能になります。

```nim:sample.nim
from system {.all.} as system2 import nil
echo system2.ThisIsSystem # ThisIsSystem is private in `system`
import os {.all.} # weirdTarget is private in `os`
echo weirdTarget # or `os.weirdTarget`
```

例えば`ThisIsSystem`は`system`モジュールで定義される非公開の定数[^2]ですし、`weirdTarget`は`os`モジュールで定義される非公開の定数[^3]です。

Nimが他のファイルで定義されたシンボルにアクセスする方法は2つあります。
多くのケースでは`import`を選択しますが、`include`を用いてファイルを直接読み込むことでもアクセス可能です[^4]。
しかし、これはモジュールシステムの管理下にない操作です。例えば、読み込み先のファイルに読み込み元と同名のシンボル名が定義されていた場合、モジュールシステムの管理下にある`import`ではモジュールの名前空間を利用して両立できます。しかし、`include`で読み込む場合は単にソースコードがすべて展開されるためコンパイルエラーが発生します。

そもそもプライベートなシンボルにアクセスするプログラムは直感的に危険に感じますが、例えばユニットテストではなるべくすべてのシンボルに対してテストしたいという需要があります。
現在は`include`が用いて展開していますが、今後は`{.all.}`を用いてモジュールシステムの管理下でプライベートなシンボルにアクセスすることになります。

`include`より本機能が優れている点として**部分的にプライベートシンボルにアクセス可能**なことが挙げられます[^5]。

[^1]: [Nim - changelog_1_6_0](https://github.com/nim-lang/Nim/blob/version-1-6/changelogs/changelog_1_6_0.md)より引用。
[^2]: デバッグ用の定数。https://github.com/nim-lang/Nim/blob/c239db4817586f769327a6d27c244682111f903a/lib/system.nim#L261
[^3]: 奇怪なターゲット。NimScriptか、JavaScriptターゲットである場合に`true`となります。https://github.com/nim-lang/Nim/blob/62701cd3b92677c2014cf70b1cf5a5ba6e0468bf/lib/pure/os.nim#L37
[^4]: [karax](https://github.com/karaxnim/karax)はExampleで`include karax / prelude`を採用しています。karaxの実装を深く理解しているわけではないため意図は不明ですが、`import`、`export`を兼ねた動作が期待できます。
[^5]: `from`・`except`を用いてモジュールからシンボルを選択的に読み込みます。

---

Nim 1.6で追加された`importutils`モジュールで`privateAccess`プロシージャが提供されています。
これは、型の非公開メンバにアクセスするためのプロシージャです。

```nim:std/importutils.nim
proc privateAccess(t: typedesc) {.magic: "PrivateAccess", raises: [], tags: [].}
```

例えば、次のような型を定義した`foo.nim`を示します。
`member1`は非公開メンバですから他のモジュールからそのままではアクセスすることはできません。

```nim:foo.nim
type
  Foo = object
    member1: int

proc initFoo* (): Foo = Foo(member1: 1)
```

しかし、`bar.nim`からでも`privateAccess`を適用することで非公開メンバにアクセス可能です。

```nim:bar.nim
import foo
import std/importutils

var foo = initFoo()
privateAccess(foo.type)
foo.member1 = 10 # ok
```

# 感想
プライベートアクセスの紹介でした。
じわじわと少しずつではありますが、NimはNimの路線でコードを安全に保つ方法が模索されて進化しています。特にユニットテストなどにおいて、ぜひ`include`から脱却してモジュールシステム上でプライベートシンボルにアクセスしましょう。

## Advent Calender 2021
- 昨日(12/11): [NimでElectron（デスクトップアプリ）](https://qiita.com/pianopia/items/e0a1453245a4355ac422)
- 明日(12/13): お楽しみに！