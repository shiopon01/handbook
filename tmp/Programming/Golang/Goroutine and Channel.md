# Goroutine and Channel

並列処理プログラミングには大きく分けて2つのアプローチがある。

- Shared-memory communication
  - ワーカ間でメモリ (リソース) を共有する
  - レースコンディションが起こらないようにロックをとることが多い
- Message-passing communication
  - ワーカ間でメッセージパッシングを行う
  - Erlangに実装されたActorモデルなどが代表的

Golangは後者であり、並列処理やメッセージパッシングを実現するために `goroutine` と `channel` といった機能が実装されている。加えて `select` や `closure` の実装が、Golangの並列処理をより強力にしている。

## Goroutine

`Goroutine (ゴルーチン）` は、Goランタイムを利用した軽量のスレッドのようなもの。goroutineはスレッドのように振る舞うがスレッドではなく、プロセスでもない。制御権を譲り合う（yield）機能の `coroutine` ともまた違ったものであるが、まるでスレッドでcoroutineを実行するかのように利用することが可能となっている。

goroutineの実態はGoランタイム内に確保されたネイティブスレッドの中でスケジューリングされて動くプログラムであり、Golangが動かす十数個のgoroutineはGoランタイムが底レイヤーで確保した5、6個のスレッド内で実現されている。また、プロセスを跨いだメッセージパッシングは難しいという理由もあり、プログラム内で利用するスレッド、goroutineは全て1つのプロセスに収まっている。

```go
go hello (a, b, c)
```

goroutineは起動したい関数かメソッドの呼び出しに対して `go` キーワードを使うことで起動できる。非同期処理を開始するための特殊な構文のように思えるが、 `go` キーワードは単なる普通の関数であり、これを使うことでGoランタイム、スレッドコントローラからgoroutineを起動することができる。

```go
func say (s string) {
  fmt.Println(s)
}

func main () {
  // 新しいgoroutineを起動
  go say ("Hello")
  time.Sleep(time.Second)
}
```

goroutineで呼び出した関数は非同期で通常通り動作するが、プログラム本体はgoroutineで呼び出した関数の完了を待たずにmainを終了させる。mainが終了するということはプログラムが終了するということで、goroutineは途中だったとしても強制的に終了させられる。そのため、この例では `say("Hello")` がPrintlnを実行するのを待つため、mainで処理を1秒間ブロックしている。（ `time.Sleep()` がこの部分だ）

起動したgoroutineは関数の処理の終了（最終行まで実行）か、 `return` で関数を抜けるか、 `runtime.Goexit ()` を呼び出すことで終了できる。goroutineで呼び出した関数が終了する時に返り値があっても、それは関数の終了時に破棄される。（受け取ることができないため）

### Goroutineの管理

goroutineを実行するならmainはgoroutineの終了を待つべきだが、処理時間の分からない処理に対して `time.Sleep` を使うわけにもいかない。また、goroutineを複数起動する場合、その数や処理の終了など全てを管理しなければならない。

そこで利用できるものが、排他や同期の制御を提供する `sync` パッケージだ。起動したgoroutineの数を管理したい場合は `sync.WaitGroup` を利用できる。

```go
func main() {
  var wg sync.WaitGroup

  for i := 0; i < 3; i++ {
    // goroutineの生成時にインクリメント
    wg.Add(1)
    go func(i int) {
      fmt.Println(i)
      // goroutineの終了時にデクリメント
      wg.Done()
    }(i)
  }

  wg.Wait() // WaitGroupが0になるまでブロック
}
```

`runtime.NumGoroutine` を使えば現在起動しているgoroutineの数を表示できる。main関数自身もgoroutineで起動しているため、これでは必ず1以上返される。

## Channels

`Channel（チャネル）` はgoroutine間でメッセージパッシング (メッセージの送受信) を行うための仕組み。

goroutineは同じプロセスのアドレス空間で実行される。そのため、共有されたメモリへのアクセスはかならず同期されていなければならない。（非同期のgoroutine同士が好き勝手メッセージを投げ合えないということであり、それなりの仕組みが必要ということだ）

そこでGoでは、 `channel` という便利な通信機構を使ってgoroutine同士の通信を実現している。次の例は、channelを作成するソースコードだ。channelは `make()` を使って作成し、この際に渡される予定の型も宣言しておく。

```go
cf := make(chan int)
cs := make(chan string)
ct := make(chan interface{})
```

これで通信の受け口が作成される。ここで定義したchannelに対して `<-` 演算子を使うことでデータの送受信が可能だ。

```go
ch := make(chan int)

ch <- 1   // 1をchannel chに送る
v := <-ch // chの中からデータを取り出し、vに代入する
```

