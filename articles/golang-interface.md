---
title: "Go言語 インタフェースのメリットと使いどころを分かりやすく解説"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go]
published: false
---

## 概要

**本記事の流れは以下の通りです。**
1. インタフェースによる抽象化について、簡単な例で解説します
2. インタフェースのメリットについて、以下3つの観点から解説します
   - ソースコードの再利用性
   - ソースコードの拡張性
   - ソースコードのテスタビリティ
3. インタフェースの使いどころについて解説します

**本記事の想定読者は以下の通りです。**
- Go言語でのインタフェースの定義方法など、基本的な部分は勉強済みである
- インタフェースのメリットや使いどころが分からない
- 他言語と Go言語のインタフェースの違いについて知りたい

## インタフェースによる抽象化

インタフェースの役割は抽象化です。この抽象化が様々なメリットをもたらします。

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
```

この例では、異なる型である`Circle`と`Rectangle`が同じインタフェース`Shape`を通して、同一のシグネチャ`Area() float64`で異なる動作をします。

「面積を計算すること」を「`Area`というメソッド名で、`float64`の値を返すこと」に抽象化しているわけですね。これで、図形の種類に関わらず、同じ方法（シグネチャ）で、型を操作できるようになりました。

しかし、このような簡易的な例ですと、あまりそのメリットが伝わりません。

次章では、より実践的な例で、インタフェースの抽象化によるメリットを見ていきましょう。

## インタフェースのメリット

前章では、インタフェースが型の振る舞いを抽象化することについて解説しました。

本章では、以下3つの観点からインタフェースの抽象化によるメリットを解説します。
- ソースコードの再利用性
- ソースコードの拡張性
- ソースコードのテスタビリティ

### ソースコードの再利用性の向上（共通化）

まずは以下のサンプルコードを見てください。

```go
func BufferToUpper(buf *bytes.Buffer) {
	data := make([]byte, buf.Len())
	len, _ := buf.Read(data)
	str := string(data[:len])

	result := strings.ToUpper(str)
	fmt.Println(result)
}

func StringToUpper(s *strings.Reader) {
	data := make([]byte, 300)
	len, _ := s.Read(data)
	str := string(data[:len])

	result := strings.ToUpper(str)
	fmt.Println(result)
}

func FileToUpper(f *os.File) {
	data := make([]byte, 300)
	len, _ := f.Read(data)
	str := string(data[:len])

	result := strings.ToUpper(str)
	fmt.Println(result)
}

func main() {
	// バッファからの読み取り
	var buf bytes.Buffer
	buf.WriteString("This is data from bytes.Buffer.")
	BufferToUpper(&buf)

	// 文字列リーダーからの読み取り
	strReader := strings.NewReader("This is a sample string.")
	StringToUpper(strReader)

	// ファイルからの読み込み
	file, err := os.Open("example.txt")
	if err != nil {
		fmt.Println("Error opening file:", err)
	}
	defer file.Close()

	FileToUpper(file)
}
```

`BufferToUpper()`、`StringToUpper()`、`FileToUpper()`は、いずれも引数として受け取った値を大文字に変換する関数です。実装をよく見ると、引数が異なるだけで、処理内容はほとんど同じなので、できれば共通化したいです。

現状、それぞれが具体的なデータソース（バッファ、文字列、ファイル）を読み込む関数として実装されているので、「読み込むこと」を抽象化したインタフェース`io.Reader`を利用すれば共通化できそうです。

`io.Reader`は、Go言語標準の `io`パッケージに定義されているインタフェースで、この例にあるすべての型（`bytes.Buffer`、`strings.Reader`、`os.File`）は`io.Reader`を実装しています。

:::details 詳細
```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```
出典:https://github.com/golang/go/blob/master/src/io/io.go

```go
func (b *Buffer) Read(p []byte) (n int, err error) {
	b.lastRead = opInvalid
	if b.empty() {
		// Buffer is empty, reset to recover space.
		b.Reset()
		if len(p) == 0 {
			return 0, nil
		}
		return 0, io.EOF
	}
	n = copy(p, b.buf[b.off:])
	b.off += n
	if n > 0 {
		b.lastRead = opRead
	}
	return n, nil
}
```
出典：https://github.com/golang/go/blob/master/src/bytes/buffer.go

```go
func (r *Reader) Read(b []byte) (n int, err error) {
	if r.i >= int64(len(r.s)) {
		return 0, io.EOF
	}
	r.prevRune = -1
	n = copy(b, r.s[r.i:])
	r.i += int64(n)
	return
}
```
出典：https://github.com/golang/go/blob/master/src/strings/reader.go

```go
func (f *File) Read(b []byte) (n int, err error) {
	if err := f.checkValid("read"); err != nil {
		return 0, err
	}
	n, e := f.read(b)
	return n, f.wrapErr("read", e)
}
```
出典：https://github.com/golang/go/blob/master/src/os/file.go
:::


よって、以下のように1つの関数に共通化できます。

```go
func ToUpper(r io.Reader) {
	data := make([]byte, 300)
	len, _ := r.Read(data)
	str := string(data[:len])

	result := strings.ToUpper(str)
	fmt.Println(result)
}

