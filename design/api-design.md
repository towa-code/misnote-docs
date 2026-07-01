# API設計

## 基本情報

| 項目 | 内容 |
|-----|-----|
| ベースURL | `https://api.misnote.com/v1` |
| 認証方式 | Amazon Cognito (JWTトークン) |
| データ形式 | JSON|

### 認証ヘッダー
```
Authorization: Bearer {CognitoのJWTトークン}
```

---

## エンドポイント一覧

### 科目（subjects）

| メソッド | パス | 説明 |
|---------|------|------|
| GET | `/subjects` | 科目一覧取得 |
| POST | `/subjects` | 科目作成 |
| PUT | `/subjects/{id}` | 科目更新 |
| DELETE | `/subjects/{id}` | 科目削除 |

### 単元（units）

| メソッド | パス | 説明 |
|---------|------|------|
| GET | `/subjects/{subject_id}/units` | 単元一覧取得 |
| POST | `/subjects/{subject_id}/units` | 単元作成 |
| PUT | `/units/{id}` | 単元更新 |
| DELETE | `/units/{id}` | 単元削除 |

### 問題（questions）

| メソッド | パス | 説明 |
|---------|------|------|
| GET | `/questions` | 問題一覧取得 |
| GET | `/questions/{id}` | 問題詳細取得 |
| POST | `/questions` | 問題作成 |
| PUT | `/questions/{id}` | 問題更新 |
| DELETE | `/questions/{id}` | 問題削除 |

### 回答（attempts）

| メソッド | パス | 説明 |
|---------|------|------|
| POST | `/questions/{id}/attempts` | 回答を記録 |
| GET | `/questions/{id}/attempts` | 回答履歴一覧取得 |

### 間違いノート（mistake_notes）

| メソッド | パス | 説明 |
|---------|------|------|
| GET | `/mistake-notes` | 苦手問題一覧取得 |
| GET | `/mistake-notes/today` | 今日の復習一覧取得 |
| GET | `/mistake-notes/mastered` | 克服済み一覧取得 |
| GET | `/mistake-notes/{id}` | 間違いノート詳細取得 |
| PUT | `/mistake-notes/{id}` | メモ・復習日を更新 |
| PUT | `/mistake-notes/{id}/status` | ステータス変更（克服済み化・苦手に戻す） |

### ページネーション（一覧系API共通）

一覧を返すAPI（`GET /questions`、`GET /mistake-notes` 系）は以下のクエリパラメータを受け付ける。

| パラメータ | 型 | デフォルト | 説明 |
|-----------|-----|----------|------|
| limit | INTEGER | 100 | 取得件数の上限 |
| offset | INTEGER | 0 | 取得開始位置 |

---

## 詳細仕様

### 科目一覧取得
`GET /subjects`

**レスポンス**
```json
[
  { "id": "uuid", "name": "数学" },
  { "id": "uuid", "name": "英語" }
]
```

---

### 科目作成
`POST /subjects`

**リクエスト**
```json
{ "name": "数学" }
```

**レスポンス**
```json
{ "id": "uuid", "name": "数学" }
```

---

### 科目更新
`PUT /subjects/{id}`

**リクエスト**
```json
{ "name": "数学（改訂）" }
```

**レスポンス**
```json
{ "id": "uuid", "name": "数学（改訂）" }
```

---

### 科目削除
`DELETE /subjects/{id}`

**レスポンス**
- `204 No Content`（成功時）
- `409 Conflict`（紐づく単元または問題が存在する場合）

```json
// 409 の場合
{ "detail": "Subject has related units or questions" }
```

---

### 単元一覧取得
`GET /subjects/{subject_id}/units`

**レスポンス**
```json
[
  { "id": "uuid", "subject_id": "uuid", "name": "二次方程式" },
  { "id": "uuid", "subject_id": "uuid", "name": "因数分解" }
]
```

---

### 単元作成
`POST /subjects/{subject_id}/units`

**リクエスト**
```json
{ "name": "二次方程式" }
```

