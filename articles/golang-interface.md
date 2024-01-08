---
title: "Go言語 インタフェース活用の具体例（多態性と依存性の注入について）"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go]
published: false
---



## インタフェースによる抽象化

インタフェースの役割は抽象化にあり、その抽象化が様々な恩恵をもたらします。
まずは以下のサンプルコードを見てください。

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

この例では、異なる型である`Circle`と`Rectangle`が同じインタフェース`Shape`を通して、同一のシグネチャ`Area() float64`で異なる動作をします。
「面積を求める」ことを抽象化すること（`Area`というメソッド名で、`float64`の値を返しさえすればOK）で、図形の種類に関わらず、同じ方法（シグネチャ）で、その型を操作できるようになったわけです。

:::message
「型」はオブジェクト指向言語における「オブジェクト」と捉えてください。
:::

## インタフェースの抽象化によるメリット

インタフェースが型の振る舞いを抽象化することは分かりましたが、一体これが何の役に立つのでしょうか。今回はインタフェースの抽象化による恩恵を2つ紹介しようと思います。
1. ソースコードの共通化
2. 具体的な実装との分離

先程の簡易的な例では、その恩恵を説明するのに限界があるため、より実用的な例を見てみましょう。
普段我々がお世話になっている Goの標準ライブラリの中から `io.Reader` を例にしてみます。

### ソースコードの共通化

異なるデータソースからの読み取りを共通のインターフェースを通じて行うことで、コードの共通化が可能になります。
インターフェースによる抽象化が、この共通化を可能にしています。

### 具体的な実装との分離

まずは以下のサンプルコードを見てください。

```go
type ServiceA struct{}

type ServiceB struct{}

func (a ServiceA) ProcessingA() (string, error) {
	// 型「ServiceB」を実装
	b := ServiceB{}

	res, err := b.ProcessingB()
	if err != nil {
		return "", err
	}

	switch res {
	case 0:
		return "hello", nil
	case 1:
		return "world", nil
	}
}

func (b ServiceB) ProcessingB() (int, error) {
	// 何か複雑でややこしい処理
	// 0 or 1、エラーが返ることもあるよ
	return 0, nil
}

func main() {
	a := ServiceA{}
	a.ProcessingA()
}
```

上記のコードでは、型`ServiceA`のメソッド`ProcessingA`内で、型`ServiceB`を実装しています。
そして、型`ServiceB`のメソッド`ProcessingB`の処理結果に応じて`ProcessingA`の処理結果が変わります。
つまり、型`ServiceA`が型`ServiceB`に依存してしまっており、`ProcessingA`の単体テストが不可能な状態であるといえます。

`ServiceB`が簡単な処理で、自チームで開発されているのであれば問題ないかもしれません。
しかし、複雑な処理であったり、他チームが開発しているとなると、これは開発上の大きな問題になります。

これを解決するのが、オブジェクト指向言語でいうところの依存性の注入（Dependency Injection）になります。
DIは難しくとらわれがちですが、ただ単に外部からインスタンスを渡してあげることに他なりません。
Go はオブジェクト指向言語ではないため、オブジェクトやインスタンスという概念がありません。
そのため「型」や「型の実装」という表現をここでは使用しますが、オブジェクト指向言語に慣れている方は「型」を「オブジェクト」、「型の実装」を「インスタンス」と読み替えていただくと咀嚼しやすいと思います。

では、依存性(`ServiceB`への依存)を外部から注入するようにコードを変更してみます。

