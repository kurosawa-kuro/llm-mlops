# Phase 1 - コア（RAGパイプライン）

**Enterprise Knowledge RAG Platform：仕様・基本設計**

---

## 1. 概要・スコープ

### 目的

社内ドキュメント（PDF / Markdown / CSV）をベクトル化し、RAG による高精度な検索・回答生成を実現する。
Phase 1 ではパイプラインのコア機能をローカル環境で完結させ、評価基盤まで構築する。

### 完成定義

- ドキュメントをアップロードし、チャンク分割・Embedding 生成・pgvector 保存が完了する
- 自然言語クエリに対し、関連チャンクを検索し LLM で回答を生成できる
- 各クエリの評価指標が MLflow に記録される
- FastAPI エンドポイント経由で上記操作が可能

### スコープ外（Phase 2 以降）

| 機能 | Phase |
|------|-------|
| Cross-Encoder 再ランキング | 2 |
| メタデータフィルタ検索 | 2 |
| トークンコスト可視化 | 2 |
| プロンプトテンプレート管理 | 2 |
| RBAC（ロールベースアクセス制御） | 3 |
| S3 連携（MinIO → AWS 移行） | 3 |
| CI/CD（GitHub Actions） | 3 |
| Docker Compose 本番構成 | 3 |

---

## 2. システム構成図

### コンポーネント図

```
┌─────────────────────────────────────────────────────┐
│                    Client                           │
│              (curl / Swagger UI)                    │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP
                       ▼
┌─────────────────────────────────────────────────────┐
│                FastAPI Server                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ /ingest  │  │ /query   │  │ /health          │  │
│  └────┬─────┘  └────┬─────┘  └──────────────────┘  │
│       │              │                              │
│  ┌────▼─────────────▼────────────────────────┐     │
│  │          RAG Pipeline (LlamaIndex)         │     │
│  │  ┌─────────┐ ┌──────────┐ ┌────────────┐  │     │
│  │  │ Chunker │ │ Embedder │ │ Generator  │  │     │
│  │  └─────────┘ └──────────┘ └────────────┘  │     │
│  └───────────────────────────────────────────┘     │
└──────────┬───────────┬──────────────┬───────────────┘
           │           │              │
     ┌─────▼───┐ ┌─────▼─────┐ ┌─────▼──────┐
     │ MinIO   │ │ PostgreSQL│ │ OpenAI API │
     │ (S3)    │ │ (pgvector)│ │            │
     └─────────┘ └───────────┘ └────────────┘
                       │
                 ┌─────▼──────┐
                 │  MLflow    │
                 │  Tracking  │
                 └────────────┘
```

### ローカル開発環境（Docker Compose）

| サービス | イメージ | ポート | 用途 |
|---------|---------|--------|------|
| `app` | Python 3.12 | 8000 | FastAPI アプリケーション |
| `db` | pgvector/pgvector:pg16 | 5432 | ベクトル DB |
| `minio` | minio/minio | 9000 / 9001 | S3 互換ストレージ |
| `mlflow` | ghcr.io/mlflow/mlflow | 5050 | 実験トラッキング |

---

## 3. 技術スタック

| レイヤー | 技術 | バージョン方針 |
|---------|------|--------------|
| 言語 | Python | 3.12+ |
| LLM | OpenAI API（gpt-4o-mini） | 最新安定版 |
| Embedding | OpenAI（text-embedding-3-small） | 最新安定版 |
| Vector DB | PostgreSQL + pgvector | PostgreSQL 16 / pgvector 0.7+ |
| RAG Framework | LlamaIndex | 0.11+ |
| API | FastAPI + Uvicorn | FastAPI 0.115+ |
| Storage | MinIO（S3 互換） | 最新安定版 |
| MLOps | MLflow | 2.17+ |
| PDF 解析 | PyMuPDF (fitz) | 最新安定版 |
| CSV 解析 | pandas | 2.2+ |
| DB クライアント | asyncpg + SQLAlchemy | asyncpg 0.30+ / SQLAlchemy 2.0+ |
| テスト | pytest + pytest-asyncio | 最新安定版 |
| Linter / Formatter | Ruff | 最新安定版 |

