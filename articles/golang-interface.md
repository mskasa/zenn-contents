---
title: "Go言語 インタフェース活用の具体例（多態性と依存性の注入について）"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go]
published: false
---

## 概要

Go言語におけるインタフェースの活用方法について、実際のコードを交えながら解説します。

想定読者は「Go言語のインタフェースについて勉強したものの、実際どういう時に使うの？」と疑問に感じている方々です。

そのため、所謂「インタフェースとは」的な説明は省略させていただきます。

## インタフェースの特徴（メリット）

インタフェースの特長は、抽象化によるコードの拡張性と再利用性の向上です。
この特長を語る上でのキーワードが多態性（ポリモーフィズム）と依存性の注入（Dependency Injection）になります。
順番に見ていきましょう。

### 多態性（ポリモーフィズム）

インタフェースを用いて実現するポリモーフィズムとは、すなわち「異なる型のオブジェクトが同じインタフェースを通して異なる動作をする」ことです。

#### 簡単な例

以下の例では、異なる型である`Circle`と`Rectangle`が同じインタフェース`Shape`を通して、同一のシグネチャであるメソッド`Area() float64`で異なる動作をします。
「面積を求める」ことを抽象化することで、図形の種類に関わらず、同じ方法（シグネチャ）で、そのオブジェクトを操作できるようになったわけです。

```go
type Shape interface {
    Area() float64
}

type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

type Rectangle struct {
    Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func main() {
    shapes := []Shape{Circle{Radius: 5}, Rectangle{Width: 3, Height: 4}}
    for _, shape := range shapes {
        fmt.Printf("Area: %.2f\n", shape.Area())
    }
}
```

ただし、このような簡易的な例では、その恩恵を説明するのに限界があるため、普段我々がお世話になっている Goの標準ライブラリを例にしてみます。

#### エラー

例えば、新しいエラーを追加する時、

errorインターフェースは非常にシンプルです。唯一必要なのはError() stringメソッドを持つことです。この単純さにより、どんな型でも容易にエラーとしての機能を持たせることができます。

一度定義したエラータイプは、異なるプロジェクトやライブラリ間で再利用することができます。

#### ハンドラー

例えば、新しいハンドラーを追加する時、我々は以下のように実装します。
```go
func hello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, world!")
}
func main() {
    http.HandleFunc("/hello", hello)
}
```
このように簡単に利用できる裏には、インタフェースの存在があります。
httpパッケージのserver.goのソースを確認してみましょう。

https://github.com/golang/go/blob/master/src/net/http/server.go

`http.HandleFunc`はパッケージレベルの関数で、`DefaultServeMux`というデフォルトの`ServeMux`インスタンスを使用して、メソッド`ServeMux.HandleFunc`を呼び出しています。

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}

func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}
```

そして、メソッド`ServeMux.Handle`でハンドラーを登録します。注目すべきは引数の`Handler`です。

```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {
    ・・・（略）・・・
}
```

これは、以下の様にインタフェース型で定義されています。

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

今一度、メソッド`ServeMux.HandleFunc`を確認してみましょう。
今回の例でいうところの、`hello(w http.ResponseWriter, r *http.Request)`を、`HandlerFunc(handler)`部分で、`HandlerFunc`型にキャストしています。

```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}
```

`HandlerFunc`型を確認してみると、シグネチャ`ServeHTTP(w ResponseWriter, r *Request)`を持つメソッドを持っているため、`Handler`インタフェースを実装しているということになります。

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

以上が我々が簡単にハンドラーを追加できている裏の仕組みです。




拡張性と再利用性

このように、標準ライブラリはインタフェースを用いて実装されているため、我々利用者は拡張して利用できるというわけです。
逆に言うと、多くのユーザーの利用が想定されるライブラリを開発する際には、インタフェースを活用した実装が求められるでしょう。

#### 補足：関数型プログラミングのアプローチによるポリモーフィズムの実現

Go では、関数は第一級関数であるため、高階関数（関数が引数として関数を取ったり、関数を返したりすること）を実装できます。

以下の記事のコード例では、`moveByWord(fn func() bool)`という同一シグネチャで異なる挙動を実現しているため、これもポリモーフィズムといえるでしょう。

https://zenn.dev/kasa/articles/golang-pacvim#%E9%AB%98%E9%9A%8E%E9%96%A2%E6%95%B0

### 依存性の注入（DI）

「依存性の注入（Dependency Injection）」という言葉は普及しているのでタイトルに使用しましたが、分かりにくいので、「インスタンス渡し」に置き換えてください。
なぜか難しい用語で広まってしまっているため、ややこしく感じている人も多いですが、結局やっていることはオブジェクトAにオブジェクトBのインスタンスを外から渡しているだけです（下図参照）。

このテクニックを使うとどんなメリットがあるのでしょうか。DIを利用していない場合と、利用した場合で比較してみましょう。

:::message
また、誤解している方も多いですが、DI自体にインタフェースが必須というわけではありません。あくまで、DI でインタフェースを受け取り構造体を返すようにコード
:::

```go
type Application struct {
	logger ConsoleLogger // 直接 ConsoleLogger に依存
}