```diff go
- type ServiceA struct{}
+ type ServiceA struct {
+ 	b ServiceB
+ }

type ServiceB struct{}

func (a ServiceA) ProcessingA() (string, error) {
-	// A の中で Bインスタンスを生成
-	b := ServiceB{}
-	res, err := b.ProcessingB()
+	res, err := a.b.ProcessingB()
	if err != nil {
		return "", err
	}
	switch res {
	case 0:
		return "hello", nil
	case 1:
		return "world", nil
	}
}

func (b ServiceB) ProcessingB() (int, error) {
	// 何か複雑でややこしい処理
	// 0 or 1、エラーが返ることもあるよ
	return 0, nil
}

func main() {
-	a := ServiceA{}
+	a := ServiceA{
+		b: ServiceB{},	// 型「ServiceB」の実装を注入
+	}
	a.ProcessingA()
}
```

型`ServiceB`の実装を外部から渡すように変更できました。
しかし、まだ不完全です。なぜなら、具象型(`struct`)への依存だからです。抽象型である`interface`に依存することで、

```diff go
type ServiceA struct {
-   b ServiceB
+	b ServiceBInterface
}

+type ServiceBInterface interface {
+	ProcessingB() (int, error)
+}

type ServiceB struct{}

func (a ServiceA) ProcessingA() (string, error) {
	// ProcessingB の結果が影響
	res, err := a.b.ProcessingB()
	if err != nil {
		return "", err
	}
	switch res {
	case 0:
		return "hello", nil
	case 1:
		return "world", nil
	}
}

func (b ServiceB) ProcessingB() (int, error) {
	// 何か複雑でややこしい処理
	// 0 or 1、エラーが返ることもあるよ
	return 0, nil
}

func main() {
	a := ServiceA{
		b: ServiceB{},
	}
	a.ProcessingA()
}
```

```go
type MockServiceB struct {
	Result int
	Err    error
}

func (m MockServiceB) ProcessingB() (int, error) {
	return m.Result, m.Err
}

func TestServiceAProcessingAWithTableDriven(t *testing.T) {
	testCases := []struct {
		name     string
		mock     MockServiceB
		expected string
		hasError bool
	}{
		{
			name: "Success with 'hello'",
			mock: MockServiceB{
				Result: 0,
				Err:    nil,
			},
			expected: "hello",
			hasError: false,
		},
		{
			name: "Success with 'world'",
			mock: MockServiceB{
				Result: 1,
				Err:    nil,
			},
			expected: "world",
			hasError: false,
		},
		{
			name: "Error with empty result",
			mock: MockServiceB{
				Result: 1,
				Err:    errors.New("Error in ProcessingB"),
			},
			expected: "",
			hasError: true,
		},
	}

	for _, tc := range testCases {
		t.Run(tc.name, func(t *testing.T) {
			// テストケースごとに ServiceA にモックを注入
			serviceA := ServiceA{b: tc.mock}

			result, err := serviceA.ProcessingA()

			if result != tc.expected {
				t.Errorf("Expected %s, but got %s", tc.expected, result)
			}

			if (err != nil) != tc.hasError {
				t.Errorf("Expected error status: %v, but got: %v", tc.hasError, err)
			}
		})
	}
}
```

- 単体テストが困難
- 拡張性の欠如

ConsoleLogger の動作が Application のテスト結果に直接影響を与えるため、純粋な単体テストを行うのが難しくなります。
テスト中に ConsoleLogger の振る舞いを制御することができないため、テストの再現性や精度が低下します。また、外部リソース（例えばコンソール出力）に依存するコードのテストは、モックやスタブを使用して内部依存関係を隔離する場合と比べて、一般的に複雑で時間がかかります。依存性注入を使用することで、これらの問題を回避し、より効率的で再現性の高いテストを実現できます。

## 概要

Go言語におけるインタフェースの活用方法について、実際のコードを交えながら解説します。

想定読者は「Go言語のインタフェースについて勉強したものの、実際どういう時に使うの？」と疑問に感じている方々です。

そのため、所謂「インタフェースとは」的な説明は省略させていただきます。

## インタフェースの特長（メリット）

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

#### errorインタフェース

```go
type error interface {
	Error() string
}
```

例えば、新しいエラーを追加する時、