---

## 4. データフロー

### 4.1 ドキュメント登録フロー（Ingestion）

```
ファイルアップロード (PDF / Markdown / CSV)
    │
    ▼
┌─────────────────────────┐
│ 1. ファイル保存 (MinIO)  │  入力: UploadFile
│                         │  出力: S3 key (str)
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 2. テキスト抽出          │  入力: S3 key
│    (PDF→fitz / MD→raw / │  出力: raw_text (str)
│     CSV→pandas)         │
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 3. チャンク分割          │  入力: raw_text, metadata
│    (固定サイズ+overlap)  │  出力: List[Chunk]
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 4. Embedding 生成       │  入力: List[Chunk.text]
│    (OpenAI API batch)   │  出力: List[Vector(1536)]
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 5. DB 保存              │  入力: Document + List[Chunk+Vector]
│    (pgvector INSERT)    │  出力: document_id (UUID)
└─────────────────────────┘
```

### 4.2 検索・回答生成フロー（Query）

```
ユーザークエリ (自然言語)
    │
    ▼
┌─────────────────────────┐
│ 1. クエリ Embedding     │  入力: query (str)
│    (OpenAI API)         │  出力: query_vector (1536)
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 2. ベクトル類似検索      │  入力: query_vector, top_k
│    (pgvector cosine)    │  出力: List[Chunk+score]
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 3. コンテキスト合成      │  入力: List[Chunk]
│    (重複除去+トークン制御)│  出力: context_text (str)
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 4. LLM 回答生成         │  入力: system_prompt + context + query
│    (OpenAI Chat API)    │  出力: answer (str) + usage
└───────────┬─────────────┘
            ▼
┌─────────────────────────┐
│ 5. 評価ログ記録          │  入力: query, answer, chunks, latency
│    (MLflow)             │  出力: run_id
└─────────────────────────┘
```

---

## 5. Chunking 設計

### 5.1 分割戦略

固定サイズ + オーバーラップ方式を採用する。
既存 rag-system（chunk_size=1000, overlap=0）の知見を踏まえ、オーバーラップを追加して文脈の断絶を軽減する。

### 5.2 パラメータ

| パラメータ | 値 | 根拠 |
|-----------|-----|------|
| `chunk_size` | 1000 文字 | rag-system 実績値。embedding コスト と 検索精度のバランス |
| `chunk_overlap` | 200 文字 | chunk_size の 20%。文脈の断絶を防ぐ推奨値 |
| `separator` | `\n\n` → `\n` → ` ` | 段落 → 改行 → スペースの優先順位で分割 |
| `length_function` | `len` | 文字数ベース（トークン数ではなく文字数で簡潔に） |

### 5.3 チャンクメタデータ

```python
@dataclass
class ChunkMetadata:
    source_file: str       # S3 key（例: "documents/report.pdf"）
    chunk_index: int       # ソースファイル内の通し番号（0始まり）
    file_type: str         # "pdf" | "markdown" | "csv"
    page_number: int | None  # PDF の場合のみ
    total_chunks: int      # ソースファイルの総チャンク数
```

### 5.4 対応ファイル形式

| 形式 | 拡張子 | 抽出方法 | 備考 |
|------|--------|---------|------|
| PDF | `.pdf` | PyMuPDF（fitz） | ページ単位でテキスト抽出後に結合 |
| Markdown | `.md` | raw テキスト読み込み | フロントマター除去 |
| CSV | `.csv` | pandas → 行ごとにテキスト化 | ヘッダー + 行の key-value 形式 |

### 5.5 CSV テキスト化例

```
# 元 CSV
名前,部署,入社年
田中太郎,開発部,2020

# テキスト化結果
名前: 田中太郎
部署: 開発部
入社年: 2020
```

---

## 6. Embedding 設計

### 6.1 モデル選定