**レスポンス**
```json
{ "id": "uuid", "subject_id": "uuid", "name": "二次方程式" }
```

---

### 単元更新
`PUT /units/{id}`

**リクエスト**
```json
{ "name": "二次方程式（応用）" }
```

**レスポンス**
```json
{ "id": "uuid", "subject_id": "uuid", "name": "二次方程式（応用）" }
```

---

### 単元削除
`DELETE /units/{id}`

**レスポンス**
- `204 No Content`（成功時）
- `409 Conflict`（紐づく問題が存在する場合）

```json
// 409 の場合
{ "detail": "Unit has related questions" }
```

---

### 問題一覧取得
`GET /questions`

**クエリパラメータ**

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| subject_id | UUID | No | 科目で絞り込む |
| unit_id | UUID | No | 単元で絞り込む |
| limit / offset | INTEGER | No | ページネーション（共通仕様参照） |

**レスポンス**
```json
[
  {
    "id": "uuid",
    "subject": { "id": "uuid", "name": "数学" },
    "unit": { "id": "uuid", "name": "二次方程式" },
    "question_text": "次の方程式を解け: 2x + 3 = 7",
    "correct_answer": "x = 2",
    "created_at": "2024-01-01T00:00:00"
  }
]
```

> `unit` は登録時に未指定の場合 `null` になる

---

### 問題詳細取得
`GET /questions/{id}`

**レスポンス**
```json
{
  "id": "uuid",
  "subject": { "id": "uuid", "name": "数学" },
  "unit": { "id": "uuid", "name": "二次方程式" },
  "question_text": "次の方程式を解け: 2x + 3 = 7",
  "correct_answer": "x = 2",
  "created_at": "2024-01-01T00:00:00"
}
```

---

### 問題作成
`POST /questions`

**リクエスト**
```json
{
  "subject_id": "uuid",
  "unit_id": "uuid",
  "question_text": "次の方程式を解け: 2x + 3 = 7",
  "correct_answer": "x = 2",
  "memo": "符号のミスに注意",
  "learning": "移項するとき符号が反転することを公式として覚える",
  "next_review_at": "2024-01-08"
}
```

> `unit_id`・`memo`・`learning`・`next_review_at` は省略可。`unit_id` を指定する場合は `subject_id` の科目に属する単元であること（違反時は 400）。
> `memo`・`learning`・`next_review_at` の**いずれかが指定された場合**、`mistake_notes` レコードが自動で作成される（初期値: `status=active`、`wrong_count=1`、`correct_streak=0`、`next_review_at` 未指定時は `null`）。

**レスポンス**
```json
{
  "id": "uuid",
  "subject": { "id": "uuid", "name": "数学" },
  "unit": { "id": "uuid", "name": "二次方程式" },
  "question_text": "次の方程式を解け: 2x + 3 = 7",
  "correct_answer": "x = 2",
  "created_at": "2024-01-01T00:00:00",
  "mistake_note_id": "uuid"
}
```

> `memo`・`learning`・`next_review_at` をいずれも渡さなかった場合、`mistake_note_id` は `null`

---

### 問題更新
`PUT /questions/{id}`

**リクエスト**
```json
{
  "subject_id": "uuid",
  "unit_id": "uuid",
  "question_text": "次の方程式を解け: 3x + 6 = 12",
  "correct_answer": "x = 2"
}
```

**レスポンス** — `GET /questions/{id}` と同じ形

> POST と同様、`unit_id` は `subject_id` の科目に属する単元であること（違反時は 400）

---

### 問題削除
`DELETE /questions/{id}`

**レスポンス**
- `204 No Content`

> 紐づく `attempts`・`mistake_notes` もカスケード削除される

---

### 回答を記録
`POST /questions/{id}/attempts`

**リクエスト**
```json
{
  "user_answer": "x = 3",
  "is_correct": false
}
```

> `user_answer` は省略可（自己採点方式のため、答えを書かずに正誤だけ記録できる）

