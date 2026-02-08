# API一覧

## 概要
ポートフォリオサイトのバックエンドAPI仕様書

## Base URL
```
https://rqkwsjwvmnoozsfpupnt.supabase.co
```

## 認証
すべてのリクエストに以下のヘッダーが必要です。

```javascript
{
  'apikey': 'YOUR_SUPABASE_ANON_KEY',
  'Authorization': 'Bearer YOUR_SUPABASE_ANON_KEY',
  'Content-Type': 'application/json'
}
```

---

## エンドポイント一覧

### 1. 実績データ取得 (PostgREST)

#### `GET /rest/v1/mst_work`

公開されている実績データを取得します。

**エンドポイント:**
```
GET https://rqkwsjwvmnoozsfpupnt.supabase.co/rest/v1/mst_work
```

**DB種別:** PostgREST (Auto-generated API)

**クエリパラメータ:**

| パラメータ | 型 | 必須 | 説明 |
|---------|-----|------|------|
| select | string | No | 取得するカラムを指定 (例: `id,title,image`) |
| is_public | boolean | No | 公開フラグでフィルタ (例: `eq.true`) |
| order | string | No | ソート順 (例: `order_num.asc`, `created_at.desc`) |
| limit | integer | No | 取得件数の制限 |

**`order`パラメータの使用例:**
- `order=order_num.asc` - 表示順序の昇順(推奨)
- `order=order_num.asc,created_at.desc` - 表示順序の昇順、同じ場合は作成日の降順
- `order=created_at.desc` - 作成日の降順

**リクエスト例:**
```bash
# 公開済みの実績を表示順序(order_num)でソートして取得
curl -X GET \
  'https://rqkwsjwvmnoozsfpupnt.supabase.co/rest/v1/mst_work?is_public=eq.true&deleted_at=is.null&order=order_num.asc,created_at.desc' \
  -H 'apikey: YOUR_ANON_KEY' \
  -H 'Authorization: Bearer YOUR_ANON_KEY'
```

**レスポンス例:**
```json
[
  {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "title": "ECサイト構築プロジェクト",
    "image": "https://rqkwsjwvmnoozsfpupnt.supabase.co/storage/v1/object/public/works/sample.jpg",
    "description": "Reactを使用したECサイトの開発",
    "order_num": 1,
    "is_public": true,
    "created_at": "2025-01-15T10:00:00.000Z",
    "updated_at": "2025-01-15T10:00:00.000Z",
    "deleted_at": null
  }
]
```

**TypeScript型定義:**
```typescript
interface Work {
  id: string;
  title: string;
  image: string | null;
  description: string | null;
  is_public: boolean;
  created_at: string;
  updated_at: string;
  deleted_at: string | null;
}
```

---

### 2. 問い合わせデータ作成 (RPC)

#### `POST /rest/v1/rpc/create_contact`

問い合わせフォームから送信されたデータをデータベースに保存します。

**エンドポイント:**
```
POST https://rqkwsjwvmnoozsfpupnt.supabase.co/rest/v1/rpc/create_contact
```

**DB種別:** RPC (PostgreSQL Function)

**リクエストボディ:**

| パラメータ | 型 | 必須 | 説明 |
|---------|-----|------|------|
| p_user_name | string | Yes | 問い合わせ者名 |
| p_email | string | Yes | メールアドレス |
| p_content | string | Yes | 問い合わせ内容 |
| p_description | string | No | 補足説明 |

**リクエスト例:**
```bash
curl -X POST \
  'https://rqkwsjwvmnoozsfpupnt.supabase.co/rest/v1/rpc/create_contact' \
  -H 'apikey: YOUR_ANON_KEY' \
  -H 'Authorization: Bearer YOUR_ANON_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "p_user_name": "山田太郎",
    "p_email": "yamada@example.com",
    "p_content": "ポートフォリオサイトについての問い合わせです",
    "p_description": "制作期間について詳しく知りたいです"
  }'
```

