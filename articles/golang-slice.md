---
title: "Go言語のスライスを深掘り：内部構造とパフォーマンス最適化のポイント"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go]
published: false
---

## 概要
Go言語のスライスは、柔軟で強力なデータ構造です。しかし、その内部構造やパフォーマンスへの影響を理解していないと、予期せぬメモリ消費やパフォーマンスの低下に悩まされることがあります。本記事では、スライスの内部構造から、appendによる動作、パフォーマンス最適化のための初期容量の設定、構造体スライスの効率的な扱い方、さらにGarbage Collection（GC）への影響について詳細に解説します。また、Go 1.21で導入されたslices.Clone関数についても触れ、copyとの違いを具体例で示します。これにより、スライスを使いこなし、Goプログラムのパフォーマンスを最適化するための知識を得られるでしょう。

## スライスの内部構造の理解
スライスは、Go言語で提供される柔軟なデータ構造で、その実態は構造体です。この構造体は、以下の3つのフィールドを持ちます。これらを理解することが、スライスの動作、特に`append()`などの操作を深く理解する上で重要です。

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

この構造からわかるように、スライスは配列のラッパーに過ぎません。スライスは動的にサイズを変更できるため、固定サイズの配列とは異なり、要素の追加や削除が柔軟に行えます。この柔軟性こそが、Go言語においてスライスが広く使用される理由です。

## スライスとパフォーマンスの最適化

スライスを使用する際に注意すべき重要な点は、容量が不足した場合に自動的に新しい配列が割り当てられることです。具体的には、スライスの容量を超える要素を追加すると、新しい配列が作成され、既存の要素がその新しい配列にコピーされます。この操作は、スライスのサイズを動的に拡張するために必要ですが、同時にこのコピー操作がパフォーマンスに悪影響を与える場合があります。

特に大規模なデータを扱う場合、初期容量を適切に設定することが重要です。適切な容量を設定することで、メモリの再割り当てや不要なコピー操作を減らし、効率的なメモリ管理が可能になります。

以下のコード例では、スライスの容量不足によって新しい配列が割り当てられ、要素がコピーされる挙動を確認することができます。

```go
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
    fmt.Printf("%s - 長さ: %d, 容量: %d, ポインタ: %p, 要素: %v\n", label, len(s), cap(s), s, s)
}
```

```txt:実行結果
初期スライス - 長さ: 0, 容量: 2, ポインタ: 0xc00010e010, 要素: []
追加後 1 - 長さ: 1, 容量: 2, ポインタ: 0xc00010e010, 要素: [1]
追加後 2 - 長さ: 2, 容量: 2, ポインタ: 0xc00010e010, 要素: [1 2]
追加後 3 - 長さ: 3, 容量: 4, ポインタ: 0xc000122000, 要素: [1 2 3]
追加後 4 - 長さ: 4, 容量: 4, ポインタ: 0xc000122000, 要素: [1 2 3 4]
追加後 5 - 長さ: 5, 容量: 8, ポインタ: 0xc000124000, 要素: [1 2 3 4 5]
```

この結果から、スライスの容量が不足すると、新しい配列が割り当てられ、容量が2倍に拡張されることが分かります。また、容量が2から4、そして8に増加する際に、配列のポインタが変わっていることから、新しい配列が作成され、既存の要素がその配列にコピーされていることが確認できます。

### 初期容量とパフォーマンス向上のコツ
スライスを作成する際には、あらかじめ予測される最大要素数に基づいて初期容量を設定することが望ましいです。これにより、不要な再割り当てやコピーの回数を減らすことができます。以下に、make関数を使って初期容量を設定する例を示します。

```go
// 予測される最大要素数が100の場合
slice := make([]int, 0, 100)
```
このように設定することで、スライスが初期容量内に収まっている限り、メモリの再割り当ては発生しません。その結果、パフォーマンスの向上が期待できます。

ただし、必要以上に大きな初期容量を設定すると、未使用のメモリ領域が無駄に確保され、システム全体のメモリ使用量が増加し、他のプロセスに悪影響を及ぼす可能性があります。そのため、スライスの容量が事前に明確に決まらない場合は、容量を過度に気にする必要はありません。スライスが持つ動的なサイズ変更の柔軟性を活かしましょう。

### 構造体のスライスとポインタの活用
配列の各要素がフィールド数の多い構造体である場合、再割り当て時のデータコピー量が増大し、パフォーマンスに悪影響を及ぼすことがあります。これを解決するためには、構造体そのものを配列の要素にするのではなく、構造体のポインタを要素として使用する方法が有効です。

