---
title: "Go言語 deferの理解を確認する基本問題3選"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go, 初心者]
published: false
---

## 概要
Go言語の`defer`の挙動や仕様を確認する簡単なコードを作成しました。
初心者の方は勉強のために、中級者の方は理解の確認のために解いてみてください。
熟練者の方や、そうでない方でも、何か間違いがあればご指摘お願いします。

## 問題
それぞれ、コンソールに表示される文字列を考えてみてください。
### 問題１

```go
func funcX(s string) {
	fmt.Print(s)
}
func funcY(s string) func() {
	fmt.Print(s)
	return func() { fmt.Print("Y") }
}
func main() {
	s := "A"
	defer funcX(s)
	s = "B"
	defer funcX(s)
	defer funcY(s)
	defer funcY(s)()
	defer func() {
		s = "C"
		fmt.Print(s)
	}()
}
```

:::details 正解を見る
* 正解:
	```
	BCYBBA
	```
* Go Playground: https://go.dev/play/p/zZdvqsi6Onv
:::

### 問題２
```go
func funcX() int {
	x := 0
	defer func() { x = 1 }()
	return x
}
func funcY() (y int) {
	y = 0
	defer func() { y = 1 }()
	return y
}
func main() {
	fmt.Println("x:", funcX())
	fmt.Println("y:", funcY())
}
```
:::details 正解を見る
* 正解:
	```
	x: 0
	y: 1
	```
* Go Playground: https://go.dev/play/p/tdkqZFEksYR
:::

### 問題３
```go
func funcX() {
	defer func() {
		err := recover()
		fmt.Println("funcX recover:", err)
	}()
	log.Panic("panic")
}
func funcY() {
	defer func() {
		err := recover()
		fmt.Println("funcY recover:", err)
	}()
	log.Fatal("fatal")
}
func main() {
	funcX()
	funcY()
}
```
:::details 正解を見る
* 正解:
	```
	funcX recover: panic
	```
* Go Playground: https://go.dev/play/p/mRxpdoAAM7g
:::

## 解説
### 番外編：無名関数について
そもそも無名関数の理解が曖昧だと、各問題の解説を読んでもピンとこないと思います。例えば、`defer`でよく見る下の形は無名関数を使っています。
```go
defer func() {
	s = "C"
	fmt.Print(s)
}()
```

まずは、下記のコードを見てみましょう。
変数`f`に無名関数を格納し、`f()`で格納した無名関数を呼び出しています。
```go
f := func() {
	s := "C"
	fmt.Println(s)
}
f()
```
変数を介さずに無名関数を即時呼び出ししたい場合は下記の様になります。
```go
func() {
	s := "C"
	fmt.Println(s)
}()
```
後は`defer`が前に付いているか付いていないかの違いだけですね。

### 問題１
下記の理解を確認する問題です。
1. `defer`の実行順序
1. `defer`に渡した関数の引数の評価タイミング

ポイント別に見ていきましょう。
#### １．defer の実行順序
`defer`は、関数が`return`された後や関数の末尾に到達した後に、与えられた処理を実行する制御構文です。また、`defer`は LIFO（スタック）のデータ構造になっています。
![](/images/golang-defer/golang-defer.drawio.png)

上図の紫の線が通常の実行順序で、青の線が`defer`の実行順序になります。
よって、`funcY(s)` → `func()` → `funcY(s)()` → `funcY(s)` → `funcX(s)` → `funcX(s)` の順に実行されます。

問題１のコードを再掲します。
```go
func funcX(s string) {
	fmt.Print(s)
}
func funcY(s string) func() {
	fmt.Print(s)
	return func() { fmt.Print("Y") }
}
func main() {
	s := "A"
	defer funcX(s)
	s = "B"
	defer funcX(s)
	defer funcY(s)
	defer funcY(s)()
	defer func() {
		s = "C"
		fmt.Print(s)
	}()
}
```

1. `funcY(s)`
	* `B`を表示して、無名関数を返します
1. `func()`
	* `C`を表示します
1. `funcY(s)()`
	* `funcY`の戻り値である無名関数を実行し、`Y`を表示します
1. `funcY(s)`
	* `B`を表示して、無名関数を返します
	* ここで返された無名関数は実行されていません
1. `funcX(s)`
	* `B`を表示します
1. `funcX(s)`
	* `A`を表示します
	* ここが`A`になる理由が２つ目のポイントです

#### ２．defer に渡した関数の引数は即時評価される
```go
func funcX(s string) {
	fmt.Print(s)
}
func main() {
	s := "A"
	defer funcX(s)
	s = "B"
	defer funcX(s)
}
```
復習ですが、`defer`は、関数が`return`された後や関数の末尾に到達した後に、与えられた処理を実行する制御構文でした。
関数が最後まで実行された段階では、変数`s`には`B`という値が入っています。その場合、１つ目の`funcX(s)`の処理でも`B`が表示されそうですが、実際は`A`が表示されます。
これは **「defer に渡した関数の引数は即時評価される」** という仕様があるからです。
実際、この仕様がなければ、コードを読み解くのが複雑になったり、意図しない挙動が起こったりするでしょう。そもそも変数への再代入は避けるべきですが…。

### 問題２

`defer`された関数が、外側の関数の戻り値にアクセスするためには、名前付き戻り値を使う必要があります。この仕組みはエラー制御で実用されています。

```go
func sendRequest(req Request) (err error) {
	conn, err := openConnection()
	if err != nil {
		return err
	}
	defer func() {
		err = multierr.Append(err, conn.Close())
	}()
	// ...
}
```
上記は、ライブラリ [uber-go/multierr](https://pkg.go.dev/go.uber.org/multierr#hdr-Deferred_Functions) のサンプルコードです。関数内で発生したエラー情報を握り潰さずに `defer`内で発生したエラー情報を呼び出し元に伝えています。

### 問題３

