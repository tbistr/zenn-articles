---
title: "Type aliasの使い所を提案"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

:::message
この記事は、過去に自分のブログで公開していたものの移植です。
:::

## Type alias

Type aliasはGo 1.9で追加された機能です。
以下のようなsyntaxで既存の型に別名を定義できます。

```go
type aliasT = string
```

普通のDefined typeと何が違うのかというと、定義された型と元になる型との同一性が異なります。
具体的にDefined typeでは以下のように独自メソッドを生やすことができる一方、Type aliasにはメソッドを追加できません。

```go
type defT string

func (d defT) SomeMethod() {
    // これは出来る
}

type aliasT = string

func (d aliasT) SomeMethod() {
    // これは出来ない!
}
```

これはaliasT型がstringと完全に同じとみなされているからで、以下のようなコードはコンパイルが通ります。

```go
type aliasT = string

func f() string {
    var a aliasT
    return a
}
```

## 使い所

導入経緯的には、大規模コードのAPI変更をサポートするために作られた機能だそうで、普通のコードベースだと一生体験する機会がなさそうです。
そんな感じで微妙にどこに使えば良いか迷うこの機能ですが、最近書いたコードで良さそうなユースケースを見つけました。

### 導入対象

対象となるプロジェクトは自作の[AtCoder関連のライブラリ「atcoder-go」](https://github.com/tbistr/pig)で、注目したい部分は以下のようなファイル群です。

```plaintext
atcodergo
    |---model // 外部に公開する構造体を定義
    |   |---model.go
    |---parse // HTMLパーサー、セレクタを使ってHTML→構造体の変換
    |   |---task.go
    |   |---(etc.go...)
    |---task.go // 外部から操作するメソッドを定義
    |---(etc.go...)
```

各ファイルの内容はざっくり、こんな感じです。

```go
// model/model.go
type Task struct {
 Name   string
 IdName string
 ID     string
}

// parse/task.go
func Task(doc pig.Node) []*model.Task {
    tasks := []*model.Task{}
    // pig(自作ライブラリ)を使ってHTMLをパース
    return tasks
}

// /task.go
func (c *Client) Task() []*model.Task {
    tasks := []*model.Task{}
    // 通信してページを取得
    tasks = parse.Task(doc)
    return tasks
}
```

以下のようなことを考慮してこの構成になりました。

- parseだけするパッケージを分けたい→`parse/`追加
  - テストを書く場合parse関数は分離するべき
  - `parse~()`みたいなprefixを付けた関数が量産されるのは嫌
- `parse/`の中の関数で返す型と、`atcodergo/`で返す型は共通化したい→`model/`追加

### 導入

`atcodergo/task.go`をこのように変更しました。

```go:/task.go

type Task = model.Task

func (c *Client) Task() []*Task {
    tasks := []*Task{}
    // 通信してページを取得
    tasks = parse.Task(doc)
    return tasks
}
```

## まとめ

この書き方には以下のような利点があります。

- 外部から見える型が`model.Task`のように一段深くならない
- Defined typeと異なり型変換が不要

型変換については、もしDefine typeを使った場合には`unsafe.Pointer`や`unsafe.Slice`を使用してキャスト処理を書く必要があります。
この処理自体はそんなに長くもないのですが、やはり元の型との整合性を自分で担保しないといけないのはちょっと怖いです。

一方この書き方には、型のドキュメント(godoc)が空になり、基になる型の情報がわかりにくくなるというデメリットがあります。
また、Go的には割りと大規模なコードベースでも[1パッケージにフラットに書く風潮があったりする](https://future-architect.github.io/articles/20210427a/)ので、そちらを採用する場合は完全に不要になります。
今回はHTMLセレクタを自作しており、それを使っている部分を最小化、なるべくテスタブルにしたいという事情があります。

こんな感じで、外部に公開したい型を参照しやすくするという用途では結構使えるんじゃないかと思っています。

## おまけ(Alias typeの導入経緯)

Type aliasの導入経緯は[このページ](https://go.dev/talks/2016/refactor.article)から読むことが出来ます。
ざっくり言うと大規模コードベースでパッケージを整理する際、一気にすべてのコードを変更しなくて済むように`type newT = oldT`のように書くことを想定しています。