func main() {
	// バッファからの読み取り
	var buf bytes.Buffer
	buf.WriteString("This is data from bytes.Buffer.")
	ToUpper(&buf)

	// 文字列リーダーからの読み取り
	strReader := strings.NewReader("This is a sample string.")
	ToUpper(strReader)

	// ファイルからの読み込み
	file, err := os.Open("example.txt")
	if err != nil {
		fmt.Println("Error opening file:", err)
	}
	defer file.Close()

	ToUpper(file)
}
```

結果的に、関数`ToUpper()`が複数回呼び出されるようになります。
ソースコードの共通化、これ即ちソースコードの再利用性向上というわけです。

以上が、インタフェースのメリットの1つであるソースコードの共通化で、これがソースコードの再利用性向上に寄与しています。

### ソースコードの拡張性の向上

本章では、Go言語のビルトインの`error`インタフェースを例に、ソースコードの拡張性について見ていきます。

`error`インタフェースは以下の様に非常にシンプルな定義です。唯一のメソッドは`Error()`で、これはエラーメッセージを文字列で返すだけです。このシンプルさが拡張性を高める一因となっています。

```go
type error interface {
	Error() string
}
```

例として、前章のサンプルコードにあった、`os.Open()`で`no such file or directory`のエラーが発生した場合を紐解いてみます。

```go
file, err := os.Open("example.txt")
if err != nil {
	errType := reflect.TypeOf(err)
	fmt.Println("Error opening file:", err)
}
```

`err`の型が知りたいため、reflectパッケージの`TypeOf()`処理を追加しています。
結果、`fs`パッケージの`PathError`型が返ってきているようです。

```
Error type: *fs.PathError
Error opening file: open example.txt: no such file or directory
```

ソースコードを辿っていくと、以下の実装に辿り着きました。`PathError`型が `error`インタフェースを実装していることが分かりますね。

```go
type PathError struct {
	Op   string
	Path string
	Err  error
}

func (e *PathError) Error() string { return e.Op + " " + e.Path + ": " + e.Err.Error() }
```
出典：https://github.com/golang/go/blob/master/src/io/fs/fs.go#L244-L250

この例が示すことは、誰でも簡単に独自のカスタムエラーを作成できるということです。例えば、`MyError`型を定義する場合は以下の様になります。

```go
type MyError struct {
	Code    int
	Message string
}

func (e *MyError) Error() string {
	return fmt.Sprintf("error: code=%d, message=%s", e.Code, e.Message)
}

func doSomething() error {
	return &MyError{
		Message: "An error occurred",
		Code:    1,
	}
}

func main() {
	err := doSomething()
	if err != nil {
		fmt.Println(err)
	}
}
```

:::message
補足：`Error()`はどのタイミングで実行される？
fmtパッケージに errorインタフェースの実装を渡すことで、内部的に`err.Error()`が実行されます。上のコードでは`fmt.Println(err)`の部分です。
:::

このように、独自のエラー型を定義し、`Error()`メソッドを実装することで、特定の情報や振る舞いを持つ新しいカスタムエラーを簡単に作成できます。以上のことから、インタフェースがソースコードの拡張性に寄与していることが分かります。

### ソースコードのテスタビリティ向上

インタフェースを利用することで、コンポーネント間を疎結合にすることができます。

まずは以下のサンプルコードを見てください。

```go
type ServiceA struct{}

