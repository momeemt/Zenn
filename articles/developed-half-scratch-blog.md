---
title: "自作マークアップ言語からはじめる自作ブログ"
emoji: "✏️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Nim, blog, githubactions, Docker, Netlify]
published: true
---

# ブログを自作した
こんにちは。プログラマといえば自作ブログ[^shosetsu]、自作ブログとはMarkdownで文章を書いて静的Webサイトジェネレータでよしなに変換してCIで自動デプロイしてVercelなどで配信するアレのことで、だいたい3割くらいのプログラマがやっていて残りの5割は憧れていることでお馴染みです。
ところで皆さまは反骨精神をお持ちだと思いますが[^gyakubari]、モダンな技術を組み合わせて物を作ることへの反発心を抱いたことがあるのではないでしょうか。本記事では自作ブログを自作マークアップ言語から作った話をします。

https://blog.momee.mt

[^shosetsu]: 諸説
[^gyakubari]: あなたは逆張りオタクですか？偶然ですね、私もです！

# マークアップ言語の開発
まず、こちらが開発したマークアップ言語 BrackのHTMLへのトランスパイラです。

https://github.com/momeemt/brack

## なぜマークアップ言語を作るのか
自作ブログではたいていMarkdownが採用されます。Zennの記事もMarkdownで記述されていますし、多くのプロジェクトのREADMEやドキュメントでも目にします。
軽量マークアップ言語はデータ記述の整合性を保ちつつ、ソースコードの可読性も意識して設計されていますが、変換後のコンテンツのみが重要で、かつユーザーにより多様な表現力が求められるブログ記事の記述にはMarkdownはあまり向かないと考えています[^html-in-md]。
Markdownは標準的で明文化された仕様を持たない[^md-spec]ゆえに、運用されている方言が無数に存在しています[^md-dialect]。そのような現状を踏まえると、ブログを書くために必要なマークアップ言語は、特定の装飾に文法を対応させた言語ではなく、**文法はコマンドと引数の解釈のみを規定し、装飾をユーザーが定義することで自由に拡張可能な言語なのではないか**と思いました。

