# ent. ORM

## 1. スキーマファイル作成（手動 or `ent init`）

```bash
go run entgo.io/ent/cmd/ent init Link
```

`ent/schema/link.go` が生成される（中身はほぼ空）

## 2. スキーマ定義（手動）

あなたが今見てるコード！`Fields()`とか`Indexes()`を書く

```go
package schema

import (
    "time"

    "entgo.io/ent"
    "entgo.io/ent/schema/field"
    "entgo.io/ent/schema/index"
    "github.com/google/uuid"
)

// Link holds the schema definition for the links table.
type Link struct {
    ent.Schema
}

// Fields of the Link.
func (Link) Fields() []ent.Field {
    return []ent.Field{
        field.UUID("id", uuid.UUID{}).
            Default(uuid.New),
        field.String("user_id").
            Optional().
            Nillable(),
        field.String("url").
            NotEmpty(),
        field.String("title").
            Optional().
            Nillable(),
        field.String("description").
            Optional().
            Nillable(),
        field.String("domain").
            Optional().
            Nillable(),
        field.String("og_image").
            Optional().
            Nillable(),
        field.String("page_url").
            Optional().
            Nillable(),
        field.String("note").
            Optional().
            Nillable(),
        field.Strings("tags").
            Optional(),
        field.JSON("metadata", map[string]any{}).
            Default(map[string]any{}),
        field.Time("saved_at").
            Default(time.Now),
        field.Time("created_at").
            Default(time.Now),
        field.Time("published_at").
            Optional().
            Nillable(),
    }
}

// Indexes of the Link.
func (Link) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("user_id", "saved_at"),
        index.Fields("domain"),
        index.Fields("published_at"),
    }
}
```

## 3. コード生成（`go generate`）

```bash
go generate ./...
```

↓
**ここで大量のファイルが生成される！**

生成されるのは：

- `ent/link.go` - Link エンティティの本体
- `ent/link_create.go` - 作成用
- `ent/link_update.go` - 更新用
- `ent/link_delete.go` - 削除用
- `ent/link_query.go` - クエリ用
- などなど...

つまり、**スキーマファイル（`ent/schema/*.go`）→ 大量の実装ファイル（`ent/*.go`）** って順番で生成されるってこと！

スキーマはあなたが設計図を描いて、ent がそれを元に実装コード全部作ってくれるイメージ

## 📁 フォルダ編

### `schema/`: あなたが書く設計図の場所

- `link.go` とか入ってるやつ
- ここだけは手動で編集するよ
- 新しいエンティティ追加したらここに増える

### `hook/`: ライフサイクルフック用

- 保存前/後に何か処理したい時に使う
- 例：保存前にバリデーション、保存後に通知送るとか
- 最初は空だけど、自分でフック追加できるよ

### `predicate/`: クエリの条件指定用の型定義

- `Where()` で使う条件の型が入ってる
- 例：`link.UserIDEQ("user123")` みたいなやつ
- 基本触らない

### `migrate/`: データベースマイグレーション関連

- スキーマから DB のテーブル作るコードが入ってる
- `AutoMigration` とか使う時にここが活躍する
- これも基本触らない

### `runtime/`: 実行時の設定やヘルパー

- ent の内部で使う設定とか
- 普段は気にしなくて OK

### `enttest/`: テスト用のヘルパー

- テスト書く時に便利な関数が入ってる
- インメモリ DB とか簡単に作れるよ

### `link/`: Link エンティティ専用の定数とか

- フィールド名の定数とかが入ってる
- 例：`link.FieldUserID` みたいなやつ

## 📄 ファイル編

### `client.go`: ent のメインクライアント

```go
client, err := ent.Open("sqlite3", "file:ent?mode=memory")
```

これで使うやつ

### `ent.go`: 基本的な型定義

- `ent.Link` とかの基本構造

### `link.go`: Link エンティティの本体

- 構造体定義とかメソッドとか

### `link_create.go`: 新規作成用のビルダー

```go
client.Link.Create().SetURL("https://...").Save(ctx)
```

### `link_update.go`: 更新用のビルダー

```go
client.Link.UpdateOneID(id).SetTitle("new title").Save(ctx)
```

### `link_delete.go`: 削除用

```go
client.Link.DeleteOneID(id).Exec(ctx)
```

### `link_query.go`: 検索用のクエリビルダー

```go
client.Link.Query().Where(link.UserIDEQ("user123")).All(ctx)
```

### `mutation.go`: ミューテーション（変更操作）の内部実装

- フック実装する時とかに使うけど、普段は触らない

### `tx.go`: トランザクション用

```go
client.Tx(ctx) // トランザクション開始
```

### `generate.go`: これがトリガー

```go
//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate ./schema
```

これがあるから `go generate` で全部生成されるんだよ〜

**まとめ**：普段使うのは主に `client.go` と `link_query.go`, `link_create.go` あたり！他は自動生成されたヘルパーだから、基本読むだけで OK
