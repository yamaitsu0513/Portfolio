# データベース設計書

## 概要
ポートフォリオサイト用のデータベース設計

## 技術スタック
- データベース: Supabase (PostgreSQL)
- 認証: Supabase Auth
- ストレージ: Supabase Storage

---

## テーブル一覧

### 1. mst_work (実績管理テーブル)

ポートフォリオとして表示する実績・作品情報を管理するテーブル

#### カラム定義

| カラム名 | 型 | NULL | デフォルト | 制約 | 説明 |
|---------|-----|------|-----------|------|------|
| id | uuid | NO | gen_random_uuid() | PRIMARY KEY | 実績ID |
| title | text | NO | - | - | 実績のタイトル |
| image | text | YES | - | - | 画像URL (Supabase Storage) |
| description | text | YES | - | - | 実績の詳細説明 |
| order_num | integer | NO | 0 | - | 表示順序 (昇順でソート、小さいほど上位) |
| is_public | boolean | NO | false | - | 公開フラグ (true: 公開, false: 非公開) |
| created_at | timestamptz | NO | now() | - | 作成日時 |
| updated_at | timestamptz | NO | now() | - | 更新日時 |
| deleted_at | timestamptz | YES | - | - | 削除日時 (論理削除) |

#### インデックス
```sql
-- 主キー
PRIMARY KEY (id)

-- 公開中の実績を表示順でソート
CREATE INDEX idx_work_public_order ON mst_work(order_num ASC, created_at DESC) WHERE is_public = true AND deleted_at IS NULL;

-- 論理削除されていないレコードを取得
CREATE INDEX idx_work_deleted ON mst_work(deleted_at) WHERE deleted_at IS NULL;
```

#### RLS (Row Level Security) ポリシー案
```sql
-- 公開されている実績は誰でも閲覧可能
CREATE POLICY "公開実績は誰でも閲覧可能"
ON mst_work FOR SELECT
USING (is_public = true AND deleted_at IS NULL);

-- 管理者のみ全データにアクセス可能 (今後の拡張用)
```

---

### 2. mst_contact (問い合わせ管理テーブル)

サイト訪問者からの問い合わせ内容を保存するテーブル

#### カラム定義

| カラム名 | 型 | NULL | デフォルト | 制約 | 説明 |
|---------|-----|------|-----------|------|------|
| id | uuid | NO | gen_random_uuid() | PRIMARY KEY | 問い合わせID |
| user_name | text | NO | - | - | 問い合わせ者名 |
| email | text | NO | - | CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$') | メールアドレス |
| content | text | NO | - | - | 問い合わせ内容 |
| description | text | YES | - | - | 補足説明・備考 |
| created_at | timestamptz | NO | now() | - | 作成日時 |
| updated_at | timestamptz | NO | now() | - | 更新日時 |
| deleted_at | timestamptz | YES | - | - | 削除日時 (論理削除) |

#### インデックス
```sql
-- 主キー
PRIMARY KEY (id)

-- 作成日時での絞り込み・ソート
CREATE INDEX idx_contact_created ON mst_contact(created_at DESC) WHERE deleted_at IS NULL;

-- メールアドレスでの検索
CREATE INDEX idx_contact_email ON mst_contact(email) WHERE deleted_at IS NULL;
```

#### RLS (Row Level Security) ポリシー案
```sql
-- 問い合わせの投稿は誰でも可能
CREATE POLICY "誰でも問い合わせ投稿可能"
ON mst_contact FOR INSERT
WITH CHECK (true);

-- 閲覧は管理者のみ (今後の拡張用)
```

---

## データベース設計の方針

### 1. 論理削除の採用
すべてのテーブルに `deleted_at` カラムを設け、論理削除を採用しています。これにより:
- データの復元が容易
- 監査証跡の保持
- 参照整合性の維持

### 2. タイムスタンプ管理
- `created_at`: レコード作成日時
- `updated_at`: レコード更新日時 (トリガーで自動更新)
- `deleted_at`: 論理削除日時

### 3. UUID の採用
主キーには UUID を使用し、セキュリティと拡張性を確保しています。

### 4. RLS (Row Level Security)
Supabase の機能を活用し、テーブルレベルでアクセス制御を実装します。

---

## 今後の拡張予定

- 認証機能実装時の管理者テーブル追加
- 実績へのタグ・カテゴリ機能
- 問い合わせステータス管理 (未対応/対応中/完了)
- 問い合わせへの返信履歴テーブル

---

## 補足

### updated_at 自動更新トリガー

```sql
-- updated_at を自動更新する関数
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- mst_work テーブルのトリガー
CREATE TRIGGER update_mst_work_updated_at
    BEFORE UPDATE ON mst_work
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- mst_contact テーブルのトリガー
CREATE TRIGGER update_mst_contact_updated_at
    BEFORE UPDATE ON mst_contact
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```