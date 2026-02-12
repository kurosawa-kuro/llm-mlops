# Phase 3 - 企業向け強化

**Enterprise Knowledge RAG Platform：仕様・基本設計**

---

## 1. 概要・スコープ

### 目的

Phase 1〜2 で構築した RAG パイプラインを企業環境で運用可能な状態に引き上げる。
認証・認可によるアクセス制御、AWS への移行パス、CI/CD パイプライン、本番品質の Docker Compose 構成を整備する。

### 前提条件

Phase 1・Phase 2 が完了していること（全チェックリスト項目がクリア済み）。

### 完成定義

- ロールベースのアクセス制御（RBAC）により、ユーザーごとに操作権限が制御されている
- ストレージ層が MinIO / AWS S3 のどちらでも動作する抽象化が完了している
- GitHub Actions による CI パイプライン（lint・test・build）が動作している
- Docker Compose で全サービスがワンコマンドで起動し、本番環境と同等の構成で動作する

### スコープ外（将来拡張）

| 機能 | 備考 |
|------|------|
| Kubernetes デプロイ | EKS / Kind 対応は将来検討 |
| ArgoCD GitOps | K8s 移行後に検討 |
| MFA / 外部 IdP 連携 | OAuth / OIDC は将来拡張 |
| DuckDB 分析基盤 | README ロードマップには記載あるが別フェーズ |

---

## 2. システム構成図

### 本番構成図

```
┌──────────────────────────────────────────────────────────────┐
│                       Client                                 │
│                 (Browser / Swagger UI)                        │
└──────────────────────────┬───────────────────────────────────┘
                           │ HTTPS
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                    FastAPI Server                             │
│                                                              │
│  ┌──────────┐                                                │
│  │ Auth     │  JWT 検証 → RBAC ミドルウェア                    │
│  │ Middleware│─────────────────────────────────┐              │
│  └──────────┘                                  │              │
│                                                ▼              │
│  ┌──────────┐ ┌──────────┐ ┌───────┐ ┌──────────────────┐   │
│  │ /auth    │ │ /ingest  │ │/query │ │ /costs /templates│   │
│  └────┬─────┘ └────┬─────┘ └───┬───┘ └────────┬─────────┘   │
│       │            │           │               │             │
│  ┌────▼────────────▼───────────▼───────────────▼──────┐      │
│  │              RAG Pipeline (LlamaIndex)              │      │
│  │   + Reranker + Template Manager + Cost Tracker      │      │
│  └────────────────────────────────────────────────────┘      │
└──────────┬───────────┬──────────────┬────────────────────────┘
           │           │              │
     ┌─────▼───┐ ┌─────▼─────┐ ┌─────▼──────┐
     │ S3      │ │ PostgreSQL│ │ OpenAI API │
     │ 抽象層  │ │ (pgvector)│ │            │
     │ MinIO/  │ └───────────┘ └────────────┘
     │ AWS S3  │       │
     └─────────┘ ┌─────▼──────┐
                 │  MLflow    │
                 └────────────┘
```

### ローカル開発環境（Docker Compose 本番構成）

| サービス | イメージ | ポート | 用途 |
|---------|---------|--------|------|
| `app` | llm-mlops:latest（自前ビルド） | 8000 | FastAPI アプリケーション |
| `db` | pgvector/pgvector:pg16 | 5432 | ベクトル DB |
| `minio` | minio/minio | 9000 / 9001 | S3 互換ストレージ |
| `mlflow` | ghcr.io/mlflow/mlflow | 5050 | 実験トラッキング |
| `createbuckets` | minio/mc | - | MinIO バケット自動作成（init コンテナ） |
| `migrate` | llm-mlops:latest | - | DB マイグレーション（init コンテナ） |

---

## 3. 技術スタック（Phase 3 追加分）

| レイヤー | 技術 | バージョン方針 | 用途 |
|---------|------|--------------|------|
| 認証 | PyJWT | 最新安定版 | JWT 生成・検証 |
| パスワード | passlib[bcrypt] | 最新安定版 | パスワードハッシュ化 |
| S3 クライアント | boto3 | 最新安定版 | AWS S3 / MinIO 統一クライアント |
| CI/CD | GitHub Actions | - | lint / test / build / push |
| Docker | Docker + Docker Compose | Compose v2 | 本番構成 |
| セキュリティスキャン | Ruff + bandit + safety | 最新安定版 | 静的解析・脆弱性チェック |

