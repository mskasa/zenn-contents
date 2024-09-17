---
title: "Goスライス入門から応用まで：構造理解とパフォーマンスの最適化を解説"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go]
published: true
---

## 概要
Go言語のスライスは、柔軟で強力なデータ構造です。

しかし、その内部構造やパフォーマンスへの影響を理解していないと、予期せぬメモリ消費やパフォーマンスの低下に悩まされることがあります。

本記事では、スライスの内部構造から、`append`による動作、パフォーマンス最適化のための初期容量の設定、構造体スライスの効率的な扱い方、さらにGarbage Collection（GC）への影響について解説します。また、Go 1.21で導入された`slices.Clone`関数についても触れ、`copy`との違いを具体例で示します。

本記事が、初学者やスライスを雰囲気で使っている方の助けになれば幸いです。

## スライスの内部構造の理解
スライスは、Go言語で提供される柔軟なデータ構造で、その実態は構造体です。

この構造体は、以下の3つのフィールドを持ちます。これを理解することが、スライスの動作、特に`append`などの操作を深く理解する上で重要です。

- ポインタ（スライスが参照する配列の先頭を指すポインタ）
- 長さ（スライスが現在参照している要素数）
- 容量（スライスが参照できる配列の総要素数）

具体的には、以下のように定義されています。

