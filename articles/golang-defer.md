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
func funcX() {
	fmt.Print("A")
}
func funcY(s string) {
	fmt.Print(s)
}
func funcZ(s string) func() {
	fmt.Print(s)
	return func() { fmt.Print("B") }
}
func main() {
	defer funcX()
	s := "C"
	defer funcY(s)
	s = "D"
	defer funcZ(s)
	defer funcZ(s)()
}
```

:::details 正解を見る
* 正解:
	```
	DBDCA
	```
* Go Playground: https://go.dev/play/p/bxxdw3bRf10
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
### 無名関数
`defer` は下記の様に、無名関数とよく組み合わせて使用されます。
最後の括弧の必要性が分かっている人は、この章を読み飛ばしてください。
```go
defer func() {
	処理
}()
```

さて、下記のコードを見てみましょう。
```go
func funcX() func() {
	return func() { // 無名関数（クロージャ）
		s := "hoge"
		fmt.Println(s)
	}
}
func main() {
	// funcXの戻り値である無名関数を変数に格納してから、無名関数を呼び出す
	f := funcX()
	f()
	// funcXの戻り値である無名関数を即時呼び出す（上記2行をワンライナーで表現）
	funcX()()

	defer func() {
		s := "fuga"
		fmt.Println(s)
	}() // ← 即時呼び出しの括弧
}
```
`funcX()`は関数（クロージャ）を返す関数です。よって、`f := funcX()`で、`f`には関数が格納されます。格納された関数を呼び出すためには、`f()`と書きます。

ここまでの処理を一括で表現すると、一見気持ち悪いですが、`funcX()()`となります。
そして、この括弧がまさしく`defer`の最後にある括弧と同じ意味になります。

イメージしにくい人のために、引数を追加してみました↓

```go
func funcX() func(s string) {
	return func(s string) {
		fmt.Println(s)
	}
}
func main() {
	f := funcX()
	f("hoge")

	funcX()("hoge")
}
```


### 問題１
ポイント別に分解して見ていきましょう。
#### １．defer の実行順序
```go
func main() {
	defer funcX()	  // 4番目
	defer funcY(s)	  // 3番目
	defer funcZ(s)	  // 2番目
	defer funcZ(s)()  // 1番目
}
```
`defer`は LIFO（スタック）のデータ構造になっています。
よって、`funcZ(s)()` → `funcZ(s)` → `funcY(s)` → `funcX()` の順に実行されます。

#### ２．関数から関数（クロージャ）を返す
```go
func funcZ(s string) func() {
	fmt.Print(s)
	return func() { fmt.Print("B") }
}
func main() {
	defer funcZ(s)()
}
```
`defer` から若干話がそれてしまいますが `funcZ(s)()` について補足しておきます。


#### ３．defer に渡した関数の引数は即時評価される

### 問題２
### 問題３