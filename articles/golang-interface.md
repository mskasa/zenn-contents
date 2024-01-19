---
title: "Go言語 インタフェースのメリットと使いどころを分かりやすく解説"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go, interface, di]
published: true
---

## 概要

- 本記事の想定読者は以下の通りです
  - インタフェースの定義方法など、基本的な部分は勉強済みである
  - インタフェースのメリットが分からない
  - インタフェースの使いどころが分からない

- 本記事の流れは以下の通りです
  1. インタフェースによる抽象化について解説
  2. インタフェースのメリットについて解説
  3. インタフェースの使いどころについて解説

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

この例では、異なる型である`Circle`と`Rectangle`がインタフェース`Shape`を通して、同一シグネチャ`Area() float64`で異なる動作をします。

「面積を計算すること」を「`Area`というメソッド名で、`float64`の値を返すこと」に抽象化しているわけですね。これで、図形の種類に関わらず、同じ方法（シグネチャ）で、型を操作できるようになりました。

しかし、このような簡易的な例ですと、あまりそのメリットが伝わりません。

次章では、より実践的な例で、インタフェースの抽象化によるメリットを見ていきましょう。

## インタフェースのメリット

前章では、インタフェースが型の振る舞いを抽象化することについて解説しました。

本章では、以下3つの観点からインタフェースの抽象化によるメリットを解説します。
- ソースコードの再利用性
- ソースコードの拡張性
- ソースコードのテスタビリティ

### ソースコードの再利用性向上（共通化）

本章では、インタフェースによるソースコードの共通化と、その結果もたらされる再利用性の向上について見ていきます。

まずは以下のサンプルコードを見てください。

```go
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

`StringToUpper()`と`FileToUpper()`は、どちらも引数として受け取った値を大文字に変換する関数です。引数が異なりますが処理内容は同じなので、可能であれば共通化したいですね。

現状、それぞれが具体的なデータソースを読み込む関数として実装されているので、「読み込むこと」を抽象化したインタフェース`io.Reader`を利用すれば共通化できそうです。

`io.Reader`は、Go言語標準の `io`パッケージに定義されているインタフェースで、`strings.Reader`型と`os.File`型はどちらも`io.Reader`を実装しています。

:::details 詳細
- `io.Reader`：https://github.com/golang/go/blob/master/src/io/io.go
```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```
- `strings.Reader`：https://github.com/golang/go/blob/master/src/strings/reader.go
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
- `os.File`：https://github.com/golang/go/blob/master/src/os/file.go
```go
func (f *File) Read(b []byte) (n int, err error) {
	if err := f.checkValid("read"); err != nil {
		return 0, err
	}
	n, e := f.read(b)
	return n, f.wrapErr("read", e)
}
```
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

結果的に、`ToUpper()`が複数回呼び出されるようになります。
ソースコードの共通化、これ即ちソースコードの再利用性向上というわけです。

以上が、インタフェースのメリットの1つであるソースコードの共通化であり、これがソースコードの再利用性向上に寄与しています。

### ソースコードの拡張性向上

本章では、Go言語のビルトインの`error`インタフェースを例に、ソースコードの拡張性について見ていきます。

`error`インタフェースは以下の様に非常にシンプルな定義です。唯一のメソッドは`Error()`で、これはエラー情報を文字列で返すだけです。このシンプルさが拡張性を高める一因となっています。

```go
type error interface {
	Error() string
}
```

例として、前章のサンプルコードにあった、`os.Open()`で`no such file or directory`のエラーが発生した場合を紐解いてみます。

```go diff
 file, err := os.Open("example.txt")
 if err != nil {
+	errType := reflect.TypeOf(err)
+	fmt.Println("Error type:", errType)
 	fmt.Println("Error opening file:", err)
 }
```

`err`の型を見たいので、`reflect`パッケージの`TypeOf()`処理を追加しています。
結果、`fs`パッケージの`PathError`型が返ってきているようです。

```
Error type: *fs.PathError
Error opening file: open example.txt: no such file or directory
```

`PathError`型は以下の様に定義されています。`Error()`メソッドを備えており、`error`インタフェースを満たしていますね。

```go
type PathError struct {
	Op   string
	Path string
	Err  error
}