**レスポンス例:**
```json
{
  "id": "456e7890-e89b-12d3-a456-426614174001",
  "user_name": "山田太郎",
  "email": "yamada@example.com",
  "content": "ポートフォリオサイトについての問い合わせです",
  "description": "制作期間について詳しく知りたいです",
  "created_at": "2025-02-07T12:00:00.000Z"
}
```

**PostgreSQL関数の実装例:**
```sql
CREATE OR REPLACE FUNCTION create_contact(
  p_user_name TEXT,
  p_email TEXT,
  p_content TEXT,
  p_description TEXT DEFAULT NULL
)
RETURNS TABLE (
  id UUID,
  user_name TEXT,
  email TEXT,
  content TEXT,
  description TEXT,
  created_at TIMESTAMPTZ
) AS $$
BEGIN
  RETURN QUERY
  INSERT INTO mst_contact (user_name, email, content, description)
  VALUES (p_user_name, p_email, p_content, p_description)
  RETURNING 
    mst_contact.id,
    mst_contact.user_name,
    mst_contact.email,
    mst_contact.content,
    mst_contact.description,
    mst_contact.created_at;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

### 3. 運営への問い合わせメール送信 (Edge Function)

#### `POST /functions/v1/send_email_contact_admin`

問い合わせ内容を運営者(管理者)にメールで通知します。

**エンドポイント:**
```
POST https://rqkwsjwvmnoozsfpupnt.supabase.co/functions/v1/send_email_contact_admin
```

**DB種別:** Edge Function (Deno)

**リクエストボディ:**

| パラメータ | 型 | 必須 | 説明 |
|---------|-----|------|------|
| p_to | string | Yes | 運営者のメールアドレス |
| user_name | string | Yes | 問い合わせ者名 |
| user_email | string | Yes | 問い合わせ者のメールアドレス |
| content | string | Yes | 問い合わせ内容 |
| description | string | No | 補足説明 |

**リクエスト例:**
```bash
curl -X POST \
  'https://rqkwsjwvmnoozsfpupnt.supabase.co/functions/v1/send_email_contact_admin' \
  -H 'apikey: YOUR_ANON_KEY' \
  -H 'Authorization: Bearer YOUR_ANON_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "p_to": "admin@example.com",
    "user_name": "山田太郎",
    "user_email": "yamada@example.com",
    "content": "ポートフォリオサイトについての問い合わせです",
    "description": "制作期間について詳しく知りたいです"
  }'
```

**レスポンス例:**
```json
{
  "success": true,
  "message": "管理者へのメール送信が完了しました",
  "messageId": "msg_abc123xyz"
}
```

**エラーレスポンス例:**
```json
{
  "success": false,
  "error": "メール送信に失敗しました",
  "details": "SendGrid API error: Invalid API key"
}
```

**Edge Function実装例:**
```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts'

serve(async (req) => {
  try {
    const { p_to, user_name, user_email, content, description } = await req.json()

    // SendGrid API でメール送信
    const response = await fetch('https://api.sendgrid.com/v3/mail/send', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${Deno.env.get('SENDGRID_API_KEY')}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        personalizations: [{
          to: [{ email: p_to }],
          subject: `【問い合わせ】${user_name}様からのお問い合わせ`,
        }],
        from: { email: 'noreply@example.com' },
        content: [{
          type: 'text/plain',
          value: `
お問い合わせ者: ${user_name}
メールアドレス: ${user_email}

【お問い合わせ内容】
${content}

【補足】
${description || 'なし'}
          `
        }]
      })
    })

    if (!response.ok) {
      throw new Error('SendGrid API error')
    }

    return new Response(
      JSON.stringify({ success: true, message: '管理者へのメール送信が完了しました' }),
      { headers: { 'Content-Type': 'application/json' } }
    )
  } catch (error) {
    return new Response(
      JSON.stringify({ success: false, error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    )
  }
})
```

---

### 4. ユーザーへの問い合わせ完了メール送信 (Edge Function)

#### `POST /functions/v1/send_email_contact_user`

問い合わせを送信したユーザーに自動返信メールを送信します。

**エンドポイント:**
```
POST https://rqkwsjwvmnoozsfpupnt.supabase.co/functions/v1/send_email_contact_user
```

**DB種別:** Edge Function (Deno)

**リクエストボディ:**

| パラメータ | 型 | 必須 | 説明 |
|---------|-----|------|------|
| p_to | string | Yes | 問い合わせ者のメールアドレス |
| user_name | string | Yes | 問い合わせ者名 |

**リクエスト例:**
```bash
curl -X POST \
  'https://rqkwsjwvmnoozsfpupnt.supabase.co/functions/v1/send_email_contact_user' \
  -H 'apikey: YOUR_ANON_KEY' \
  -H 'Authorization: Bearer YOUR_ANON_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "p_to": "yamada@example.com",
    "user_name": "山田太郎"
  }'
