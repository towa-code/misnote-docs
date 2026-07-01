# misnote 実装ロードマップ

クラウド（AWS）は最終フェーズまで使わず、ローカル環境で動くものを段階的に完成させる方針。

---

## フェーズ概要

```
Phase 0  環境構築          ─ Docker + 骨格作成
Phase 1  バックエンドAPI   ─ FastAPI + PostgreSQL（ローカル）
Phase 2  フロントエンド    ─ Next.js + 生成クライアント
Phase 3  認証              ─ シンプルJWT（ローカル）
Phase 4  クラウド移行      ─ AWS（RDS / ECS / Cognito）
```

---

## Phase 0 — 環境構築

**目標：** `docker compose up` で FastAPI と PostgreSQL が起動し、Next.js dev server が API を叩ける状態にする。

### リポジトリ構成

```
misnote/
├── backend/          # FastAPI
│   ├── app/
│   │   ├── main.py
│   │   ├── models/   # SQLAlchemy
│   │   ├── schemas/  # Pydantic
│   │   └── routers/
│   ├── alembic/
│   └── requirements.txt
├── frontend/         # Next.js
│   └── src/
│       └── generated/  # openapi-generator が出力するクライアント
└── docker-compose.yml
```

### docker-compose.yml（最小構成）

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: misnote
      POSTGRES_USER: misnote
      POSTGRES_PASSWORD: misnote
    ports:
      - "5432:5432"

  api:
    build: ./backend
    ports:
      - "8000:8000"
    depends_on:
      - db
    environment:
      DATABASE_URL: postgresql://misnote:misnote@db:5432/misnote
```

### チェックリスト

- [ ] `docker compose up` で PostgreSQL 起動確認
- [ ] FastAPI 起動・`http://localhost:8000/docs` で Swagger UI 確認
- [ ] Next.js `npm run dev` で `http://localhost:3000` 確認
- [ ] frontend から `http://localhost:8000` への CORS 許可設定

---

## Phase 1 — バックエンドAPI

**目標：** 設計書通りの全エンドポイントをローカルで動かす。認証はいったん不要（全リクエストを単一のシードユーザーとして処理）。

### Step 1-1: DB モデル・マイグレーション

SQLAlchemy モデルを作成し、Alembic でマイグレーション管理する。

実装順序（外部キー依存の順）：

1. `users`
2. `subjects`
3. `units`（`subject_id` → `subjects`）
4. `questions`（`subject_id`, `unit_id` → `subjects`, `units`）
5. `mistake_notes`（`question_id` → `questions`、UNIQUE制約）
6. `attempts`（`question_id` → `questions`）

### Step 1-2: エンドポイント実装

機能の依存関係が少ない順に実装する：

| 順序 | エンドポイント群 | 理由 |
|------|-----------------|------|
| 1 | `GET/POST/PUT/DELETE /subjects` | 他に依存しない |
| 2 | `GET/POST/PUT/DELETE /subjects/{id}/units` | subjects に依存 |
| 3 | `POST /questions` | subjects・units に依存、mistake_notes の自動生成含む |
| 4 | `GET/PUT /mistake-notes`, `GET /mistake-notes/today`, `GET /mistake-notes/{id}` | questions に依存 |
| 5 | `POST/GET /questions/{id}/attempts` | mistake_notes の更新ロジック（wrong_count 加算・correct_streak 更新・mastered→active 復帰）を含む |
| 6 | `PUT /mistake-notes/{id}/status` | mistake_notes に依存（克服済み化・苦手に戻す） |

### Step 1-3: OpenAPI エクスポート確認

```bash
# FastAPI が /openapi.json を自動生成する
curl http://localhost:8000/openapi.json -o frontend/openapi.json
```

### チェックリスト

- [ ] Alembic マイグレーション成功
- [ ] 全エンドポイントを Swagger UI から手動テスト
- [ ] `POST /questions`で `memo` を渡したとき `mistake_notes` が自動生成されること
- [ ] 不正解の attempt で既存ノートの `wrong_count` が加算され、重複ノートが作られないこと
- [ ] 3回連続正解で attempt レスポンスの `mastery_suggested` が `true` になること（mastered へは自動遷移しないこと）
- [ ] `GET /mistake-notes/today` が `next_review_at <= today` の結果のみ返すこと
- [ ] `openapi.json` が出力されること