errorインターフェースは非常にシンプルです。唯一必要なのはError() stringメソッドを持つことです。この単純さにより、どんな型でも容易にエラーとしての機能を持たせることができます。

一度定義したエラータイプは、異なるプロジェクトやライブラリ間で再利用することができます。

これらはすべて、インタフェースが為せる技です。

#### Handlerインタフェース

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

TODO
以下は折り畳んで、要点だけ伝える

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

TODO 以下も折り畳む

#### 番外編：関数型プログラミングのアプローチによるポリモーフィズムの実現

インタフェースを利用しない形でもポリモーフィズムを実装できます。本筋から外れるので、興味のある方だけどうぞ。

Go では、関数は第一級関数であるため、高階関数（関数が引数として関数を取ったり、関数を返したりすること）を実装できます。

以下の記事のコード例では、`moveByWord(fn func() bool)`という同一シグネチャで異なる挙動を実現しているため、これもポリモーフィズムといえるでしょう。

https://zenn.dev/kasa/articles/golang-pacvim#%E9%AB%98%E9%9A%8E%E9%96%A2%E6%95%B0

### 依存性の注入（DI）

「依存性の注入（Dependency Injection）」という言葉は普及しているのでタイトルに使用しましたが、分かりにくいので、「インスタンス渡し」に置き換えてください。
なぜか難しい用語で広まってしまっているため、ややこしく感じている人も多いですが、結局やっていることはオブジェクトAにオブジェクトBのインスタンスを外から渡しているだけです（下図参照）。

このテクニックを使うとどんなメリットがあるのでしょうか。DIを利用していない場合と、利用した場合で比較してみましょう。

オブジェクトが別のオブジェクトのインスタンスを直接作成するのではなく、その依存オブジェクトを外部から注入される（例：コンストラクタインジェクション、セッターインジェクション）。これにより、オブジェクトの作成と使用を分離し、柔軟性とテスタビリティを高めます。

初期コード
問題点説明
DIコード
テストコード
インタフェースにして拡張

```go
type ServiceA struct{}

type ServiceB struct{}

func (a ServiceA) ProcessingA() (string, error) {
	// A の中で Bインスタンスを生成
	b := ServiceB{}
	// ProcessingB の結果が影響
	res, err := b.ProcessingB()
	if err != nil {
		return "", err
	}
	switch res {
	case 0:
		return "hello", nil
	case 1:
		return "world", nil
	}
}

func (b ServiceB) ProcessingB() (int, error) {
	// 何か複雑でややこしい処理
	// 0 or 1、エラーが返ることもあるよ
	return 0, nil
}

func main() {
	a := ServiceA{}
	a.ProcessingA()
}
```

上記のコードは`ServiceA`の処理内で`ServiceB`を直接生成しています。
そして、`ServiceB`の処理結果に応じて`ServiceA`の処理結果が変わります。

つまり、`ServiceA`が`ServiceB`に依存しているわけです。
その結果、`ServiceA`のテストをする際は`ServiceB`もテストすることとなるため、テストが失敗した際の問題の切り分けが難しくなります（単体テスト不可能な状態）。

`ServiceB`が簡単な処理で、自チームで開発されているのであれば問題ないかもしれませんが、複雑な処理であったり、他チームが開発しているとなると、これは開発上の大きな問題になります。

これを解決するのが DIです。

まずは以下のように、外部からインスタンスを渡してあげましょう。
依存性(`ServiceB`への依存)を外から注入しています。

```diff go
-type ServiceA struct{}
+type ServiceA struct {
+	b ServiceB
+}

type ServiceB struct{}

func (a ServiceA) ProcessingA() (string, error) {
-	// A の中で Bインスタンスを生成
-	b := ServiceB{}
	// ProcessingB の結果が影響
-   res, err := b.ProcessingB()
+	res, err := a.b.ProcessingB()
	if err != nil {
		return "", err
	}
	switch res {
	case 0:
		return "hello", nil
	case 1:
		return "world", nil
	}
}

func (b ServiceB) ProcessingB() (int, error) {
	// 何か複雑でややこしい処理
	// 0 or 1、エラーが返ることもあるよ
	return 0, nil
}

func main() {
-   a := ServiceA{}
+	a := ServiceA{
+		b: ServiceB{},
+	}
	a.ProcessingA()
}
```