```

**レスポンス例:**
```json
{
  "success": true,
  "message": "ユーザーへの確認メール送信が完了しました",
  "messageId": "msg_def456uvw"
}
```

**エラーレスポンス例:**
```json
{
  "success": false,
  "error": "メール送信に失敗しました",
  "details": "Invalid email address"
}
```

**Edge Function実装例:**
```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts'

serve(async (req) => {
  try {
    const { p_to, user_name } = await req.json()

    // SendGrid API でメール送信
    const response = await fetch('https://api.sendgrid.com/v3/mail/send', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${Deno.env.get('SENDGRID_API_KEY')}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        personalizations: [{
          to: [{ email: p_to }],
          subject: 'お問い合わせありがとうございます',
        }],
        from: { email: 'noreply@example.com' },
        content: [{
          type: 'text/plain',
          value: `
${user_name} 様

この度はお問い合わせいただき、誠にありがとうございます。

お問い合わせ内容を確認いたしました。
2〜3営業日以内に担当者より返信させていただきます。

今しばらくお待ちくださいますようお願い申し上げます。

---
ポートフォリオサイト運営チーム
          `
        }]
      })
    })

    if (!response.ok) {
      throw new Error('SendGrid API error')
    }

    return new Response(
      JSON.stringify({ success: true, message: 'ユーザーへの確認メール送信が完了しました' }),
      { headers: { 'Content-Type': 'application/json' } }
    )
  } catch (error) {
    return new Response(
      JSON.stringify({ success: false, error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    )
  }
})
```

---

## React (TypeScript) での実装例

### 実績一覧の取得

```typescript
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  'https://rqkwsjwvmnoozsfpupnt.supabase.co',
  'YOUR_ANON_KEY'
)

interface Work {
  id: string
  title: string
  image: string | null
  description: string | null
  order_num: number
  is_public: boolean
  created_at: string
  updated_at: string
  deleted_at: string | null
}

async function fetchWorks(): Promise<Work[]> {
  const { data, error } = await supabase
    .from('mst_work')
    .select('*')
    .eq('is_public', true)
    .is('deleted_at', null)
    .order('order_num', { ascending: true })
    .order('created_at', { ascending: false })

  if (error) {
    console.error('Error fetching works:', error)
    return []
  }

  return data as Work[]
}
```

### 問い合わせフォームの送信

```typescript
interface ContactFormData {
  userName: string
  email: string
  content: string
  description?: string
}