| 項目 | 値 |
|------|-----|
| モデル | `text-embedding-3-small` |
| ベクトル次元数 | 1536 |
| 最大入力トークン | 8191 |
| 正規化 | API 側で正規化済み（単位ベクトル） |

**選定理由**: rag-system では `text-embedding-ada-002` を使用していたが、`text-embedding-3-small` は同等コストで精度が向上しており、次元数削減オプション（`dimensions` パラメータ）も利用可能。

### 6.2 バッチ処理設計

```python
EMBEDDING_BATCH_SIZE = 100   # 1 API コールあたりの最大テキスト数
EMBEDDING_MAX_TOKENS = 8191  # 1 テキストあたりの最大トークン数
```

| 項目 | 値 | 根拠 |
|------|-----|------|
| バッチサイズ | 100 | OpenAI API の推奨上限 |
| リトライ | 最大 3 回（指数バックオフ） | API レート制限対策 |
| タイムアウト | 30 秒 | バッチ単位 |

### 6.3 処理フロー

```python
async def embed_chunks(texts: list[str]) -> list[list[float]]:
    """テキストリストをバッチ分割してEmbedding生成"""
    all_embeddings = []
    for batch in chunked(texts, EMBEDDING_BATCH_SIZE):
        response = await openai_client.embeddings.create(
            model="text-embedding-3-small",
            input=batch,
        )
        all_embeddings.extend([item.embedding for item in response.data])
    return all_embeddings
```

---

## 7. データベース設計（pgvector）

### 7.1 テーブル定義

#### documents テーブル

```sql
CREATE TABLE documents (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    filename    VARCHAR(500) NOT NULL,
    s3_key      VARCHAR(1000) NOT NULL UNIQUE,
    file_type   VARCHAR(20) NOT NULL,          -- 'pdf' | 'markdown' | 'csv'
    file_size   BIGINT NOT NULL,               -- バイト数
    chunk_count INTEGER NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_documents_s3_key ON documents (s3_key);
CREATE INDEX idx_documents_created_at ON documents (created_at);
```

#### chunks テーブル

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE chunks (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id   UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    chunk_index   INTEGER NOT NULL,
    content       TEXT NOT NULL,
    embedding     vector(1536) NOT NULL,
    token_count   INTEGER,
    page_number   INTEGER,                     -- PDF の場合のみ
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE (document_id, chunk_index)
);

CREATE INDEX idx_chunks_document_id ON chunks (document_id);
```

### 7.2 インデックス設計

Phase 1 ではデータ量が限定的（〜数千チャンク）のため HNSW インデックスを採用する。

```sql
-- HNSW インデックス（cosine 距離）
CREATE INDEX idx_chunks_embedding ON chunks
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

| パラメータ | 値 | 説明 |
|-----------|-----|------|
| `m` | 16 | グラフの接続数（デフォルト推奨値） |
| `ef_construction` | 64 | インデックス構築時の探索幅（精度 vs 速度） |
| 距離関数 | `vector_cosine_ops` | cosine 距離（正規化済みベクトル前提） |

**参考**: IVFFlat は数万チャンク以上になった場合に検討する（Phase 2 以降）。

### 7.3 類似検索クエリ

```sql
-- cosine 類似度で上位 k 件を取得
SELECT
    c.id,
    c.content,
    c.chunk_index,
    c.page_number,
    d.filename,
    d.s3_key,
    1 - (c.embedding <=> :query_vector) AS similarity
FROM chunks c
JOIN documents d ON c.document_id = d.id
ORDER BY c.embedding <=> :query_vector
LIMIT :top_k;
```

**`<=>`**: pgvector の cosine 距離演算子。値が小さいほど類似度が高い。

---

## 8. コンテキスト合成設計

### 8.1 パラメータ

| パラメータ | 値 | 根拠 |
|-----------|-----|------|
| `top_k` | 3 | rag-system 実績値。精度とコストのバランス |
| `max_context_tokens` | 3000 | gpt-4o-mini の 128k コンテキスト内で十分な余裕を確保 |
| `similarity_threshold` | 0.5 | これ以下の類似度のチャンクは除外 |