type ConsoleLogger struct{}

func (l ConsoleLogger) Log(message string) {
	fmt.Println("Log to console:", message)
}

func (app *Application) Run() {
	app.logger.Log("Application is running")
}

func main() {
	app := Application{
		logger: ConsoleLogger{},
	}

	app.Run() // 出力: Log to console: Application is running
}
```

- 単体テストが困難
- 拡張性の欠如

ConsoleLogger の動作が Application のテスト結果に直接影響を与えるため、純粋な単体テストを行うのが難しくなります。
テスト中に ConsoleLogger の振る舞いを制御することができないため、テストの再現性や精度が低下します。また、外部リソース（例えばコンソール出力）に依存するコードのテストは、モックやスタブを使用して内部依存関係を隔離する場合と比べて、一般的に複雑で時間がかかります。依存性注入を使用することで、これらの問題を回避し、より効率的で再現性の高いテストを実現できます。

```go
package main

import (
    "fmt"
)

// Logger インタフェースはログ記録の機能を定義します。
type Logger interface {
    Log(message string)
}

// ConsoleLogger は Logger インタフェースの一つの実装です。
type ConsoleLogger struct{}

func (l ConsoleLogger) Log(message string) {
    fmt.Println("Log to console:", message)
}

// Application は Logger の実装に依存しています。
type Application struct {
    logger Logger
}

func (app *Application) SetLogger(logger Logger) {
    app.logger = logger
}

func (app *Application) Run() {
    app.logger.Log("Application is running")
}

func main() {
    app := &Application{}
    
    // コンソールロガーを注入
    consoleLogger := ConsoleLogger{}
    app.SetLogger(consoleLogger)

    app.Run() // 出力: Log to console: Application is running
}
```

```go
package main

import (
    "testing"
)

// MockLogger はテスト用の Logger 実装です。
type MockLogger struct {
    LoggedMessages []string
}

func (l *MockLogger) Log(message string) {
    l.LoggedMessages = append(l.LoggedMessages, message)
}

func TestApplicationRun(t *testing.T) {
    mockLogger := &MockLogger{}
    app := &Application{
        logger: mockLogger,
    }

    app.Run()

    if len(mockLogger.LoggedMessages) != 1 || mockLogger.LoggedMessages[0] != "Application is running" {
        t.Errorf("Expected 'Application is running' log message, got %v", mockLogger.LoggedMessages)
    }
}
```

インタフェースに依存している場合、そのインタフェースのIN/OUT（つまり、メソッドのシグネチャや振る舞いの契約）が変わらない限り、依存している側のコードの振る舞いは変わらないというメリットがあります。これにより、インタフェースの実装を変更しても、インタフェースを使用するコードには影響が及びません。

一方で、インタフェースのIN/OUT自体が変更される場合、それは依存しているコードに大きな影響を与える可能性があります。このため、インタフェースの変更は慎重に行う必要があり、通常は互換性を保つために既存のインタフェースを維持しながら新しいインタフェースを追加する形を取ることが一般的です。

```go
// 既存のインタフェース
type Logger interface {
    Log(message string)
}