async function submitContact(formData: ContactFormData): Promise<boolean> {
  try {
    // 1. 問い合わせデータをDBに保存
    const { data, error: dbError } = await supabase.rpc('create_contact', {
      p_user_name: formData.userName,
      p_email: formData.email,
      p_content: formData.content,
      p_description: formData.description || null,
    })

    if (dbError) throw dbError

    // 2. 管理者へメール送信
    const adminEmailResponse = await supabase.functions.invoke(
      'send_email_contact_admin',
      {
        body: {
          p_to: 'admin@example.com',
          user_name: formData.userName,
          user_email: formData.email,
          content: formData.content,
          description: formData.description,
        },
      }
    )

    // 3. ユーザーへ確認メール送信
    const userEmailResponse = await supabase.functions.invoke(
      'send_email_contact_user',
      {
        body: {
          p_to: formData.email,
          user_name: formData.userName,
        },
      }
    )

    return true
  } catch (error) {
    console.error('Error submitting contact:', error)
    return false
  }
}
```

---

## エラーハンドリング

### 共通エラーコード

| HTTPステータス | 説明 | 対処法 |
|--------------|------|--------|
| 400 | Bad Request - リクエストパラメータが不正 | パラメータの型や必須項目を確認 |
| 401 | Unauthorized - 認証エラー | APIキーを確認 |
| 403 | Forbidden - アクセス権限なし | RLSポリシーを確認 |
| 404 | Not Found - リソースが見つからない | エンドポイントURLを確認 |
| 500 | Internal Server Error - サーバーエラー | ログを確認し、管理者に連絡 |

### PostgRESTエラー例

```json
{
  "code": "PGRST116",
  "details": "The result contains 0 rows",
  "hint": null,
  "message": "JSON object requested, multiple (or no) rows returned"
}
```

### Edge Functionエラー例

```json
{
  "success": false,
  "error": "メール送信に失敗しました",
  "details": "SendGrid API error: Invalid API key"
}
```

---

## セキュリティ考慮事項

### RLS (Row Level Security) ポリシー

**mst_work:**
```sql
-- 公開されている実績は誰でも閲覧可能
CREATE POLICY "公開実績は誰でも閲覧可能"
ON mst_work FOR SELECT
USING (is_public = true AND deleted_at IS NULL);
```

**mst_contact:**
```sql
-- 誰でも問い合わせ投稿可能
CREATE POLICY "誰でも問い合わせ投稿可能"
ON mst_contact FOR INSERT
WITH CHECK (true);

-- 閲覧は管理者のみ(将来実装)
```

### APIキーの管理

- **Anon Key**: フロントエンド用(公開可能)
- **Service Role Key**: バックエンド専用(絶対に公開しない)
- **SendGrid API Key**: Edge Function環境変数で管理

### レート制限

- PostgREST: プロジェクト設定で制限可能
- Edge Functions: 1リクエスト/秒 推奨
- SendGrid: 送信上限に注意(フリープランは100通/日)

---

## 環境変数

Edge Functionで使用する環境変数:

```bash
SENDGRID_API_KEY=SG.xxxxxxxxxxxxx
ADMIN_EMAIL=admin@example.com
SUPABASE_URL=https://rqkwsjwvmnoozsfpupnt.supabase.co
SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

設定方法:
```bash
# Supabase CLI
supabase secrets set SENDGRID_API_KEY=your_api_key
```

---

## テスト方法

### cURLでのテスト

```bash
# 実績一覧取得(表示順序順)
curl -X GET \
  'https://rqkwsjwvmnoozsfpupnt.supabase.co/rest/v1/mst_work?select=*&order=order_num.asc' \
  -H 'apikey: YOUR_ANON_KEY'

# 問い合わせ送信
curl -X POST \
  'https://rqkwsjwvmnoozsfpupnt.supabase.co/rest/v1/rpc/create_contact' \
  -H 'apikey: YOUR_ANON_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"p_user_name":"テストユーザー","p_email":"test@example.com","p_content":"テスト問い合わせ"}'
```

### Postmanでのテスト

1. Collection作成
2. 環境変数設定(`SUPABASE_URL`, `ANON_KEY`)
3. 各エンドポイントをリクエスト追加
4. Testsタブでアサーション追加

---

## 今後の拡張予定

- [ ] 実績の詳細取得API (`GET /rest/v1/mst_work?id=eq.{id}`)
- [ ] 画像アップロードAPI (Supabase Storage)
- [ ] 管理者認証API
- [ ] 問い合わせステータス更新API
- [ ] ページネーション対応
- [ ] 全文検索機能

---

## 変更履歴

| 日付 | バージョン | 変更内容 |
|------|----------|---------|
| 2025-02-07 | 1.0.0 | 初版作成 |

---

## 参考リンク

- [Supabase Documentation](https://supabase.com/docs)
- [PostgREST API Reference](https://postgrest.org/en/stable/api.html)
- [SendGrid API Documentation](https://docs.sendgrid.com/api-reference)
- [Supabase Edge Functions](https://supabase.com/docs/guides/functions)