[^html-in-md]: 端的に言えばMarkdownにHTMLに混在しまくっている状態に良い思い出がない
[^md-spec]: それを解決するために、後発で[CommonMark](https://commonmark.org/)という厳密な定義を持つ規格がある。
[^md-dialect]: Markdown方言で最も有名なものの1つにGitHub Flavored Markdownがある。[GitHub Flavored Markdown は何であって何でないか](https://qiita.com/tk0miya/items/6b81e0e4563199037018)などを読むとCommon Markとの違いがわかりやすい。それはそれとして脚注を実装している方言があるから、こうして記事を書く気になっているのでいいこともある。

```bash:Brackの解釈
{コマンド名 引数1}
文章1[コマンド名 引数1, 引数2]文章2
文章3[コマンド名 引数1, 引数2, 引数3]
```

Brackは上のように、`[]`・`{}`・`<>`という3種類の括弧に囲まれた文字列を**コマンド**と認識し、別の文字列に置換します。ここで、コマンド名は記号である必要はなく任意の文字列[^except-brackets]を設定できます。これはかなり大事で、たとえばMarkdownで画像は`![alt](href)`と記述しますがこれが画像とは記号から想起されにくいです[^bold-md]。一方で、`img`などコマンド名が任意に記号、アルファベット、その他Unicode文字を受け入れることができると多数の装飾を呼び出す執筆者側の体験が向上すると考えました。

```bash:インライン構造とブロック構造
{* 見出し1}
こんにちは、[@ momeemt, https://twitter.com/momeemt]です！
```

```html:変換結果
<h1>見出し1</h1>
<p>こんにちは、<a href="https://twitter.com/momeemt">momeemt</a>です！</p>
```

波括弧（Curly Brackets, `{}`）はブロック構造を、角括弧（Square Brackets, `[]`）はインライン構造を取ります。HTMLに変換される場合はインライン構造は文書内に展開され、1つの段落（`p`タグ）にまとめられます。

[^except-brackets]: **実装上では。** 執筆中に気が付いたが、例えば括弧の終端文字（ここでは`]`）やエスケープを行う`\`が含まれるとダメっぽい。ああ...
[^bold-md]: それに対してボールド（`**text**`）や打ち消し線（`~~text~~`）は記号がそれらの意味を想起しやすいので良いと思う。前者はMarkdown以外にもよく採用されているので単に慣れな気もするが...

## トークン列への分割
まずはトークン列への分割を行います。Brackは括弧と引数の解釈、エスケープのみを行うので簡単です。
`[`、`]`、`{`、`}`、`<`、`>`をトークンとして解釈します。コマンド内では最初に現れる空白までをコマンド名として、その後は`,`によって引数を分割してトークン列とします。
そのほかの文字列はこれまでのルールによってトークン列に分割されるまでトークン列に追加されます。また、エスケープ文字（`\`）を発見した後にはその次の1文字はトークン列として解釈されず、探索中のトークン列に追加します。Brackはこの操作を`lex`というプロシージャで提供します。

```bash:hello.[]
{* 見出し1}
こんにちは、[@ momeemt, https://twitter.com/momeemt]です！
```

```nim:lexer
import brack

let src = block:
  var f = open("tests/hello.[]")
  defer: f.close()
  f.readAll()
echo lex(src)
```

```nim:stdout
@["{", "*", "見出し1", "}", "こんにちは、", "[", "@", "momeemt", ",", "https://twitter.com/momeemt", "]", "です！"]
```

## 抽象構文木への変換
次はトークン列を抽象構文木に変換します。
ノードはいくつか種類があり、重要なのは段落である`bnkParagraph`、角括弧（`[]`）である`bnkSquareBracket`、波括弧（`{}`）である`bnkCurlyBracket`、山括弧（`<>`）である`bnkAngleBracket`、引数である`bnkArgument`、コマンドの識別子である`bnkIdent`、文章である`bnkText`です。
Object variants[^ov]により、コマンド識別子と文章は文字列の値を持ち、その他のノードは子要素としてノードへの参照の配列を持ちます。

[^ov]: 構造体にあるフィールドによる条件分岐でフィールドを設定できる。これがNimにおける代数的データ型と主張されることもあるが静的に決定されないので違うと思う。今は異なる分岐で同名のフィールドを設定できないのが辛すぎるがRFCでそれをサポートすることは決定している。ADT欲しい。

```nim:構文木
type
  BrackNodeKind* = enum
    bnkInvalid
    bnkRoot
    bnkParagraph
    bnkSquareBracket
    bnkCurlyBracket
    bnkAngleBracket
    bnkArgument
    bnkIdent
    bnkText
  
  BrackNodeObj* = object
    id*: string
    case kind*: BrackNodeKind
    of bnkText, bnkIdent:
      val*: string
    else:
      children*: seq[BrackNode]
  
  BrackNode* = ref BrackNodeObj
```

パーサは左括弧（`[`・`{`・`<`）を見つけると終端を発見するまでトークン列を読み進め、`bnkXXXBracket`のノードを生成します。読み進めている最中に再度左括弧を見つけた場合は、その結果を子要素に持つ形で再帰を行います。
ノードはそれぞれ`id`を持っており、これはOID[^oid]を標準ライブラリの`std/oids`で生成し文字列に変換したものです。

[^oid]: Object Identifier

```bash:nest.[]
{* 見出し1}
[* [/ イタリック] [~ 打ち消し線 [/ イタリック]]]
```

```nim:parser
import brack
import brack/ast

let src = block:
  var f = open("tests/nest.[]")
  defer: f.close()
  f.readAll()
echo lex(src).parse()
```

```nim:stdout
bnkRoot (635e44d1ad39f876755905ba)
  bnkCurlyBracket (635e44d1ad39f876755905bc)
    bnkIdent (635e44d1ad39f876755905bd)
      *
    bnkArgument (635e44d1ad39f876755905be)
      bnkText (635e44d1ad39f876755905bf)
        見出し1
  bnkParagraph (635e44d1ad39f876755905bb)
    bnkText (635e44d1ad39f876755905c0)
    bnkSquareBracket (635e44d1ad39f876755905c1)
      bnkIdent (635e44d1ad39f876755905c2)
        *
      bnkArgument (635e44d1ad39f876755905c6)
        bnkSquareBracket (635e44d1ad39f876755905c7)
          bnkIdent (635e44d1ad39f876755905c3)
            /
          bnkArgument (635e44d1ad39f876755905c4)
            bnkText (635e44d1ad39f876755905c5)
              イタリック
        bnkText (635e44d1ad39f876755905c8)
        bnkSquareBracket (635e44d1ad39f876755905d0)
          bnkIdent (635e44d1ad39f876755905c9)
            ~
          bnkArgument (635e44d1ad39f876755905ca)
            bnkText (635e44d1ad39f876755905cb)
              打ち消し線
            bnkSquareBracket (635e44d1ad39f876755905cf)
              bnkIdent (635e44d1ad39f876755905cc)
                /
              bnkArgument (635e44d1ad39f876755905cd)
                bnkText (635e44d1ad39f876755905ce)
                  イタリック
```

パースできました！上手く木構造に変換されています。関係ないですが`brack/ast`というのはBrackの抽象構文木（AST）関連の型・便利プロシージャが定義されているモジュールで、Nimにおける文字列変換演算子である`$`が定義されているのでimportしています。標準出力である`echo`は内部的に`$`演算子を呼んでいるので`BrackNode`型をそのまま出力しようとするとコンパイルエラーになります。

## ジェネレータの生成
次に、抽象構文木をHTMLに変換します。しかしBrackはコマンドとその引数のみを解釈して具体的な装飾は実装しません。ユーザーが実装したジェネレータコンポーネントを統合して1つのジェネレータを組み立てるマクロを実装します。

```nim:Brackの標準ライブラリで実装されたジェネレータコンポーネント
proc h1* (text: string): string {.curly: "*".} =
  result = htmlgen.h1(text)

proc bold* (text: string): string {.square: "*".} =
  const style = style"""
    font-weight: bold;
  """
  result = htmlgen.span(text, style=style)

proc anchorLink* (text, url: string): string {.square: "@".} =
  result = htmlgen.a(text, href=url)
```

先にジェネレータコンポーネントがどのように実装されるかについて説明します。`square`・`curly`というプラグマが提供されており、これに渡した文字列がBrackにおけるコマンド名になります。
たとえば`h1`プロシージャは標準ライブラリの`std/htmlgen`を利用して`h1`タグを生成しています。つまり、`{* hello}`のようなBrackマークアップは`<h1>hello</h1>`に変換されます。
重要なのはパースして得られたコマンド名と引数の組み合わせからどうやってユーザーが実装したプロシージャを呼び出すかです。Nimは静的型付き言語なのでコンパイル時に整合性が取れる必要がありますが、`anchorLink`プロシージャを見るとわかるように各コマンドは取り得る引数の型や個数が違いますし、愚直にマップに突っ込んで動的に解決することは難しいでしょう。
そこで、マクロによりジェネレータコンポーネントのプロシージャ名・引数の組み合わせからプロシージャを呼び出すプログラムを生成します。

```nim:プラグマの実装
macro curly* (name: static[string], body: untyped): untyped =
  result = copy(body)
  let procNameIdent = newIdentNode("curly_" & resolveProcedureName(name))
  if result[0][1].kind == nnkAccQuoted:
    result[0][1][0] = procNameIdent
  elif result[0][1].kind == nnkIdent:
    result[0][1] = procNameIdent
```

ジェネレータコンポーネントに付与したプラグマはマクロで定義されています。
Nimの言語仕様では`#`はコメントの予約語でありプロシージャ名にすることはできません[^sharp]が、Brackのコマンド名としては利用したい場合があります。そこで、プラグマが受け取ったコマンド名をNim上で適切な識別子に変換（`resolveProcedureName`）し、プロシージャ名を書き換えます。こうすることで、Brackのソースコードから得られるコマンド名からプロシージャがそのまま対応する上、ライブラリ開発者はプロシージャに任意の名前をつけられるのでよりよくなります。

[^sharp]: VSCodeのNimの拡張機能であるkosz78.nimは``` `#` ```という名前のプロシージャを定義すると適当であるかのようにハイライトされますが、実際には`#`以下はコメント扱いされます

```nim:識別子の変換
func resolveProcedureName* (command_name: string): string =
  for ch in command_name:
    result.add $int(ch)
```

これにより、`*`というコマンド名を持つ`h1`プロシージャは`curly_42`というプロシージャに書き換えられます。Square Brackets、Curly Bracketsなど括弧の種類ごとにプラグマが用意されていることで、同名・同引数のコマンドを定義できる点も利点です[^resolve]。

[^resolve]: `[* foo]`からは`square_42`プロシージャが呼ばれるし、`{* foo}`からは`curly_42`プロシージャが呼ばれる

次に、ユーザーによる定義されたコマンド情報を`macrocache`に登録します。
Brackとユーザーにより定義されるコマンド定義はモジュールを隔てますが、マクロがモジュールを超えてNimの抽象構文木にアクセスするには[`std/macrocache`](https://nim-lang.org/docs/macrocache.html)という標準ライブラリで定義された`CacheSeq`というキャッシュ配列を利用する必要があります。Brackでは各Nimライブラリが定義したプロシージャのASTをキャッシュに登録することで、Brackがそれを解析してそれに合わせたプロシージャ呼び出しを生成します。

```nim:macrocacheへの登録
macro brackModule* (body: untyped): untyped =
  var stmtlist = copy(body)
  for statement in stmtlist:
    if statement.kind == nnkProcDef:
      let kind = $statement[4][0][0]
      var statement = statement
      statement[0][1] = newIdentNode(kind & "_" & resolveProcedureName($statement[4][0][1]))
      case kind
      of "square", "curly":
        mcCommandSyms.add statement
      of "angle":
        mcMacroSyms.add statement
  result = body
```

`initBrack`はBrackを利用するために`import`文の後に置く必要があるマクロです。
これにより先ほど登録した`macrocache`からプロシージャを解析して`generate`というBrackの抽象構文木からHTMLに変換するジェネレータを生成します。


```nim:ジェネレータの生成
macro initBrack* (): untyped =
  let
    generate = newIdentNode("generate")
    procedureName = newIdentNode("procedureName")
    arguments = newIdentNode("arguments")
    commandBranchAST = getCommandBranch()
  result = quote do:
    proc commandGenerator (ast: BrackNode, prefix: string): string =
      var
        `procedureName` = ""
        `arguments`: seq[string] = @[]
      for node in ast.children:
        if node.kind == bnkIdent:
          `procedureName` = prefix & resolveProcedureName(node.val)
        elif node.kind == bnkArgument:
          var argument = ""
          for argNode in node.children:
            if argNode.kind == bnkCurlyBracket:
              argument.add commandGenerator(argNode, "curly_")
            elif argNode.kind == bnkSquareBracket:
              argument.add commandGenerator(argNode, "square_")
            elif argNode.kind == bnkText:
              argument.add argNode.val
          `arguments`.add argument
      `commandBranchAST`
    
    proc squareBracketCommandGenerator (ast: BrackNode): string =
      result = commandGenerator(ast, "square_")
    
    proc curlyBracketCommandGenerator (ast: BrackNode): string =
      result = commandGenerator(ast, "curly_")

    proc paragraphCommandGenerator (ast: BrackNode): string =
      for node in ast.children:
        if node.kind == bnkText:
          result &= node.val
        elif node.kind == bnkSquareBracket:
          result &= squareBracketCommandGenerator(node)
    
    proc `generate`* (ast: BrackNode): string =
      for node in ast.children:
        if node.kind == bnkCurlyBracket:
          result &= curlyBracketCommandGenerator(node)
        elif node.kind == bnkParagraph:
          result &= "<p>" & paragraphCommandGenerator(node).replace("\n", "<br />") & "</p>"
```

これで任意[^any-cmdname]の名前のコマンドが定義でき、そのコマンドからコンパイル時にプロシージャへの呼び出しへ対応づけることができました。Brackの標準ライブラリで定義されているコマンドを使って変換してみます！

[^any-cmdname]: 実装上は

```nim:
import brack

initBrack()
let src = block:
  var f = open("tests/nest.[]")
  defer: f.close()
  f.readAll()
echo lex(src).parse().generate()
```

```html:生成されたHTML（整形は手動）
<h1>見出し1</h1>
<p>
  <span style="font-weight: bold;">
    <span style="font-style: italic;">
      イタリック
    </span>
    <span style="text-decoration: line-through;">
      打ち消し線
      <span style="font-style: italic;">
        イタリック
      </span>
    </span>
  </span>
</p>
```

できました！

## 脚注の実装
試しにブログの必須要素である脚注を実装していきます。多くのMarkdown方言では次のようにマークアップできます。

```md:脚注
こんにちは！[^1]

[^1]: 朝の挨拶
```

一方で、**インライン脚注**という自動採番によりシンプルに脚注を表現する文法を採用している方言もあります。

```md:インライン脚注
こんにちは！[^ 朝の挨拶]
```

書きやすい方がいいので、インライン脚注を実装してみましょう。
実装してみたいのはやまやまですが、できません。お気づきの方もいらっしゃると思いますが、Brackのコマンド（Square Bracktes, Curly Brackets）は引数を受け取って何らかの文字列、HTMLタグを返すシンプルな置換機構なので、脚注のようにフッターにタグを挿入するような、置換の範疇を超えた操作はできないのです。どうすれば良いでしょうか。

ここで、ここまで触れなかった山括弧（`<>`）、マクロが必要となります。マクロは、Brackの抽象構文木を受け取って新しい抽象構文木を返す仕組みのことです。パース後、マクロの展開をしてからジェネレータに構文木を渡します。
先に脚注の実装をお見せします。

```nim:脚注
proc id* (text: string, id: string): string {.square: "&".} =
  result = htmlgen.span(text, id=id)

proc footnoteSup* (text: string): string {.square: "footnoteSup".} =
  result = htmlgen.sup(text)

proc footnoteFooter* (texts: seq[string]): string {.square: "footnoteFooter".} =
  var footnoteList = ""
  for text in texts:
    footnoteList.add htmlgen.li(
      htmlgen.span(text),
      id=(&"fn-{$text}"),
      class="footnote_ordered-list"
    )
  result = htmlgen.div(
    htmlgen.div("脚注", class="footnote_header"),
    htmlgen.ol(footnoteList, class="footnote_ordered-list")
  )

proc footnote* (ast: BrackNode, id: string): BrackNode {.angle: "^".} =
  result = ast
  let
    text = ast[id][1][0].val
    n = ast.count(bnkSquareBracket, "footnoteSup")
    sup = bnkSquareBracket.newTree(
      newIdentNode("footnoteSup"),
      bnkArgument.newTree(
        bnkSquareBracket.newTree(
          newIdentNode("@"),
          bnkArgument.newTree(
            newTextNode(&"[{$n}]")
          ),
          bnkArgument.newTree(
            newTextNode(&"#fn-{text}")
          )
        )
      ),
    )
  result.insert(id, sup)
  result.delete(id)
  if not ast.exists("footnote"):
    result.children.add BrackNode(
      id: "footnote",
      kind: bnkParagraph,
      children: @[
        bnkSquareBracket.newTree(
          newIdentNode("footnoteFooter"),
        )
      ]
    )
  result["footnote"][0].add bnkArgument.newTree(
    newTextNode(text)
  )
```

長い！長いのですが、マクロを実装して日がないためユーティリティが揃っていないので仕方ありません。マクロは文書全体の`BrackNode`（AST）と`bnkAngleBracket`の`id`を受け取り、書き換え済みの文書全体の`BrackNode`を返します。
`[]`演算子がオーバーロードされていて、`id`は一意であるため`ast[id]`のように呼ぶことで対象の部分木を取り出せます。右上に表示される脚注番号（`sup`）と、存在しなければ`footnote`という`id`を持つ要素を作成して返却します。

```bash:footnote.[]
{* 夜ご飯}
今日のご飯はカレーライス<^ ハヤシライスのことを言ってる場合もある>よ！
```

```html:生成結果（整形は手動）
<h1>夜ご飯</h1>
<p>
  今日のご飯はカレーライス
  <sup>
    <a href="#fn-ハヤシライスのことを言ってる場合もある">[1]</a>
  </sup>
  よ！
</p>
<div>
  <div class="footnote_header">脚注</div>
  <ol class="footnote_ordered-list">
    <li id="fn-ハヤシライスのことを言ってる場合もある" class="footnote_ordered-list">
      <span>ハヤシライスのことを言ってる場合もある</span>
    </li>
  </ol>
</div>
```

さて、マクロはユーザーによって定義される要素なのでジェネレータと同じく`initBrack`マクロ内で解決されます。

```nim:マクロを展開する
proc angleBracketMacroExpander (ast, node: BrackNode, id: string): BrackNode =
  result = ast
  var
    `procedureName` = ""
    `id` = id
  for childNode in node.children:
    if childNode.kind == bnkIdent:
      `procedureName` = "angle_" & resolveProcedureName(childNode.val)
    elif childNode.kind == bnkArgument:
      for argNode in childNode.children:
        case argNode.kind
        of bnkAngleBracket:
          result = angleBracketMacroExpander(result, argNode, argNode.id)
        of bnkSquareBracket, bnkCurlyBracket:
          result = otherwiseMacroExpander(result, argNode, argNode.id)
        else: discard
  `macroBranchAST`

proc otherwiseMacroExpander (ast, node: BrackNode, id: string): BrackNode =
  result = ast
  for childNode in node.children:
    if childNode.kind == bnkAngleBracket:
      result = angleBracketMacroExpander(result, childNode, childNode.id)
    elif childNode.kind == bnkArgument:
      for argNode in childNode.children:
        case argNode.kind
        of bnkAngleBracket:
          result = angleBracketMacroExpander(result, argNode, argNode.id)
        of bnkSquareBracket, bnkCurlyBracket:
          result = otherwiseMacroExpander(result, argNode, argNode.id)
        else: discard

proc `expand`* (node: BrackNode): BrackNode =
  result = node
  for childNode in node.children:
    if childNode.kind == bnkAngleBracket:
      result = angleBracketMacroExpander(result, childNode, childNode.id)
    else:
      result = otherwiseMacroExpander(result, childNode, childNode.id)
```

これで任意のHTML要素をBrackのコマンドとマクロで表現できるようになりました！

# ブログジェネレータの実装
執筆のためのマークアップ言語ができたので、ここから自作ブログの話になります。

![](https://storage.googleapis.com/zenn-user-upload/6d45759b2ce6-20221030.png)

リポジトリが更新された際に記事ディレクトリのBrackファイル（`*.[]`）を変換し、記事用のテンプレートファイルに変換結果を埋め込み、Netlifyで配信しています。設定ファイルにはNimScript[^nimscript]を利用しています。

[^nimscript]: Nimのサブセットで、NimVM上で動作する。Nimのマクロ周辺の解決や、Nimbleファイルにおけるタスク定義などで利用されている。

## テンプレートファイル
ページ関連の情報を渡し、Brackの変換結果を`article`引数で埋め込んでいます。
これは[Source Code Filters](https://nim-lang.org/docs/filters.html)という仕組みを利用していて、ブログジェネレータから`nimf`ファイルを`include`して、`generateArticleHtml`プロシージャを呼ぶことで埋め込み済みの文字列情報を入手できるようにしています。

```html:article.html.nimf
#? stdtmpl(subsChar = '$', metaChar = '#')
#proc generateArticleHtml(article: string, page: Page): string =
#  result = ""
<!DOCTYPE html>
<html lang="ja">
  <head>
    <!-- 略 -->
    <title>$page.title | blog.momee.mt</title>
  </head>
  <body>
    <div id="uniArticleRoot">
      <div class="title">
        <h1>$page.title</h1>
        <p class="title_overview">$page.overview</p>
        <p class="title_date">$page.date</p>
      </div>
      <article>
        $article
      </article>
      <!-- 略 -->
    </div>
    <!-- 略 -->
  </body>
</html>
```

## 設定ファイルの解釈
設定ファイルにはNimScriptを利用しています。

```nim
let
  title* = "テスト記事"
  overview* = "タルタルソースはギリギリ食べられる"
  tags* = @["ブログ"]
  thumbnail* = 2
  publish* = true
```

Nimと同じ文法が利用できるので（個人的には）書きやすいです。
これはブログジェネレータ内で`compiler`というNimコンパイラが提供するAPIを利用してインタプリタを構築して解釈しています。現在はあるキーに対して値を設定することしかしていないですが、Nimの型システムやマクロがそのまま使えるほか、`createInterpreter`時に任意のモジュールを差し込むことができるのでより高度な文脈が記述できる設定ファイルを作ることも可能です。

```nim
import compiler/nimeval
import compiler/renderer
import compiler/ast

proc getInterpreter* (path: string): Interpreter =
  when defined(macosx):
    let stdlib = "/opt/homebrew/Cellar/nim/1.6.6/nim/lib/"
  else:
    let stdlib = "/nim/lib"
  result = createInterpreter(path, [stdlib, stdlib / "pure", stdlib / "core"])
  result.evalScript()

proc destroy* (intr: Interpreter) =
  intr.destroyInterpreter()

proc getString* (intr: Interpreter, name: string): string =
  result = intr.getGlobalValue(
    intr.selectUniqueSymbol(name)
  ).strVal

for (dayInDir, year, month, day) in dateInDir("../articles"):
  for dir in walkDir(dayInDir.path):
    let
      name = dir.path.split('/')[^1]
      intr = getInterpreter(dir.path & "/setting.nims")
      title = intr.getString("title")
      overview = intr.getString("overview")
      thumbnail = intr.getInt("thumbnail")
      tags = intr.getSeqString("tags")
      publish = intr.getBool("publish")
```

### Dockerコンテナを作成する
ところで`createInterpreter`する際に標準ライブラリのパスを指定していますが、これはOSや同じmacOSでもIntelかApple Siliconかでパスが変動します。そこで、Dockerコンテナを作成しています。特別なことはしていなくて、Nimが管理しているコンテナを引っ張ってきて、タイムゾーンを設定しているだけです。一応Dev Containerの設定ファイルも書きましたがコンパイラを引っ張ってくるのに時間がかかるので普段はローカルで開発しています。

```Dockerfile:Dockerfile
FROM nimlang/nim:latest
ENV TZ Asia/Tokyo
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
RUN apt-get update && apt install -y tzdata && apt-get clean && rm -rf /var/lib/apt/lists/*
```

```yml:docker-compose.yml
version: '3.8'

services:
  blog.momee.mt:
    build:
      context: .
      dockerfile: docker/nim/Dockerfile
    tty: true
    working_dir: /workspace
    volumes:
      - .:/workspace
```

# デプロイ
GitHub Actionsでリポジトリが更新された際に、docker-composeでブログをビルドして`build`というブランチに結果をpushするようなワークフローを記述しています。ビルド結果はNetlifyによって配信されています。

```yml:deploy.yml
name: Deploy

on:
  push:
    branches: ["main"]

  workflow_dispatch:

permissions:
  contents: write

jobs:  
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: satackey/action-docker-layer-caching@v0.0.11
        continue-on-error: true
      - run: docker-compose build
      - run: docker-compose up -d
      - run: docker-compose exec -T blog.momee.mt nimble generate
      - name: Push
        uses: s0/git-publish-subdir-action@develop
        env:
          REPO: self
          BRANCH: build
          FOLDER: dist
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          MESSAGE: "Build: ({sha}) {msg}"
```

# おわりに
自作マークアップ言語による自作ブログを作りました。
自作ブログは園芸[^dotfiles]のように少しずつ手入れして良いブログにしていくことが楽しみでもあります。現在は毎日日報用のブランチを切って定時になったらオートマージするワークフローを書いたり、テンプレートファイルがあまりに辛いのでBrackの変換先をJSONを追加して、JavaScriptバックエンドで仮想DOMを自作したりをやりたいなと思っていて、今年1年でちょっとずつ進めていきたいなと思います。
もし自作マークアップ言語 on 自作ブログに関心があればリポジトリにスターをいただけると励みになります。

https://github.com/brack-lang/transpiler

https://github.com/momeemt/blog.momee.mt

[^dotfiles]: や、dotfiles。dotfiles管理したい。まだしてない