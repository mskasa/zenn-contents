---
title: "Go言語 スライスの内部構造とパフォーマンスについて理解する"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go]
published: false
---

## 概要



## スライスの構造

https://github.com/golang/go/blob/master/src/runtime/slice.go#L15-L19

### 配列

「Goでは配列を使わずにスライスを使いましょう」とは色々な
え…？じゃあ何で配列なんてあるの？いらなくない？

配列はスライスの内部で使われているからです。
要するに、スライスは配列のラッパーです。便利なラッパーな方を使おうね、それだけです。

## スライスの生成方法

### 初期値がある場合
```go
s := []int{1, 2, 3}
```

### バッファ
バッファを再利用
一度メモリ領域を確保するだけで済む

### それ以外




## 関数の引数としてスライスを渡す

### Go は値渡し

> As in all languages in the C family, everything in Go is passed by value. That is, a function always gets a copy of the thing being passed, as if there were an assignment statement assigning the value to the parameter. (引用元：https://go.dev/doc/faq#pass_by_value)

Goでは、関数に引数を渡した際、必ず引数の値のコピーを作成します。参照渡しはありません。

関数内のappend()に注意する
