# 空インターフェース interface{}

Goには全ての型との互換性を持つ空インターフェース `interface{}` 型というものが存在する。空インターフェース型の変数を用意しておくと、そこにはどの型の値でも代入できる魔法の変数が生まれる。

```go
var hoge interface{}

hoge = 123
hoge = "hello"
hoge = []string{"tamago", "kake"}
```

本来、intの値が入る変数に文字列を新しく代入しようものなら `cannot use "hello" (type string) as type int in assignment` などと怒られるものだが、この例では全ての型との互換がある `interface{}` 型に代入しているためエラーにはならない。

（実際のコードではこの例のような使い方をすることは少ないと思うので、まだ使いみちが良く分かっていないだろう。）

コード内の空インターフェース型は、関数の引数や返り値などでよく利用される。空インターフェースにはどんな型の値でも代入することができるため、関数が空インターフェース型を返すように宣言することで任意の型の値を返すことができるようになるためだ。

つまり、本来は決まった型しか返せない関数であったが、全ての型との互換性を持つ空インターフェース型を返すようにすることでどんな値でも返却することができるようになるのだ。

ここに、関数の返り値を `interface{}` とした例を挙げる。関数 `say` は、受け取った数値が偶数である場合に `string` 型の値を返し、奇数であるなら `int` 型の値を返す。返り値は勝手に空インターフェース型とされるため、 `return` 部では値の型などを気にする必要は一切ない。

```go
func say(n int) interface{} {
	if n%2 == 0 {
		return 123
	} else {
		return "hello"
	}
}

func main() {
	fmt.Println(say(1)) //=> (int) 123
	fmt.Println(say(2)) //=> (string) "hello"
}
```

同じように、関数の引数を `interface{}` とした例も挙げておく。引数の値を空インターフェース `interface{}` にしておくことで、呼び出しからどの型の値でも受け取ることが出来るようになる。

```go
func say(any interface{}) { ... }

func main() {
  say(123)
  say("hello")
  say([]string{"tamago", "kake"})
}
```

大変便利ではあるが、ここで渡された値をどうやって処理するのかが気になるところであろう。Goではそれぞれの型に `型アサーション` と呼ばれる型チェックの仕組みが組み込まれている。変数（レシーバ）の型と `(<型>)` の部分で指定した型が一致すれば `true` 、しなければ `false` だ。

```
<変数>.(<型>)
```

この型アサーションの返り値は実際の値と、型チェックの結果が返される。このチェックから条件分岐させるやり方は、公式からも `"comma ok" idiom` と呼ばれている。

次の例は `"comma ok" idiom` を用いた条件分岐の例。関数 `say` に渡された値はifの前に型アサーションを行われ、その結果を格納した変数 `ok` の値の条件分岐が行われている。

```go
func say(any interface{}) {
	if value, ok := any.(int); ok {
    fmt.Println("(int)", value)
    return
	}

	if value, ok := any.(string); ok {
    fmt.Println("(string)", value)
    return
  }

  fmt.Println("(nomatch)", any)
}

func main() {
	say(123)     //=> (int) 123
	say("HELLO") //=> (string) Hello
}
```

条件分岐はswitchにも対応している。

```go
func say(any interface{}) {
  switch value := any.(type) {
    case int:
      fmt.Println("(int)", value)
    case string:
      fmt.Println("(string)", value)
    default:
      fmt.Pritnln("(nomatch)", value)
  }
}
```