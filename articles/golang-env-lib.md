---
title: "Go言語で環境変数を扱う"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["golang"]
published: true
publication_name: "kurusugawa"
---

Go言語で環境変数を扱う方法についてまとめました。
osパッケージを使う方法と、外部ライブラリを使い構造体にマッピングする方法を紹介します。
また、`.env`ファイルを読み込む方法もあわせて紹介します。

メインはライブラリとその使い方の紹介です。

# TL;DR

- 書捨てコード
  - `os`パッケージを使う
- 普通のコード
  - `caarlos0/env`を使う
- `.env`を使いたいとき
  - `joho/godotenv`を併用

# `os`パッケージを使う

書捨てコード等の簡単な処理で環境変数を利用する場合、`os`パッケージを使うのが簡単です。

`os.Getenv`で環境変数を取得できます。
以下の処理では環境変数`HOGE`に設定された値を取得しています。

```go
hoge := os.Getenv("HOGE")
```

`os.Getenv`は環境変数が設定されていない場合も、空文字列`""`を返します。
環境変数が設定されているかどうかを判定する場合は、`os.LookupEnv`を使います。
設定されているかは2つ目の戻り値の`bool`型で返ります。

```go
hoge, ok = os.LookupEnv("HOGE")
if !ok {
    fmt.Println("HOGE is not set")
}
```

`os.Getenv`, `os.LookupEnv`が使えれば、原理的には環境変数を扱う上で必要な処理はすべて書けます。
**しかし、扱う変数が増えてきたら外部ライブラリを使うことを推奨します(個人的には3つ以上)。
また、`os.LookupEnv`が必要だったり、構造体のフィールドと結びついている場合も外部ライブラリが便利です。**

# ライブラリ(`caarlos0/env`)を使う

この節では、外部ライブラリを使って環境変数を扱う方法を紹介します。