### 8.2 重複除去ロジック

同一ドキュメントから隣接チャンクが複数選出された場合、オーバーラップ部分を除去する。

```python
def deduplicate_chunks(chunks: list[RetrievedChunk]) -> list[RetrievedChunk]:
    """隣接チャンクのオーバーラップ部分を除去"""
    seen_content_hashes: set[str] = set()
    result = []
    for chunk in chunks:
        content_hash = hashlib.md5(chunk.content.encode()).hexdigest()
        if content_hash not in seen_content_hashes:
            seen_content_hashes.add(content_hash)
            result.append(chunk)
    return result
```

### 8.3 プロンプト構成

```
[System]
あなたは社内文書に基づいて質問に回答するアシスタントです。
以下の参考情報のみを使って回答してください。
情報が不足している場合はその旨を伝えてください。
回答には参考にした情報の出典（ファイル名）を含めてください。

【参考情報】
[1] (report.pdf - p.3)
チャンクのテキスト内容...

[2] (manual.md)
チャンクのテキスト内容...

[3] (data.csv)
チャンクのテキスト内容...

[User]
ユーザーの質問文
```

---

## 9. RAG 生成設計

### 9.1 LLM モデル・パラメータ

| パラメータ | 値 | 根拠 |
|-----------|-----|------|
| モデル | `gpt-4o-mini` | rag-system 実績。コスト効率が高い |
| `temperature` | 0.1 | 事実ベースの回答を重視し、創造性を抑制 |
| `max_tokens` | 1024 | 回答長の上限 |
| `top_p` | 1.0 | デフォルト |

### 9.2 システムプロンプト

```python
SYSTEM_PROMPT = """あなたは社内文書に基づいて質問に回答するアシスタントです。

## ルール
1. 以下の【参考情報】のみを使って回答してください
2. 参考情報に含まれない内容については「提供された情報からは回答できません」と回答してください
3. 回答の根拠となった出典（ファイル名・ページ番号）を末尾に記載してください
4. 推測や一般知識による補完は行わないでください

## 回答形式
- 簡潔かつ正確に回答する
- 箇条書きを適切に使用する
- 出典を【出典】セクションに記載する

【参考情報】
{context}"""
```

### 9.3 レスポンス形式

```json
{
  "answer": "回答テキスト...",
  "sources": [
    {
      "filename": "report.pdf",
      "chunk_index": 2,
      "page_number": 3,
      "similarity": 0.87
    }
  ],
  "usage": {
    "prompt_tokens": 1500,
    "completion_tokens": 200,
    "total_tokens": 1700
  },
  "latency_ms": 1234
}
```

---

## 10. 評価設計（MLflow）

### 10.1 記録する指標

| 指標 | 型 | 説明 |
|------|-----|------|
| `latency_ms` | float | クエリ → 回答の総レイテンシ（ms） |
| `retrieval_latency_ms` | float | ベクトル検索のレイテンシ（ms） |
| `generation_latency_ms` | float | LLM 生成のレイテンシ（ms） |
| `top_k` | int | 検索件数 |
| `chunk_count` | int | コンテキストに含めたチャンク数 |
| `prompt_tokens` | int | プロンプトトークン数 |
| `completion_tokens` | int | 生成トークン数 |
| `total_tokens` | int | 合計トークン数 |
| `similarity_scores` | list[float] | 各チャンクの類似度スコア |
| `has_answer` | bool | 「回答できません」でないか |

### 10.2 MLflow 実験構成

```python
MLFLOW_EXPERIMENT_NAME = "rag-pipeline-v1"
MLFLOW_TRACKING_URI = "http://localhost:5050"
```

| 項目 | 値 |
|------|-----|
| Experiment 名 | `rag-pipeline-v1` |
| Run 命名規則 | `query-{timestamp}` |
| Artifact | クエリログ JSON |

### 10.3 クエリログスキーマ

各クエリの詳細を MLflow Artifact として JSON 保存する。

