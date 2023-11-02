---
title: "Go言語で独自エラーを実装するときの実例(ライブラリ編)"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
published_at: 2022-11-21
---

:::message
この記事は、過去に自分のブログで公開していたものの移植です。
:::

## 概要

Goでライブラリを作るとき、呼び出し元でどのようにエラーハンドリングするか想像して実装することが出来ていますか?
この記事では、minimalなサンプルを提示し、Goでの独自エラー実装について紹介します。

TODO: Asについては、僕がユースケースを理解できていないので書いていません。

## 前提

Goの文法は分かっているということを仮定します。

## ライブラリ利用者側から見たGoのエラー処理について

ご存知の通り、Goには高級?なエラー処理機構がありません。
関数が正常に終了したかどうかは、if文によって判定します。

```go
func someFunc() error {...}

err := someFunc()
if err != nil {
    // 何か処理
}
```

また、`error`インターフェースの定義は以下のようになっていて、`.Error()`によって出力する文字列を作成できれば、どんなものでも`error`型として扱うことが出来ます。

```go
type error interface{
    Error() string
}
```

そのため、ライブラリ作成者には独自エラーの構造体に対して、`Error() string`を実装することが要求されます。

```go
type myError struct{
    // some member
}

func (err *myError) Error() string{
    return ""
}
```

### Unwrapの利用

上記のような独自エラーには、**エラーが起こった原因となるエラー**が何か判別出来ないという欠点がありました。
(正確には、判別のための標準的なインターフェースがなかった。)

Go1.13以降、エラー原因の特定インターフェースとして、`errors.Unwrap(err) error`という関数が使えます。
これによって、ライブラリ使用者は**エラーを引き起こしたエラー**を取得することが出来ます。

```go
err := someFunc()
if err != nil {
    inner := errors.Unwrap(err)
    fmt.Printf("内部エラーはこれ: %v", inner)
}
```

### Isの利用

エラー原因特定のユースケースとして、エラー種別によって処理を分けるというケースがあります。
このとき、エラーの原因の原因の原因...のようにエラーが連鎖的にwrapされている場合、あるエラーと、その原因となったエラーの種別が同じものかを判定する必要があります。

そこで、`errors.Is(err, target error) bool`という関数が用意されています。
これによって、`err`が`nil`になるまで`Unwrap`し続け、`target`と等しいかチェック、処理を分けることが出来ます。

```go
err := someFunc()
if err != nil {
    if errors.Is(err, ErrNotFound) {
        // なにか処理
    }else if errors.Is(err, ErrPermissionDenied) {
        // なにか処理
    }
}
```

## ライブラリの実装者がすべき独自エラーの実装について

### Unwrapの実装

`Unwrap`機能を提供するために、独自エラーはエラーを返すきっかけとなった内部エラーを保持しておく必要があります。
また、`error`型として扱うために、`Error() string`メソッドを実装する必要があります。

```go
type myError struct {
    innner error
}

func (err *myError)Error() string {
    return "myError: " + err.innner.Error()
}
```

ただ、これだけでは正しく`Unwrap`してくれません。
そこで、`errors.Unwrap`の実装を見てどうすればいいか確認してみましょう。

https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/errors/wrap.go;l=14

```go
// Unwrap returns the result of calling the Unwrap method on err, if err's
// type contains an Unwrap method returning error.
// Otherwise, Unwrap returns nil.
func Unwrap(err error) error {
    u, ok := err.(interface {
        Unwrap() error
    })
    if !ok {
        return nil
    }
    return u.Unwrap()
}
```

関数内の最初に、`err.(interface {Unwrap() error})`という式があります。
これは`err`が、無名interfaceである`interface {Unwrap() error}`を実装している型に変換可能かどうか、実行時にチェックするロジックです。
(型アサーション)

すなわち、`Unwrap`に投げられるエラーには、`Unwrap() error`が実装されていることが期待されます。
(注意!`errors.Unwrap`とはシグネチャが違います)

myErrorにも、`Unwrap() error`を実装してみます。

```go
func (err *myError)Unwrap() error {
    return err.innner
}

func main() {
    err := someMyFunc() // 適当な関数
    innerErr := errors.Unwrap(err)
    fmt.Printf("内部エラー発見可能: %v", innerErr)
}
```

### Isの実装

次に、`errors.Is`が`myError`について呼び出された状況を考えます。

```go
err := someMyFunc() // *myErrorを返す関数
if errors.Is(err, fs.ErrNotExist) {
    // どこかでファイルが存在しないことによるエラーが発生した
}
```

前節で`Unrwap`を実装したため、再帰的に`err`を辿っていって、どこかで`fs.ErrNotExist`に当たらないかをチェックすることが出来ます。

しかし、**独自エラー自体に種別があり**、その種別について判定したい場合にはどうすれば良いでしょう。
とりあえず、`errors.Is`の実装を見てみます。