---

## Phase 2 — フロントエンド

**目標：** openapi-generator で生成した TypeScript クライアントを使い、全5画面を実装する。

### Step 2-1: openapi-generator セットアップ

```bash
# frontend/package.json に追加
npx @openapitools/openapi-generator-cli generate \
  -i ../backend/openapi.json \
  -g typescript-fetch \
  -o src/generated
```

`src/generated/` には型定義と fetch クライアントが自動生成される。手書き fetch は書かない。

### Step 2-2: 画面実装順序

データ依存の少ない画面から実装する：

| 順序 | 画面 | 依存するAPI |
|------|------|------------|
| 1 | 科目・単元管理 | subjects / units |
| 2 | 問題登録 | subjects / units / questions |
| 3 | ホーム（今日の復習） | mistake-notes/today |
| 4 | 苦手問題一覧 | mistake-notes |
| 5 | 復習 | attempts / mistake-notes |

### チェックリスト

- [ ] openapi-generator が型エラーなく生成されること
- [ ] 科目・単元を追加・編集・削除できること
- [ ] 問題を登録すると苦手問題一覧に表示されること
- [ ] ホームに今日の復習問題が表示されること
- [ ] 復習フロー（問題 → 答え → 正解/不正解 → 保存）が一通り動くこと

---

## Phase 3 — 認証（ローカルJWT）

**目標：** ユーザー登録・ログインを実装し、全APIリクエストに JWT を付与する。Cognito は使わず、FastAPI 側で JWT を発行・検証する。

### バックエンド

- `POST /auth/register`（メール + パスワード → ユーザー作成）
- `POST /auth/login`（メール + パスワード → JWT 返却）
- 全エンドポイントに `Depends(get_current_user)` を追加
- ライブラリ: `python-jose`, `passlib[bcrypt]`

### フロントエンド

- ログイン画面を追加
- JWT を `localStorage` または `httpOnly Cookie` に保存
- 生成クライアントのリクエストヘッダーに `Authorization: Bearer {token}` を設定
- 未認証時はログイン画面にリダイレクト

### チェックリスト

- [ ] 新規ユーザーが登録・ログインできること
- [ ] 認証なしのAPIアクセスが 401 を返すこと
- [ ] ログアウト後に再アクセスするとログイン画面に戻ること

---

## Phase 4 — クラウド移行（AWS）

**目標：** ローカルで動いているアプリをそのまま AWS に載せる。コードの変更は認証部分（Cognito 差し替え）のみに留める。

### インフラ構成

```
ユーザー
  │
  ├─ Next.js (Vercel or Amplify)
  │
  └─ API Gateway
       │
       └─ ECS + Fargate（FastAPIコンテナ）
            │
            ├─ RDS PostgreSQL
            └─ CloudWatch（ログ）
```

### 移行手順

1. **RDS PostgreSQL** を作成 → `DATABASE_URL` を差し替え、Alembic マイグレーションを実行
2. **ECR** に FastAPI Docker イメージをプッシュ
3. **ECS + Fargate** でコンテナを起動
4. **Cognito** ユーザープールを作成 → FastAPI の JWT 検証を Cognito 公開鍵に切り替え
5. **API Gateway** で ECS にルーティング
6. **フロントエンド** の `NEXT_PUBLIC_API_BASE_URL` を `https://api.misnote.com/v1` に変更

### チェックリスト

- [ ] RDS に接続した状態でマイグレーション成功
- [ ] ECS コンテナが起動し、`/health` エンドポイントが 200 を返すこと
- [ ] Cognito で発行した JWT で API が認証されること
- [ ] フロントエンドから本番 API に対して全フローが動くこと

---

## 関連ドキュメント

- [アプリ概要](./design/overview.md)
- [DB設計](./design/db-design.md)
- [API設計](./design/api-design.md)
- [画面設計](./design/screen-design.md)
