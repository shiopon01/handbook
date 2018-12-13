
# メソッド

インターフェース型以外の、どんな型にでもメソッドを生やすことができる。

```go
type Hex int

func (h Hex) String() string {
  return fmt.Sprintf("0x%x", int(h))
}
```

`int` 型を新たに `Hex` 型として定義し、 `String` というメソッドを設定した。

# Struct

宣言

```go
type Response struct {
  Name string
  Status int
}
```

初期化

```go
var resp Response = Response{}
var resp Response
resp := Response{}
```

```go
resp := new(Response)
resp := &Response{}
```

生成した構造体への　ポインタを返す。 `new(struct)` を利用した場合、生成時に初期化ができない。逆に、 `&struct{}` を利用した場合は初期化が可能。

```go
resp := &Response{"hey", 200}
```


# etc

`make` はチャネルを初期化、構造体は `new` で初期化。

# 参考

- http://golang.jp/effective_go