以下に、構造体のスライスとポインタのスライスを比較する例を示します。

```go
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

結果からわかるように、ポインタのサイズは構造体のサイズに比べて固定であるため、構造体が大きくなるほど、メモリ使用量の差が顕著になります。大量のデータを持つ構造体を扱う場合、構造体そのものではなくポインタを使用することで、再割り当て時のオーバーヘッドを軽減し、パフォーマンスを向上させることができます。

## スライス操作の注意点とベストプラクティス

スライスを関数に渡して操作する際、期待通りの結果が返らないことがあります。これはスライスの内部構造やメモリ管理に関わる特性によるものです。以下のコード例を使って、スライスの挙動を詳しく見ていきましょう。

```go
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

スライスには、配列の先頭を指すポインタが含まれています。Goには参照渡しはありませんが、スライスの内部構造上、ポインタが値渡しされることで、参照渡しに似た挙動になります。このため、関数外と関数内の両方で同じ配列を参照しています（`0xc00001a018`）。結果として、関数内で変更された内容が関数外にも反映され、スライスは`[0 1 0]`となります。

### 問2

このケースでは、関数内と関数外で異なる配列が参照されています。`append()`によってスライスの容量が不足し、新しい配列が割り当てられたため、関数内の操作結果は関数外に反映されません。そのため、関数外のスライスは変更されず、`[0 0 0]`のままです。

### 問3

スライスの容量を増やすとどうなるでしょうか。関数内と関数外で同じ配列（`0xc00010a060`）を参照していますが、結果は期待通りにはなりません。関数内のスライスは要素が追加されて長さが4になりますが、関数外では長さが3のままです。そのため、関数外では4番目の要素を参照できません。

試しに以下のようにコードを変更してみると、期待通りの結果が得られます。
```diff go
- fmt.Printf("slice：%v\n", slice3)
+ fmt.Printf("slice：%v\n", slice3[0:4])
```

### 問4

関数内で`append()`を正しく動作させたい場合、スライスのポインタを渡す方法があります。しかし、スライス自体がポインタのような役割を果たしているため、スライスのポインタを渡すのはコードの可読性を損なうことがあります。

### ベストプラクティス

代わりに、関数の返り値としてスライスを返す方が、直感的でわかりやすいコードになるでしょう。以下にそのサンプルコードを示します。

```go
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

このコードでは、`appendElements3`関数がスライスを受け取り、新たに要素を追加したスライスを返します。関数内でスライスが新しい配列に再割り当てされた場合でも、関数の返り値を受け取ることで、呼び出し元のスライスが正しく更新されます。これにより、コードの可読性が向上し、挙動も直感的になります。

## スライスのメモリ管理とGCの挙動

Goのスライスは、動的に長さや容量を変更できる柔軟なデータ構造です。そのため、様々な場面で広く使用されています。しかし、スライスのメモリ管理の特性を理解していないと、予期せぬメモリ使用量の増加やGC（ガベージコレクション）が期待通りに動作しないといった問題に直面することがあります。この章では、スライスのメモリ管理とGCへの影響について解説します。

### GCの動作について
まず、非常に大きなスライス（1億要素）を作成し、その一部（最初の1要素）のみを別のスライスとして参照するケースを考えます。この場合、元の大きなスライス（`largeSlice`）を`nil`にしても、参照された部分スライス（`subSlice`）があるため、GCを実行してもメモリが解放されません。

```go
package main

import (
	"fmt"
	"runtime"
	"unsafe"
)

