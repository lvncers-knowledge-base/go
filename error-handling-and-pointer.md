# エラーハンドリングとポインタ

## エラーハンドリング編

エラーハンドリングのポイント：

Go は error を戻り値で返すスタイル。
最初は if err != nil が多くてウザいかもだけど、慣れると「エラー処理してない箇所」が一目でわかって安心感ある。
fmt.Errorf の%w でエラーをラップすると、どこでエラーが起きたか追跡しやすくなるよ。
自分でエラー型作ると、もっと詳細な情報を持たせられる。

## 1. 基本的なエラーハンドリング

```go
func divide(a, b float64) (float64, error) {
  if b == 0 {
    return 0, errors.New("0で割ることはできません")
  }
  return a / b, nil
}

func main() {
  // 正常なケース
  result, err := divide(10, 2)
  if err != nil {
    fmt.Println("エラー:", err)
    return
  }
  fmt.Printf("10 / 2 = %.2f\n", result)

  // エラーケース
  result, err = divide(10, 0)
  if err != nil {
    fmt.Println("エラー:", err)
    // エラーの時は処理を中断するか、デフォルト値を使う
  } else {
    fmt.Printf("10 / 0 = %.2f\n", result)
  }
}
```

## 2. カスタムエラー型

```go
type ValidationError struct {
  Field string
  Message string
}

func (e \*ValidationError) Error() string {
  return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

func validateUser(name string, age int) error {
  if name == "" {
    return &ValidationError{
      Field: "name",
      Message: "名前は必須です",
    }
  }
  if age < 0 || age > 150 {
    return &ValidationError{
      Field: "age",
      Message: "年齢が不正です",
    }
  }
  return nil
}

func main() {
  err := validateUser("", 25)
  if err != nil {
    fmt.Println("バリデーションエラー:", err)
  }
  err = validateUser("太郎", 200)
  if err != nil {
    fmt.Println("バリデーションエラー:", err)
  }
  err = validateUser("花子", 25)
  if err != nil {
    fmt.Println("バリデーションエラー:", err)
  } else {
    fmt.Println("バリデーション成功！")
  }
}
```

// 3. エラーのラップ（Go 1.13 以降）
func fetchData(id int) error {
if id < 0 {
return fmt.Errorf("データ取得失敗: %w", errors.New("無効な ID"))
}
return nil
}

func processData(id int) error {
err := fetchData(id)
if err != nil {
return fmt.Errorf("処理中にエラー発生: %w", err)
}
return nil
}

func errorWrappingExample() {
fmt.Println("=== エラーのラップ ===")
err := processData(-1)
if err != nil {
fmt.Println("エラー:", err)
// errors.Unwrap で元のエラーを取得できる
}
fmt.Println()
}

// ============================================
// ポインタ編
// ============================================

type User struct {
Name string
Age int
}

// 4. 値渡しとポインタ渡しの違い
func updateUserByValue(u User) {
u.Age = 30 // これは元の構造体には影響しない
fmt.Println("関数内（値渡し）:", u)
}

func updateUserByPointer(u *User) {
u.Age = 30 // これは元の構造体を変更する
fmt.Println("関数内（ポインタ渡し）:", *u)
}

func pointerBasics() {
fmt.Println("=== 値渡し vs ポインタ渡し ===")
user1 := User{Name: "太郎", Age: 25}
fmt.Println("変更前:", user1)
updateUserByValue(user1)
fmt.Println("値渡し後:", user1) // 変わらない！
updateUserByPointer(&user1)
fmt.Println("ポインタ渡し後:", user1) // 変わってる！
fmt.Println()
}

// 5. メソッドレシーバー（バリュー vs ポインタ）
type Counter struct {
count int
}

// バリューレシーバー（元の値は変更されない）
func (c Counter) IncrementByValue() {
c.count++
fmt.Println("バリューレシーバー内:", c.count)
}

// ポインタレシーバー（元の値が変更される）
func (c \*Counter) IncrementByPointer() {
c.count++
fmt.Println("ポインタレシーバー内:", c.count)
}

func receiverExample() {
fmt.Println("=== メソッドレシーバー ===")
counter := Counter{count: 0}
fmt.Println("初期値:", counter.count)
counter.IncrementByValue()
fmt.Println("バリューレシーバー後:", counter.count) // 変わらない！
counter.IncrementByPointer()
fmt.Println("ポインタレシーバー後:", counter.count) // 変わる！
fmt.Println()
}

// 6. nil ポインタのチェック
func printUser(u \*User) {
if u == nil {
fmt.Println("ユーザーは nil です")
return
}
fmt.Printf("ユーザー: %s (%d 歳)\n", u.Name, u.Age)
}

func nilCheckExample() {
fmt.Println("=== nil ポインタのチェック ===")
var user \*User // nil ポインタ
printUser(user)
user = &User{Name: "次郎", Age: 28}
printUser(user)
fmt.Println()
}

// 7. 実践例：データベース風の操作
type Product struct {
ID int
Name string
Price float64
}

// ポインタを返すことで、nil で「見つからない」を表現できる
func findProduct(id int, products []Product) (\*Product, error) {
for i := range products {
if products[i].ID == id {
return &products[i], nil
}
}
return nil, fmt.Errorf("商品 ID %d が見つかりません", id)
}

func updateProductPrice(p \*Product, newPrice float64) error {
if p == nil {
return errors.New("商品が nil です")
}
if newPrice < 0 {
return errors.New("価格は 0 以上である必要があります")
}
p.Price = newPrice
return nil
}

func practicalExample() {
fmt.Println("=== 実践例：商品管理 ===")
products := []Product{
{ID: 1, Name: "りんご", Price: 100},
{ID: 2, Name: "バナナ", Price: 150},
{ID: 3, Name: "オレンジ", Price: 120},
}
// 商品を検索
product, err := findProduct(2, products)
if err != nil {
fmt.Println("エラー:", err)
return
}
fmt.Printf("見つかった商品: %s (%.0f 円)\n", product.Name, product.Price)
// 価格を更新
err = updateProductPrice(product, 180)
if err != nil {
fmt.Println("更新エラー:", err)
return
}
fmt.Printf("更新後: %s (%.0f 円)\n", product.Name, product.Price)
// 存在しない商品を検索
product, err = findProduct(99, products)
if err != nil {
fmt.Println("エラー:", err)
}
fmt.Println()
}

func main() {
// エラーハンドリング
errorBasics()
customErrorExample()
errorWrappingExample()
// ポインタ
pointerBasics()
receiverExample()
nilCheckExample()
practicalExample()
fmt.Println("全部の例が完了！")
}