func (e *PathError) Error() string { return e.Op + " " + e.Path + ": " + e.Err.Error() }
```
出典：https://github.com/golang/go/blob/master/src/io/fs/fs.go#L244-L250

このように、Go言語の`error`インターフェースは、標準ライブラリやサードパーティのライブラリにおいて広く利用されています。

また、この例が示すことは、誰でも簡単に新しいカスタムエラー型を作成できるということです。

試しに、新しいカスタムエラー型を作成し、エラーハンドリングを拡張してみましょう。

以下はそのサンプルコードです。新しいカスタムエラー型である`MyError`型を定義し、エラーが致命的かどうかを判断する処理を追加してみました。非常に簡単に拡張できていますね。

```go
package main

import (
	"fmt"
)

type MyError struct {
	Code    int
	Message string
}

func (e *MyError) Error() string {
	return fmt.Sprintf("error: code=%d, message=%s", e.Code, e.Message)
}

func (e *MyError) IsCritical() bool {
	return e.Code >= 100
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

		if myErr, ok := err.(*MyError); ok {
			if myErr.IsCritical() {
				fmt.Println("Critical error encountered!")
			} else {
				fmt.Println("Non-critical error encountered.")
			}
		}
	}
}

```

:::message
**補足：`Error()`メソッドはどこで実行される？**
`fmt`パッケージに`error`インタフェースを実装するオブジェクトを渡すことで、内部的に`Error()`メソッドが実行されます。この例では、`fmt.Println(err)`の部分で実行されています。
:::

ここまでの内容をまとめると、以下になります。

- シンプルなインターフェース定義
  - `Error()`メソッドを実装するだけで、`error`インターフェースを満たすことができます。このシンプルさのおかげで、開発者は新しいカスタムエラー型を簡単に追加できます。

- 広範囲な互換性の提供
  - Go言語の`error`インターフェースは、標準ライブラリやサードパーティのライブラリにおいて広く利用されています。そのため、新しいカスタムエラー型を導入しても、既存のエラーハンドリングを大幅に変更する必要はありません。

以上、インタフェースによるソースコードの拡張性向上について、`error`インタフェースを例に解説しました。

### ソースコードのテスタビリティ向上

本章では、インタフェースによるテスタビリティ向上について見ていきます。

あるオブジェクトが具体的な実装に依存している場合、それを抽象（インタフェース）への依存に変更することで、コンポーネント間を疎結合にすることができます。

疎結合にすることでコンポーネントは交換可能になり、テスタビリティが向上します。

まずは以下のサンプルコードを見てください。

```go
type ServiceA struct{}

type ServiceB struct{}

func (a ServiceA) ProcessingA() string {
	// ServiceB型のインスタンスを生成
	b := ServiceB{}
	res := b.ProcessingB()
	// ProcessingBの処理結果に応じて何か色々する
	if res == 0 {
		return "hello"
	} else {
		return "world"
	}
}

func (b ServiceB) ProcessingB() int {
	// 何か複雑な処理
	return 0
}

func main() {
	a := ServiceA{}
	result := a.ProcessingA()
	fmt.Println(result)
}
```

`ServiceA`内で`ServiceB`のインスタンスを生成しており、`ProcessingB()`の処理結果に応じて`ProcessingA()`の処理結果が変わるイメージを伝えるためのコードです（色々簡略化していますがご愛嬌ということで）。

つまりは、`ServiceA`が`ServiceB`に依存しており、`ProcessingA()`の単体テストが不可能な状態であるといえます。

`ProcessingB()`が簡単な処理で、自チームで開発されているのであれば問題ないかもしれません。しかし、複雑な処理であったり、他チームが開発しているとなると、開発上の大きな問題になってしまいます。

これを解決するのが、俗に言う依存性の注入（Dependency Injection, DI）になります。DIは言葉が先行しているせいで何やら難しくとらわれがちですが、外部からインスタンスを渡してあげる程度の理解で問題ないと思います。

では、依存性(`ServiceB`への依存)を外部から注入するように変更してみます。

```diff go
-type ServiceA struct{}
+type ServiceA struct {
+	b ServiceB
+}

 type ServiceB struct{}

 func (a ServiceA) ProcessingA() string {
-	// ServiceB型のインスタンスを生成
-	b := ServiceB{}
-	res := b.ProcessingB()
+	res := a.b.ProcessingB()
 	// ProcessingBの処理結果に応じて何か色々する
 	if res == 0 {
 		return "hello"
 	} else {
 		return "world"
 	}
 }

 func (b ServiceB) ProcessingB() int {
 	// 何か複雑な処理
 	return 0
 }

 func main() {
-	a := ServiceA{}
+	a := ServiceA{
+		b: ServiceB{},	// ServiceB型のインスタンスを渡す
+	}
 	result := a.ProcessingA()
 	fmt.Println(result)
 }