:::details 差分表示なし TODO
```go
func main() {
	a := ServiceA{
		b: ServiceB{},
	}
	a.ProcessingA()
}
```
:::

しかし、まだ不完全です。
なぜなら、具象型(`struct`)への依存だからです。抽象型である`interface`に依存することで、

```diff go
type ServiceA struct {
-   b ServiceB
+	b ServiceBInterface
}

+type ServiceBInterface interface {
+	ProcessingB() (int, error)
+}

type ServiceB struct{}

func (a ServiceA) ProcessingA() (string, error) {
	// ProcessingB の結果が影響
	res, err := a.b.ProcessingB()
	if err != nil {
		return "", err
	}
	switch res {
	case 0:
		return "hello", nil
	case 1:
		return "world", nil
	}
}

func (b ServiceB) ProcessingB() (int, error) {
	// 何か複雑でややこしい処理
	// 0 or 1、エラーが返ることもあるよ
	return 0, nil
}

func main() {
	a := ServiceA{
		b: ServiceB{},
	}
	a.ProcessingA()
}
```

```go
type MockServiceB struct {
	Result int
	Err    error
}

func (m MockServiceB) ProcessingB() (int, error) {
	return m.Result, m.Err
}

func TestServiceAProcessingAWithTableDriven(t *testing.T) {
	testCases := []struct {
		name     string
		mock     MockServiceB
		expected string
		hasError bool
	}{
		{
			name: "Success with 'hello'",
			mock: MockServiceB{
				Result: 0,
				Err:    nil,
			},
			expected: "hello",
			hasError: false,
		},
		{
			name: "Success with 'world'",
			mock: MockServiceB{
				Result: 1,
				Err:    nil,
			},
			expected: "world",
			hasError: false,
		},
		{
			name: "Error with empty result",
			mock: MockServiceB{
				Result: 1,
				Err:    errors.New("Error in ProcessingB"),
			},
			expected: "",
			hasError: true,
		},
	}

	for _, tc := range testCases {
		t.Run(tc.name, func(t *testing.T) {
			// テストケースごとに ServiceA にモックを注入
			serviceA := ServiceA{b: tc.mock}

			result, err := serviceA.ProcessingA()

			if result != tc.expected {
				t.Errorf("Expected %s, but got %s", tc.expected, result)
			}

			if (err != nil) != tc.hasError {
				t.Errorf("Expected error status: %v, but got: %v", tc.hasError, err)
			}
		})
	}
}
```

- 単体テストが困難
- 拡張性の欠如

ConsoleLogger の動作が Application のテスト結果に直接影響を与えるため、純粋な単体テストを行うのが難しくなります。
テスト中に ConsoleLogger の振る舞いを制御することができないため、テストの再現性や精度が低下します。また、外部リソース（例えばコンソール出力）に依存するコードのテストは、モックやスタブを使用して内部依存関係を隔離する場合と比べて、一般的に複雑で時間がかかります。依存性注入を使用することで、これらの問題を回避し、より効率的で再現性の高いテストを実現できます。

```go
type Application struct{}

type ConsoleLogger struct{}

func (l ConsoleLogger) Log(message string) {
	fmt.Println("Log to console:", message)
}

func (app *Application) Run() {
	logger := ConsoleLogger{}
	logger.Log("Application is running")
}

func main() {
	app := Application{}
	app.Run()
}
```

- 単体テストが困難
- 拡張性の欠如