使用するライブラリは`caarlos0/env`です。
リポジトリは[こちら](https://github.com/caarlos0/env)。
構造体へのマッピングをサポートしており、jsonの読み込みのように構造体のフィールドにタグをつけることで結びつけます。

go getでインストール(バージョンは元リポジトリを要確認)しましょう。

```bash
go get github.com/caarlos0/env/v10
```

## 基本の使い方

フィールドに`env:"HOGE"`のようなタグをつけた構造体を用意します。
`HOGE`の部分は環境変数の名前です。

`var cfg config`のように確保した構造体のポインタを`env.Parse`に渡します。
戻り値でエラーがなければ、フィールドの値が環境変数の値で上書きされます。

```go
type config struct {
    Host string `env:"HOST"`
    Port string `env:"PORT"`
}

// 読み込み
var cfg config
if err := env.Parse(&cfg); err != nil {
    fmt.Println(err)
}

// 結果を利用
fmt.Println(cfg.Host)
fmt.Println(cfg.Port)
```

非常に簡単ですね。
ここからは、よく使う便利な機能を紹介します。

## デフォルト値

`envDefault`タグを使うと、環境変数が設定されていない場合のデフォルト値を設定できます。
以下の例では、`PORT`が設定されていない場合に`8080`をデフォルト値として設定しています。

```go
type config struct {
    Host string `env:"HOST"`
    Port string `env:"PORT" envDefault:"8080"`
}
```

予めフィールドに値を入れておくことでもデフォルト値を設定できます。
以下の例では、Hostに`localhost`、Portに`3030`をデフォルト値として設定しています。

```go
cfg := config{
    Host: "localhost",
    Port: "3030",
}

if err := env.Parse(&cfg); err != nil {
    fmt.Println(err)
}
```

## ネストした構造体

ネストされたフィールドにもタグがついていれば値が代入されます。
以下の例では、ネストされた`userConfig`という構造体の各フィールドにも値が代入されます。

```go
type userConfig struct {
    Name string `env:"NAME"`
    Pass string `env:"PASS"`
}

type config struct {
    Host string `env:"HOST"`
    Port string `env:"PORT"`
    admin userConfig
}

var cfg config
if err := env.Parse(&cfg); err != nil {
    fmt.Println(err)
}

fmt.Println(cfg.admin.User)
```

`envPrefix`タグを使うことで、ネストされたフィールドに共通の接頭辞をつけることができます。
以下の例では、`userConfig`のフィールドの接頭辞として`ADMIN_`がつきます。
つまり、`userConfig.Name`の値は`ADMIN_NAME`の環境変数から取得されます。

```go
type userConfig struct {
    Name string `env:"NAME"`
    Pass string `env:"PASS"`
}

type config struct {
    Host  string     `env:"HOST"`
    Port  string     `env:"PORT"`
    admin userConfig `envPrefix:"ADMIN_"`
}
```

## 必須の値をチェック

`required`オプションを使うと、環境変数が設定されていることを強制できます。
以下の例では、`HOST`に値が設定されていないとパース時にエラーを返します。

```go
type config struct {
    Host string `env:"HOST"`
    Port string `env:"PORT"`
}

var cfg config
if err := env.Parse(&cfg); err != nil {
    fmt.Println(err)
    // 環境変数が設定されていないとエラー
    // env: required environment variable "HOST" is not set
}
```

環境変数が空文字列で設定されている場合もエラーを返すには`notEmpty`オプションを使います。
以下の例では、`HOST`に空文字列が設定されているとパース時にエラーを返します。

```go
type config struct {
    Host string `env:"HOST,notEmpty"`
    Port string `env:"PORT"`
}

var cfg config
if err := env.Parse(&cfg); err != nil {
    fmt.Println(err)
    // 環境変数が空文字列だとエラー
    // env: environment variable "HOST" should not be empty
}
```

:::message
ちなみに`required`オプションと`notEmpty`オプションは同時に使うことができますが、`notEmpty`の指定だけで設定されていない場合もエラーになります。

:::

## 使える型色々

`caarlos0/env`は以下のような型のパースをサポートしています。

- `string`
- `bool`
- `int`系
- `uint`系
- `float`系
- `time.Duration`
  - `1s`や`1h`のような表現が使えます。
- `url.URL`

これらのシンプルな型に加えてスライスやマップもサポートしています。
`SLICE="1:2:3"`や、`MAP="k1:v1,k2:v2"`のように、`:`や`,`で区切られた文字列をパースします。

:::message
ここまでよく使う機能を紹介しました。
より高度な機能については[ドキュメント](https://github.com/caarlos0/env)を参照してください。

:::

## よくある落とし穴

### 空文字列と未設定は違う

環境変数全般に言えることですが、値が空文字列なのと変数の未設定は別物です。
空文字列を許容したくない場合、先に紹介した`notEmpty`を使いましょう。

またデフォルト値がある場合、環境変数が設定されていない場合に加えて、値が空文字列の場合もデフォルト値が使われます。
意図的に空文字列を設定したいフィールドには、デフォルト値を設定しないようにしましょう。

### タグの書き方のミス

タグを書くとき、余計なスペースを入れないようにしましょう。
また、`required`などのオプションの位置にも注意です。

:::message alert
特に`:`の前後にスペースを入れてしまうとパース時にすらエラーにならず、値が設定されていないことに気づけません！

:::

```go
type envs struct {
    Hoge string `env:"HOGE"`  // OK
    Fuga string `env: "FUGA"` // NG
    Piyo string `env :"PIYO"` // NG
}

type envsWithOpt struct {
    Hoge string `env:"HOGE,required"`  // OK
    Fuga string `env:"FUGA, required"` // NG
    Piyo string `env:"PIYO ,required"` // OKだけど、やらないほうがいい
    Puyo string `env:"PUYO",required`  // NG
}
```

タグの書き方の一部は、静的解析ツールを使うとチェックしてくれます。
ですが、完全ではないので気をつけましょう。

### 公開されていないフィールドは無視される

これが一番あるミスかもしれません。
構造体のフィールドのうち、外部パッケージに対して公開されていないフィールドは無視されます。
つまり、設定したいフィールドの名前は必ず大文字で始まる必要があります。

:::message alert
パース時のエラーでも静的解析でも検知できません！
超注意です！

:::

```go
type envs struct{
    hoge string `env:"HOGE"` // 無視される
    Fuga string `env:"FUGA"` // OK
}
```

一見罠っぽい挙動ですが`json`パッケージ等も同じ挙動なので、タグを使ったリフレクションを使う際は気をつけましょう。

## 他ライブラリの検討

`caarlos0/env`以外のメジャーな環境変数ライブラリに`kelseyhightower/envconfig`があります。
また、この2つ以外でスター数が1kを超えるライブラリは存在しません。
`caarlos0/env`githubのスター数は4kですが、`kelseyhightower/envconfig`は4.7kと人気は`kelseyhightower/envconfig`のほうが上のようです。

:::message
2023/12/3時点の検索結果です。

:::

### 比較

`kelseyhightower/envconfig`のサンプルコードを、READMEから抜粋します(説明のため改変)。

```bash
# シェルで環境変数を設定
export MYAPP_DEBUG=false
export MYAPP_PORT=8080
export MYAPP_USER=Kelsey
```

```go
type Specification struct {
    Debug       bool
    Port        int
    User        string
}

var s Specification
err := envconfig.Process("myapp", &s)
if err != nil {
    log.Fatal(err.Error())
}
```

簡単に`caarlos0/env`と比較すると、以下のような違いがあります。

- 構造体へのタグがデフォルトで必須ではない
  - 対応する環境変数名は自動で決定される
- すべての公開フィールドがパース対象になる
- パース関数`envconfig.Process`の第一引数にプレフィックスが必要

一方で、対応する型やオプションは`caarlos0/env`とほぼ同じです。
つまり、実現できることは同じです。

### 選定理由

:::message alert
個人の感想です！
プロジェクトごとに自分なりの正解を考えましょう！

:::

この2つから`caarlos0/env`を選定しているのは、インタフェースが明示的だからです。
一方`kelseyhightower/envconfig`は暗黙に処理される部分が多いです。
これには以下のようなデメリットがあります。

- 環境変数の名前を把握するために参照すべきコードが分散する
- 意図しないフィールドが上書きされる


個人的な感触として、ツールとして固いのは`caarlos0/env`のほうだと判断しています。
設定項目が少なく暗黙に高機能な処理が走るのは、なんとなくGoのコードベースとは馴染まないと感じます。

また、`kelseyhightower/envconfig`は2023年に入ってからコードの更新がありません。
Issueも溜まっているようです。
かなり基本的なライブラリなので単純に更新の頻度が高ければ良いとは思いませんが、考慮する必要はあると思います。

# `.env`ファイルを読み込む (`joho/godotenv`を使う)

DockerやDocker Composeを使うとき、`.env`ファイルを使って環境変数を設定することがあります。
コンテナランタイム側で環境変数をセットする場合は紹介したライブラリがそのまま使えます。
しかし、例えばサブディレクトリにGoのプロジェクトルートと`.env`がある場合は、プログラムから読み込む必要があります。

`.env`を読み込むライブラリとして`joho/godotenv`があります。
リポジトリは[こちら](https://github.com/joho/godotenv)。

このように`godotenv.Load`を呼ぶと、`.env`ファイルが読み込まれます。
明示的なファイル名の指定もできます。

```go
err := godotenv.Load()
if err != nil {
    log.Fatal("Error loading .env file")
}

// ファイル名を指定できる
godotenv.Load("some.env")
godotenv.Load("one.env", "two.env")
```

# まとめ

Go言語で環境変数を扱う方法についてまとめました。
基本的な部分ですが、「golang 環境変数」で検索したときにライブラリを使うことが推奨されている記事が少なかったので書いてみました。
参考になったら嬉しいです。