```

`ServiceB`型のインスタンスを外部から渡すように変更しました。

しかし、具象型（構造体）である`ServiceB`に依存していることに変わりはありません。

抽象型（インタフェース）である`ServiceBInterface`を定義し、そのインタフェースに依存するように変更します。

```diff go
 type ServiceA struct {
-	b ServiceB
+	b ServiceBInterface
 }

+type ServiceBInterface interface {
+	ProcessingB() int
+}
```

:::details コード全体
```go
type ServiceA struct {
	b ServiceBInterface
}

type ServiceB struct{}

type ServiceBInterface interface {
	ProcessingB() int
}

func (a ServiceA) ProcessingA() string {
	res := a.b.ProcessingB()
	// ProcessingBの処理結果に応じて何か色々する
	if res == 0 {
		return "hello"
	} else {
		return "world"
	}
}

func (b ServiceB) ProcessingB() int {
	// 何か複雑な処理
	return 0
}

func main() {
	a := ServiceA{
		b: ServiceB{}, // ServiceB型のインスタンスを渡す
	}
	result := a.ProcessingA()
	fmt.Println(result)
}
```
:::

これで、具体的な実装から分離することができました。

以下の様に`ProcessingB()`のモックを作成することで、`ProcessingA()`の単体テストが可能になります。

```go
type MockServiceB struct {
	ProcessingBFunc func() int
}

func (m *MockServiceB) ProcessingB() int {
	return m.ProcessingBFunc()
}

func TestProcessingA(t *testing.T) {
	tests := []struct {
		name          string
		mockBResponse int
		want          string
	}{
		{"ProcessingB returns 0", 0, "hello"},
		{"ProcessingB returns non-zero", 1, "world"},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			mockB := &MockServiceB{
				ProcessingBFunc: func() int { return tt.mockBResponse },
			}

			a := ServiceA{b: mockB}

			result := a.ProcessingA()

			if result != tt.want {
				t.Errorf("ProcessingA() = %v, want %v", result, tt.want)
			}
		})
	}
}
```

以上が、インタフェースによるソースコードのテスタビリティ向上についての解説になります。

## インタフェースの使いどころ

ここまで、インタフェースの役割とメリットについて解説してきました。

では、インタフェースをいつ使えばよいのでしょうか。
答えは非常にシンプルで、「必要になったときに使う」です。

Go言語のインタフェースの最大の特徴は、インタフェースが暗黙的に実装されることです。

この Go言語特有の設計からは「後からでもインタフェースを追加しやすいようにしているから、必要だと判断した時に追加してください」というメッセージが汲み取れます。

詳しく解説しましょう。

今更ですが、Go言語では、型が特定のインタフェースを実装しているかどうかは、その型がインタフェースで定義された全てのメソッドを持っているかどうかに基づいて自動的に判断されます。

これに対して、例えば Java言語では、クラスが特定のインターフェースを実装する場合、そのインターフェースを明示的に`implements`キーワードを使って宣言する必要があります。

Go言語では、そのような明示的な宣言が不要であるため、後からインタフェースを追加する際の既存コードへの影響が少ないのです。

以上のことから、Go言語では、インタフェースを中心に設計するのではなく、開発過程でインタフェースを発見し、必要に応じて適用するアプローチが合理的です。

これによって、インタフェースが必要以上に増えて、実装が複雑化することを抑止できます。

また、開発者が事前に定義されたインタフェースに縛られることなく、実際のニーズやパターンに応じて柔軟にインタフェースを定義できます。

## まとめ

- インタフェースの役割は抽象化である
  - 抽象化によるメリットは以下の通り
    - ソースコードの再利用性向上（共通化）
    - ソースコードの拡張性向上
    - ソースコードのテスタビリティ向上
- Go言語での開発において、インタフェースは必要に応じて追加するのが合理的である
  - Go言語のインタフェースは暗黙的に実装されるため、後からインタフェースを追加するのが比較的容易であるため
  - インタフェース中心の設計を避けることで以下のメリットがある
    - インタフェースが必要以上に増えて、実装が複雑化することを抑止できる
    - 開発段階で、実際のニーズやパターンに応じて柔軟にインタフェースを定義できる

## 参考

https://book.impress.co.jp/books/1122101133