func main() {
	// GC実行前のメモリ統計を取得
	var memStats runtime.MemStats
	runtime.ReadMemStats(&memStats)
	fmt.Printf("GC前: Alloc = %v MiB\n", bToMb(memStats.Alloc))
	fmt.Printf("GC前: TotalAlloc = %v MiB\n", bToMb(memStats.TotalAlloc))
	fmt.Printf("GC前: Sys = %v MiB\n", bToMb(memStats.Sys))
	fmt.Printf("GC前: NumGC = %v\n\n", memStats.NumGC)

	// 大きなスライスを作成してから部分スライスを取得
	largeSlice := make([]int, 100000000) // 1億要素の大きなスライス
	subSlice := largeSlice[:1]           // 最初の1要素だけを参照する

	fmt.Printf("largeSliceのポインタ: %p\n", unsafe.Pointer(&largeSlice[0]))
	fmt.Printf("subSliceのポインタ: %p\n", unsafe.Pointer(&subSlice[0]))

	// largeSliceを解放してGCが解放されるかチェック
	largeSlice = nil

	// メモリを強制的に解放
	runtime.GC()

	// GC実行後のメモリ統計を取得
	runtime.ReadMemStats(&memStats)
	fmt.Printf("GC後: Alloc = %v MiB\n", bToMb(memStats.Alloc))
	fmt.Printf("GC後: TotalAlloc = %v MiB\n", bToMb(memStats.TotalAlloc))
	fmt.Printf("GC後: Sys = %v MiB\n", bToMb(memStats.Sys))
	fmt.Printf("GC後: NumGC = %v\n\n", memStats.NumGC)

	// subSliceを保持し続ける
	fmt.Printf("subSliceを参照中: %v\n", subSlice)

	// もう一度GCを強制実行して確認
	runtime.GC()
	runtime.ReadMemStats(&memStats)
	fmt.Printf("再度GC後: Alloc = %v MiB\n", bToMb(memStats.Alloc))
	fmt.Printf("再度GC後: TotalAlloc = %v MiB\n", bToMb(memStats.TotalAlloc))
	fmt.Printf("再度GC後: Sys = %v MiB\n", bToMb(memStats.Sys))
	fmt.Printf("再度GC後: NumGC = %v\n\n", memStats.NumGC)
}

func bToMb(b uint64) uint64 {
	return b / 1024 / 1024
}
```

```txt:実行結果
GC前: Alloc = 0 MiB
GC前: TotalAlloc = 0 MiB
GC前: Sys = 6 MiB
GC前: NumGC = 0

largeSliceのポインタ: 0xc000180000
subSliceのポインタ: 0xc000180000
GC後: Alloc = 763 MiB
GC後: TotalAlloc = 763 MiB
GC後: Sys = 771 MiB
GC後: NumGC = 2

subSliceを参照中: [0]
再度GC後: Alloc = 0 MiB
再度GC後: TotalAlloc = 763 MiB
再度GC後: Sys = 771 MiB
再度GC後: NumGC = 3
```

これは、`subSlice`が`largeSlice`のメモリを依然として参照しているためです。つまり、`subSlice`がある限り、`largeSlice`全体のメモリが保持され続け、結果としてメモリは解放されず、使用され続けます。

### Cloneによるメモリ解放
この問題を解決するために、Go 1.21以降で提供されている`slices.Clone()`を使う方法があります。`Clone()`を使用すると、`subSlice`が`largeSlice`から完全に独立した新しいスライスとして作成され、元の`largeSlice`のメモリが不要になります。これにより、`largeSlice`を`nil`に設定した後にGCを実行すると、メモリが正しく解放されるようになります。

```diff go
 package main

 import (
 	"fmt"
 	"runtime"
 	"slices"
 	"unsafe"
 )
 
 func main() {
 	// GC実行前のメモリ統計を取得
 	var memStats runtime.MemStats
 	runtime.ReadMemStats(&memStats)
 	fmt.Printf("GC前: Alloc = %v MiB\n", bToMb(memStats.Alloc))
 	fmt.Printf("GC前: TotalAlloc = %v MiB\n", bToMb(memStats.TotalAlloc))
 	fmt.Printf("GC前: Sys = %v MiB\n", bToMb(memStats.Sys))
 	fmt.Printf("GC前: NumGC = %v\n\n", memStats.NumGC)
 
 	// 大きなスライスを作成してから部分スライスを取得
 	largeSlice := make([]int, 100000000)     // 1億要素の大きなスライス
-	subSlice := largeSlice[:1]               // 最初の1要素だけを参照する
+	subSlice := slices.Clone(largeSlice[:1]) // 最初の1要素だけを参照する
 
 	fmt.Printf("largeSliceのポインタ: %p\n", unsafe.Pointer(&largeSlice[0]))
 	fmt.Printf("subSliceのポインタ: %p\n", unsafe.Pointer(&subSlice[0]))
 
 	// largeSliceを解放してGCが解放されるかチェック
 	largeSlice = nil
 
 	// メモリを強制的に解放
 	runtime.GC()
 
 	// GC実行後のメモリ統計を取得
 	runtime.ReadMemStats(&memStats)
 	fmt.Printf("GC後: Alloc = %v MiB\n", bToMb(memStats.Alloc))
 	fmt.Printf("GC後: TotalAlloc = %v MiB\n", bToMb(memStats.TotalAlloc))
 	fmt.Printf("GC後: Sys = %v MiB\n", bToMb(memStats.Sys))
 	fmt.Printf("GC後: NumGC = %v\n\n", memStats.NumGC)
 
 	// subSliceを保持し続ける
 	fmt.Printf("subSliceを参照中: %v\n", subSlice)
 
 	// もう一度GCを強制実行して確認
 	runtime.GC()
 	runtime.ReadMemStats(&memStats)
 	fmt.Printf("再度GC後: Alloc = %v MiB\n", bToMb(memStats.Alloc))
 	fmt.Printf("再度GC後: TotalAlloc = %v MiB\n", bToMb(memStats.TotalAlloc))
 	fmt.Printf("再度GC後: Sys = %v MiB\n", bToMb(memStats.Sys))
 	fmt.Printf("再度GC後: NumGC = %v\n\n", memStats.NumGC)
 }
 
 func bToMb(b uint64) uint64 {
 	return b / 1024 / 1024
 }
