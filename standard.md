# Go 並列処理の基本概念

## 1. 基本的な goroutine

```go
func main() {
  // 普通の関数呼び出し（同期的）
  sayHello("同期")

  // goroutine で実行（非同期的）
  go sayHello("非同期1")
  go sayHello("非同期2")
  go sayHello("非同期3")

  // メイン goroutine が終わると全部終わるため、待つ
  time.Sleep(1 * time.Second)
}

func sayHello(name string) {
  for i := 0; i < 3; i++ {
    fmt.Printf("%s: Hello %d\n", name, i)
    time.Sleep(200 * time.Millisecond)
  }
}
```

## 2. WaitGroup を使った同期

WaitGroup は「全部の goroutine が終わるまで待つ」ための道具。
Add でカウント増やして、Done で減らして、Wait で待つ。これめっちゃ使う！

```go
func main() {
  var wg sync.WaitGroup

  // 3つの goroutine を起動
  for i := 1; i <= 3; i++ {
    wg.Add(1) // カウンターを増やす

    go func(id int) {
      defer wg.Done() // 終わったらカウンターを減らす

      fmt.Printf("Worker %d: 処理開始\n", id)
      time.Sleep(time.Duration(id) / 2 * time.Second)
      fmt.Printf("Worker %d: 処理完了\n", id)
    }(i) // iを引数として渡すの大事！
  }

  wg.Wait() // 全部のgoroutineが終わるまで待つ
}
```

## 3. Channel を使った通信

Channel は goroutine 間でデータをやり取りする仕組み。<-の矢印で送受信するんだけど、慣れると直感的。

```go
func main() {
  // チャネルを作成
  ch := make(chan string)

  // データを送信する goroutine
  go func() {
    time.Sleep(1 * time.Second)
    ch <- "Hello from goroutine!" // チャネルに送信
  }()

  // データを受信（ブロックする）
  msg := <-ch
  fmt.Println("受信:", msg)
}
```

## 4. Buffered Channel

```go
func main() {
  // バッファサイズ3 のチャネル
  ch := make(chan int, 3)

  // バッファがあるので、すぐにブロックしない
  ch <- 1
  ch <- 2
  ch <- 3

  fmt.Println("受信:", <-ch)
  fmt.Println("受信:", <-ch)
  fmt.Println("受信:", <-ch)
}
```

## 5. 実践的な例：並列で API っぽい処理

```go
func main() {
  type Result struct {
    ID    int
    Value string
    Err   error
  }

  tasks := []int{1, 2, 3, 4, 5}
  results := make(chan Result, len(tasks))

  // 各タスクを並列実行
  for _, id := range tasks {
    go func(taskID int) {
      // 何か重い処理をシミュレート
      time.Sleep(time.Duration(taskID*100) * time.Millisecond)

      results <- Result{
        ID:    taskID,
        Value: fmt.Sprintf("Task %d completed",   taskID),
          Err:   nil,
        }
    }(id)
  }

  // 結果を収集
  for i := 0; i < len(tasks); i++ {
    result := <-results
    fmt.Printf("結果: ID=%d, Value=%s\n", result.ID,   result.Value)
  }
}
```

## 6. select を使った複数チャネルの待機

select は複数のチャネルを同時に待てるやつ。どれか一つでも準備できたら処理が進む。

```go
func main() {
  ch1 := make(chan string)
  ch2 := make(chan string)

  go func() {
    time.Sleep(1 * time.Second)
    ch1 <- "ch1からのメッセージ"
  }()

  go func() {
    time.Sleep(500 * time.Millisecond)
    ch2 <- "ch2からのメッセージ"
  }()

  // どちらか早い方を受信
  for i := 0; i < 2; i++ {
    select {
      case msg1 := <-ch1:
        fmt.Println("受信:", msg1)
      case msg2 := <-ch2:
        fmt.Println("受信:", msg2)
    }
  }
}
```