```json
{
  "run_id": "mlflow-run-id",
  "timestamp": "2025-01-15T10:30:00.000Z",
  "query": "有給休暇の申請方法は？",
  "answer": "有給休暇の申請は...",
  "retrieved_chunks": [
    {
      "chunk_id": "uuid",
      "document": "hr_policies.pdf",
      "chunk_index": 5,
      "similarity": 0.92,
      "content_preview": "有給休暇の申請については..."
    }
  ],
  "metrics": {
    "latency_ms": 1234,
    "retrieval_latency_ms": 45,
    "generation_latency_ms": 1189,
    "prompt_tokens": 1500,
    "completion_tokens": 200,
    "total_tokens": 1700
  },
  "parameters": {
    "model": "gpt-4o-mini",
    "embedding_model": "text-embedding-3-small",
    "top_k": 3,
    "temperature": 0.1,
    "chunk_size": 1000,
    "chunk_overlap": 200
  }
}
```

---

## 11. API 設計（FastAPI）

### 11.1 エンドポイント一覧

| メソッド | パス | 説明 |
|---------|------|------|
| `POST` | `/ingest` | ドキュメントのアップロード・登録 |
| `POST` | `/query` | RAG 検索・回答生成 |
| `GET` | `/health` | ヘルスチェック |

### 11.2 POST /ingest

ドキュメントをアップロードし、チャンク分割・Embedding 生成・DB 保存を実行する。

**Request**

```
Content-Type: multipart/form-data
```

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| `file` | UploadFile | Yes | PDF / Markdown / CSV ファイル |

**Response 200**

```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "filename": "hr_policies.pdf",
  "file_type": "pdf",
  "chunk_count": 15,
  "message": "Document ingested successfully"
}
```

**Response 400**

```json
{
  "detail": "Unsupported file type: .docx. Supported: .pdf, .md, .csv"
}
```

**Response 500**

```json
{
  "detail": "Failed to process document: error message"
}
```

### 11.3 POST /query

自然言語クエリに対して RAG で回答を生成する。

**Request**

```json
{
  "query": "有給休暇の申請方法は？",
  "top_k": 3
}
```

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-----------|-----|------|-----------|------|
| `query` | string | Yes | - | 検索クエリ（自然言語） |
| `top_k` | integer | No | 3 | 取得するチャンク数（1〜10） |

**Response 200**

```json
{
  "answer": "有給休暇の申請は、社内ポータルの「休暇申請」メニューから行います。申請は希望日の3営業日前までに提出してください。\n\n【出典】\n- hr_policies.pdf (p.12)",
  "sources": [
    {
      "filename": "hr_policies.pdf",
      "chunk_index": 5,
      "page_number": 12,
      "similarity": 0.92
    },
    {
      "filename": "hr_policies.pdf",
      "chunk_index": 6,
      "page_number": 12,
      "similarity": 0.85
    },
    {
      "filename": "company_manual.md",
      "chunk_index": 3,
      "page_number": null,
      "similarity": 0.71
    }
  ],
  "usage": {
    "prompt_tokens": 1500,
    "completion_tokens": 200,
    "total_tokens": 1700
  },
  "latency_ms": 1234
}
```

**Response 400**

```json
{
  "detail": "Query must not be empty"
}
```

### 11.4 GET /health

**Response 200**

```json
{
  "status": "ok",
  "timestamp": "2025-01-15T10:30:00.000Z",
  "components": {
    "database": "ok",
    "minio": "ok",
    "openai": "ok"
  }
}
```

---

## 12. ディレクトリ構成

Phase 1 時点の推奨ディレクトリ構造。

