
# Arrays

長さ (length) と要素の型を指定して宣言する。この時、渡す値がなければ全て型のゼロ値で初期化されるため、明示的な初期化は不要。
C言語と同じように、 `a[n]` でn番目の要素にアクセス可能。

```go
var a [4]int
a[0] = 1
```

配列変数は、配列全体を表している。そのため、C言語のように先頭の配列要素へのポインタではないことに注意。

## 宣言

要素数を指定する方法と、要素数を指定しない (コンパイラに数えさせる) 方法がある。要素数を指定しない場合は、要素数に `...` を指定してやる。

```
a := [2]string{"pen", "apple"}
b := [...]string{"pen", "apple"}
```


# slices

スライスはgolangにおける可変長配列のようなもの。
配列を定義する際、要素数の部分を空にすると定義できる。

可変長配列と呼ばれるが、実態は、固定長配列へのView (構造体) 。 ([SliceHeader](https://golang.org/pkg/reflect/#SliceHeader)) として実装されている。

```go
type SliceHeader struct {
  Data uintptr // 配列へのポインタ
  Len  int     // 配列の長さ
  Cap  int     // 配列の容量
}
```

スライスは配列へ参照なので、スライスで値を変更すると変更元となる配列にも反映される。
配列への参照の他にも、参照している配列に含まれる要素の数 `長さ (Length)` と、参照している場所から数えて最後までの要素数 `容量 (Capacity)` の2つを持っている。この長さと容量は、 `len(s)` `cap(s)` という式で求めることが出来る。

## 宣言

スライスは新しく配列を作って参照するか、既存の配列を参照するかで作成することができる。配列からは、 `ary[1:4]` などとしてスライスを作成できる。

```go
var a []int

ary := [6]int{2, 3, 5, 7, 11, 13}
var b []int = ary[1:4] //=> 3, 5, 7
```

```go
//これは配列
[3]bool{true, true, false}

//これは、上記と同様の配列を持ち、それを参照するスライス
[]bool{true, true, false}
```

スライスのゼロ値は `nil` になる。このとき、スライスは0の長さと配列を持ち、元となる配列は持っていない。

```go
s[]int == nil //=> true
```


スライスの作成では組み込みの `make` 関数を使用しても作成できる。make関数は、指定の型の配列を割り当てて初期化し、その配列を指すスライスを返す。この時、スライスの長さ、容量も同時に指定することも可能。

```go
s := make([]int)

s := make([]int, 0, 5) //=> len 0, cap 5
```

長さで指定された部分はゼロ値埋めされる。

## スライスへのデータ追加

スライスへ新しい要素を追加するには、組み込みの `append` を使う。

```go
func append(s []T, x ...T]) []T
```

```go
s := []int{1, 3, 5}
a := append(s, 7) //=> {1, 3, 5, 7}
```

渡す値は、スライス `s` と、スライスと同じ型の変数 `x`。
返り値は、元のスライスと追加する変数群をあわせたスライスになる。

スライスのappendは基本的にいま参照している配列へ値を追加してくれるが、もし配列の容量が小さくて変数が入らなかった場合、自動的により大きいサイズの配列を割り当てる。その場合、戻り値となるスライスは新しい割当先配列を参照することになる。

関数に渡したスライスをappendした場合、この自動割り当て機能によって参照先が変わる可能性があるため、bang method的な利用では注意が必要。

## 拡張

長さ < 容量 の場合、一度スライスした配列でも再スライスすることによって、スライスの長さを伸ばすことが出来る。容量を超えて拡張することはできない。この時、容量を指定すると最大まで拡張することが可能。

```go
s := []int{2, 3, 5, 7, 11, 13}
s = s[:0] //=> []
s = s[:4] //=> [2 3 5 7]
s = s[:cap(s)] //=> {2, 3, 5, 7, 11, 13}
```