https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/errors/wrap.go;l=40
```go
// Is reports whether any error in err's chain matches target.
//
// The chain consists of err itself followed by the sequence of errors obtained by
// repeatedly calling Unwrap.
//
// An error is considered to match a target if it is equal to that target or if
// it implements a method Is(error) bool such that Is(target) returns true.
//
// An error type might provide an Is method so it can be treated as equivalent
// to an existing error. For example, if MyError defines
//
//    func (m MyError) Is(target error) bool { return target == fs.ErrExist }
//
// then Is(MyError{}, fs.ErrExist) returns true. See syscall.Errno.Is for
// an example in the standard library. An Is method should only shallowly
// compare err and the target and not call Unwrap on either.
func Is(err, target error) bool {
    if target == nil {
        return err == target
    }

    isComparable := reflectlite.TypeOf(target).Comparable()
    for {
        if isComparable && err == target {
            return true
        }
        if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
            return true
        }
        // TODO: consider supporting target.Is(err). This would allow
        // user-definable predicates, but also may allow for coping with sloppy
        // APIs, thereby making it easier to get away with them.
        if err = Unwrap(err); err == nil {
            return false
        }
    }
}
```

`Unwrap`より少し複雑ですが、

1. `target`が比較可能で`err`と等しければ`true`
2. `err`が`Is(error) bool`を実装する型であれば、`err.Is(target)`を使って判定
3. 上のステップが`false`なら、`Unwrap`してループ→1へ

といった処理になっています。
実装から分かるように、独自エラーに種別がある場合`Is(error) bool`メソッドの実装が必要です。

まずは、独自エラーに種別を定義してみます。
`Error() string`も改良してみましょう。

```go
type myError struct {
    kind  myStatus
    inner error
}

type myStatus string

const (
    statusNotFound      = myStatus("NotFound")
    statusUnauthorized  = myStatus("Unauthorized")
    statusCatInterfered = myStatus("CatInterfered")
)

func (err *myError) Error() string {
    return string(err.kind) + ": " + err.inner.Error()
}
```

`myStatus`型を作るor作らない、`string`ではなく`int`にする等、状況によって実装は変わりますが、`myError`に
種別を保持するメンバ(`kind`)を加えるというのは共通するはずです。

続いて、`myError`に対して`Is(error) bool`メソッドを実装します。
また、ライブラリ利用者が`errors.Is`の引数`target`に入れるための**エラー種別を公開する変数**を定義する必要があります。
(例えば、先程例示した`fs.ErrNotExist`のようなものです)

```go
var (
    ErrNotFound      = myError{statusNotFound, nil}
    ErrUnauthorized  = myError{statusUnauthorized, nil}
    ErrCatInterfered = myError{statusCatInterfered, nil}
)

func (err *myError) Is(target error) bool {
    // `kind`で比較する前に、型変換で`myError`として扱えるかをチェック
    t, ok := target.(*myError)
    return ok && err.kind == t.kind
}
```

このように、「myError型として扱える」、「kindメンバが等しい」という2条件が揃ったときに2つのエラーが正しいとすると良いと思います。
またライブラリの利用者は、`errors.Is(err, ErrNotFound)`のように利用するものと想定しています。

## 実装のまとめ

これまでの実装で、以下のようなコードが出来上がります。

{{< gist tbistr 53e3905667cc3c3afd949e4d57357b5a >}}

## おまけ`fmt.Errorf()`を使うべきか

`fmt.Errorf()`のフォーマット指定子に`%w`を使うことで、渡されたエラーをwrapしたエラーを作ってくれます。
ただし、実装を見るとわかりますが、

https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/fmt/errors.go;l=17

```go
type wrapError struct {
    msg string
    err error
}

func (e *wrapError) Error() string {
    return e.msg
}

func (e *wrapError) Unwrap() error {
    return e.err
}
```
のように、`wrapError`という構造体に表示文字列と内部エラーを保持しているだけです。
すなわち、一番外側のエラーの種別情報に関して完全に捨てることになります。
個人的には、ライブラリのコードでは`fmt.Errorf`は使わずに独自エラーを定義してあげて、有り得るエラーをvarで提示した方が親切だと思います。

## 独自エラーを作らないという考え

薄いラッパーや、関数毎に返すエラーが決まりきっている場合、下のレイヤのエラーをそのまま返すという選択もありだと思います。

例えば、`os.Open`のような関数では`os.PathError`を返すとしています。
しかし、[実体は`fs.PathError`です](https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/os/error.go;l=48)。

また、そんなfsパッケージの中でも一部のエラーはoserrorのものをそのまま返しています。

https://cs.opensource.google/go/go/+/refs/tags/go1.19.3:src/io/fs/fs.go;l=136

```go
// Generic file system errors.
// Errors returned by file systems can be tested against these errors
// using errors.Is.
var (
    ErrInvalid    = errInvalid()    // "invalid argument"
    ErrPermission = errPermission() // "permission denied"
    ErrExist      = errExist()      // "file already exists"
    ErrNotExist   = errNotExist()   // "file does not exist"
    ErrClosed     = errClosed()     // "file already closed"
)

func errInvalid() error    { return oserror.ErrInvalid }
func errPermission() error { return oserror.ErrPermission }
func errExist() error      { return oserror.ErrExist }
func errNotExist() error   { return oserror.ErrNotExist }
func errClosed() error     { return oserror.ErrClosed }
```
