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
* 正解: `BCYBBA`
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
	fmt.Print("x:", funcX())
	fmt.Print("y:", funcY())
}
```
:::details 正解を見る
* 正解: `x:0y:1`
* Go Playground: https://go.dev/play/p/673ui2Dvoje
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
* 正解: `funcX recover: panic`
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
後は`defer`に渡しているかどうかの違いだけですね。

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

下記の理解を確認する問題です。
1. `defer`の実行順序
1. `defer`に渡した関数の引数の評価タイミング

#### １．defer の実行順序
`defer`に渡した処理は`return`された後や関数の末尾に到達した後に実行されます。そして、`defer`は LIFO（スタック）のデータ構造になっています。下図の紫の線が通常のプログラムの実行順序で、青の線が`defer`の実行順序になります。
![](/images/golang-defer/golang-defer.drawio.png)

よって、`funcY(s)` → `func()` → `funcY(s)()` → `funcY(s)` → `funcX(s)` → `funcX(s)` の順に実行されます。

1. `funcY(s)`
	* `B`を表示して、無名関数を返します
1. `func()`
	* `C`を表示します
1. `funcY(s)()`
	* `funcY(s)`の戻り値である無名関数を実行し、`Y`を表示します
1. `funcY(s)`
	* `B`を表示して、無名関数を返します
	* ここで返された無名関数は実行されていません
1. `funcX(s)`
	* `B`を表示します
1. `funcX(s)`
	* `A`を表示します
	* ここが`A`になる理由について、続けて解説していきます。

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
関数が最後まで実行された段階では、変数`s`には`B`という値が入っています。その場合、１つ目の`funcX(s)`の処理でも`B`が表示されそうですが、実際は`A`が表示されます。

これは **「defer に渡した関数の引数は即時評価される」** という仕様があるからです。

`defer`に渡した処理が`return`された後や関数の末尾に到達した後に実行されるということに囚われると混乱しますが、逆にこの仕様が無かった場合を想像してみましょう。

この仕様がなければ、変数を追ってコードを読み解くのが難しくなったり、意図しない挙動が起こったりすることが予測できると思います（そもそも変数への再代入は避けるべきですが…）。

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
`defer`に渡した関数が、外側の関数の戻り値にアクセスするためには、名前付き戻り値を使う必要があります。よって、`funcX()`は戻り値にアクセスできず、値は`0`のまま、`funcY()`はアクセスでき、値は`1`となります。

この仕組みはエラー制御で多用されています。

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
上記は、ライブラリ [uber-go/multierr](https://pkg.go.dev/go.uber.org/multierr#hdr-Deferred_Functions) のサンプルコードです。関数内で発生したエラー情報を握り潰さずに`defer`内で発生したエラー情報を加えた上で呼び出し元に返すよう実装されています。

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

`panic`の方のみが表示され、`fatal`は表示されません。これは、`os.Exit`を使ってプログラムを強制終了すると`defer`が実行されないからです。実は`log.Fatal`系は`os.Exit(1)`を呼び出しています。
https://github.com/golang/go/blob/master/src/log/log.go#L260-L276

よって、コードレビューで`log.Fatal()`を見たら注視して、誤っていたら指摘しましょう。`log.Fatal()`が許されるのは`main()`や`init()`、初期化処理くらいだと思います。

## 参考

色々読みましたが、下の2冊が最高にオススメです！

https://www.oreilly.co.jp/books/9784814400041/

https://www.oreilly.co.jp/books/9784873119694/