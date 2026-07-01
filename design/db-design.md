# DB設計：間違いノートアプリ

## テーブル一覧

| テーブル | 役割 |
|---------|------|
| `users` | ユーザー情報 |
| `subjects` | 科目 |
| `units` | 単元（科目の下の階層） |
| `questions` | 問題 |
| `attempts` | 回答履歴 |
| `mistake_notes` | 間違いノート＋復習管理 |

---

## 各テーブル詳細

### `users`（ユーザー）

| カラム名 | 型 | 説明 |
|---------|-----|------|
| id | UUID | 主キー |
| cognito_sub | VARCHAR | Cognito のユーザーID（UNIQUE、nullable） |
| email | VARCHAR | メールアドレス（UNIQUE） |
| name | VARCHAR | 表示名 |
| created_at | TIMESTAMP | 作成日時 |

> 認証は段階的に移行する（[ROADMAP](../ROADMAP.md) 参照）。Phase 3 のローカルJWT期間は `password_hash VARCHAR` を追加して使い、Phase 4 で Cognito に切り替えた時点で `cognito_sub` に紐付ける。

---

### `subjects`（科目）

| カラム名 | 型 | 説明 |
|---------|-----|------|
| id | UUID | 主キー |
| user_id | UUID | → users |
| name | VARCHAR | 科目名（数学・英語など） |
| created_at | TIMESTAMP | 作成日時 |
| updated_at | TIMESTAMP | 更新日時 |

---

### `units`（単元）

| カラム名 | 型 | 説明 |
|---------|-----|------|
| id | UUID | 主キー |
| subject_id | UUID | → subjects |
| name | VARCHAR | 単元名（二次方程式など） |
| created_at | TIMESTAMP | 作成日時 |
| updated_at | TIMESTAMP | 更新日時 |

---

### `questions`（問題）

| カラム名 | 型 | 説明 |
|---------|-----|------|
| id | UUID | 主キー |
| user_id | UUID | → users |
| subject_id | UUID | → subjects |
| unit_id | UUID | → units（nullable） |
| question_text | TEXT | 問題文 |
| correct_answer | TEXT | 正解 |
| created_at | TIMESTAMP | 作成日時 |
| updated_at | TIMESTAMP | 更新日時 |

> `unit_id` を指定する場合、その単元は `subject_id` の科目に属していなければならない（アプリケーション側で検証する）

---

### `attempts`（回答履歴）

| カラム名 | 型 | 説明 |
|---------|-----|------|
| id | UUID | 主キー |
| user_id | UUID | → users |
| question_id | UUID | → questions |
| user_answer | TEXT | 自分の答え（任意・nullable） |
| is_correct | BOOLEAN | 正解かどうか |
| answered_at | TIMESTAMP | 回答日時 |

---

### `mistake_notes`（間違いノート）

| カラム名 | 型 | 説明 |
|---------|-----|------|
| id | UUID | 主キー |
| user_id | UUID | → users |
| question_id | UUID | → questions（**UNIQUE**：1問題につきノートは1件） |
| memo | TEXT | 間違えた理由・ポイント |
| learning | TEXT | 今回学んだこと |
| status | ENUM | active / mastered（デフォルト active） |
| wrong_count | INTEGER | 間違えた回数（作成時 1） |
| correct_streak | INTEGER | 連続正解回数（デフォルト 0） |
| next_review_at | DATE | ユーザーが設定する次の復習日（nullable） |
| created_at | TIMESTAMP | 作成日時 |
| updated_at | TIMESTAMP | 更新日時 |

---

## テーブル関係図

```
users
 ├──< subjects
 │      └──< units
 │
 ├──< questions >── subjects
 │                └── units（nullable）
 │
 ├──< attempts >── questions
 │
 └──< mistake_notes >── questions
```

---

## インデックス

| テーブル | インデックス | 用途 |
|---------|------------|------|
| mistake_notes | (user_id, status, next_review_at) | 「今日の復習」クエリ（`GET /mistake-notes/today`） |
| mistake_notes | UNIQUE (question_id) | 1問題1ノートの保証 |
| questions | (user_id, subject_id) | 問題一覧の科目絞り込み |
| attempts | (question_id, answered_at) | 回答履歴の取得 |
| subjects | (user_id) | 科目一覧 |
| units | (subject_id) | 単元一覧 |

---

## 設計のポイント

- 全テーブルの主キーは UUID。全データは `user_id` でスコープし、他ユーザーのデータにはアクセスできない（`units` は `subject_id` 経由で所有者を判定する）
- `unit_id` はnullableにすることで、単元未分類の問題も登録できる
- `attempts` は同じ問題を何度解いても履歴として残る
- `mistake_notes` は `question_id` にUNIQUE制約を持ち、1問題につき1件。不正解の attempt を記録したとき、ノートが無ければ自動作成し、あれば既存ノートを更新する（重複は作らない）
- `next_review_at` はユーザーが自分で次の復習日を設定する（自動計算しない）

### correct_streak と克服（mastered）のルール

- 正解の attempt → `correct_streak` を +1
- 不正解の attempt → `correct_streak` を 0 にリセットし、`wrong_count` を +1
- `correct_streak` が閾値（**3回連続正解**）に達すると、UIが「克服済みにしますか？」と**提案**する
- `mastered` への遷移は常にユーザー操作（自動では遷移しない）。提案の有無にかかわらず手動で克服済みにできる
- `mastered` のノートの問題に不正解の attempt が記録された場合、`status` を `active` に戻し、`correct_streak` をリセットする
- `status` が `mastered` の問題は復習リストから外れる