**Phase 1〜2 から継続:**
Python 3.12+, FastAPI, LlamaIndex, pgvector, MLflow, OpenAI API, sentence-transformers, Jinja2, tiktoken（変更なし）

---

## 4. RBAC 設計

### 4.1 概要

auth-gateway プロジェクトの RBAC パターンを参考に、FastAPI ミドルウェアとしてロールベースのアクセス制御を実装する。
Phase 3 ではシンプルな JWT + ロール方式を採用し、将来の IdP 連携（Cognito / Keycloak）に対応可能な設計とする。

### 4.2 ロール定義

| ロール | 説明 |
|--------|------|
| `admin` | 全権限。ユーザー管理・システム設定が可能 |
| `editor` | ドキュメント登録・テンプレート管理が可能 |
| `viewer` | 検索・閲覧のみ |

### 4.3 権限定義

| 権限 | 説明 |
|------|------|
| `ingest:write` | ドキュメント登録 |
| `query:read` | RAG 検索 |
| `templates:read` | テンプレート閲覧 |
| `templates:write` | テンプレート作成・更新・削除 |
| `costs:read` | コスト閲覧 |
| `users:read` | ユーザー一覧閲覧 |
| `users:write` | ユーザー作成・更新・削除 |
| `admin:all` | 全操作権限 |

### 4.4 ロール→権限マッピング

```python
ROLE_PERMISSIONS: dict[str, list[str]] = {
    "admin": [
        "admin:all",
        "ingest:write",
        "query:read",
        "templates:read",
        "templates:write",
        "costs:read",
        "users:read",
        "users:write",
    ],
    "editor": [
        "ingest:write",
        "query:read",
        "templates:read",
        "templates:write",
        "costs:read",
    ],
    "viewer": [
        "query:read",
        "templates:read",
        "costs:read",
    ],
}
```

### 4.5 エンドポイント別権限マトリクス

| エンドポイント | 必要権限 | admin | editor | viewer |
|--------------|---------|-------|--------|--------|
| `POST /auth/login` | なし（公開） | o | o | o |
| `GET /auth/me` | 認証済み | o | o | o |
| `POST /ingest` | `ingest:write` | o | o | - |
| `POST /query` | `query:read` | o | o | o |
| `GET /costs` | `costs:read` | o | o | o |
| `GET /templates` | `templates:read` | o | o | o |
| `POST /templates` | `templates:write` | o | o | - |
| `PUT /templates/{id}` | `templates:write` | o | o | - |
| `DELETE /templates/{id}` | `templates:write` | o | o | - |
| `GET /users` | `users:read` | o | - | - |
| `POST /users` | `users:write` | o | - | - |
| `GET /health` | なし（公開） | o | o | o |

### 4.6 JWT 設計

#### Access Token Payload

```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "email": "admin@example.com",
  "role": "admin",
  "type": "access",
  "iat": 1735196400,
  "exp": 1735200000
}
```

#### JWT 設定

| 項目 | 値 |
|------|-----|
| アルゴリズム | HS256 |
| Access Token TTL | 3600 秒（1 時間） |
| Refresh Token TTL | 604800 秒（7 日） |
| 署名キー | 環境変数 `JWT_SECRET` |

### 4.7 認証ミドルウェア

```python
from fastapi import Depends, HTTPException, Request
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
) -> TokenPayload:
    """JWT を検証して現在のユーザー情報を返す"""
    try:
        payload = jwt.decode(
            credentials.credentials,
            settings.JWT_SECRET,
            algorithms=["HS256"],
        )
        return TokenPayload(**payload)
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token has expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")


def require_permission(*permissions: str):
    """指定された権限のいずれかを持つことを要求するデコレータ"""
    async def checker(user: TokenPayload = Depends(get_current_user)):
        user_permissions = ROLE_PERMISSIONS.get(user.role, [])
        if "admin:all" in user_permissions:
            return user
        if not any(p in user_permissions for p in permissions):
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user
    return checker
```

### 4.8 ルーターでの使用例

```python
@router.post("/ingest")
async def ingest_document(
    file: UploadFile,
    user: TokenPayload = Depends(require_permission("ingest:write")),
):
    ...

@router.post("/query")
async def query_rag(
    request: QueryRequest,
    user: TokenPayload = Depends(require_permission("query:read")),
):
    ...
```

### 4.9 ダミーユーザー（開発用）