**レスポンス**
```json
{
  "id": "uuid",
  "question_id": "uuid",
  "user_answer": "x = 3",
  "is_correct": false,
  "answered_at": "2024-01-01T00:00:00",
  "mistake_note_id": "uuid",
  "correct_streak": 0,
  "mastery_suggested": false
}
```

**mistake_notes への副作用**

| 条件 | 挙動 |
|------|------|
| 不正解・ノート無し | ノートを自動作成（`wrong_count=1`、`correct_streak=0`、`next_review_at=null`。復習日はユーザーが後から設定する） |
| 不正解・ノート有り | 既存ノートを更新：`wrong_count` +1、`correct_streak` を 0 にリセット。`mastered` だった場合は `active` に戻す（重複ノートは作らない） |
| 正解・ノート有り | `correct_streak` +1 |
| 正解・ノート無し | 何もしない（`mistake_note_id`・`correct_streak`・`mastery_suggested` は `null`） |

> `mastery_suggested` は更新後の `correct_streak` が **3 以上**のとき `true`。フロントエンドはこれを見て「克服済みにしますか？」と提案する（mastered への変更は別途 `PUT /mistake-notes/{id}/status` で行う。自動では遷移しない）

---

### 回答履歴一覧取得
`GET /questions/{id}/attempts`

**レスポンス**
```json
[
  {
    "id": "uuid",
    "question_id": "uuid",
    "user_answer": "x = 3",
    "is_correct": false,
    "answered_at": "2024-01-01T00:00:00"
  }
]
```

> `answered_at` の降順で返す

---

### 苦手問題一覧取得
`GET /mistake-notes`

status が `active` の一覧を返す。

**レスポンス**
```json
[
  {
    "id": "uuid",
    "question": {
      "id": "uuid",
      "subject": { "id": "uuid", "name": "数学" },
      "unit": { "id": "uuid", "name": "二次方程式" },
      "question_text": "次の方程式を解け: 2x + 3 = 7",
      "correct_answer": "x = 2"
    },
    "memo": "符号のミスに注意",
    "learning": "移項するとき符号が反転することを公式として覚える",
    "status": "active",
    "wrong_count": 3,
    "correct_streak": 1,
    "next_review_at": "2024-01-08"
  }
]
```

> 画面で科目名・単元名を表示するため、`question` には `subject`・`unit` を含める（`unit` は未設定なら `null`）。以降の mistake-notes 系レスポンスも同じ形

---

### 今日の復習一覧取得
`GET /mistake-notes/today`

`next_review_at` が今日以前で `status` が `active` の一覧を返す。

> `next_review_at` が `null` のノートは含まれない。未設定分の扱い（「復習日未設定」セクション）は [画面設計](./screen-design.md) を参照

**レスポンス**
```json
[
  {
    "id": "uuid",
    "question": {
      "id": "uuid",
      "subject": { "id": "uuid", "name": "数学" },
      "unit": { "id": "uuid", "name": "二次方程式" },
      "question_text": "次の方程式を解け: 2x + 3 = 7",
      "correct_answer": "x = 2"
    },
    "memo": "符号のミスに注意",
    "learning": "移項するとき符号が反転することを公式として覚える",
    "status": "active",
    "wrong_count": 3,
    "correct_streak": 1,
    "next_review_at": "2024-01-01"
  }
]
```

---

### 克服済み一覧取得
`GET /mistake-notes/mastered`

status が `mastered` の一覧を返す。

**レスポンス**
```json
[
  {
    "id": "uuid",
    "question": {
      "id": "uuid",
      "subject": { "id": "uuid", "name": "数学" },
      "unit": { "id": "uuid", "name": "二次方程式" },
      "question_text": "次の方程式を解け: 2x + 3 = 7",
      "correct_answer": "x = 2"
    },
    "memo": "符号のミスに注意",
    "learning": "移項するとき符号が反転することを公式として覚える",
    "status": "mastered",
    "wrong_count": 3,
    "correct_streak": 3,
    "next_review_at": null
  }
]
```