```
llm-mlops/
├── README.md                           # プロジェクト概要
├── README Phase 1 - コア（RAGパイプライン）.md  # 本ドキュメント
├── docker-compose.yml                  # ローカル開発環境
├── .env.example                        # 環境変数テンプレート
├── pyproject.toml                      # Python プロジェクト設定
│
├── src/
│   ├── __init__.py
│   ├── main.py                         # FastAPI アプリケーション
│   ├── config.py                       # 設定・環境変数
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── ingest.py                   # POST /ingest
│   │   ├── query.py                    # POST /query
│   │   └── health.py                   # GET /health
│   │
│   ├── rag/
│   │   ├── __init__.py
│   │   ├── chunker.py                  # チャンク分割ロジック
│   │   ├── embedder.py                 # Embedding 生成
│   │   ├── retriever.py                # ベクトル検索
│   │   ├── context.py                  # コンテキスト合成
│   │   └── generator.py               # LLM 回答生成
│   │
│   ├── storage/
│   │   ├── __init__.py
│   │   ├── s3.py                       # MinIO / S3 操作
│   │   └── database.py                 # pgvector 操作
│   │
│   ├── evaluation/
│   │   ├── __init__.py
│   │   └── mlflow_logger.py            # MLflow ログ記録
│   │
│   └── models/
│       ├── __init__.py
│       └── schemas.py                  # Pydantic スキーマ
│
├── db/
│   └── migrations/
│       └── 001_init.sql                # 初期テーブル作成
│
├── tests/
│   ├── __init__.py
│   ├── test_chunker.py
│   ├── test_embedder.py
│   ├── test_retriever.py
│   ├── test_api_ingest.py
│   └── test_api_query.py
│
└── data/
    └── sample/                         # テスト用サンプルファイル
        ├── sample.pdf
        ├── sample.md
        └── sample.csv
```

---

## 13. 環境変数

| 変数名 | デフォルト値 | 説明 |
|--------|-------------|------|
| `OPENAI_API_KEY` | （必須） | OpenAI API キー |
| `DATABASE_URL` | `postgresql+asyncpg://postgres:postgres@localhost:5432/ragdb` | PostgreSQL 接続 URL |
| `MINIO_ENDPOINT` | `localhost:9000` | MinIO エンドポイント |
| `MINIO_ACCESS_KEY` | `minioadmin` | MinIO アクセスキー |
| `MINIO_SECRET_KEY` | `minioadmin` | MinIO シークレットキー |
| `MINIO_BUCKET` | `documents` | S3 バケット名 |
| `MLFLOW_TRACKING_URI` | `http://localhost:5050` | MLflow トラッキング URI |
| `CHUNK_SIZE` | `1000` | チャンクサイズ（文字数） |
| `CHUNK_OVERLAP` | `200` | チャンクオーバーラップ（文字数） |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | Embedding モデル名 |
| `LLM_MODEL` | `gpt-4o-mini` | LLM モデル名 |
| `TOP_K` | `3` | デフォルト検索件数 |

---

## 14. 実装チェックリスト

### インフラ

- [ ] Docker Compose 構成（PostgreSQL + MinIO + MLflow）
- [ ] pgvector 拡張有効化・初期マイグレーション
- [ ] MinIO バケット自動作成

### Ingestion パイプライン

- [ ] PDF テキスト抽出（PyMuPDF）
- [ ] Markdown テキスト読み込み
- [ ] CSV テキスト化（pandas）
- [ ] 固定サイズ + オーバーラップ チャンク分割
- [ ] OpenAI Embedding バッチ生成
- [ ] MinIO ファイルアップロード
- [ ] pgvector チャンク保存（documents + chunks テーブル）
- [ ] `POST /ingest` エンドポイント

### Query パイプライン

- [ ] クエリ Embedding 生成
- [ ] pgvector cosine 類似検索
- [ ] コンテキスト合成（重複除去・トークン制御）
- [ ] LLM 回答生成（gpt-4o-mini）
- [ ] `POST /query` エンドポイント

### 評価・運用

- [ ] MLflow 実験・Run 記録
- [ ] クエリログ JSON 保存
- [ ] `GET /health` エンドポイント

### テスト

- [ ] チャンク分割ユニットテスト
- [ ] Embedding 生成ユニットテスト（モック）
- [ ] 検索ユニットテスト
- [ ] API 統合テスト（/ingest, /query, /health）

---

## 15. 変更履歴

| バージョン | 日付 | 変更内容 |
|-----------|------|----------|
| 1.0 | 2025-02-12 | 初版作成 |
