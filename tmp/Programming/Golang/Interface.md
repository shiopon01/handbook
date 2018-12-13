# インターフェース

オブジェクトの振る舞いを規定する。インターフェースに格納された型に関する情報へのポインタと、関連データへのポインタとして表される。

```go
type Reader interface {
  // メソッド(引数の型) 返り値の型
  Read(int) string
  Stop()
}
```

# Duck Typing

インターフェースは型に対してメソッドの制限を行うための仕組みであり、Go言語でDuck Typingを行うための仕組みだ。Duck Typingとは「アヒルのように歩き、アヒルのように鳴くのなら、それはアヒルである」という言い回しの単語であり、ここでは「ある型がインターフェースで定義されたメソッドを全て実装しているのであれば、その型はそのインターフェースを実装しているものとする」という意味で使われる。

```go
type Reader interface {
	Read() string
}

type User struct {
	Id   int
	Name string
}

func (u User) Read() string {
	return fmt.Sprintf(`{ "Id": %d, "Name": "%s" }`, u.Id, u.Name)
}
```

この時点で構造体 `User` はインターフェース `Reader` で定義されているメソッドと同じ名前のメソッド `Read` を実装しているため、この構造体UserはインターフェースReaderとして扱うことができる。

これは例えば、インターフェースReaderを引数として取る関数に構造体Userを渡すことができるということだ。この使い方は、渡される型に実装されているメソッドを保証したい場合に利用できる。

さっきのインターフェースや構造体を定義しているものとして、次の例を見てみよう。これはインターフェースReaderとは無関係の型が、インターフェースReaderとして扱われる例だ。

```go
func say(r Reader) {
  fmt.Println(r.Read())
}

func main() {
  user := &User{1, "Jack"}
  say(user)
}
```

sayはインターフェースReaderを引数として取るが、実際に渡しているのは構造体Userだ。これは「構造体UserはメソッドReadを実装しているから、インターフェースReaderと実質同義！」というDuck Typingが働いているためにエラーが起きない。もちろん、メソッドReadを実装していなかった場合、インターフェースReaderとは別物と扱われるためにこれはエラーとなる。

このDuck Typing、関数の引数だけでなく、構造体を生成する際にも有用だ。インターフェース型の変数に好きな型を代入すると値をチェックされ、その型がインターフェースと同義かどうかを確認される。もちろん、同義でなかった場合はエラーだ。

```go
func main() {
  var user Reader = &User{1, "Jack"}
  ...
}
```

この場合もやはり、望むメソッドが実装されていることを保証することができる。どちらかというと、こっちの「interfaceを意識した型設計」が求められているのかもしれない。

# インターフェースへのインターフェース埋込み

Go言語にはサブクラスや継承という概念は存在しない。その代わり、構造体やインターフェースに型を埋め込み、実装を借りる仕組みがある。

次の例は、インターフェースReaderとWriterの特徴（メソッド）を併せ持った、新たなインターフェース `ReadWriter` を生み出す際の例。

```go
type Reader interface {
  Read() string
}

type Writer interface {
  Write(id int, name string)
}

// Reader + Writer
type ReadWriter interface {
  Reader
  Writer
}
```

ReadWriterを実装した型を作ろうと思った時、想像しているように `Read` と `Write` の両方のメソッドが必要だ。ReadWriterが宣言されているとして、それを実装する例を挙げてみよう。

```go
type User struct {
	Id   int
	Name string
}

func (u User) Read() string {
	return fmt.Sprintf(`{ "Id": %d, "Name": "%s" }`, u.Id, u.Name)
}

func (u User) Write(id int, name string) {
	u.Id = id
	u.Name = name
}

func main() {
  var user ReadWriter = &User{1, "Jack"}
  ...
}
```

# 構造体への構造体埋込み

Goの構造体はクラスではないため継承という概念は存在しないが、インターフェースのように埋込みという形で他の構造体の特徴を引き継ぐことが出来る。

```go
type User struct {
	Id   int
	Name string
}

func (u User) Read() string {
	return fmt.Sprintf(`{ "Id": %d, "Name": "%s" }`, u.Id, u.Name)
}

type Admin struct {
	User  // Id, Name, Read() を引き継ぐ
	Admin bool
}

func main() {
	admin := &Admin{User{1, "jack"}, true}
	fmt.Println(admin.Read())
}
```

実行してみると分かるが、Userから引き継いだ `admin.Read()` ではIdとNameしか出力できず、Admin独自のパラメータ `Admin bool` を出力できない。この対策として、Adminでは `Read()` をオーバーライドして独自のパラメータも出力できるようにする。

```go
func (a Admin) Read() string {
	return fmt.Sprintf(`{ "Id": %d, "Name": "%s", "Admin": "%t" }`, a.Id, a.Name, a.Admin)
}
```

オーバーライドされていないReadも呼び出すことが出来る。

```go
func main() {
  admin := &Admin{User{1, "jack"}, true}

  fmt.Println(admin.Read()) //=>  {"Id": 1, "Name": "jack", "Admin": "true" }
  fmt.Println(admin.User.Read()) //=> { "Id": 1, "Name": "jack" }
}
```

# 構造体へのインターフェース埋込み

構造体にもインターフェースを埋め込むことができる。この場合、mainでUserを作成した時点でインターフェースに宣言されている関数を実装できていないとエラーとなる。

```go
type Crying interface {
	Cry() string
}

type User struct {
	Crying
}

func (u User) Cry() string {
	return "ouou.."
}

func main() {
	user := &User{}
	fmt.Println(user.Cry())
}
```