---
title: "[和訳]Go Code Review Comments"
emoji: "👀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go, review]
published: true
---

この記事は、[Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)の翻訳記事です。
原典の情報に加えて、部分的に補足説明を追加しています。
文の順序や構造は自然な日本語になるように調整していますが、段落は原典と対応するようにしています。

日本語にすることが適切でない単語や表現については、`このように`インラインコードで示しています(通常のコード片も同様に表示)。

:::message
翻訳上の意味、日本語特有の情報や、内容に関する補足はこのように注釈を表示します。
:::

# 著作権表示

Go言語のソースとドキュメントは、修正BSDライセンスで提供されています。
Go Code Review Comments, およびGo wikiに直接表示があるわけではありませんが、Go wikiのドメインである`go.dev`の[ライセンスページ](https://go.dev/LICENSE)に同様のライセンス表示を確認できるため、以下に記載します。
本記事は、基となる記事[Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)に、修正BSDライセンスで許可されている改変を加えたものです。

```plaintext
Copyright (c) 2009 The Go Authors. All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are
met:

   * Redistributions of source code must retain the above copyright
notice, this list of conditions and the following disclaimer.
   * Redistributions in binary form must reproduce the above
copyright notice, this list of conditions and the following disclaimer
in the documentation and/or other materials provided with the
distribution.
   * Neither the name of Google Inc. nor the names of its
contributors may be used to endorse or promote products derived from
this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```

# Go Wiki: Go Code Review Comments

<!-- This page collects common comments made during reviews of Go code, so that a single detailed explanation can be referred to by shorthands. This is a laundry list of common style issues, not a comprehensive style guide. -->

この記事は、Go言語でよくあるコードレビューコメントをまとめたものです。
実際にレビューで指摘する際に、各項目のリンクを提示することで詳細な説明を参照できるようにしています。
典型的な問題に関するリストであり、網羅的なスタイルガイドではないことに注意してください。

<!-- You can view this as a supplement to Effective Go. -->

[Effective Go](https://go.dev/doc/effective_go)の補足として利用できます。

<!-- Additional comments related to testing can be found at Go Test Comments -->

テストに関するコメントは、[Go Test Comments](https://go.dev/wiki/TestComments)で確認できます。

<!-- Google has published a longer Go Style Guide. -->

コードスタイルに関しては、Googleが公開している[Go Style Guide](https://google.github.io/styleguide/go/decisions)も参考になります。

<!-- Please discuss changes before editing this page, even minor ones. Many people have opinions and this is not the place for edit wars. -->

*軽微なものであっても、このページを[編集する前に必ず議論](https://github.com/golang/go/issues/new?title=wiki%3A+CodeReviewComments+change&body=&labels=Documentation)してください。*
多くの人が意見を持っています。
このページは編集戦争の場所ではありません。

## Gofmt

<!-- Run gofmt on your code to automatically fix the majority of mechanical style issues. Almost all Go code in the wild uses gofmt. The rest of this document addresses non-mechanical style points. -->

[`gofmt`](https://pkg.go.dev/cmd/gofmt)を実行してください。
これにより、スタイルに関する問題のほとんどが自動的に修正されます。
世にあるGoコードのほとんどは`gofmt`を使用しています。
このページでは以降、機械的に修正できないスタイルに絞って説明します。

<!-- An alternative is to use goimports, a superset of gofmt which additionally adds (and removes) import lines as necessary. -->

`gofmt`の代替として、`goimports`があります。
`goimports`は`gofmt`の上位互換のツールであり、必要に応じてインポート行を追加(または削除)します。

## Comment Sentences

<!-- See https://go.dev/doc/effective_go#commentary. Comments documenting declarations should be full sentences, even if that seems a little redundant. This approach makes them format well when extracted into godoc documentation. Comments should begin with the name of the thing being described and end in a period: -->

<https://go.dev/doc/effective_go#commentary>も参照してください。
多少冗長に思えても、ドキュメント化されるコメントは完全な文にしてください。
これにより、godocで抽出されたときに適切にフォーマットされます。
コメントは下記の例のように、説明する対象の名前で始まり、ピリオドで終わるべきです。

```golang
// Request represents a request to run a command.
type Request struct { ...

// Encode writes the JSON encoding of req to w.
func Encode(w io.Writer, req *Request) { ...
```

:::message
日本語コメントを採用している場合でも、godocやエディタの補完を考慮して「説明する対象の名前で始まり、ピリオドで終わる」に従うべきです。
文頭の説明対象の名称の後には、半角スペース`" "`を入れてください。
ピリオドは句点(。)に置き換えても良いです。

```golang
// Request はコマンド実行のためのリクエストを表します。
type Request struct { ...

// Encode はwにreqのJSONエンコーディングを書き込みます。
func Encode(w io.Writer, req *Request) { ...
```

:::

## Contexts

<!-- Values of the context.Context type carry security credentials, tracing information, deadlines, and cancellation signals across API and process boundaries. Go programs pass Contexts explicitly along the entire function call chain from incoming RPCs and HTTP requests to outgoing requests. -->

`context.Context`型の値は、セキュリティクレデンシャル、トレース情報、デッドライン、キャンセルシグナルをAPIやプロセスの境界を越えて運びます。
Goのプログラムでは、入力RPC・HTTPリクエストから外部へのリクエスト発行まで、関数呼び出しチェイン全体で`Context`を明示的に渡します。

<!-- Most functions that use a Context should accept it as their first parameter: -->

大抵の関数では、`Context`を第1引数として受け取るべきです。

```golang
func F(ctx context.Context, /* その他の引数 */) {}
```

<!-- A function that is never request-specific may use context.Background(), but err on the side of passing a Context even if you think you don’t need to. The default case is to pass a Context; only use context.Background() directly if you have a good reason why the alternative is a mistake. -->

関数の処理が絶対に`request-specific`でない場合は`context.Background()`を使用できますが、必要がないと思っても`Context`を渡す方が無難です。
デフォルトでは`Context`を渡すようにしておき、`context.Background()`を直接使用するのは、代替手段が誤りであるという理由がある場合に限ります。

:::message
`request-specific`とは、HTTPやRPCなどのリクエストの文脈に依存する処理を意味します。
たとえば、HTTPリクエストの受信に紐づいて同時に実行する関数は、そのリクエストがキャンセルされたり、タイムアウトしたりすることで一緒に終了する必要があります。
これはタイミングや状態など、個別のリクエストの状況(文脈)に依存する処理です。
:::

<!-- Don’t add a Context member to a struct type; instead add a ctx parameter to each method on that type that needs to pass it along. The one exception is for methods whose signature must match an interface in the standard library or in a third party library. -->

構造体のメンバーに`Context`を追加しないでください。
代わりに、それを渡す必要がある各メソッドに`ctx`引数を追加してください。
例外は、標準・サードパーティライブラリのインタフェースにシグネチャを合わせる必要があるメソッドです。

<!-- Don’t create custom Context types or use interfaces other than Context in function signatures. -->

独自の`Context`型を作成したり、`Context`以外のインタフェースを関数シグネチャで使用しないでください。

<!-- If you have application data to pass around, put it in a parameter, in the receiver, in globals, or, if it truly belongs there, in a Context value. -->

アプリケーションデータを引き回す場合、基本的には引数、レシーバ、グローバル変数に格納することを検討し、本当に`Context`に属する場合にのみ`Context`に格納してください。

<!-- Contexts are immutable, so it’s fine to pass the same ctx to multiple calls that share the same deadline, cancellation signal, credentials, parent trace, etc. -->

`Context`はイミュータブル(不変)なので、同じデッドライン、キャンセルシグナル、クレデンシャル、親トレースなどを共有する複数の呼び出しに同じ`ctx`変数を渡すことは問題ありません。

## Copying

<!-- To avoid unexpected aliasing, be careful when copying a struct from another package. For example, the bytes.Buffer type contains a []byte slice. If you copy a Buffer, the slice in the copy may alias the array in the original, causing subsequent method calls to have surprising effects. -->

予期しない`aliasing`を避けるために、他のパッケージから構造体をコピーする際には注意してください。
たとえば、`bytes.Buffer`型には`[]byte`スライスが含まれています。
このとき単に`Buffer`をコピーすると、コピーしたスライスは元のスライスと同じ配列を参照する可能性があります。
これにより、後続のメソッド呼び出しに予期しない副作用が生じるかもしれません。

:::message
`aliasing`とは、同じメモリ領域を指す変数が複数作られることです。
:::

<!-- In general, do not copy a value of type T if its methods are associated with the pointer type, *T. -->

一般的に、ポインタ型`*T`に関連付けられたメソッドを持つ型`T`の値をコピーしないでください。

## Crypto Rand

<!-- Do not use package math/rand to generate keys, even throwaway ones. Unseeded, the generator is completely predictable. Seeded with time.Nanoseconds(), there are just a few bits of entropy. Instead, use crypto/rand’s Reader, and if you need text, print to hexadecimal or base64: -->

使い捨てであっても、`math/rand`パッケージを使用して鍵を生成しないでください。
シードが設定されていない場合、生成器は完全に予測可能です。
`time.Nanoseconds()`でシードを設定した場合でも、エントロピーはわずか数ビットしかありません。
代わりに`crypto/rand`の`Reader`を使用し、テキストが必要な場合は16進数やbase64に変換してください。

```golang
import (
    "crypto/rand"
    // "encoding/base64"
    // "encoding/hex"
    "fmt"
)

func Key() string {
    buf := make([]byte, 16)
    _, err := rand.Read(buf)
    if err != nil {
        panic(err)  // out of randomness, should never happen
    }
    return fmt.Sprintf("%x", buf)
    // or hex.EncodeToString(buf)
    // or base64.StdEncoding.EncodeToString(buf)
}
```

## Declaring Empty Slices

<!-- When declaring an empty slice, prefer -->

空のスライスを宣言する場合、以下のように宣言することを推奨します。

```golang
var t []string
```

<!-- over -->

以下のように宣言するのは避けてください。

```golang
t := []string{}
```

<!-- The former declares a nil slice value, while the latter is non-nil but zero-length. They are functionally equivalent—their len and cap are both zero—but the nil slice is the preferred style. -->

前者は値が`nil`のスライスを宣言し、後者は`nil`ではないが長さがゼロのスライスを宣言します。
機能的には等価(`len()`と`cap()`がどちらもゼロ)ですが、値が`nil`のスライスの方が好ましいスタイルです。

<!-- Note that there are limited circumstances where a non-nil but zero-length slice is preferred, such as when encoding JSON objects (a nil slice encodes to null, while []string{} encodes to the JSON array []). -->

特定の状況では、長さゼロのスライス(`[]string{}`)が適切な場合もあります。
たとえば、JSONオブジェクトをする場合、`nil`スライスは`null`にエンコードされますが、`[]string{}`はJSON配列`[]`にエンコードされます。

<!-- When designing interfaces, avoid making a distinction between a nil slice and a non-nil, zero-length slice, as this can lead to subtle programming errors. -->

インタフェースを設計する際、`nil`スライスと`nil`ではない長さゼロのスライスの区別しないようにしてください。

<!-- For more discussion about nil in Go see Francesc Campoy’s talk Understanding Nil. -->

Goの`nil`に関する詳細な議論については、Francesc Campoyの[Understanding Nil](https://www.youtube.com/watch?v=ynoY2xz-F8s)を参照してください。

## Doc Comments

<!-- All top-level, exported names should have doc comments, as should non-trivial unexported type or function declarations. See https://go.dev/doc/effective_go#commentary for more information about commentary conventions. -->

トップレベルのエクスポートされる名前には、すべてにドキュメントコメントを付けるべきです。
また、エクスポートされていない型や関数宣言でも、重要なものにはドキュメントコメントを付けるべきです。
コメントの慣習についての詳細は<https://go.dev/doc/effective_go#commentary>を参照してください。

## Don’t Panic

<!-- See https://go.dev/doc/effective_go#errors. Don’t use panic for normal error handling. Use error and multiple return values. -->

<https://go.dev/doc/effective_go#errors>も参照してください。
通常のエラーハンドリングに`panic`を使用しないでください。
`error`を含む複数の戻り値を使用してください。

## Error Strings

<!-- Error strings should not be capitalized (unless beginning with proper nouns or acronyms) or end with punctuation, since they are usually printed following other context. That is, use fmt.Errorf("something bad") not fmt.Errorf("Something bad"), so that log.Printf("Reading %s: %v", filename, err) formats without a spurious capital letter mid-message. This does not apply to logging, which is implicitly line-oriented and not combined inside other messages. -->

後続のコンテキストに続いて表示されることが多いため、エラー文字列は大文字で始めたり、句読点で終わらせたりすべきではありません(固有名詞や略語で始まる場合を除く)。
つまり、`fmt.Errorf("something bad")`を使用し、`fmt.Errorf("Something bad")`を使用しないでください。
そうすると、`log.Printf("Reading %s: %v", filename, err)`がメッセージ中に余分な大文字が含まれないようにフォーマットされます。
ログ出力は暗黙に行志向であり、他のメッセージ内に組み込まれることはないため、このルールは適用されません。

## Examples

<!-- When adding a new package, include examples of intended usage: a runnable Example, or a simple test demonstrating a complete call sequence. -->

新しいパッケージを追加する際には、実行可能な`Example`や完全な呼び出し手順を示す単純なテストを書くことで、意図した使用方法の例を含めてください。

<!-- Read more about testable Example() functions. -->

こちらも参考にしてください: [testable Example() functions](https://go.dev/blog/examples).

## Goroutine Lifetimes

<!-- When you spawn goroutines, make it clear when - or whether - they exit. -->

`groutine`を生成する際には、それがいつ終了するのか、またそもそも終了するのかを明確にしてください。

<!-- Goroutines can leak by blocking on channel sends or receives: the garbage collector will not terminate a goroutine even if the channels it is blocked on are unreachable. -->

`goroutine`は、チャネルの送受信でブロックされることによってリークする可能性があります。
これは、GCがブロックされたチャネルが到達不能であっても`goroutine`を終了させないためです。

<!-- Even when goroutines do not leak, leaving them in-flight when they are no longer needed can cause other subtle and hard-to-diagnose problems. Sends on closed channels panic. Modifying still-in-use inputs “after the result isn’t needed” can still lead to data races. And leaving goroutines in-flight for arbitrarily long can lead to unpredictable memory usage. -->

不要になった`groutine`をそのままにしておくと、`goroutine`がリークしなかったとしても次のような微妙で原因究明が難しい問題を引き起こす可能性があります。
`Close`されたチャネルに対する`Send`は`panic`を引き起こします。
結果が不要になった後でも、使用中の入力を変更することでデータ競合を引き起こすことがあります。
そして、`goroutine`が任意の時間実行したままになっていると、メモリ使用量が予測不能になる可能性があります。

<!-- Try to keep concurrent code simple enough that goroutine lifetimes are obvious. If that just isn’t feasible, document when and why the goroutines exit. -->

`goroutine`のライフタイムを明確にするため、並行コードをできるだけシンプルに保つようにしてください。
それができない場合は、`goroutine`がいつ、なぜ終了するのかをドキュメント化してください。

## Handle Errors

<!-- See https://go.dev/doc/effective_go#errors. Do not discard errors using _ variables. If a function returns an error, check it to make sure the function succeeded. Handle the error, return it, or, in truly exceptional situations, panic. -->

<https://go.dev/doc/effective_go#errors>も参照してください。
`_`を使用してエラーを破棄しないでください。
関数がエラーを返す場合は、関数が成功したことを確認するためにエラーをチェックしてください。
エラーに対処するか、エラーを返すか、本当に例外的な状況であれば`panic`してください。

## Imports

<!-- Avoid renaming imports except to avoid a name collision; good package names should not require renaming. In the event of collision, prefer to rename the most local or project-specific import. -->

名前の衝突を避ける目的以外で、インポート名を変更しないでください。
適切なパッケージ名であれば、インポート名を変更する必要はないはずです。
衝突時には、よりローカルでプロジェクト固有な方のインポート名の変更を推奨します。

<!-- Imports are organized in groups, with blank lines between them. The standard library packages are always in the first group. -->

インポートはグループごとに整理され、間に空行が入ります。
標準ライブラリのパッケージは常に最初のグループに配置します。

```golang
package main

import (
    "fmt"
    "hash/adler32"
    "os"

    "github.com/foo/bar"
    "rsc.io/goversion/version"
)
```

<!-- goimports will do this for you. -->

これは[`goimports`](https://pkg.go.dev/golang.org/x/tools/cmd/goimports)が自動で行ってくれます。

## Import Blank

<!-- Packages that are imported only for their side effects (using the syntax import _ "pkg") should only be imported in the main package of a program, or in tests that require them. -->

副作用のためだけにインポートされるパッケージ(`import _ "pkg"`を使用)は、プログラムの`main`パッケージか、それを必要とするテストでのみインポートすべきです。

## Import Dot

<!-- The import . form can be useful in tests that, due to circular dependencies, cannot be made part of the package being tested: -->

`import .`形式は、循環依存によってテスト対象パッケージの一部にできないテストで便利です。

```golang
package foo_test

import (
    "bar/testutil" // このパッケージもfooをインポートしている
    . "foo"
)
```

<!-- In this case, the test file cannot be in package foo because it uses bar/testutil, which imports foo. So we use the ‘import .’ form to let the file pretend to be part of package foo even though it is not. Except for this one case, do not use import . in your programs. It makes the programs much harder to read because it is unclear whether a name like Quux is a top-level identifier in the current package or in an imported package. -->

この例では、`foo`をインポートしている`bar/testutil`をインポートしているため、テストファイルを`foo`パッケージに配置できません。
ここで、`import .`形式を使用するとテストファイルが`foo`パッケージの一部であるかのように振る舞わせることができます。
この1つのケースを除いて、`import .`を使用しないでください。
`import .`によって`Quux`のような名前がそのパッケージのトップレベル識別子なのか、インポートされたパッケージの識別子なのかが不明確になり、可読性が低下します。

:::message
`Quux`というのは例示用変数名で、英語版`Hoge`です。
:::

## In-Band Errors

:::message
タイトル中の`In-Band`は、「帯域内の」という意味です。
信号処理の文脈で、エラー情報や管理情報が通常のデータと同じ帯域を使って伝送されることを指します。
この記事の場合、正常系の結果と同じ経路(変数)によってエラー情報が返されることを指しています。
<https://e-words.jp/w/%E3%82%A4%E3%83%B3%E3%83%90%E3%83%B3%E3%83%89%E7%AE%A1%E7%90%86.html>
:::

<!-- In C and similar languages, it’s common for functions to return values like -1 or null to signal errors or missing results: -->

C言語などの言語では、エラーや結果が見つからないことを示すために、`-1`や`null`のような値を返すことが一般的です。

```golang
// Lookup はKeyに対応する値を返し、見つからない場合は""を返す。
func Lookup(key string) string

// in-bandなエラーをチェックしないことで以下のようなバグにつながる。
Parse(Lookup(key))  // "keyに対応する値がありません"を返すべきところ、"パースに失敗しました"というエラーになる
```

<!-- Go’s support for multiple return values provides a better solution. Instead of requiring clients to check for an in-band error value, a function should return an additional value to indicate whether its other return values are valid. This return value may be an error, or a boolean when no explanation is needed. It should be the final return value. -->

Goは複数戻り値をサポートしているので、これが良い解決になります。
呼び出し側に`in-band`なエラーをチェックさせる代わりに、関数は他の戻り値が有効かどうかを示すための追加の値を返すべきです。
この返り値は`error`か、説明が不要な場合は`bool`であるべきです。
また、`error`や`bool`は最後の戻り値であるべきです。

```golang
// Lookup はKeyに対応する値を返し、見つからない場合はok=falseを返す。
func Lookup(key string) (value string, ok bool)
```

<!-- This prevents the caller from using the result incorrectly: -->

これで、呼び出し元が結果を不正に使用することを防ぐことができます。

```golang
Parse(Lookup(key))  // コンパイル時エラー
```

<!-- And encourages more robust and readable code: -->

そして、より堅牢で読みやすいコードになります。

```golang
value, ok := Lookup(key)
if !ok {
    return fmt.Errorf("no value for %q", key)
}
return Parse(value)
```

<!-- This rule applies to exported functions but is also useful for unexported functions. -->

このルールはエクスポートされた関数に適用されますが、そうでない関数にも有用です。

<!-- Return values like nil, “”, 0, and -1 are fine when they are valid results for a function, that is, when the caller need not handle them differently from other values. -->

`nil`、`""`、`0`、`-1`などの返り値は、関数の有効な結果である場合には問題ありません。
つまり、呼び出した場所で他の値と異なる処理を行う必要がない場合です。

<!-- Some standard library functions, like those in package “strings”, return in-band error values. This greatly simplifies string-manipulation code at the cost of requiring more diligence from the programmer. In general, Go code should return additional values for errors. -->

`strings`パッケージのような標準ライブラリの一部の関数は、`in-band`なエラー値を返します。
これは、プログラマにより多くの注意を要求する代わりに、文字列操作コードを大幅に簡素化します。
一般的には、Goのコードはエラーのために追加の値を返すべきです。

## Indent Error Flow

<!-- Try to keep the normal code path at a minimal indentation, and indent the error handling, dealing with it first. This improves the readability of the code by permitting visually scanning the normal path quickly. For instance, don’t write: -->

通常のコードパスを最小限のインデントに保ち、エラーハンドリングを先に実行して、そこにインデントを付けるようにしましょう。
これによって、正常系のコードパスを素早く流し読みできるため、コードの可読性が向上します。

```golang
if err != nil {
    // エラー処理
} else {
    // 正常系のコード
}
```

<!-- Instead, write: -->

上の例は、以下のように書き換えてください。

```golang
if err != nil {
    // エラー処理
    return // または`continue`など
}
// 正常系のコード
```

<!-- If the if statement has an initialization statement, such as: -->

以下のように、`if`文に初期化ステートメントがある場合は、

```golang
if x, err := f(); err != nil {
    // エラー処理
    return
} else {
    // use x
}
```

<!-- then this may require moving the short variable declaration to its own line: -->

変数宣言を独立した行に移動する必要があるかもしれません。

```golang
x, err := f()
if err != nil {
    // エラー処理
    return
}
// use x
```

## Initialisms

<!-- Words in names that are initialisms or acronyms (e.g. “URL” or “NATO”) have a consistent case. For example, “URL” should appear as “URL” or “url” (as in “urlPony”, or “URLPony”), never as “Url”. As an example: ServeHTTP not ServeHttp. For identifiers with multiple initialized “words”, use for example “xmlHTTPRequest” or “XMLHTTPRequest”. -->

頭文字語や略語(例:`URL`や`NATO`)を含む単語は一貫したケースを持つべきです。
たとえば、`URL`は`URL`や`url`として表示されるべきで(`urlPony`や`URLPony`)、`Url`にはしないでください。
例として、`serve http`は`ServeHttp`ではなく`ServeHTTP`とします。
複数の単語を持つ識別子の場合は、`xmlHTTPRequest`や`XMLHTTPRequest`のようにします。

<!-- This rule also applies to “ID” when it is short for “identifier” (which is pretty much all cases when it’s not the “id” as in “ego”, “superego”), so write “appID” instead of “appId”. -->

このルールは、`ID`が`identifier`の略語である場合にも適用されます(`id`が`ego`や`superego`の並びで使われる場合を除いてほとんど全ての場合で)。
そのため、`appId`ではなく`appID`としてください。

:::message
「`id`が`ego`や`superego`の並びで〜」の部分について、それぞれ心理学用語で「イド」「エゴ(自我)」「スーパーエゴ(超自我)」という意味です。
つまり、`id`が一般名詞として使われる場合を示しています。
:::

<!-- Code generated by the protocol buffer compiler is exempt from this rule. Human-written code is held to a higher standard than machine-written code. -->

`protocol buffer`コンパイラによって生成されたコードは、このルールの対象外です。
人間が書いたコードには、機械が生成したコードよりも高い水準が求められます。

## Interfaces

<!-- Go interfaces generally belong in the package that uses values of the interface type, not the package that implements those values. The implementing package should return concrete (usually pointer or struct) types: that way, new methods can be added to implementations without requiring extensive refactoring. -->

Goの`interface`は通常、その`interface`型を満たす値を実装するパッケージではなく、使うパッケージに属します。
`interface`を実装するパッケージは具象型(通常はポインタや構造体)を返すべきです。
これにより、実装に新しいメソッドを追加する際に大規模なリファクタリングを必要とせずに済みます。

<!-- Do not define interfaces on the implementor side of an API “for mocking”; instead, design the API so that it can be tested using the public API of the real implementation. -->

「モックのために」APIの実装側で`interface`を定義しないでください。
代わりに、実際の実装の公開APIを使用してテストできるようにAPIを設計してください。

<!-- Do not define interfaces before they are used: without a realistic example of usage, it is too difficult to see whether an interface is even necessary, let alone what methods it ought to contain. -->

使用する前に`interface`を定義しないでください。
実際の使用例がないと、`interface`が必要かどうかさえわかりにくく、どのメソッドを含めるべきかもわかりません。

```golang
package consumer  // consumer.go

type Thinger interface { Thing() bool }

func Foo(t Thinger) string { … }
```

```golang
package consumer // consumer_test.go

type fakeThinger struct{ … }
func (t fakeThinger) Thing() bool { … }
…
if Foo(fakeThinger{…}) == "x" { … }
```

```golang
// このように実装してはいけません！！！
package producer

type Thinger interface { Thing() bool }

type defaultThinger struct{ … }
func (t defaultThinger) Thing() bool { … }

func NewThinger() Thinger { return defaultThinger{ … } }
```

<!-- Instead return a concrete type and let the consumer mock the producer implementation. -->

代わりに具象型を返し、`consumer`パッケージが`producer`の実装をモックするようにしてください。

```golang
package producer

type Thinger struct{ … }
func (t Thinger) Thing() bool { … }

func NewThinger() Thinger { return Thinger{ … } }
```

## Line Length

<!-- There is no rigid line length limit in Go code, but avoid uncomfortably long lines. Similarly, don’t add line breaks to keep lines short when they are more readable long–for example, if they are repetitive. -->

Goコードには厳格な行長制限はありませんが、不快なほど長すぎる行は避けてください。
同様に、読みやすい場合には長いままにしておき、改行入れるべきではありません。
たとえば、同じ処理が繰り返される場合などです。

<!-- Most of the time when people wrap lines “unnaturally” (in the middle of function calls or function declarations, more or less, say, though some exceptions are around), the wrapping would be unnecessary if they had a reasonable number of parameters and reasonably short variable names. Long lines seem to go with long names, and getting rid of the long names helps a lot. -->

「不自然な」改行が多い場合(例外はありますが、関数呼び出しや関数宣言の途中など)、適切な数の引数と適切に短い変数名があればそれらは不要なはずです。
長い行は長い名前とセットになることが多いため、名前を短くすることでかなり改善されます。

<!-- In other words, break lines because of the semantics of what you’re writing (as a general rule) and not because of the length of the line. If you find that this produces lines that are too long, then change the names or the semantics and you’ll probably get a good result. -->

言い換えると、(一般的なルールとして)書いている内容のセマンティクス(意味的な内容)に基づいて改行するようにし、長さそのものを理由に改行しないでください。
行が長すぎると感じた場合は、名前やセマンティクスを変更してみることで、良い結果が得られるでしょう。

<!-- This is, actually, exactly the same advice about how long a function should be. There’s no rule “never have a function more than N lines long”, but there is definitely such a thing as too long of a function, and of too repetitive tiny functions, and the solution is to change where the function boundaries are, not to start counting lines. -->

実は、関数がどれくらい長くなるべきかについても同じことが言えます。
「N行を超える関数を書いてはいけない」というルールはありませんが、確かに長すぎる関数や、小さすぎて繰り返しの多い関数があると問題になります。
そして、その解決策は関数の境界を変更することであり、行数を数えることではありません。

## Mixed Caps

<!-- See https://go.dev/doc/effective_go#mixed-caps. This applies even when it breaks conventions in other languages. For example an unexported constant is maxLength not MaxLength or MAX_LENGTH. -->

<https://go.dev/doc/effective_go#mixed-caps>も参照してください。
これは、他の言語の慣習に反する場合でも適用されます。
たとえば、エクスポートされていない定数は`maxLength`であり、`MaxLength`や`MAX_LENGTH`ではありません。

<!-- Also see Initialisms. -->

また、[Initialisms](#initialisms)も参照してください。

## Named Result Parameters

:::message
文中`Named Result Parameters`は「名前付き戻り値」と訳しています。
:::

<!-- Consider what it will look like in godoc. Named result parameters like: -->

`godoc`でどのように見えるかを意識してください。
以下のような名前付き戻り値は、

```golang
func (n *Node) Parent1() (node *Node) {}
func (n *Node) Parent2() (node *Node, err error) {}
```

<!-- will be repetitive in godoc; better to use: -->

`godoc`では繰り返しになります。
以下のようにすると良いでしょう。

```golang
func (n *Node) Parent1() *Node {}
func (n *Node) Parent2() (*Node, error) {}
```

<!-- On the other hand, if a function returns two or three parameters of the same type, or if the meaning of a result isn’t clear from context, adding names may be useful in some contexts. Don’t name result parameters just to avoid declaring a var inside the function; that trades off a minor implementation brevity at the cost of unnecessary API verbosity. -->

一方で、関数が同じ型の値を2つまたは3つ返す場合や、結果の意味がコンテキストから明確でない場合は、名前付き戻り値が有用な場合もあります。
しかし、ただ関数内で変数を宣言するのを避けるためだけに使うべきではありません。
(これは、実装を少しだけ簡潔にする代わりに、APIが不必要に冗長になります。)

```golang
func (f *Foo) Location() (float64, float64, error)
```

<!-- is less clear than: -->

上記は以下の例よりもわかりにくいです。

```golang
// Location はfの緯度と経度を返します。
// 負の値はそれぞれ南と西を意味します。
func (f *Foo) Location() (lat, long float64, err error)
```

<!-- Naked returns are okay if the function is a handful of lines. Once it’s a medium sized function, be explicit with your return values. Corollary: it’s not worth it to name result parameters just because it enables you to use naked returns. Clarity of docs is always more important than saving a line or two in your function. -->

関数が数行程度の場合は、`naked return`で問題ありませんが、中規模以上の関数になったら、戻り値の名前を明示的にするべきです。
逆に、`naked return`を使うためだけに名前付き戻り値を使う価値はありません。
ドキュメントの明確さは、関数内の1、2行を節約するよりも重要です。

:::message
`naked return`は、`return`文に引数を指定しない形式のことです。
[Naked Returns](#naked-returns)の節も参照してください。
:::

<!-- Finally, in some cases you need to name a result parameter in order to change it in a deferred closure. That is always OK. -->

最後に、`defer`して使うクロージャ内で変更するために名前付き戻り値が必要な場合もあります。
これは常に問題ありません。

## Naked Returns

<!-- A return statement without arguments returns the named return values. This is known as a “naked” return. -->

引数なしの`return`文は、自動的に名前付き戻り値を返します。
これは`naked return`として知られています。

```golang
func split(sum int) (x, y int) {
    x = sum * 4 / 9
    y = sum - x
    return
}
```

<!-- See Named Result Parameters. -->

[Named Result Parameters](#named-result-parameters)も参照してください。

## Package Comments

<!-- Package comments, like all comments to be presented by godoc, must appear adjacent to the package clause, with no blank line. -->

`godoc`で表示されるすべてのコメントと同様に、パッケージコメントは`package`節に空行を挟まず隣接する必要があります。

```golang
// math パッケージは基本的な定数と数学的関数を提供します。
package math
/*
template パッケージは、HTMLなどのテキスト出力を生成するためのデータ駆動のテンプレートを実装します。
....
*/
package template
```

<!-- For “package main” comments, other styles of comment are fine after the binary name (and it may be capitalized if it comes first), For example, for a package main in the directory seedgen you could write: -->

`package main`のコメントについては、バイナリ名の後に他のスタイルのコメントを使用しても問題ありません(最初に来る場合は大文字にできます)。
たとえば、`seedgen`ディレクトリ内の`package main`の場合、以下のように書くことができます。

```golang
// Binary seedgen ...
package main
```

or

```golang
// Command seedgen ...
package main
```

or

```golang
// Program seedgen ...
package main
```

or

```golang
// The seedgen command ...
package main
```

or

```golang
// The seedgen program ...
package main
```

or

```golang
// Seedgen ..
package main
```

<!-- These are examples, and sensible variants of these are acceptable. -->

これらの例や、これらの合理的なバリエーションは受け入れられます。

<!-- Note that starting the sentence with a lower-case word is not among the acceptable options for package comments, as these are publicly-visible and should be written in proper English, including capitalizing the first word of the sentence. When the binary name is the first word, capitalizing it is required even though it does not strictly match the spelling of the command-line invocation. -->

パッケージコメントでは、文の最初の単語を小文字で始めることは許可されていません。
これらは公開されるため、文の最初が大文字である正しい英語で書かれるべきです。
バイナリ名が最初の単語の場合、コマンドラインで呼び出すときのスペルと厳密に一致しない場合でも、大文字で始める必要があります。

<!-- See https://go.dev/doc/effective_go#commentary for more information about commentary conventions. -->

コメントの慣習についての詳細は<https://go.dev/doc/effective_go#commentary>を参照してください。

## Package Names

<!-- All references to names in your package will be done using the package name, so you can omit that name from the identifiers. For example, if you are in package chubby, you don’t need type ChubbyFile, which clients will write as chubby.ChubbyFile. Instead, name the type File, which clients will write as chubby.File. Avoid meaningless package names like util, common, misc, api, types, and interfaces. See https://go.dev/doc/effective_go#package-names and https://go.dev/blog/package-names for more. -->

あるパッケージ内の名前への参照は、必ずそのパッケージ名を付けて行われるため、識別子からその名前を省略できます。
たとえば、`chubby`パッケージ内にいる場合、`chubby.ChubbyFile`と呼び出されるため`ChubbyFile`という型名は冗長です。
代わりに、`chubby.File`と呼び出せるように`File`という名前を付けてください。
`util`、`common`、`misc`、`api`、`types`、`interfaces`のような意味のないパッケージ名は避けてください。
詳細は<https://go.dev/doc/effective_go#package-names>と<https://go.dev/blog/package-names>を参照してください。

## Pass Values

<!-- Don’t pass pointers as function arguments just to save a few bytes. If a function refers to its argument x only as *x throughout, then the argument shouldn’t be a pointer. Common instances of this include passing a pointer to a string (*string) or a pointer to an interface value (*io.Reader). In both cases the value itself is a fixed size and can be passed directly. This advice does not apply to large structs, or even small structs that might grow. -->

数バイトを節約するためだけに、関数の引数としてポインタを渡さないでください。
関数が引数`x`を`*x`としてのみ参照する場合、ポインタ引数にすべきではありません。
これには、文字列(`*string`)やインタフェース値(`*io.Reader`)へのポインタを渡す場合も含まれます。
どちらの場合も、値自体は固定長であり、直接渡すことができます。
このアドバイスは、大きな構造体やサイズが大きくなりうる構造体には適用されません。

## Receiver Names

<!-- The name of a method’s receiver should be a reflection of its identity; often a one or two letter abbreviation of its type suffices (such as “c” or “cl” for “Client”). Don’t use generic names such as “me”, “this” or “self”, identifiers typical of object-oriented languages that gives the method a special meaning. In Go, the receiver of a method is just another parameter and therefore, should be named accordingly. The name need not be as descriptive as that of a method argument, as its role is obvious and serves no documentary purpose. It can be very short as it will appear on almost every line of every method of the type; familiarity admits brevity. Be consistent, too: if you call the receiver “c” in one method, don’t call it “cl” in another. -->

メソッドのレシーバの名前は、そのレシーバのアイデンティティを反映するべきです。
多くの場合1, 2文字の略語で十分です(`Client`の場合は`c`や`cl`など)。
メソッドに特別な意味を持たせるオブジェクト指向言語のような、`me`、`this`、`self`などの名前は使用しないでください。
Goでは、メソッドのレシーバは単なる引数とは別のパラメータであるため、それに応じた名前を付けるべきです。
役割が明確で、ドキュメントとしての目的を果たさないため、メソッドの引数ほど説明的である必要はありません。
その型のすべてのメソッドのほぼすべての行に表示されるため、非常に短くても構いません。
親しみやすさによって簡潔さは許容されます。
また、一貫性を保つことも重要です。
あるメソッドでレシーバを`c`とする場合、別のメソッドで`cl`とはしないでください。

:::message
「親しみやすさによって簡潔さは許容されます」の原文は`familiarity admits brevity`です。
レシーバの名前は、型やファイルの文脈から自然に推測できる、つまり文脈に親しんでいるため、簡潔でも良いという意味です。
:::

## Receiver Type

<!-- Choosing whether to use a value or pointer receiver on methods can be difficult, especially to new Go programmers. If in doubt, use a pointer, but there are times when a value receiver makes sense, usually for reasons of efficiency, such as for small unchanging structs or values of basic type. Some useful guidelines: -->

特に新人Goプログラマにとって、メソッドで値またはポインタレシーバのどちらを使用するか選択することは難しいかもしれません。
迷った場合はポインタレシーバを使うべきですが、効率のために値レシーバを使用する方が適切なこともあります。
たとえば、小さな変更されない構造体や基本型の値などです。
以下にいくつかの有用なガイドラインを示します。

<!-- If the receiver is a map, func or chan, don’t use a pointer to them. If the receiver is a slice and the method doesn’t reslice or reallocate the slice, don’t use a pointer to it. -->

<!-- If the method needs to mutate the receiver, the receiver must be a pointer. -->

<!-- If the receiver is a struct that contains a sync.Mutex or similar synchronizing field, the receiver must be a pointer to avoid copying. -->

<!-- If the receiver is a large struct or array, a pointer receiver is more efficient. How large is large? Assume it’s equivalent to passing all its elements as arguments to the method. If that feels too large, it’s also too large for the receiver. -->

<!-- Can function or methods, either concurrently or when called from this method, be mutating the receiver? A value type creates a copy of the receiver when the method is invoked, so outside updates will not be applied to this receiver. If changes must be visible in the original receiver, the receiver must be a pointer. -->

<!-- If the receiver is a struct, array or slice and any of its elements is a pointer to something that might be mutating, prefer a pointer receiver, as it will make the intention clearer to the reader. -->

<!-- If the receiver is a small array or struct that is naturally a value type (for instance, something like the time.Time type), with no mutable fields and no pointers, or is just a simple basic type such as int or string, a value receiver makes sense. A value receiver can reduce the amount of garbage that can be generated; if a value is passed to a value method, an on-stack copy can be used instead of allocating on the heap. (The compiler tries to be smart about avoiding this allocation, but it can’t always succeed.) Don’t choose a value receiver type for this reason without profiling first. -->

<!-- Don’t mix receiver types. Choose either pointers or struct types for all available methods. -->

<!-- Finally, when in doubt, use a pointer receiver. -->

- レシーバが`map`、`func`、`chan`の場合や、レシーバが`slice`でメソッドがそれを再スライスしたり再割り当てしない場合、ポインタレシーバにしないでください。
- メソッドがレシーバを変更する必要がある場合、ポインタレシーバでなければなりません。
- レシーバが`sync.Mutex`などの同期フィールドを含む構造体の場合、コピーを避けるためにポインタレシーバでなければなりません。
- レシーバが大きな構造体や配列の場合、ポインタレシーバの方が効率的です。どの程度を大きいとするかは、その要素をすべてメソッドに引数として渡すことを考えてください。それが大きすぎると感じる場合、レシーバとしても大きすぎます。
- 他の関数やメソッドが、同時に実行されていたり、メソッドから呼び出されたりして、レシーバを変更する可能性がありますか？　値型はメソッドが呼び出されるときにレシーバのコピーを作成するため、外部の更新はそのレシーバには適用されません。元のレシーバに変更を反映する必要がある場合、ポインタレシーバでなければなりません。
- レシーバが構造体、配列、スライスで、その要素のいずれかが変更される可能性のあるものへのポインタである場合、読み手に意図が明確になるようにポインタレシーバを選択してください。
- レシーバが小さな配列や構造体で、元々値型(たとえば`time.Time`型)の場合や、変更可能なフィールドやポインタを持たない場合、あるいは`int`や`string`などの単純な基本型である場合、値レシーバが適しています。値がメソッドに渡される場合、ヒープではなくスタック上にコピーされ、生成されるガベージが減ることがあるからです（コンパイラはこのヒープ割り当てを避けようとしますが、必ずしも成功するわけではありません）。ただし、この理由だけで値レシーバを選ぶ前に、必ずプロファイリングを行ってください。
- レシーバの種類を混在させないでください。すべてのメソッドのレシーバをポインタまたは値のどちらかに統一してください。
- 最後に、迷った場合はポインタレシーバを使用してください。

## Synchronous Functions

<!-- Prefer synchronous functions - functions which return their results directly or finish any callbacks or channel ops before returning - over asynchronous ones. -->

非同期関数よりも同期関数(結果を直接返すか、コールバックやチャネル操作を完了してから終了する関数)を選択してください。

<!-- Synchronous functions keep goroutines localized within a call, making it easier to reason about their lifetimes and avoid leaks and data races. They’re also easier to test: the caller can pass an input and check the output without the need for polling or synchronization. -->

同期関数は`goroutine`を呼び出しの間に局在させるため、そのライフタイムを理解しやすくし、リークやデータ競合を回避しやすくなります。
また、テストもしやすくなります。
呼び出し元は入力を渡し、出力を確認するだけで済み、ポーリングや同期が必要ありません。

<!-- If callers need more concurrency, they can add it easily by calling the function from a separate goroutine. But it is quite difficult - sometimes impossible - to remove unnecessary concurrency at the caller side. -->

呼び出し側がより多くの並行性を必要とする場合、別の`goroutine`から関数を呼び出すことで簡単に追加できます。
反対に、呼び出し側で不要な並行性を削除することは非常に難しいか、不可能です。

## Useful Test Failures

<!-- Tests should fail with helpful messages saying what was wrong, with what inputs, what was actually got, and what was expected. It may be tempting to write a bunch of assertFoo helpers, but be sure your helpers produce useful error messages. Assume that the person debugging your failing test is not you, and is not your team. A typical Go test fails like: -->

テストは、何が間違っていたのか、どのような入力があったのか、実際に得られた結果は何か、期待された結果は何か、を示す有用なメッセージとともに失敗するべきです。
`assertFoo`のようなヘルパをたくさん書くのは魅力的かもしれませんが、そのヘルパが有用なエラーメッセージを生成することを確認してください。
失敗したテストをデバッグする人があなたでなく、あなたのチームでもないと仮定してください。
典型的なGoテストは以下のように失敗します。

```golang
if got != tt.want {
    t.Errorf("Foo(%q) = %d; want %d", tt.in, got, tt.want) // これ以降何もテストできない場合は、Fatalfを使用
}
```

<!-- Note that the order here is actual != expected, and the message uses that order too. Some test frameworks encourage writing these backwards: 0 != x, “expected 0, got x”, and so on. Go does not. -->

ここでの順序は`actual != expected`であり、メッセージもその順序になっています。
一部のテストフレームワークは、逆に書くことを奨励しています。
`0 != x`、`“expected 0, got x”`などですが、Goではそうではありません。

<!-- If that seems like a lot of typing, you may want to write a table-driven test. -->

もしそれが多くの入力を必要とするようであれば、[`table-driven test`](https://go.dev/wiki/TableDrivenTests)を書くことを検討してください。

:::message
`table-driven test`は日本語では「テーブル駆動テスト」と訳されるか、そのまま「テーブルドリブンテスト」と呼ばれることもあります。
:::

<!-- Another common technique to disambiguate failing tests when using a test helper with different input is to wrap each caller with a different TestFoo function, so the test fails with that name: -->

テスト失敗時の曖昧さをなくすためのテクニックとして、それぞれの呼び出しを異なる`TestFoo`関数でラップする方法もあります。
これにより、テストがその名前で失敗するようになります。

```golang
func TestSingleValue(t *testing.T) { testHelper(t, []int{80}) }
func TestNoValues(t *testing.T)    { testHelper(t, []int{}) }
```

<!-- In any case, the onus is on you to fail with a helpful message to whoever’s debugging your code in the future. -->

いずれにせよ、将来あなたのコードデバッグする人に対して、有用なメッセージとともに失敗する責任はあなたにあります。

## Variable Names

<!-- Variable names in Go should be short rather than long. This is especially true for local variables with limited scope. Prefer c to lineCount. Prefer i to sliceIndex. -->

Goの変数名は長いよりも短い方が良いです。
特にスコープが限られたローカル変数について顕著であり、`lineCount`よりも`c`、`sliceIndex`よりも`i`を選ぶべきです。

<!-- The basic rule: the further from its declaration that a name is used, the more descriptive the name must be. For a method receiver, one or two letters is sufficient. Common variables such as loop indices and readers can be a single letter (i, r). More unusual things and global variables need more descriptive names. -->

使われる場所が宣言から遠いほど説明的な名前が必要になるという基本的なルールがあります。
メソッドのレシーバの場合、1、2文字で十分です。
ループのインデックスや`reader`などの一般的な変数は1文字で構いません(`i`、`r`)。
より珍しいものやグローバル変数には、より説明的な名前が必要です。

<!-- See also the longer discussion in the Google Go Style Guide. -->

より詳細については、[Google Go Style Guide](https://google.github.io/styleguide/go/decisions#variable-names)を参照してください。