```

```diff txt:実行結果
GC前: Alloc = 0 MiB
GC前: TotalAlloc = 0 MiB
GC前: Sys = 6 MiB
GC前: NumGC = 0

largeSliceのポインタ: 0xc000180000
subSliceのポインタ: 0xc000012080
- GC後: Alloc = 763 MiB
+ GC後: Alloc = 0 MiB
GC後: TotalAlloc = 763 MiB
GC後: Sys = 771 MiB
GC後: NumGC = 2

subSliceを参照中: [0]
再度GC後: Alloc = 0 MiB
再度GC後: TotalAlloc = 763 MiB
再度GC後: Sys = 771 MiB
再度GC後: NumGC = 3
```

実際に、GC後にメモリ使用量が減少していることが確認できました。

### メモリリーク防止のポイント
スライスの一部を参照し続けると、GCが期待通りに動作しないことを確認しました。この問題を回避するためには、以下の方法が有効です。

- `slices.Clone()`を使用する
  - スライスの一部を再スライスする際に`Clone()`を使うことで、元のスライスから独立した新しいスライスを作成し、メモリを解放できます
- スライスの扱いに注意する
  - 特に大規模なスライスを操作する場合、意図せずにメモリを保持し続けないよう、再スライスや部分スライスの使用に注意が必要です

スライスのメモリ管理とGCの関係を正しく理解することで、パフォーマンス向上やメモリリークの防止に繋がります。Goで効率的なコードを書くためには、これらのポイントを押さえておくことが非常に重要です。

## 番外編：スライスのコピー方法 — copy と Clone の違い
Go 1.21から導入された`slices.Clone()`は、スライスをコピーする際には`copy()`関数よりもシンプルで便利です。違いを具体的に見てみましょう。

```go
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

### copy関数の問題点

`copy()`関数は、スライスの要素を他のスライスにコピーするために使われますが、正しく動作させるにはコピー先スライスのメモリを事前に確保しておく必要があります。上記のコードでは、`cp`というスライスが初期化されていないため、`copy()`は何もコピーしません。

```go
var cp = make([]int, len(src))
copy(cp, src)
```

このように書けば、`copy()`も正しく動作しますが、コードがやや煩雑になり、余計な手間が増えます。

### Clone関数の便利さ

```go
cl := slices.Clone(src)
```

一方、`slices.Clone()`はスライスをそのままクローンするため、コピー先のメモリ確保を心配する必要がありません。`Clone()`関数は、内部的に適切なサイズの新しいスライスを自動的に作成し、元のスライスのすべての要素を新しいスライスにコピーします。

Clone関数のメリットをまとめます。

- コードがシンプル
  - メモリを手動で確保する必要がないため、コードが短く読みやすくなります。
- エラーの可能性が減る
  - `copy()`で起こりがちな「コピー先スライスが初期化されていない」という問題を回避できます。

## まとめ
スライスはGo言語において非常に柔軟で強力なデータ構造ですが、その背後にあるメモリ管理やパフォーマンスに与える影響を理解することが重要です。スライスの内部構造、`append`の動作、初期容量の適切な設定、構造体のポインタスライスの利用といったポイントを押さえることで、効率的で最適化されたコードを書くことができます。また、GCの挙動や`slices.Clone`の利用法を学ぶことで、メモリリークの防止やスライスのコピー処理の改善にも役立てられるでしょう。これらの知識を活かし、よりパフォーマンスに優れたGoプログラムを構築していきましょう。

## 参考