channelを利用することで、マルチスレッドプログラミングの難点である排他制御や同期化を、シンプルで簡単な記法と構文で実現できる。

## [WIP]

```go
func f(ch chan int) {
  ch <- 1234
}

func main() {
  ch := make(chan int)
  go f(ch)

  fmt.Println(<-ch) // ここでデータが来るまでブロックする
}
```

デフォルトでは、channelがやり取りするデータはブロックされている。ブロックとは、もし `value := <-ch` を読み取った時、まだデータが入っていない場合は `ch` にデータが送信されるまで処理を停止すること。 (データが送信される見込みがない場合はエラーが発生する)
これを利用することで、lockを利用せずともgoroutine間の同期処理を実現することができる。 `sync` パッケージにはlockをとれる仕組みはあるが、できるだけメッセージングでリソース共有/同期をすることが推奨されている。

データの送信と受信は1セットで行われ、データを送信した後にまた送信するとエラーが発生する。また、goroutineによってデータが送信される見込みが無い時に、データを受信した後すぐ受信するとエラーが発生する。

以下、エラー例。

```go
func main() {
  ch := make(chan int)
  ch <- 12
  ch <- 34 // Error 送信後、また送信
}
```

```go
func main() {
  ch := make(chan int)
  fmt.Println(<-ch) // Error 送信される見込みがない
}
```

```go
func main() {
  ch := make(chan int)
  ch <- 12
  fmt.Println(<-ch)
  fmt.Println(<-ch) // Error 受信後、また受信
}
```

## Buffered Channels

Goでは、channelのバッファリングの大小を指定することが可能だ。つまり、channelはいくつもの要素を保存することができる。取り出しの順番はFIFO。

```go
func main() {
  ch := make(chan int, 2) // 2つのint型の要素を持つ
  ch <- 1
  ch <- 2

  fmt.Println(<-ch) //=> 1
  fmt.Println(<-ch) //=> 2
}
```

対応するvalue値は2以上でなくてはならず、1を指定するとエラーが発生する。

## RangeとClose

先程の例はchannelの値を2回読み込む必要があったので、これはあまり便利ではない。Goはこの点を考慮し、 `range` によってsliceやmapを操作するのと同じ感覚でバッファリング型のchannelを操作することができる。

```go
func f(ch chan int) {
  for i := 0; i < 5; i++ {
    ch <- i
  }
  close(ch) // 明示的なクローズ
}

func main() {
  ch := make(chan int, 5)
  go f(ch)

  for i := range ch {
    fmt.Println(i)
  }
}
```

`for i := range ch` で、このchannelがクローズを明示されるまで連続してchannel内のデータを読み込むことが出来る。メッセージの送信側では、ビルドイン関数 `close` を使ってchannelを閉じる事ができる。閉じられたchannelでは、いかなるデータの送受信もできなくなることに注意。

受信側は、 `value, ok := <-ch` という式でchannelが既に閉じられているかテストすることができる。もしokにfalseが返ってきたら、channelにはデータは入っておらず、閉じられているということになる。

また、channelはファイルのようなものでは無いため、頻繁に閉じる必要はない。何のデータも送ることがない場合、またはrangeループを終了させたい場合などだけでよい。


# select

select (セレクト) は、複数のchannelを扱うための仕組み。

複数のchannelがある場合、Goが提供する `select` キーワードが有効だ。selectはswitchのように複数の `case` を用意することで、channelの中でやりとりされるデータを監視する。

selectはデフォルトではブロックされており、channelの準備が整ったときに該当する `case` の処理が動作する。複数のchannelの準備が整った時、selectはランダムにひとつ選択し、実行する。

```go
func fibonacci(c, quit chan int) {
  x, y := 1, 1

  for {
    select {
    case c <- x:
      x, y = y, x + y
    case <-quit:
      fmt.Println("quit")
      return
    }
  }
}

func main() {
  c := make(chan int)
  quit := make(chan int)
  go func() {
    for i := 0; i < 10; i++ {
      fmt.Println(<-c)
    }
    quit <- 0
  }()

  fibonacci(c, quit)
}
```

selectはswitchによく似ており、 `default` 文も存在する。defaultは監視しているchannelがどれも準備が整っていない時に、デフォルトで実行される。 (selectはchannelを待ってブロックされない)

```go
select {
case c <- x:
default:
  // ブロックされた時にここが実行される
}
```

また、プログラム全体がブロックされないよう、selectにはタイムアウトを設定することができる。

```go
select {
case c <- x:
case <- time.After(5 * time.Second):
  fmt.Println("timeout")
  break
}
```


