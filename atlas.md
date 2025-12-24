# Atlas

## Atlas って何?

データベースのマイグレーション(変更管理)を超賢くやってくれるツール。

### Atlas 使う場合

```md
1. Ent のスキーマ定義を変更(Go コード)
2. Atlas「あ、変わったね!差分の SQL を自動生成するね」
3. 生成された SQL を確認して実行
```

👉 **Atlas が勝手に差分を計算してくれる!**

### 具体例で説明

例えば、Ent で新しいフィールド追加したとする

```go
// ent/schema/user.go
func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
        field.String("email"),    // これを追加!
    }
}
```

この状態で...

```bash
atlas migrate diff add_email \
  --to "ent://ent/schema" \        # Entのスキーマを見て
  --dir "file://ent/migrate/migrations" \  # ここに保存
  --dev-url "docker://postgres/15"     # 一時DBで計算
```

すると、Atlas が自動で生成してくれる ↓

```sql
-- ent/migrate/migrations/20231212_add_email.sql
ALTER TABLE users ADD COLUMN email varchar NOT NULL;
```

**人間が書いたわけじゃないのに、ちゃんと差分 SQL ができてる!** これが Atlas の凄いところ ✨

### baseline って何?

**問題**: すでに Supabase に本番 DB がある状態で、「今から Atlas 使いたい!」ってなったとき、どうする?

Atlas は「何も知らない状態」からスタートするから、既存のテーブル全部を「新規作成」しようとしちゃう。
でもそれってもう存在してる。
困るよね 💦

**解決策が baseline!**

```bash
# 今のSupabase DBの状態を「スタート地点」として記録
atlas schema inspect \
  --url "postgresql://supabase..." \
  --format "{{ sql . }}" > baseline.sql
```

これで「ここまでは既に適用済みだよ」って Atlas に教えてあげる。以降の差分だけ管理できるようになる!

## ent/migrate/migrations/ と ent/schema/　のファイルの違い

### `ent/schema/*.go` (Ent のスキーマファイル)

👉 **「理想の状態」を定義するファイル**

```go
// ent/schema/user.go
type User struct {
    ent.Schema
}

func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
        field.String("email"),
        field.Int("age"),
    }
}
```

これは「User テーブルはこういう形であるべき!」っていう**設計図**。

- Go のコードで書かれてる
- **現在の理想形**を表現
- 過去の履歴は含まれてない

### `ent/migrate/migrations/*.sql` (マイグレーションファイル)

👉 **「どうやって変化させたか」の履歴**

```sql
-- 20231201_create_users.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR NOT NULL
);

-- 20231205_add_email.sql
ALTER TABLE users ADD COLUMN email VARCHAR NOT NULL;

-- 20231210_add_age.sql
ALTER TABLE users ADD COLUMN age INTEGER NOT NULL;
```

こっちは「実際に DB を変更した SQL」の**記録**だね。

- SQL で書かれてる
- **時系列の変更履歴**
- 「いつ・何を・どう変えたか」がわかる

### 比喩で説明するね

**家のリフォームに例えると...**

#### `ent/schema/*.go` = 完成予想図

「3LDK、キッチン広め、バルコニー付き」みたいな**最終的な理想の間取り図**

#### マイグレーションファイル = 工事記録

```md
2023/01/10: 基礎工事完了
2023/02/15: 壁を立てた
2023/03/20: キッチンを拡張した
2023/04/05: バルコニー追加
```

完成予想図だけ見ても「どういう順番で工事したか」わかんない。
でも工事記録があれば、「あの時こう変えたんだな」って追える。

### なんで両方必要なの？

#### `ent/schema/*.go` があれば

- 開発者が「今どういう構造なの?」って一発でわかる
- Ent が自動で Go のコード生成してくれる
- 型安全に DB 操作できる

#### マイグレーションファイルがあれば

- 本番 DB に順番に適用できる(ロールバックも可能)
- チーム全員が同じ変更を追える
- 「なんでこのカラム追加したんだっけ?」って履歴見れる

### Atlas の役割はここ

```md
Ent スキーマ(理想)
`api/ent/schema/user.go`
(現在の理想形)

↑

Atlas 自動で差分計算

↑

マイグレーションファイル（履歴）
`ent/migrate/migrations/20251224_xxx.sql`
（変更手順の SQL）
```

Atlas は「理想の状態」と「今の DB」を比較して、「じゃあこの SQL 実行すれば理想に近づく!」って教えてくれる ✨
