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
### 問題１
### 問題２
### 問題３