```python
DUMMY_USERS = [
    {
        "email": "admin@example.com",
        "password": "Password!1",   # bcrypt ハッシュ化して保存
        "role": "admin",
    },
    {
        "email": "editor@example.com",
        "password": "Password!1",
        "role": "editor",
    },
    {
        "email": "viewer@example.com",
        "password": "Password!1",
        "role": "viewer",
    },
]
```

---

## 5. 認証 API 設計

### 5.1 エンドポイント一覧

| メソッド | パス | 説明 |
|---------|------|------|
| `POST` | `/auth/login` | ログイン（JWT 発行） |
| `POST` | `/auth/refresh` | トークンリフレッシュ |
| `GET` | `/auth/me` | 現在のユーザー情報取得 |
| `GET` | `/users` | ユーザー一覧（admin のみ） |
| `POST` | `/users` | ユーザー作成（admin のみ） |

### 5.2 POST /auth/login

**Request**

```json
{
  "email": "admin@example.com",
  "password": "Password!1"
}
```

**Response 200**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**Response 401**

```json
{
  "detail": "Invalid credentials"
}
```

### 5.3 POST /auth/refresh

**Request**

```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Response 200**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**Response 401**

```json
{
  "detail": "Invalid or expired refresh token"
}
```

### 5.4 GET /auth/me

**Headers**

```
Authorization: Bearer <access_token>
```

**Response 200**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "admin@example.com",
  "role": "admin",
  "permissions": [
    "admin:all",
    "ingest:write",
    "query:read",
    "templates:read",
    "templates:write",
    "costs:read",
    "users:read",
    "users:write"
  ]
}
```

### 5.5 GET /users（admin のみ）

**Response 200**

```json
{
  "users": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "admin@example.com",
      "role": "admin",
      "created_at": "2025-01-15T10:30:00.000Z"
    },
    {
      "id": "660e8400-e29b-41d4-a716-446655440001",
      "email": "editor@example.com",
      "role": "editor",
      "created_at": "2025-01-16T10:30:00.000Z"
    }
  ]
}
```

### 5.6 POST /users（admin のみ）

**Request**

```json
{
  "email": "newuser@example.com",
  "password": "SecurePass!1",
  "role": "viewer"
}
```

**Response 201**

```json
{
  "id": "770e8400-e29b-41d4-a716-446655440002",
  "email": "newuser@example.com",
  "role": "viewer",
  "message": "User created successfully"
}
```

**Response 400**

```json
{
  "detail": "Email already exists"
}
```

---

## 6. データベース設計（Phase 3 差分）

### 6.1 マイグレーション一覧

| ファイル | 内容 |
|---------|------|
| `005_users.sql` | users テーブル作成 |

### 6.2 005_users.sql

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(300) NOT NULL UNIQUE,
    password_hash   VARCHAR(200) NOT NULL,
    role            VARCHAR(20) NOT NULL DEFAULT 'viewer',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_users_email ON users (email);

-- ダミーユーザー挿入（パスワードは bcrypt ハッシュ）
-- Password!1 → $2b$12$ で始まるハッシュ
INSERT INTO users (email, password_hash, role) VALUES
    ('admin@example.com',  '$2b$12$LJ3m4ys3Lg9Yc3RzHSqKg.VDf9MXlYFhXvGBVpXc4fFJnQHIxqNSe', 'admin'),
    ('editor@example.com', '$2b$12$LJ3m4ys3Lg9Yc3RzHSqKg.VDf9MXlYFhXvGBVpXc4fFJnQHIxqNSe', 'editor'),
    ('viewer@example.com', '$2b$12$LJ3m4ys3Lg9Yc3RzHSqKg.VDf9MXlYFhXvGBVpXc4fFJnQHIxqNSe', 'viewer');
```

---

## 7. S3 連携設計（MinIO → AWS 移行）

### 7.1 概要

ストレージ層を抽象化し、環境変数の切り替えのみで MinIO（ローカル）と AWS S3（本番）を透過的に利用できるようにする。
boto3 は MinIO と AWS S3 の両方に対応しており、`endpoint_url` パラメータの有無で接続先を切り替える。

### 7.2 ストレージ抽象層

```python
import boto3
from botocore.config import Config

