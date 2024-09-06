---
title: "Go言語 スライスの内部構造とパフォーマンスについて理解する"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go]
published: false
---

## 概要
Go言語のスライスは、柔軟で強力なデータ構造です。しかし、その内部構造やパフォーマンスへの影響を理解していないと、予期せぬメモリ消費やパフォーマンスの低下に悩まされることがあります。本記事では、スライスの内部構造から、appendによる動作、パフォーマンス最適化のための初期容量の設定、構造体スライスの効率的な扱い方、さらにGarbage Collection（GC）への影響について詳細に解説します。また、Go 1.21で導入されたslices.Clone関数についても触れ、copyとの違いを具体例で示します。これにより、スライスを使いこなし、Goプログラムのパフォーマンスを最適化するための知識を得られるでしょう。

## スライスの内部構造の理解
スライスの実態は構造体で、以下の3つのフィールドを持ちます。  
これを理解しているかどうかが後述する`append()`の挙動など、スライスの理解度に直結します。

1. ポインタ（スライスが参照する配列のポインタ）
1. 長さ（スライスが参照する配列の現在の要素数）
1. 容量（スライスが参照する配列の最大要素数）

```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```
[Goソースコード内のslice.go](https://github.com/golang/go/blob/master/src/runtime/slice.go#L15-L19)より

すなわち、スライスは配列のラッパーです。動的にサイズを変更できるため、固定サイズの配列と異なり、要素の追加や削除が簡単に行えます。この柔軟性により、Go言語ではスライスが一般的に使用されます。

## スライスとパフォーマンスの最適化

スライスの挙動で注意すべき点があります。それは、スライスの容量が不足した場合、新しい配列が自動的に割り当てられるということです。

つまり、既存の要素が新しい配列にコピーされます。これにより、スライスのサイズを動的に拡張できますが、この内部で発生するコピー操作がパフォーマンスに影響することがあります。

そのため、特に大規模なデータを扱う際には、初期容量を適切に設定することが重要です。適切な容量の設定により、不要なメモリ再割り当てとコピーの回数を減らし、効率的なメモリ管理が可能になります。

スライスの容量不足時、新しい配列が割り当てられ、既存の要素がコピーされる挙動を実際に確認してみましょう。以下、具体的なコード例とその実行結果です。

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

この結果から、スライスの容量が不足すると新しい配列が割り当てられ、容量が2倍に拡張されていることが分かります。容量が2から4、そして8に増加する際に、配列のポインタが変わっている点に注目してください。新しい配列が作成され、既存の要素が新しい配列にコピーされていることが確認できます。

### 初期容量とパフォーマンス向上のコツ
スライスを作成する際、最大要素数をある程度予測して初期容量を設定しましょう。これにより、再割り当てとコピーの回数を減らすことができます。以下に、make関数を使用して初期容量を設定する例を示します。

```go
// 予測される最大要素数が100の場合
slice := make([]int, 0, 100)
```
このようにすることで、スライスが初期容量内であればメモリ再割り当てが発生しないため、パフォーマンスが向上します。

ただし、これは事前にメモリ領域を確保しているということなので、当然のことながら、必要以上に大きく設定すると、実際には使用されないメモリ領域が確保されることになります。これは、システム全体のメモリ使用量を増加させ、他のプロセスに影響を与える可能性があります。

よって、容量が明確に決まらず、`append()`を繰り返すような処理でもなければ、特に初期容量を設定する必要はないでしょう。

### 構造体のスライスとポインタの活用
配列の各要素が多くのフィールドを持つ構造体の場合、その分、再割り当て時のデータコピー量は多くなります。これは、構造体そのものを配列の要素にするのではなく、構造体のポインタを配列の要素にすることで解決できます。

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

結果から分かる通り、ポインタのサイズは固定であり、構造体のサイズが大きくなるほど、その差が顕著になります。このように、大量のデータを持つ構造体のスライスを操作する場合、ポインタのスライスを使用することで再割り当て時のオーバーヘッドが軽減され、処理速度が向上します。

## スライス操作の注意点とベストプラクティス

関数にスライスを渡して、そのスライスを操作する際、意図したとおりの結果が返ってこないことがあります。

以下、それぞれについて、スライスの中身がどのようになるか予想してみてください。

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

冒頭で記載した通り、スライスには配列の先頭のポインタが含まれています。
Goには参照渡しはなく、ポインタが値渡しされることで、参照渡しのような挙動となっています（ポインタ渡しという名称が一般的）。
関数外と関数内では同じポインタ（0xc00001a018）で配列を参照することとなるため、変更が反映された[0 1 0]

### 問2

関数内と関数外で参照する配列が違います。これは `append()` により容量不足となり、新しい配列が作成されたためです。よって、結果は変わらずの[0 1 0]

### 問3

では、容量を増やしたら上手くいくのでしょうか。関数外と関数内で同じ配列を見ているようです（0xc00010a060）。しかし、結果は上手くいきません。なぜでしょう。この問題も結局は冒頭で確認したスライスの実体を理解しているかどうかが鍵になります。関数内の長さは4になったのですが、関数外の長さは3のままなのです。その結果、関数内のスライスから4番目の要素を参照できないのです。

試しに以下の様に変更してみてください。`slice：[0 0 0 1]`となるはずです。
```diff go
- fmt.Printf("slice：%v\n", slice3)
+ fmt.Printf("slice：%v\n", slice3[0:4])
```

### 問4

では、関数内で`append()`したい場合は、スライスのポインタを渡すべきなのでしょうか？
個人的にはあまり書きたくないコードです。スライス自体がポインタのようなものなので、スライスのポインタというのは、コードを読む人を混乱させがちだと思います。
よって、関数の返り値でスライスを返してあげるのが、直感的で明確な良いコードだと思います。

## スライスのメモリ管理とGCの挙動

Goのスライスは、非常に便利なデータ構造で、動的に長さや容量を変更できるため、多くの場面で使用されます。しかし、そのメモリ管理の特性を理解しておかないと、予期せぬメモリ使用量や、GC（Garbage Collection: ガベージコレクション）が期待通りに動作しないことがあります。この節では、スライスのメモリ管理とGCへの影響について、サンプルコードを使って解説します。

サンプルコードの説明
まず、非常に大きなスライス（1億要素）を作成し、その一部（最初の1要素）のみをsubSliceとして参照します。この際、largeSlice全体のメモリが解放されるかどうかを確認するために、largeSliceをnilに設定してGCを強制的に実行します。

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

### GCの動作について
まず、largeSliceを作成した時点では、1億個の要素がヒープに割り当てられています。しかし、その後、subSliceはlargeSliceの最初の1要素のみを参照しているため、largeSlice自体をnilにしても、subSliceがlargeSliceのメモリを一部保持している状態です。

この時点で強制的にGCを実行しても、subSliceがlargeSliceのメモリを一部保持しているため、メモリは完全には解放されません。結果として、Allocが依然として763 MiBのまま残っています。

Cloneによるメモリ解放
次に、subSliceをlargeSliceから独立させるために、slices.Clone()を使用してsubSliceのクローンを作成します。Clone()によって、subSliceは元のlargeSliceから切り離され、新しいメモリ領域にコピーされるため、largeSliceのメモリは不要になります。この状態で再度GCを実行すると、Allocの値が0 MiBに減少し、元のlargeSliceのメモリが解放されていることが確認できます。

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

### メモリリーク防止のポイント
この例からわかるように、スライスの一部を参照し続けると、GCが期待通りに動作せず、大量のメモリが解放されないことがあります。この問題を回避するためには、以下のような対応が有効です。

slices.Clone()を使用する: スライスの部分を参照するのではなく、新しいスライスに要素をクローンすることで、元のスライスのメモリを完全に解放できます。
スライスの扱いに注意する: 大規模なスライスを操作する際には、意図しないメモリ保持を避けるため、再スライスや部分スライスの使用に注意が必要です。
このように、スライスのメモリ管理とGCの関係を理解することで、パフォーマンスの向上やメモリリークの防止に役立てることができます。

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

`copy()`関数は、スライスの要素を他のスライスにコピーするために使われますが、正しく動作させるにはコピー先スライスのメモリを事前に確保しておく必要があります。上記のコードでは、cpというスライスが初期化されていないため、`copy()`は何もコピーしません。

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
スライスはGo言語において非常に柔軟で強力なデータ構造ですが、その背後にあるメモリ管理やパフォーマンスに与える影響を理解することが重要です。スライスの内部構造、appendの動作、初期容量の適切な設定、構造体のポインタスライスの利用といったポイントを押さえることで、効率的で最適化されたコードを書くことができます。また、GCの挙動やslices.Cloneの利用法を学ぶことで、メモリリークの防止やスライスのコピー処理の改善にも役立てられるでしょう。これらの知識を活かし、よりパフォーマンスに優れたGoプログラムを構築していきましょう。