```go:slice.go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```
[Goソースコード内のslice.go](https://github.com/golang/go/blob/master/src/runtime/slice.go#L15-L19)より

この構造からわかるように、スライスは配列のラッパーに過ぎません。

スライスは動的にサイズを変更できるため、固定サイズの配列とは異なり、要素の追加や削除が柔軟に行えます。この柔軟性こそが、Go言語においてスライスが広く使用される理由です。

しかし、便利が故に特に意識せずに使っていると、思わぬ落とし穴にハマってしまうかもしれません。

次章から、注意すべき点を紹介していきます。

## スライスとパフォーマンスの最適化

スライスを使用する際に注意すべき点は、容量が不足した場合に自動的に新しい配列が割り当てられることです。

具体的には、スライスの容量を超える要素を追加すると、新しい配列が作成され、既存の要素がその新しい配列にコピーされます。この操作は、スライスのサイズを動的に拡張するために必要ですが、同時にこのコピー操作がパフォーマンスに悪影響を与える場合があります。

特に大規模なデータを扱う場合、初期容量を適切に設定することが重要です。適切な容量を設定することで、メモリの再割り当てや不要なコピー操作を減らし、効率的なメモリ管理が可能になります。

以下のサンプルコードでは、スライスの容量不足によって新しい配列が割り当てられ、要素がコピーされる挙動を確認することができます。

```go:容量不足による新しい配列の割り当て
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    slice := make([]int, 0, 2)
    printSliceInfo("初期スライス", slice)
    
    for i := 1; i <= 5; i++ {
        slice = append(slice, i)
        printSliceInfo(fmt.Sprintf("追加後 %d", i), slice)
    }
}

func printSliceInfo(label string, s []int) {
    fmt.Printf("%s - 長さ: %d, 容量: %d, 配列の先頭のポインタ: %p, 要素: %v\n", label, len(s), cap(s), s, s)
}
```

```txt:実行結果
初期スライス - 長さ: 0, 容量: 2, 配列の先頭のポインタ: 0xc000012070, 要素: []
追加後 1 - 長さ: 1, 容量: 2, 配列の先頭のポインタ: 0xc000012070, 要素: [1]
追加後 2 - 長さ: 2, 容量: 2, 配列の先頭のポインタ: 0xc000012070, 要素: [1 2]
追加後 3 - 長さ: 3, 容量: 4, 配列の先頭のポインタ: 0xc000090000, 要素: [1 2 3]
追加後 4 - 長さ: 4, 容量: 4, 配列の先頭のポインタ: 0xc000090000, 要素: [1 2 3 4]
追加後 5 - 長さ: 5, 容量: 8, 配列の先頭のポインタ: 0xc000092000, 要素: [1 2 3 4 5]
```

この結果から、スライスの容量が不足すると、新しい配列が割り当てられ、容量が2倍に拡張されることが分かります。また、容量が2から4、そして8に増加する際に、配列のポインタが変わっていることから、新しい配列が作成され、既存の要素がその配列にコピーされていることが確認できます。

:::message
**補足：**
`fmt.Printf("%p", slice)` ですが、`fmt`ではリフレクションを使ってスライスの内部ポインタ（構造体のフィールドである配列の先頭のポインタ）に直接アクセスしているため、このような記述が可能となっています。
スライス自体のポインタを表示したい場合は`fmt.Printf("%p", &slice)`です。
:::

### 初期容量の設定基準
以上のことから、スライスを作成する際には、**要素数が予測できる場合に限り**、初期容量を設定することが望ましいです。

```go:例）make関数を使って初期容量を100に設定する
slice := make([]int, 0, 100)
```

このように設定することで、スライスが初期容量内に収まっている限り、メモリの再割り当ては発生しません。

ただし、必要以上に大きな初期容量を設定すると、未使用のメモリ領域が無駄に確保され、システム全体のメモリ使用量が増加し、他のプロセスに悪影響を及ぼす可能性があります。

そのため、スライスの容量が事前に明確に決まらない場合は、容量を過度に気にする必要はありません。

特に初期容量を決めず、スライスが持つ動的なサイズ変更の柔軟性を活かしましょう。

### 構造体のスライスとポインタの活用
配列の各要素がフィールド数の多い構造体である場合、再割り当て時のデータコピー量が増大し、パフォーマンスに悪影響を及ぼすことがあります。

これを解決するためには、構造体そのものを配列の要素にするのではなく、構造体のポインタを要素として使用する方法が有効です。

以下に、構造体のスライスとポインタのスライスを比較する例を示します。

```go:構造体のスライスとポインタのスライスのサイズ比較
package main

import (
	"fmt"
	"unsafe"
)

type Employee struct {
	ID        int
	Name      string
	Age       int
	Address   string
	Email     string
	Phone     string
	Position  string
	Salary    float64
	StartDate string
	IsActive  bool
}

func main() {
	// 構造体のスライスを使用する場合
	var employees1 []Employee

	for i := 1; i <= 5; i++ {
		employees1 = append(employees1, Employee{
			ID:        i,
			Name:      fmt.Sprintf("Employee %d", i),
			Age:       20 + i,
			Address:   fmt.Sprintf("Address %d", i),
			Email:     fmt.Sprintf("employee%d@example.com", i),
			Phone:     fmt.Sprintf("555-000%d", i),
			Position:  "Position",
			Salary:    30000.0 + float64(i)*1000,
			StartDate: fmt.Sprintf("2022-01-%02d", i),
			IsActive:  true,
		})
	}
	fmt.Println("構造体のスライス:")
	for _, employee := range employees1 {
		fmt.Printf("%+v\n", employee)
	}

	// 構造体のスライスのメモリ使用量を表示
	structSize := unsafe.Sizeof(employees1[0])
	totalStructSize := structSize * uintptr(len(employees1))
	fmt.Printf("構造体のスライスのメモリ使用量: %d bytes\n", totalStructSize)

	// 構造体のポインタのスライスを使用する場合
	var employees2 []*Employee

	for i := 1; i <= 5; i++ {
		employees2 = append(employees2, &Employee{
			ID:        i,
			Name:      fmt.Sprintf("Employee %d", i),
			Age:       20 + i,
			Address:   fmt.Sprintf("Address %d", i),
			Email:     fmt.Sprintf("employee%d@example.com", i),
			Phone:     fmt.Sprintf("555-000%d", i),
			Position:  "Position",
			Salary:    30000.0 + float64(i)*1000,
			StartDate: fmt.Sprintf("2022-01-%02d", i),
			IsActive:  true,
		})
	}
	fmt.Println("構造体のポインタのスライス:")
	for _, employee := range employees2 {
		fmt.Printf("%p\n", employee)
	}

	// 構造体のポインタのスライスのメモリ使用量を表示
	pointerSize := unsafe.Sizeof(employees2[0])
	totalPointerSize := pointerSize * uintptr(len(employees2))
	fmt.Printf("構造体のポインタのスライスのメモリ使用量: %d bytes\n", totalPointerSize)
}
```

```txt:実行結果
構造体のスライス:
{ID:1 Name:Employee 1 Age:21 Address:Address 1 Email:employee1@example.com Phone:555-0001 Position:Position Salary:31000 StartDate:2022-01-01 IsActive:true}
{ID:2 Name:Employee 2 Age:22 Address:Address 2 Email:employee2@example.com Phone:555-0002 Position:Position Salary:32000 StartDate:2022-01-02 IsActive:true}
{ID:3 Name:Employee 3 Age:23 Address:Address 3 Email:employee3@example.com Phone:555-0003 Position:Position Salary:33000 StartDate:2022-01-03 IsActive:true}
{ID:4 Name:Employee 4 Age:24 Address:Address 4 Email:employee4@example.com Phone:555-0004 Position:Position Salary:34000 StartDate:2022-01-04 IsActive:true}
{ID:5 Name:Employee 5 Age:25 Address:Address 5 Email:employee5@example.com Phone:555-0005 Position:Position Salary:35000 StartDate:2022-01-05 IsActive:true}
構造体のスライスのメモリ使用量: 640 bytes
構造体のポインタのスライス:
0xc0000bc300
0xc0000bc380
0xc0000bc400
0xc0000bc480
0xc0000bc500
構造体のポインタのスライスのメモリ使用量: 40 bytes
```

結果からわかるように、ポインタのサイズは構造体のサイズに比べて固定であるため、構造体が大きくなるほど、メモリ使用量の差が顕著になります。

大量のデータを持つ構造体を扱う場合、構造体そのものではなくポインタを使用することで、再割り当て時のオーバーヘッドを軽減し、パフォーマンスを向上させることができます。

## スライス操作の注意点とベストプラクティス

ここからは、スライスの操作で注意すべき点について見ていきます。

スライスを関数に渡して操作する際、期待通りの結果が返らないことがあります。
これはスライスの内部構造やメモリ管理に関わる特性によるものです。

以下のサンプルコードを使って、スライスの挙動を詳しく見ていきましょう。
ぜひ、問1〜4について、どのように表示されるか考えてみてください。

```go:スライスの挙動に関する問題
package main

import "fmt"

func main() {
	fmt.Println("1. ------------------------------")
	slice1 := make([]int, 3, 3) // [0 0 0]
	fmt.Printf("sliceのポインタ: %p\n", &slice1)
	fmt.Printf("sliceが指す配列のポインタ: %p\n", slice1)
	changeElements(slice1, 1)
	fmt.Printf("slice：%v\n", slice1) // 問1

	fmt.Println("2. ------------------------------")
	slice2 := make([]int, 3, 3) // [0 0 0]
	fmt.Printf("sliceのポインタ: %p\n", &slice2)
	fmt.Printf("sliceが指す配列のポインタ: %p\n", slice2)
	appendElements(slice2, 1)
	fmt.Printf("slice：%v\n", slice2) // 問2

	fmt.Println("3. ------------------------------")
	slice3 := make([]int, 3, 5) // [0 0 0]
	fmt.Printf("sliceのポインタ: %p\n", &slice3)
	fmt.Printf("sliceが指す配列のポインタ: %p\n", slice3)
	appendElements(slice3, 1)
	fmt.Printf("slice：%v\n", slice3) // 問3

	fmt.Println("4. ------------------------------")
	slice4 := make([]int, 3, 3) // [0 0 0]
	fmt.Printf("sliceのポインタ: %p\n", &slice4)
	fmt.Printf("sliceが指す配列のポインタ: %p\n", slice4)
	appendElements2(&slice4, 1)
	fmt.Printf("slice：%v\n", slice4) // 問4
}
func changeElements(s []int, elem int) {
	s[1] = elem
	fmt.Printf("sliceのポインタ(関数内): %p\n", &s)
	fmt.Printf("sliceが指す配列のポインタ(関数内): %p\n", s)
}

func appendElements(s []int, elem int) {
	s = append(s, elem)
	fmt.Printf("sliceのポインタ(関数内): %p\n", &s)
	fmt.Printf("sliceが指す配列のポインタ(関数内): %p\n", s)
}

func appendElements2(s *[]int, elem int) {
	*s = append(*s, elem)
	fmt.Printf("sliceのポインタ(関数内): %p\n", s)
	fmt.Printf("sliceが指す配列のポインタ(関数内): %p\n", *s)
}
```

```txt:実行結果
1. ------------------------------
sliceのポインタ: 0xc000010018
sliceが指す配列のポインタ: 0xc00001a018
sliceのポインタ(関数内): 0xc000010048
sliceが指す配列のポインタ(関数内): 0xc00001a018
slice：[0 1 0]
2. ------------------------------
sliceのポインタ: 0xc000010090
sliceが指す配列のポインタ: 0xc00001a030
sliceのポインタ(関数内): 0xc0000100c0
sliceが指す配列のポインタ(関数内): 0xc00010a030
slice：[0 0 0]
3. ------------------------------
sliceのポインタ: 0xc000010108
sliceが指す配列のポインタ: 0xc00010a060
sliceのポインタ(関数内): 0xc000010138
sliceが指す配列のポインタ(関数内): 0xc00010a060
slice：[0 0 0]
4. ------------------------------
sliceのポインタ: 0xc000010180
sliceが指す配列のポインタ: 0xc00001a048
sliceのポインタ(関数内): 0xc000010180
sliceが指す配列のポインタ(関数内): 0xc00010a090
slice：[0 0 0 1]
```

### 問1

スライスには、配列の先頭を指すポインタが含まれています。

Goには参照渡しはありませんが、スライスの内部構造上、ポインタが値渡しされることで、参照渡しに似た挙動になります。

このため、関数外と関数内の両方で同じ配列を参照しています（`0xc00001a018`）。

結果として、関数内で変更された内容が関数外にも反映され、スライスは`[0 1 0]`となります。

```txt:実行結果
1. ------------------------------
sliceのポインタ: 0xc000010018
sliceが指す配列のポインタ: 0xc00001a018
sliceのポインタ(関数内): 0xc000010048
sliceが指す配列のポインタ(関数内): 0xc00001a018
slice：[0 1 0]
```

### 問2

このケースでは、関数内と関数外で異なる配列が参照されています。

`append`によってスライスの容量が不足し、新しい配列が割り当てられたため、関数内の操作結果は関数外に反映されません。

そのため、関数外のスライスは変更されず、`[0 0 0]`のままです。

```txt:実行結果
2. ------------------------------
sliceのポインタ: 0xc000010090
sliceが指す配列のポインタ: 0xc00001a030
sliceのポインタ(関数内): 0xc0000100c0
sliceが指す配列のポインタ(関数内): 0xc00010a030
slice：[0 0 0]
```

### 問3

問2の結果を踏まえると、スライスの容量を増やせば`[0 0 0 1]`になりそうです。

実際、`append`で新しい配列の割り当ては起きず、関数内と関数外で同じ配列（`0xc00010a060`）を参照しています。しかし、結果は期待通りにはなりません。

```txt:実行結果
3. ------------------------------
sliceのポインタ: 0xc000010108
sliceが指す配列のポインタ: 0xc00010a060
sliceのポインタ(関数内): 0xc000010138
sliceが指す配列のポインタ(関数内): 0xc00010a060
slice：[0 0 0]
```

なぜなら、関数内のスライスは要素が追加されて長さが4になりますが、関数外では長さが3のままだからです。

関数外では4番目の要素を参照できていないということです。

試しに以下のようにコードを変更してみると、期待通りの結果が得られるはずです。

```diff go:4つ目の要素を確認してみる
- fmt.Printf("slice：%v\n", slice3)
+ fmt.Printf("slice：%v\n", slice3[0:4])
```

### 問4

最後に、スライスのポインタを渡してみます。

容量が不足するので、新しい配列が割り当てられますが、そもそものスライスのポインタ（`0xc000010180`）を共有しているため、期待通りの結果になります。

```txt:実行結果
4. ------------------------------
sliceのポインタ: 0xc000010180
sliceが指す配列のポインタ: 0xc00001a048
sliceのポインタ(関数内): 0xc000010180
sliceが指す配列のポインタ(関数内): 0xc00010a090
slice：[0 0 0 1]
```

しかし、スライス自体がポインタのような役割を果たしている（配列の先頭を指すポインタが含まれている）ため、「スライスのポインタ」というものがコードを読む上で理解の妨げになり、可読性の低下に繋がる恐れがあります。

### ベストプラクティス

以上のように、スライスはポインタを保持している構造体であるが故に、関数内での変更が外側にも反映されたり、されなかったりします。

この一見ややこしい問題の解決は非常に簡単で、
- 関数内でスライスを操作する場合はそのスライスを返り値として返すようにする
- 関数を呼び出した側で返り値のスライスを受け取るようにする

これだけです。

このようなコーディング方針にしておけば、難しいことを考える必要はなく、安全かつ確実にスライスを操作できる上に、直感的でわかりやすいコードになります。

以下にそのサンプルコードを示します。

```go:操作したスライスは返す
package main

import "fmt"

func main() {
	fmt.Println("5. ------------------------------")
	slice5 := make([]int, 3, 3) // [0 0 0]
	fmt.Printf("sliceのポインタ: %p\n", &slice5)
	fmt.Printf("sliceが指す配列のポインタ: %p\n", slice5)
	slice5 = appendElements3(slice5, 1)
	fmt.Printf("slice：%v\n", slice5) // 問5
}

func appendElements3(s []int, elem int) []int {
	s = append(s, elem)
	fmt.Printf("sliceのポインタ(関数内): %p\n", &s)
	fmt.Printf("sliceが指す配列のポインタ(関数内): %p\n", s)
	return s
}
```

```txt:実行結果
5. ------------------------------
sliceのポインタ: 0xc000120000
sliceが指す配列のポインタ: 0xc000122000
sliceのポインタ(関数内): 0xc000120030
sliceが指す配列のポインタ(関数内): 0xc00011e030
slice：[0 0 0 1]
```

## スライスのメモリ管理とGCの挙動
この章では、スライスのメモリ管理とGC（ガベージコレクション）について解説します。

スライスのメモリ管理の特性を理解していないと、予期せぬメモリ使用量の増加やGCが期待通りに動作しないといった問題に直面することがあります。

### GCの動作とメモリリーク
分かりやすい例として、非常に大きなスライス（1億要素）を作成し、その一部（最初の1要素）のみを別のスライスとして参照するケースを考えます。

この場合、元の大きなスライス（`largeSlice`）を`nil`にしても、参照された部分スライス（`subSlice`）があるため、GCが実行されてもメモリが解放されません。

```go:メモリリークの発生
package main

import (
	"fmt"
	"runtime"
)

func main() {
	// 大きなスライスを作成してから部分スライスを取得
	largeSlice := make([]int, 100000000) // 1億要素の大きなスライス
	subSlice := largeSlice[:1]           // 最初の1要素だけを参照する

	fmt.Printf("largeSliceが参照する配列のポインタ: %p\n", largeSlice)
	fmt.Printf("subSliceが参照する配列のポインタ: %p\n", subSlice)

	// GC実行前のメモリ統計を取得
	var memStats runtime.MemStats
	runtime.ReadMemStats(&memStats)
	fmt.Printf("GC前: Alloc = %v MiB\n", bToMb(memStats.Alloc))
	fmt.Printf("GC前: TotalAlloc = %v MiB\n", bToMb(memStats.TotalAlloc))

	// largeSliceを解放してGCが解放されるかチェック
	largeSlice = nil

	// メモリを強制的に解放
	runtime.GC()

	// GC実行後のメモリ統計を取得
	runtime.ReadMemStats(&memStats)
	fmt.Printf("GC後: Alloc = %v MiB\n", bToMb(memStats.Alloc))
	fmt.Printf("GC後: TotalAlloc = %v MiB\n", bToMb(memStats.TotalAlloc))

	// subSliceを保持し続ける
	fmt.Printf("subSliceを参照中: %v\n", subSlice)

	// もう一度GCを強制実行して確認
	runtime.GC()
	runtime.ReadMemStats(&memStats)
	fmt.Printf("再度GC後: Alloc = %v MiB\n", bToMb(memStats.Alloc))
	fmt.Printf("再度GC後: TotalAlloc = %v MiB\n", bToMb(memStats.TotalAlloc))
}

func bToMb(b uint64) uint64 {
	return b / 1024 / 1024
}
```

```txt:実行結果
largeSliceが参照する配列のポインタ: 0xc000180000
subSliceが参照する配列のポインタ: 0xc000180000
GC前: Alloc = 763 MiB
GC前: TotalAlloc = 763 MiB
GC後: Alloc = 763 MiB
GC後: TotalAlloc = 763 MiB
subSliceを参照中: [0]
再度GC後: Alloc = 0 MiB
再度GC後: TotalAlloc = 763 MiB
```

`largeSlice`と`subSlice`は同じ配列（`0xc000180000`）を参照しています。

よって、`largeSlice`を`nil`に設定しても`subSlice`がある限り、配列全体のメモリが保持され続け、メモリが解放されません。

### メモリリークの防止
このメモリリークを防止するためには、完全に独立した新しいスライスを作成すれば良さそうです。

Go 1.21以降で提供されている`slices.Clone`を使ってみましょう。

`Clone`を使用すると、`subSlice`が`largeSlice`から完全に独立した新しいスライスとして作成されます。

これにより、`largeSlice`を`nil`に設定した後にGCを実行すると、メモリが正常に解放されるようになります。

```diff go:GCによるメモリ解放確認
 package main
 
 import (
 	"fmt"
 	"runtime"
 )
 
 func main() {
 	// 大きなスライスを作成してから部分スライスを取得
 	largeSlice := make([]int, 100000000) // 1億要素の大きなスライス
-	subSlice := largeSlice[:1]               // 最初の1要素だけを参照する
+	subSlice := slices.Clone(largeSlice[:1]) // 最初の1要素だけを参照する
 
 	fmt.Printf("largeSliceが参照する配列のポインタ: %p\n", largeSlice)
 	fmt.Printf("subSliceが参照する配列のポインタ: %p\n", subSlice)
 
 	// GC実行前のメモリ統計を取得
 	var memStats runtime.MemStats
 	runtime.ReadMemStats(&memStats)
 	fmt.Printf("GC前: Alloc = %v MiB\n", bToMb(memStats.Alloc))
 	fmt.Printf("GC前: TotalAlloc = %v MiB\n", bToMb(memStats.TotalAlloc))
 
 	// largeSliceを解放してGCが解放されるかチェック
 	largeSlice = nil
 
 	// メモリを強制的に解放
 	runtime.GC()
 
 	// GC実行後のメモリ統計を取得
 	runtime.ReadMemStats(&memStats)
 	fmt.Printf("GC後: Alloc = %v MiB\n", bToMb(memStats.Alloc))
 	fmt.Printf("GC後: TotalAlloc = %v MiB\n", bToMb(memStats.TotalAlloc))
 
 	// subSliceを保持し続ける
 	fmt.Printf("subSliceを参照中: %v\n", subSlice)
 
 	// もう一度GCを強制実行して確認
 	runtime.GC()
 	runtime.ReadMemStats(&memStats)
 	fmt.Printf("再度GC後: Alloc = %v MiB\n", bToMb(memStats.Alloc))
 	fmt.Printf("再度GC後: TotalAlloc = %v MiB\n", bToMb(memStats.TotalAlloc))
 }
 
 func bToMb(b uint64) uint64 {
 	return b / 1024 / 1024
 }
```

```diff txt:実行結果
 largeSliceが参照する配列のポインタ: 0xc000100000
 subSliceが参照する配列のポインタ: 0xc0000a4040
 GC前: Alloc = 763 MiB
 GC前: TotalAlloc = 763 MiB
- GC後: Alloc = 763 MiB
+ GC後: Alloc = 0 MiB
 GC後: TotalAlloc = 763 MiB
 subSliceを参照中: [0]
 再度GC後: Alloc = 0 MiB
 再度GC後: TotalAlloc = 763 MiB
```

`largeSlice`と`subSlice`が異なる配列を参照するようになりました。結果、GC後にメモリが解放されていることが確認できます。

特に大規模なスライスを操作する場合、意図せずにメモリを保持し続けないように注意しましょう。

### 番外編：スライスのコピー方法 — copy と Clone の違い
Go 1.21から導入された`slices.Clone`は、スライスをコピーする際には`copy`関数よりもシンプルで便利です。

違いを具体的に見てみましょう。

#### copy関数の問題点

```go:copy関数の問題点
package main

import (
	"fmt"
	"slices"
)

func main() {
	src := []int{0, 1, 2}
	var cp []int
	copy(cp, src)
	fmt.Println("copy:", cp)

	cl := slices.Clone(src)
	fmt.Println("clone:", cl)
}
```

```txt:実行結果
copy: []
clone: [0 1 2]
```

`copy`関数をスライスのコピーに使用する場合、正しく動作させるにはコピー先スライスのメモリを事前に確保しておく必要があります。

上記のコードでは、`cp`というスライスが初期化されていないため、`copy`は何もコピーしません。

```go:copy関数の正しい使い方
var cp = make([]int, len(src))
copy(cp, src)
```

このように書けば、`copy`も正しく動作しますが、コードがやや煩雑になり、余計な手間が増えます。

#### Clone関数の便利さ

一方、`Clone`関数は、内部的に適切なサイズの新しいスライスを自動的に作成し、元のスライスのすべての要素を新しいスライスにコピーします。

つまり、`copy`関数とは違い、コピー先のメモリ確保を意識する必要がなく、コードもシンプルになります。

```go
cl := slices.Clone(src)
```

## まとめ
- 初期容量の設定：
  - 大規模なデータを扱う場合、予測される最大要素数を見積もり、適切な初期容量を設定することで、再割り当ての回数を減らすことが可能です
- ポインタのスライスの活用：
  - 構造体のスライスをポインタとして管理することで、再割り当て時のデータコピーによるパフォーマンス低下を防げます
- メモリ管理とGC：
  - 部分スライスが元のスライスを参照することで発生するメモリリークは、完全に独立した新しいスライスを作成することで解消できます
  - スライスのコピーの作成には、Go 1.21以降で導入された`slices.Clone`の利用をオススメします
- スライス操作のベストプラクティス：
  - スライスを関数で操作する際には、返り値としてスライスを返すことで、スライスの挙動を安全かつ直感的にコントロールすることが可能です

## 参考

https://book.impress.co.jp/books/1122101133