def create_s3_client():
    """環境変数に基づいてS3クライアントを生成"""
    kwargs = {
        "service_name": "s3",
        "region_name": settings.AWS_REGION,
    }

    if settings.S3_ENDPOINT_URL:
        # MinIO（ローカル開発）
        kwargs["endpoint_url"] = settings.S3_ENDPOINT_URL
        kwargs["aws_access_key_id"] = settings.MINIO_ACCESS_KEY
        kwargs["aws_secret_access_key"] = settings.MINIO_SECRET_KEY
    # AWS S3（本番）の場合は IAM ロール / 環境変数から自動取得

    return boto3.client(**kwargs)
```

### 7.3 環境別設定

| 環境 | `S3_ENDPOINT_URL` | 認証 | バケット |
|------|-------------------|------|---------|
| ローカル | `http://minio:9000` | Access Key / Secret Key | `documents` |
| AWS | 未設定（`None`） | IAM ロール / 環境変数 | `{project}-{env}-documents` |

### 7.4 ストレージインターフェース

```python
class StorageService:
    """S3/MinIO統一ストレージサービス"""

    async def upload_file(
        self, file_content: bytes, key: str, content_type: str
    ) -> str:
        """ファイルをアップロードしてS3 keyを返す"""
        ...

    async def download_file(self, key: str) -> bytes:
        """S3 keyからファイルをダウンロード"""
        ...

    async def delete_file(self, key: str) -> None:
        """S3 keyのファイルを削除"""
        ...

    async def file_exists(self, key: str) -> bool:
        """ファイルの存在確認"""
        ...
```

### 7.5 S3 キー命名規則

```
documents/{document_id}/{filename}
```

例: `documents/550e8400-e29b-41d4-a716-446655440000/hr_policies.pdf`

---

## 8. Docker Compose 本番構成

### 8.1 Dockerfile（マルチステージビルド）

```dockerfile
# ============================================
# Stage 1: Builder
# ============================================
FROM python:3.12-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml ./
RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir .

COPY src/ ./src/

# ============================================
# Stage 2: Production
# ============================================
FROM python:3.12-slim AS production

RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY --from=builder /app/src ./src

RUN chown -R appuser:appuser /app
USER appuser

ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 8.2 docker-compose.yml

```yaml
services:
  # ============================================
  # Application
  # ============================================
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    ports:
      - "8000:8000"
    env_file:
      - .env
    environment:
      - DATABASE_URL=postgresql+asyncpg://postgres:postgres@db:5432/ragdb
      - S3_ENDPOINT_URL=http://minio:9000
      - MLFLOW_TRACKING_URI=http://mlflow:5050
    depends_on:
      db:
        condition: service_healthy
      minio:
        condition: service_healthy
      migrate:
        condition: service_completed_successfully
      createbuckets:
        condition: service_completed_successfully
    restart: unless-stopped

  # ============================================
  # Database
  # ============================================
  db:
    image: pgvector/pgvector:pg16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: ragdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d ragdb"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # ============================================
  # Database Migration (init container)
  # ============================================
  migrate:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    command: ["python", "-m", "src.db_migrate"]
    environment:
      - DATABASE_URL=postgresql+asyncpg://postgres:postgres@db:5432/ragdb
    depends_on:
      db:
        condition: service_healthy

  # ============================================
  # Object Storage
  # ============================================
  minio:
    image: minio/minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - miniodata:/data
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # ============================================
  # MinIO Bucket Init (init container)
  # ============================================
  createbuckets:
    image: minio/mc
    depends_on:
      minio:
        condition: service_healthy
    entrypoint: >
      /bin/sh -c "
      mc alias set myminio http://minio:9000 minioadmin minioadmin;
      mc mb myminio/documents --ignore-existing;
      mc anonymous set download myminio/documents;
      exit 0;
      "

  # ============================================
  # MLflow Tracking Server
  # ============================================
  mlflow:
    image: ghcr.io/mlflow/mlflow
    ports:
      - "5050:5050"
    command: >
      mlflow server
      --host 0.0.0.0
      --port 5050
      --backend-store-uri sqlite:///mlflow/mlflow.db
      --default-artifact-root /mlflow/artifacts
    volumes:
      - mlflowdata:/mlflow
    restart: unless-stopped

volumes:
  pgdata:
  miniodata:
  mlflowdata:
```

### 8.3 docker-compose.override.yml（開発用）

```yaml
services:
  app:
    build:
      target: builder
    volumes:
      - ./src:/app/src
    command: ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
    environment:
      - LOG_LEVEL=debug