---

### 間違いノート詳細取得
`GET /mistake-notes/{id}`

復習画面への直接アクセス（URL指定・リロード）で使う。

**レスポンス** — 一覧の1要素と同じ形（`question` に `subject`・`unit` を含む）

---

### メモ・復習日を更新
`PUT /mistake-notes/{id}`

**リクエスト**
```json
{
  "memo": "符号のミスに注意。移項するとき符号が変わる",
  "learning": "移項するとき符号が反転することを公式として覚える",
  "next_review_at": "2024-01-08"
}
```

**レスポンス**
```json
{
  "id": "uuid",
  "memo": "符号のミスに注意。移項するとき符号が変わる",
  "learning": "移項するとき符号が反転することを公式として覚える",
  "next_review_at": "2024-01-08",
  "status": "active"
}
```

---

### ステータス変更
`PUT /mistake-notes/{id}/status`

克服済みへの変更（`mastered`）と、克服済みから苦手への復帰（`active`）の両方に使う。

**リクエスト**
```json
{ "status": "mastered" }
```

**レスポンス**
```json
{
  "id": "uuid",
  "status": "mastered",
  "question": {
    "id": "uuid",
    "subject": { "id": "uuid", "name": "数学" },
    "unit": { "id": "uuid", "name": "二次方程式" },
    "question_text": "次の方程式を解け: 2x + 3 = 7",
    "correct_answer": "x = 2"
  },
  "memo": "符号のミスに注意",
  "learning": "移項するとき符号が反転することを公式として覚える",
  "wrong_count": 3,
  "correct_streak": 3,
  "next_review_at": null
}
```

> `mastered` にすると `next_review_at` は `null` にクリアされる。`active` に戻した場合、`correct_streak` は 0 にリセットし、復習日はユーザーが改めて設定する

---

## エラーレスポンス

| ステータスコード | 意味 |
|----------------|------|
| 400 | ビジネスルール違反（例：単元が指定科目に属していない） |
| 401 | 認証エラー（トークンが無効） |
| 403 | 権限エラー（他のユーザーのデータにアクセス） |
| 404 | データが見つからない |
| 409 | 競合エラー（紐づくデータが存在するため削除不可） |
| 422 | バリデーションエラー（必須項目の欠落・型不一致。FastAPI が自動で返す） |
| 500 | サーバーエラー |

> 形式チェック（Pydantic）は 422、形式は正しいが業務的に不正なリクエストは 400 を使う

**エラーレスポンスの形式**
```json
{
  "detail": "Question not found"
}
```

---

## OpenAPI Generator

### 概要

FastAPIは実装するだけで `openapi.yaml` を自動生成する。
そのyamlをopenapi-generatorに渡すことで、Next.js用のコードを自動生成できる。

### 生成されるもの

| 生成物 | 使う場所 | 説明 |
|--------|---------|------|
| TypeScript型定義 | Next.js | APIのリクエスト・レスポンスの型 |
| APIクライアント | Next.js | fetchを自分で書かなくていい |

### 手順

**① FastAPI起動後、yamlを取得**
```bash
curl http://localhost:8000/openapi.json -o openapi.yaml
```

**② openapi-generatorでNext.js用コードを生成**
```bash
openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-fetch \
  -o ./frontend/src/generated
```

**③ 生成されたコードを使う**
```typescript
// 自分でfetchを書く代わりにこれだけでOK
import { QuestionsApi } from '@/generated'

const api = new QuestionsApi()
const questions = await api.getQuestions()
```

### 生成されるフォルダ構成

```
frontend/src/generated/
├── apis/
│   ├── QuestionsApi.ts
│   ├── MistakeNotesApi.ts
│   └── SubjectsApi.ts
└── models/
    ├── Question.ts
    ├── MistakeNote.ts
    └── Attempt.ts
```

## 関連ドキュメント

- [アプリ概要](./overview.md)
- [DB設計](./db-design.md)
- [画面設計](./screen-design.md)