type ServiceB struct{}

func (a ServiceA) ProcessingA() (string, error) {
	// ServiceB型を実装
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

上のコードでは、`ServiceA`型の`ProcessingA()`内で、`ServiceB`型を実装しています。そして、`ServiceB`型の`ProcessingB()`の処理結果に応じて`ProcessingA()`の処理結果が変わります。

つまり、`ServiceA`型が`ServiceB`型に依存しており、`ProcessingA()`の単体テストが不可能な状態であるといえます。

`ProcessingB()`が簡単な処理で、自チームで開発されているのであれば問題ないかもしれません。しかし、複雑な処理であったり、他チームが開発しているとなると、開発上の大きな問題になってしまいます。

これを解決するのが、オブジェクト指向言語でいうところの依存性の注入（Dependency Injection, DI）になります。DIは難しくとらわれがちですが、ただ単に外部からインスタンスを渡してあげることに他なりません。

:::message
Go言語はオブジェクト指向言語ではないため、オブジェクトやインスタンスという概念がありません。そのため「型」や「型の実装」という表現をここでは使用しますが、オブジェクト指向言語に慣れている方は「型」を「オブジェクト」、「型の実装」を「インスタンス」と読み替えていただくと咀嚼しやすいと思います。
:::

では、依存性(`ServiceB`型への依存)を外部から注入するように変更してみます。

```diff go
- type ServiceA struct{}
+ type ServiceA struct {
+ 	b ServiceB
+ }

type ServiceB struct{}

func (a ServiceA) ProcessingA() (string, error) {
-	// ServiceB型を実装
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
+		b: ServiceB{},	// ServiceB型の実装を注入
+	}
	a.ProcessingA()
}
```

`ServiceB`型の実装を外部から渡すように変更しました。

しかし、具象型（構造体）である`ServiceB`に依存していることに変わりはありません。抽象型（インタフェース）である`ServiceBInterface`を定義し、そのインタフェースに依存するように変更します。

```diff go
type ServiceA struct {
-	b ServiceB
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

これで、具体的な実装から分離することができました。以下の様に`ProcessingB()`のモックを作成することで、`ProcessingA()`の単体テストを記述することができます。

```go
type MockServiceB struct {
	Result int
	Err    error
}

func (m MockServiceB) ProcessingB() (int, error) {
	return m.Result, m.Err
}

func TestProcessingA(t *testing.T) {
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

以上が、具体的な実装ではなく抽象（インタフェース）に依存させることで疎結合を促進させ、

## インタフェースの使いどころ

Go言語におけるインタフェースの最大の特徴は、インタフェースが暗黙的に実装されるということです。この設計原則は「後からでもインタフェースを適用しやすいようにしているから、必要だと判断した時に追加してね！」という Go言語設計者からのメッセージとも読み取れるでしょう。

詳しく解説します。今更ですが、Go言語では、型が特定のインタフェースを実装しているかどうかは、その型がインタフェースで定義された全てのメソッドを持っているかどうかに基づいて自動的に判断されます。これに対して、例えば Java言語では、クラスが特定のインターフェースを実装する場合、そのインターフェースを明示的に`implements`キーワードを使って宣言する必要があります。Go言語では、そのような明示的な宣言が不要であるため、後からインタフェースを追加する際の既存コードへの影響が少ないのです。

以上のことから、Go言語では、インタフェースを中心に設計するのではなく、開発過程でインタフェースを発見し、必要に応じて適用するアプローチが合理的です。

これによって、インタフェースが必要以上に増えて、実装が複雑化することを抑止できます。また、開発者が事前に定義されたインタフェースに縛られることなく、実際のニーズやパターンに応じて柔軟にインタフェースを定義できます。

## まとめ

- インタフェースの役割は抽象化である
  - 抽象化によるメリットは以下の通りである
    1. ソースコードの再利用性向上（共通化）
    2. ソースコードの拡張性向上
    3. ソースコードのテスタビリティ向上
- Go言語における開発では、インタフェースは必要に応じて適用するのが合理的である
  - インタフェースが必要以上に増えて、実装が複雑化することを抑止できる
  - Go言語のインタフェースは暗黙的に実装されるため、後からインタフェースを追加するのが比較的容易である

## 参考