```

---

## 9. CI/CD 設計（GitHub Actions）

### 9.1 ワークフロー一覧

| ワークフロー | トリガー | 内容 |
|------------|---------|------|
| `ci.yml` | push / PR → main | lint → test → build |
| `security.yml` | push → main / 毎日 | セキュリティスキャン |

### 9.2 ci.yml

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  PYTHON_VERSION: "3.12"

jobs:
  # ============================================
  # Lint
  # ============================================
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install dependencies
        run: pip install ruff

      - name: Run Ruff linter
        run: ruff check src/ tests/

      - name: Run Ruff formatter check
        run: ruff format --check src/ tests/

  # ============================================
  # Test
  # ============================================
  test:
    runs-on: ubuntu-latest
    needs: lint
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env:
          POSTGRES_DB: ragdb_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Cache pip packages
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('pyproject.toml') }}
          restore-keys: ${{ runner.os }}-pip-

      - name: Install dependencies
        run: pip install ".[test]"

      - name: Run tests
        env:
          DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/ragdb_test
          OPENAI_API_KEY: sk-test-dummy-key
        run: pytest tests/ -v --tb=short

  # ============================================
  # Build
  # ============================================
  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Cache Docker layers
        uses: actions/cache@v4
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: ${{ runner.os }}-buildx-

      - name: Build Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: false
          tags: llm-mlops:${{ github.sha }}
          cache-from: type=local,src=/tmp/.buildx-cache
          cache-to: type=local,dest=/tmp/.buildx-cache-new,mode=max

      - name: Move cache
        run: |
          rm -rf /tmp/.buildx-cache
          mv /tmp/.buildx-cache-new /tmp/.buildx-cache
```

### 9.3 security.yml

```yaml
name: Security Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: "0 0 * * *"
  workflow_dispatch:

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install security tools
        run: pip install bandit safety

      - name: Run Bandit (SAST)
        run: bandit -r src/ -f json -o bandit-report.json || true

      - name: Run Safety (dependency audit)
        run: safety check --output json > safety-report.json || true

      - name: Upload security reports
        uses: actions/upload-artifact@v4
        with:
          name: security-reports
          path: |
            bandit-report.json
            safety-report.json
```

---

## 10. セキュリティ設計

### 10.1 方針

| 項目 | 方針 |
|------|------|
| HTTPS | 本番環境ではリバースプロキシ（ALB / Nginx）で TLS 終端 |
| CORS | 許可オリジンを環境変数で指定 |
| CSRF | 無効（API 用途、Stateless） |
| Rate Limit | FastAPI ミドルウェア（slowapi）で制御 |
| パスワード | bcrypt ハッシュ化（passlib） |
| JWT | HS256 署名、環境変数で秘密鍵管理 |
| Secret 管理 | `.env` ファイル（ローカル）/ GitHub Secrets（CI）/ AWS Secrets Manager（本番） |

### 10.2 CORS 設定

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,  # 環境変数から
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 10.3 Rate Limit

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@router.post("/query")
@limiter.limit("30/minute")
async def query_rag(request: Request, ...):
    ...

@router.post("/auth/login")
@limiter.limit("5/minute")
async def login(request: Request, ...):
    ...
```

---

## 11. 環境変数（Phase 3 追加分）

| 変数名 | デフォルト値 | 説明 |
|--------|-------------|------|
| `JWT_SECRET` | （必須） | JWT 署名キー |
| `ACCESS_TOKEN_TTL` | `3600` | Access Token 有効期限（秒） |
| `REFRESH_TOKEN_TTL` | `604800` | Refresh Token 有効期限（秒） |
| `S3_ENDPOINT_URL` | `None`（AWS S3 直接） | MinIO の場合は `http://minio:9000` |
| `AWS_REGION` | `ap-northeast-1` | AWS リージョン |
| `CORS_ORIGINS` | `["http://localhost:3000"]` | 許可オリジン（JSON 配列） |
| `RATE_LIMIT_QUERY` | `30/minute` | /query の Rate Limit |
| `RATE_LIMIT_LOGIN` | `5/minute` | /auth/login の Rate Limit |
| `LOG_LEVEL` | `info` | ログレベル |

---

## 12. ディレクトリ構成（Phase 3 差分）

Phase 1〜2 からの追加・変更ファイルを示す。