// 新しい機能を追加した拡張インタフェース
type AdvancedLogger interface {
    Logger // 既存のインタフェースを組み込む
    LogWithLevel(message string, level LogLevel)
}

// 既存のインタフェースを実装する型
type SimpleLogger struct{}

func (l SimpleLogger) Log(message string) {
    fmt.Println(message)
}

// 拡張インタフェースを実装する型
type LevelLogger struct{}

func (l LevelLogger) Log(message string) {
    // 基本的なログ記録の実装
    fmt.Println(message)
}

func (l LevelLogger) LogWithLevel(message string, level LogLevel) {
    // レベルを含むログ記録の実装
    fmt.Printf("[%s] %s\n", level, message)
}

// 使用例
func performLogging(l Logger) {
    l.Log("Basic log message")
    if advLogger, ok := l.(AdvancedLogger); ok {
        advLogger.LogWithLevel("Advanced log message", Info)
    }
}
```

## インタフェースの特徴（デメリット）

## まとめ

ここまで説明してきましたが、最後にインタフェースのデメリットを述べておくと、「実装が複雑になる」ということがあげられます。

よって、本稿の内容を理解した上で、インタフェースの要否を判断しましょう
（個人的には、インタフェースを使わずにコードを書いておき、必要だと分かった段階で利用する形に書き換える程度のスタンスでよいと思います）。

標準ライブラリのコードを読む時などに必要

過剰に複雑な実装をするぐらいなら、インタフェースを使わずにコードを書くという指針で十分です。
具体的に、実装が複数パターン出てくることが明らかになってから初めて書き換えるぐらいで良いと思います。
すべてに通ずることですが、必要なら

Go言語におけるインターフェースは、確かに抽象型であり、その使用がパフォーマンスに影響を与える可能性がありますが、この影響は通常は非常に小さいものです。インターフェースがパフォーマンスに与える影響を理解するために、以下のポイントを考慮する必要があります。

インターフェースのパフォーマンスへの影響
動的ディスパッチ: インターフェースを通じてメソッドを呼び出す際には、動的ディスパッチ（実行時のメソッド解決）が行われます。これは静的ディスパッチ（コンパイル時のメソッド解決）に比べてわずかにコストが高いです。しかし、このオーバーヘッドは通常非常に小さく、ほとんどのアプリケーションでは無視できるレベルです。

メモリオーバーヘッド: インターフェースを使用すると、追加のメモリオーバーヘッドが発生することがあります。インターフェースは内部的に型情報と具体的な値へのポインタを保持するため、これがわずかなメモリ使用量の増加につながります。

ガーベージコレクション: インターフェースを通じてオブジェクトを扱う場合、これらのオブジェクトがヒープ上に配置される可能性があります。これはガーベージコレクションの頻度や負荷に影響を与える可能性があります。

抽象型であるため、パフォーマンスに影響

Goコンパイラはエスケープ分析を実施して、オブジェクトが関数のスコープを超えて生存するかどうかを判断します。インターフェースを通じて渡されるオブジェクトが関数の外部で参照される可能性がある場合、それはヒープに割り当てられます。
ヒープに配置されると、GCが管理するメモリの量が増加します。これにより、GCの実行頻度や処理負荷が増える可能性があります。

Go言語において関数をインターフェース型の引数で呼び出す際にヒープ割り当てが発生するかどうかは、コンパイラのエスケープ分析に依存します。エスケープ分析はコンパイル時に行われ、変数やオブジェクトが関数のスコープを超えて生存するかどうかを判断するプロセスです。このプロセスの結果に基づいて、変数やオブジェクトがスタック上に配置されるか、ヒープ上に配置されるかが決定されます。

インターフェース型の引数を関数に渡す場合、コンパイラはそのインターフェースが関数の外部でどのように使用されるかを判断する必要があります。インターフェースを介して渡されるオブジェクトがエスケープすると判断されると、そのオブジェクトはヒープに割り当てられます。

ヒープとスタック: Go言語のコンパイラは、変数の割り当て場所（ヒープまたはスタック）を実行時の挙動に基づいて決定します。一般的に、関数のスコープを超えて生存する可能性がある変数はヒープに配置され、スコープ内で完結する変数はスタックに配置されます。