ConsoleLogger の動作が Application のテスト結果に直接影響を与えるため、純粋な単体テストを行うのが難しくなります。
テスト中に ConsoleLogger の振る舞いを制御することができないため、テストの再現性や精度が低下します。また、外部リソース（例えばコンソール出力）に依存するコードのテストは、モックやスタブを使用して内部依存関係を隔離する場合と比べて、一般的に複雑で時間がかかります。依存性注入を使用することで、これらの問題を回避し、より効率的で再現性の高いテストを実現できます。

```go
type Application struct {
	logger ConsoleLogger
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
	app.Run()
}
```

上記のコードの問題点は Application が直接 ConsoleLogger に依存していることです。

```go
type Application struct {
	logger ConsoleLogger // 直接 ConsoleLogger に依存
}
```

これにより、以下の問題が発生します。

1. 単体テストが困難
2. 拡張性の欠如

特に 1つ目の問題は重大でしょう。この例ですと、ConsoleLogger の動作が Application のテスト結果に直接影響を与えるため、純粋な単体テストを行うのが非常に困難です。

```go
type Application struct {
	logger ConsoleLogger
}

type ConsoleLogger struct{}

func (l ConsoleLogger) Log(message string) {
	fmt.Println("Log to console:", message)
}

func (app *Application) Run() {
	app.logger.Log("Application is running")
}

func (app *Application) SetLogger(logger ConsoleLogger) {
	app.logger = logger
}

func main() {
	app := &Application{}

	consoleLogger := ConsoleLogger{}
	app.SetLogger(consoleLogger)

	app.Run()
}
```

- 単体テストが困難
- 拡張性の欠如

ConsoleLogger の動作が Application のテスト結果に直接影響を与えるため、純粋な単体テストを行うのが難しくなります。
テスト中に ConsoleLogger の振る舞いを制御することができないため、テストの再現性や精度が低下します。また、外部リソース（例えばコンソール出力）に依存するコードのテストは、モックやスタブを使用して内部依存関係を隔離する場合と比べて、一般的に複雑で時間がかかります。依存性注入を使用することで、これらの問題を回避し、より効率的で再現性の高いテストを実現できます。

```go
type Logger interface {
	Log(message string)
}

type Application struct {
	logger Logger
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
	app.Run()
}
```

```go
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

DIを使用すると、コンポーネントは抽象型（インターフェースや抽象クラス）に依存することが一般的です。これは、LSPの原則とも一致し、多様な実装が同じインターフェースを共有することを促進します。

## インタフェースの使いどころ

これまでインタフェースのメリットのみを解説してきましたが、デメリットもあります。
もちろん抽象型が故のパフォーマンスへの影響もありますが、

以下の書籍では「インタフェース汚染」という命名で解説されています。

「インタフェースで設計するのではなく、インタフェースを発見するもの」という設計思想は、Go言語の核心的な特徴の一つにも反映されています。Go言語では、インタフェースが暗黙的に実装されるという設計原則が取り入れられており、これはプログラマーがソフトウェアの開発過程で自然に適切なインタフェースを見つけ、統合することを促進します。このアプローチは、プログラマーが事前に定義されたインタフェースに縛られることなく、実際のニーズやパターンに応じて柔軟にインタフェースを形成できるように設計されています。

Go言語において、型が特定のインタフェースを実装しているかどうかは、その型がインタフェースで定義された全てのメソッドを持っているかどうかに基づいて自動的に判断されます。これにより、開発者はインタフェースを明示的に宣言する必要がなく、コードの柔軟性と再利用性が向上します。この暗黙的なインタフェースの実装は、プログラムの設計をより動的で自然なものにし、開発者が必要に応じてインタフェースを「発見」し、適用することを容易にします。

結局のところ、この設計思想は、過度な計画や複雑さを避け、より直感的で効率的なソフトウェア開発を促進するGo言語の強力な特徴です。プログラマーは実際の問題解決に必要な機能に集中でき、より洗練された、メンテナンスしやすいコードを作成することが可能になります。


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