```
llm-mlops/
├── Dockerfile                          # マルチステージビルド（Phase 3 新規）
├── docker-compose.yml                  # 本番構成（Phase 3 新規）
├── docker-compose.override.yml         # 開発用オーバーライド（Phase 3 新規）
├── .dockerignore                       # Phase 3 新規
│
├── .github/
│   └── workflows/
│       ├── ci.yml                      # CI パイプライン（Phase 3 新規）
│       └── security.yml                # セキュリティスキャン（Phase 3 新規）
│
├── src/
│   ├── db_migrate.py                   # マイグレーション実行スクリプト（Phase 3 新規）
│   │
│   ├── api/
│   │   ├── ...（Phase 1〜2 と同一）
│   │   ├── auth.py                     # /auth エンドポイント（Phase 3 新規）
│   │   └── users.py                    # /users エンドポイント（Phase 3 新規）
│   │
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── jwt_service.py              # JWT 生成・検証（Phase 3 新規）
│   │   ├── password.py                 # bcrypt ハッシュ化（Phase 3 新規）
│   │   ├── rbac.py                     # ロール・権限定義（Phase 3 新規）
│   │   └── dependencies.py             # FastAPI Depends（Phase 3 新規）
│   │
│   └── storage/
│       ├── s3.py                       # boto3 統一クライアント（Phase 3 リファクタ）
│       └── database.py
│
├── db/
│   └── migrations/
│       ├── 001_init.sql
│       ├── 002_add_metadata.sql
│       ├── 003_token_costs.sql
│       ├── 004_prompt_templates.sql
│       └── 005_users.sql               # Phase 3 新規
│
└── tests/
    ├── ...（Phase 1〜2 と同一）
    ├── test_auth.py                     # Phase 3 新規
    ├── test_rbac.py                     # Phase 3 新規
    ├── test_s3_abstraction.py           # Phase 3 新規
    └── test_api_auth.py                 # Phase 3 新規
```

---

## 13. 実装チェックリスト

### RBAC

- [ ] users テーブル作成（マイグレーション）
- [ ] パスワード bcrypt ハッシュ化（passlib）
- [ ] JWT 生成・検証（PyJWT）
- [ ] ロール・権限マッピング定義
- [ ] 認証ミドルウェア（`get_current_user`）
- [ ] 権限チェックミドルウェア（`require_permission`）
- [ ] `POST /auth/login` エンドポイント
- [ ] `POST /auth/refresh` エンドポイント
- [ ] `GET /auth/me` エンドポイント
- [ ] `GET /users` エンドポイント（admin のみ）
- [ ] `POST /users` エンドポイント（admin のみ）
- [ ] ダミーユーザー初期データ投入
- [ ] 既存エンドポイントへの権限チェック適用

### S3 連携

- [ ] boto3 ベースのストレージ抽象層実装
- [ ] MinIO 接続（`S3_ENDPOINT_URL` 指定時）
- [ ] AWS S3 接続（`S3_ENDPOINT_URL` 未指定時）
- [ ] 既存 MinIO 直接参照の抽象層への置き換え

### Docker Compose

- [ ] マルチステージ Dockerfile（builder + production）
- [ ] 非 root ユーザー実行
- [ ] ヘルスチェック設定
- [ ] docker-compose.yml（全サービス定義）
- [ ] docker-compose.override.yml（開発用ホットリロード）
- [ ] init コンテナ（マイグレーション + バケット作成）
- [ ] `.dockerignore` 作成
- [ ] `docker compose up` でワンコマンド起動確認

### CI/CD

- [ ] `.github/workflows/ci.yml`（lint → test → build）
- [ ] CI 用 pgvector サービスコンテナ
- [ ] pip キャッシュ設定
- [ ] Docker レイヤーキャッシュ設定
- [ ] `.github/workflows/security.yml`（bandit + safety）

### セキュリティ

- [ ] CORS ミドルウェア設定
- [ ] Rate Limit 設定（slowapi）
- [ ] `.env.example` の更新（Phase 3 変数追加）

### テスト

- [ ] JWT 生成・検証ユニットテスト
- [ ] RBAC 権限チェックユニットテスト
- [ ] 認証 API 統合テスト（/auth/login, /auth/refresh, /auth/me）
- [ ] S3 抽象層ユニットテスト（MinIO / モック）

---

## 14. 変更履歴

| バージョン | 日付 | 変更内容 |
|-----------|------|----------|
| 1.0 | 2025-02-12 | 初版作成 |
