# Phase 2 - 差別化

**Enterprise Knowledge RAG Platform：仕様・基本設計**

---

## 1. 概要・スコープ

### 目的

Phase 1 で構築した RAG パイプラインのコアに、検索精度の向上・運用性の強化・コスト可視化を追加し、プロダクション品質に引き上げる。

### 前提条件

Phase 1 が完了していること（全チェックリスト項目がクリア済み）。

### 完成定義

- Cross-Encoder 再ランキングにより、検索精度が向上している（MRR / nDCG で Phase 1 比改善を MLflow で確認可能）
- メタデータ（部署・日付・ファイル種別）によるフィルタ付き検索ができる
- クエリ単位・期間単位のトークンコストが可視化されている
- 複数のプロンプトテンプレートを管理し、切り替えて使用できる

### スコープ外（Phase 3）

| 機能 | Phase |
|------|-------|
| RBAC（ロールベースアクセス制御） | 3 |
| S3 連携（MinIO → AWS 移行） | 3 |
| CI/CD（GitHub Actions） | 3 |
| Docker Compose 本番構成 | 3 |

---

## 2. システム構成図

### コンポーネント図（Phase 2 差分）

```
┌─────────────────────────────────────────────────────────────┐
│                      FastAPI Server                         │
│                                                             │
│  ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌──────────────┐  │
│  │ /ingest  │ │ /query   │ │ /costs    │ │ /templates   │  │
│  └────┬─────┘ └────┬─────┘ └─────┬─────┘ └──────┬───────┘  │
│       │            │             │               │          │
│  ┌────▼────────────▼─────────────────────────────────┐     │
│  │            RAG Pipeline (LlamaIndex)               │     │
│  │                                                    │     │
│  │  ┌─────────┐ ┌──────────┐ ┌─────────────────────┐ │     │
│  │  │ Chunker │ │ Embedder │ │ Generator           │ │     │
│  │  └─────────┘ └──────────┘ └─────────────────────┘ │     │
│  │                                                    │     │
│  │  ┌──────────────────┐  ┌──────────────────────┐   │     │
│  │  │ Reranker         │  │ Template Manager     │   │     │
│  │  │ (Cross-Encoder)  │  │                      │   │     │
│  │  └──────────────────┘  └──────────────────────┘   │     │
│  └────────────────────────────────────────────────────┘     │
│                                                             │
└──────────┬───────────┬──────────────┬───────────────────────┘
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

**Phase 2 で追加されるコンポーネント:**

| コンポーネント | 説明 |
|--------------|------|
| Reranker | Cross-Encoder による再ランキング |
| Template Manager | プロンプトテンプレートの CRUD・バージョン管理 |
| `/costs` API | トークンコスト集計エンドポイント |
| `/templates` API | テンプレート管理エンドポイント |

---

## 3. 技術スタック（Phase 2 追加分）

| レイヤー | 技術 | バージョン方針 | 用途 |
|---------|------|--------------|------|
| Reranker | sentence-transformers | 最新安定版 | Cross-Encoder モデルのロード・推論 |
| Reranker モデル | cross-encoder/ms-marco-MiniLM-L-6-v2 | HuggingFace 最新 | 再ランキングスコアリング |
| テンプレートエンジン | Jinja2 | 3.1+ | プロンプトテンプレートの変数展開 |
| コスト計算 | tiktoken | 最新安定版 | トークン数の事前計算 |

**Phase 1 から継続:**
Python 3.12+, FastAPI, LlamaIndex, pgvector, MLflow, OpenAI API（変更なし）

---

## 4. データフロー

### 4.1 検索・回答生成フロー（Phase 2 拡張）

Phase 1 のフローにステップ 2.5（再ランキング）と メタデータフィルタを追加する。

```
ユーザークエリ (自然言語 + フィルタ条件)
    │
    ▼
┌──────────────────────────────┐
│ 1. クエリ Embedding          │  入力: query (str)
│    (OpenAI API)              │  出力: query_vector (1536)
└───────────┬──────────────────┘
            ▼
┌──────────────────────────────┐
│ 2. ベクトル類似検索           │  入力: query_vector, top_k_initial,
│    + メタデータフィルタ       │        filters (optional)
│    (pgvector cosine + WHERE) │  出力: List[Chunk+score] (候補)
└───────────┬──────────────────┘
            ▼
┌──────────────────────────────┐
│ 3. Cross-Encoder 再ランキング │  入力: query, List[Chunk] (候補)
│    (ms-marco-MiniLM)         │  出力: List[Chunk+rerank_score] (top_k)
└───────────┬──────────────────┘
            ▼
┌──────────────────────────────┐
│ 4. コンテキスト合成           │  入力: List[Chunk] (再ランキング済)
│    (重複除去+トークン制御)    │  出力: context_text (str)
└───────────┬──────────────────┘
            ▼
┌──────────────────────────────┐
│ 5. テンプレート選択・適用     │  入力: template_id, context, query
│    (Jinja2)                  │  出力: formatted_prompt (str)
└───────────┬──────────────────┘
            ▼
┌──────────────────────────────┐
│ 6. LLM 回答生成              │  入力: formatted_prompt
│    (OpenAI Chat API)         │  出力: answer (str) + usage
└───────────┬──────────────────┘
            ▼
┌──────────────────────────────┐
│ 7. コスト計算・ログ記録       │  入力: usage, model, query_metadata
│    (MLflow + costs テーブル)  │  出力: run_id, cost_record
└──────────────────────────────┘
```

---

## 5. Cross-Encoder 再ランキング設計

### 5.1 概要

Phase 1 の Bi-Encoder（Embedding 類似検索）は高速だが、クエリとチャンクの微妙な意味的関係を捉えきれない場合がある。
Cross-Encoder を 2 段階目に導入し、候補チャンクをクエリとペアで精密にスコアリングする。

### 5.2 モデル選定

| 項目 | 値 |
|------|-----|
| モデル | `cross-encoder/ms-marco-MiniLM-L-6-v2` |
| パラメータ数 | 22M |
| 最大入力トークン | 512 |
| 推論デバイス | CPU（Phase 2 ではローカル CPU で十分） |
| ライブラリ | sentence-transformers |

**選定理由**: MS MARCO で学習済みの軽量モデル。CPU 推論でも数十候補のスコアリングに 100ms 以下で応答可能。日本語テキストにも一定の汎化性能がある。

### 5.3 パラメータ

| パラメータ | 値 | 説明 |
|-----------|-----|------|
| `top_k_initial` | 10 | Bi-Encoder で取得する候補数（再ランキング前） |
| `top_k_final` | 3 | Cross-Encoder 後に返す最終チャンク数 |
| `rerank_threshold` | 0.0 | この閾値以下のスコアのチャンクは除外 |
| `rerank_enabled` | true | 再ランキングの有効/無効切り替え |

### 5.4 処理フロー

```python
from sentence_transformers import CrossEncoder

class Reranker:
    def __init__(self, model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"):
        self.model = CrossEncoder(model_name)

    def rerank(
        self,
        query: str,
        chunks: list[RetrievedChunk],
        top_k: int = 3,
        threshold: float = 0.0,
    ) -> list[RankedChunk]:
        """Cross-Encoderでクエリ-チャンクペアをスコアリング"""
        pairs = [(query, chunk.content) for chunk in chunks]
        scores = self.model.predict(pairs)

        ranked = [
            RankedChunk(chunk=chunk, rerank_score=float(score))
            for chunk, score in zip(chunks, scores)
            if float(score) > threshold
        ]
        ranked.sort(key=lambda x: x.rerank_score, reverse=True)
        return ranked[:top_k]
```

### 5.5 評価指標

Phase 2 で追加する検索精度指標（MLflow に記録）:

| 指標 | 説明 |
|------|------|
| `mrr` | Mean Reciprocal Rank（再ランキング後の正解チャンク順位） |
| `ndcg@k` | Normalized Discounted Cumulative Gain |
| `rerank_latency_ms` | Cross-Encoder の推論レイテンシ |
| `rank_change` | 再ランキング前後の順位変動（平均） |

---

## 6. メタデータフィルタ検索設計

### 6.1 概要

ベクトル類似検索に加えて、ドキュメントのメタデータ（部署・日付・ファイル種別）による事前フィルタを可能にする。
pgvector の WHERE 句と組み合わせてフィルタ付きベクトル検索を実現する。

### 6.2 メタデータスキーマ

documents テーブルにメタデータカラムを追加する。

```sql
-- Phase 2 マイグレーション
ALTER TABLE documents
    ADD COLUMN department  VARCHAR(100),
    ADD COLUMN author      VARCHAR(200),
    ADD COLUMN published_at DATE,
    ADD COLUMN tags        TEXT[];    -- PostgreSQL 配列型

CREATE INDEX idx_documents_department ON documents (department);
CREATE INDEX idx_documents_published_at ON documents (published_at);
CREATE INDEX idx_documents_tags ON documents USING GIN (tags);
```

### 6.3 メタデータ登録

Ingestion 時にメタデータをオプションで指定可能にする。

```json
// POST /ingest リクエスト（Phase 2 拡張）
// Content-Type: multipart/form-data

// フォームフィールド:
// file: (binary)
// metadata: (JSON string)
{
  "department": "開発部",
  "author": "田中太郎",
  "published_at": "2025-01-15",
  "tags": ["技術", "設計書"]
}
```

### 6.4 フィルタ構文

クエリ時のフィルタ指定方式:

```json
{
  "query": "有給休暇の申請方法は？",
  "top_k": 3,
  "filters": {
    "department": "人事部",
    "file_type": "pdf",
    "published_after": "2024-01-01",
    "published_before": "2025-12-31",
    "tags": ["規程"]
  }
}
```

### 6.5 フィルタ付き検索クエリ

```sql
SELECT
    c.id,
    c.content,
    c.chunk_index,
    c.page_number,
    d.filename,
    d.s3_key,
    d.department,
    d.tags,
    1 - (c.embedding <=> :query_vector) AS similarity
FROM chunks c
JOIN documents d ON c.document_id = d.id
WHERE 1=1
    AND (:department IS NULL OR d.department = :department)
    AND (:file_type IS NULL OR d.file_type = :file_type)
    AND (:published_after IS NULL OR d.published_at >= :published_after)
    AND (:published_before IS NULL OR d.published_at <= :published_before)
    AND (:tags IS NULL OR d.tags && :tags)    -- 配列の重複チェック（AND条件）
ORDER BY c.embedding <=> :query_vector
LIMIT :top_k_initial;
```

### 6.6 フィルタ一覧

| フィルタキー | 型 | 演算 | 説明 |
|------------|-----|------|------|
| `department` | string | 完全一致 | 部署名 |
| `file_type` | string | 完全一致 | `pdf` / `markdown` / `csv` |
| `published_after` | date | >= | 公開日の下限 |
| `published_before` | date | <= | 公開日の上限 |
| `tags` | string[] | 配列重複（AND） | いずれかのタグを含む |
| `author` | string | 部分一致 | 著者名 |

---

## 7. トークンコスト可視化設計

### 7.1 概要

OpenAI API の利用コストをクエリ単位・日単位・月単位で追跡し、API 経由で可視化する。
コスト計算は OpenAI の公開料金テーブルに基づく。

### 7.2 料金テーブル

```python
# 料金は USD / 1M tokens（2025年1月時点）
COST_TABLE = {
    "gpt-4o-mini": {
        "input": 0.15,     # $0.15 / 1M input tokens
        "output": 0.60,    # $0.60 / 1M output tokens
    },
    "text-embedding-3-small": {
        "input": 0.02,     # $0.02 / 1M tokens
        "output": 0.0,
    },
}
```

### 7.3 データベース設計

```sql
CREATE TABLE token_costs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    query_id        UUID,                          -- MLflow run_id との紐付け
    operation       VARCHAR(20) NOT NULL,           -- 'embedding' | 'generation'
    model           VARCHAR(100) NOT NULL,
    prompt_tokens   INTEGER NOT NULL DEFAULT 0,
    completion_tokens INTEGER NOT NULL DEFAULT 0,
    total_tokens    INTEGER NOT NULL DEFAULT 0,
    cost_usd        NUMERIC(12, 8) NOT NULL,       -- 小数点以下8桁（微小コスト対応）
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_token_costs_created_at ON token_costs (created_at);
CREATE INDEX idx_token_costs_operation ON token_costs (operation);
CREATE INDEX idx_token_costs_model ON token_costs (model);
```

### 7.4 コスト計算ロジック

```python
def calculate_cost(
    model: str,
    prompt_tokens: int,
    completion_tokens: int,
) -> float:
    """トークン数からUSDコストを計算"""
    rates = COST_TABLE.get(model)
    if not rates:
        return 0.0
    input_cost = (prompt_tokens / 1_000_000) * rates["input"]
    output_cost = (completion_tokens / 1_000_000) * rates["output"]
    return input_cost + output_cost
```

### 7.5 コスト集計 API レスポンス

```json
// GET /costs?period=monthly&year=2025&month=1
{
  "period": "monthly",
  "year": 2025,
  "month": 1,
  "summary": {
    "total_cost_usd": 1.234,
    "total_queries": 156,
    "total_tokens": 234567,
    "avg_cost_per_query_usd": 0.0079
  },
  "by_operation": {
    "embedding": {
      "total_cost_usd": 0.012,
      "total_tokens": 45678
    },
    "generation": {
      "total_cost_usd": 1.222,
      "prompt_tokens": 156789,
      "completion_tokens": 32100
    }
  },
  "by_model": {
    "gpt-4o-mini": {
      "total_cost_usd": 1.222,
      "total_tokens": 188889
    },
    "text-embedding-3-small": {
      "total_cost_usd": 0.012,
      "total_tokens": 45678
    }
  },
  "daily_breakdown": [
    {
      "date": "2025-01-01",
      "cost_usd": 0.045,
      "query_count": 5,
      "total_tokens": 8901
    }
  ]
}
```

### 7.6 コスト集計クエリ

```sql
-- 月別サマリー
SELECT
    DATE_TRUNC('month', created_at) AS month,
    operation,
    model,
    COUNT(*) AS query_count,
    SUM(prompt_tokens) AS total_prompt_tokens,
    SUM(completion_tokens) AS total_completion_tokens,
    SUM(total_tokens) AS total_tokens,
    SUM(cost_usd) AS total_cost_usd
FROM token_costs
WHERE created_at >= :start_date AND created_at < :end_date
GROUP BY month, operation, model
ORDER BY month;
```

---

## 8. プロンプトテンプレート管理設計

### 8.1 概要

システムプロンプトをテンプレートとして DB に保存・管理し、API 経由で CRUD 操作およびバージョン管理を行う。
クエリ時にテンプレート ID を指定することで、異なるプロンプト戦略を切り替えて使用・評価できる。

### 8.2 テンプレート変数

Jinja2 テンプレートで使用可能な変数:

| 変数名 | 型 | 説明 |
|--------|-----|------|
| `{{ context }}` | str | 検索されたチャンクのフォーマット済みテキスト |
| `{{ query }}` | str | ユーザーのクエリ文 |
| `{{ sources }}` | list | ソース情報リスト（ファイル名・ページ番号） |
| `{{ top_k }}` | int | 使用したチャンク数 |
| `{{ language }}` | str | 回答言語（デフォルト: "ja"） |

### 8.3 デフォルトテンプレート

```yaml
name: "default-v1"
description: "Phase 1 から引き継ぐ標準テンプレート"
is_default: true
system_template: |
  あなたは社内文書に基づいて質問に回答するアシスタントです。

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
  {{ context }}

user_template: |
  {{ query }}
```

### 8.4 テンプレート例（追加）

```yaml
# 要約特化テンプレート
name: "summarizer-v1"
description: "ドキュメント要約に特化したテンプレート"
system_template: |
  あなたは社内文書の要約を行うアシスタントです。

  ## ルール
  1. 以下の参考情報を簡潔に要約してください
  2. 重要なポイントを箇条書きで3〜5点にまとめてください
  3. 元の文書にない情報を追加しないでください

  【参考情報】
  {{ context }}

user_template: |
  以下について要約してください: {{ query }}
```

### 8.5 データベース設計

```sql
CREATE TABLE prompt_templates (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(200) NOT NULL UNIQUE,
    description TEXT,
    system_template TEXT NOT NULL,
    user_template   TEXT NOT NULL DEFAULT '{{ query }}',
    is_default  BOOLEAN NOT NULL DEFAULT FALSE,
    version     INTEGER NOT NULL DEFAULT 1,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- デフォルトテンプレートは1つのみ
CREATE UNIQUE INDEX idx_prompt_templates_default
    ON prompt_templates (is_default) WHERE is_default = TRUE;
```

### 8.6 テンプレートレンダリング

```python
from jinja2 import Template

def render_prompt(
    template: PromptTemplate,
    context: str,
    query: str,
    sources: list[SourceInfo],
    top_k: int,
    language: str = "ja",
) -> tuple[str, str]:
    """テンプレートに変数を埋め込んでシステム・ユーザーメッセージを生成"""
    variables = {
        "context": context,
        "query": query,
        "sources": sources,
        "top_k": top_k,
        "language": language,
    }
    system_msg = Template(template.system_template).render(**variables)
    user_msg = Template(template.user_template).render(**variables)
    return system_msg, user_msg
```

---

## 9. データベース設計（Phase 2 差分）

### 9.1 マイグレーション一覧

| ファイル | 内容 |
|---------|------|
| `002_add_metadata.sql` | documents テーブルへのメタデータカラム追加 |
| `003_token_costs.sql` | token_costs テーブル作成 |
| `004_prompt_templates.sql` | prompt_templates テーブル作成 |

### 9.2 002_add_metadata.sql

```sql
ALTER TABLE documents
    ADD COLUMN department   VARCHAR(100),
    ADD COLUMN author       VARCHAR(200),
    ADD COLUMN published_at DATE,
    ADD COLUMN tags         TEXT[];

CREATE INDEX idx_documents_department ON documents (department);
CREATE INDEX idx_documents_published_at ON documents (published_at);
CREATE INDEX idx_documents_tags ON documents USING GIN (tags);
```

### 9.3 003_token_costs.sql

```sql
CREATE TABLE token_costs (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    query_id          UUID,
    operation         VARCHAR(20) NOT NULL,
    model             VARCHAR(100) NOT NULL,
    prompt_tokens     INTEGER NOT NULL DEFAULT 0,
    completion_tokens INTEGER NOT NULL DEFAULT 0,
    total_tokens      INTEGER NOT NULL DEFAULT 0,
    cost_usd          NUMERIC(12, 8) NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_token_costs_created_at ON token_costs (created_at);
CREATE INDEX idx_token_costs_operation ON token_costs (operation);
CREATE INDEX idx_token_costs_model ON token_costs (model);
```

### 9.4 004_prompt_templates.sql

```sql
CREATE TABLE prompt_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL UNIQUE,
    description     TEXT,
    system_template TEXT NOT NULL,
    user_template   TEXT NOT NULL DEFAULT '{{ query }}',
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_prompt_templates_default
    ON prompt_templates (is_default) WHERE is_default = TRUE;

-- デフォルトテンプレート挿入
INSERT INTO prompt_templates (name, description, system_template, user_template, is_default)
VALUES (
    'default-v1',
    'Phase 1 から引き継ぐ標準テンプレート',
    E'あなたは社内文書に基づいて質問に回答するアシスタントです。\n\n## ルール\n1. 以下の【参考情報】のみを使って回答してください\n2. 参考情報に含まれない内容については「提供された情報からは回答できません」と回答してください\n3. 回答の根拠となった出典（ファイル名・ページ番号）を末尾に記載してください\n4. 推測や一般知識による補完は行わないでください\n\n## 回答形式\n- 簡潔かつ正確に回答する\n- 箇条書きを適切に使用する\n- 出典を【出典】セクションに記載する\n\n【参考情報】\n{{ context }}',
    '{{ query }}',
    TRUE
);
```

---

## 10. API 設計（Phase 2 差分）

### 10.1 エンドポイント一覧（Phase 2 全体）

| メソッド | パス | Phase | 説明 |
|---------|------|-------|------|
| `POST` | `/ingest` | 1（拡張） | ドキュメント登録（メタデータ対応追加） |
| `POST` | `/query` | 1（拡張） | RAG 検索（再ランキング・フィルタ・テンプレート対応追加） |
| `GET` | `/health` | 1 | ヘルスチェック（変更なし） |
| `GET` | `/costs` | **2 新規** | トークンコスト集計 |
| `GET` | `/costs/queries/{query_id}` | **2 新規** | クエリ単位のコスト詳細 |
| `GET` | `/templates` | **2 新規** | テンプレート一覧 |
| `POST` | `/templates` | **2 新規** | テンプレート作成 |
| `GET` | `/templates/{id}` | **2 新規** | テンプレート詳細 |
| `PUT` | `/templates/{id}` | **2 新規** | テンプレート更新 |
| `DELETE` | `/templates/{id}` | **2 新規** | テンプレート削除 |

### 10.2 POST /ingest（Phase 2 拡張）

**Request**

```
Content-Type: multipart/form-data
```

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| `file` | UploadFile | Yes | PDF / Markdown / CSV ファイル |
| `metadata` | string (JSON) | No | メタデータ JSON（Phase 2 追加） |

**metadata JSON 構造**

```json
{
  "department": "人事部",
  "author": "田中太郎",
  "published_at": "2025-01-15",
  "tags": ["規程", "就業規則"]
}
```

**Response 200**

```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "filename": "hr_policies.pdf",
  "file_type": "pdf",
  "chunk_count": 15,
  "metadata": {
    "department": "人事部",
    "author": "田中太郎",
    "published_at": "2025-01-15",
    "tags": ["規程", "就業規則"]
  },
  "message": "Document ingested successfully"
}
```

### 10.3 POST /query（Phase 2 拡張）

**Request**

```json
{
  "query": "有給休暇の申請方法は？",
  "top_k": 3,
  "filters": {
    "department": "人事部",
    "file_type": "pdf",
    "published_after": "2024-01-01",
    "tags": ["規程"]
  },
  "rerank": true,
  "template_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-----------|-----|------|-----------|------|
| `query` | string | Yes | - | 検索クエリ |
| `top_k` | integer | No | 3 | 最終チャンク数（1〜10） |
| `filters` | object | No | null | メタデータフィルタ（Phase 2 追加） |
| `rerank` | boolean | No | true | 再ランキング有効/無効（Phase 2 追加） |
| `template_id` | string (UUID) | No | null（デフォルトテンプレート使用） | テンプレート ID（Phase 2 追加） |

**Response 200**

```json
{
  "answer": "有給休暇の申請は...",
  "sources": [
    {
      "filename": "hr_policies.pdf",
      "chunk_index": 5,
      "page_number": 12,
      "similarity": 0.92,
      "rerank_score": 8.45,
      "department": "人事部",
      "tags": ["規程", "就業規則"]
    }
  ],
  "usage": {
    "prompt_tokens": 1500,
    "completion_tokens": 200,
    "total_tokens": 1700
  },
  "cost_usd": 0.000345,
  "latency_ms": 1456,
  "rerank_applied": true,
  "template_used": "default-v1",
  "filters_applied": {
    "department": "人事部",
    "file_type": "pdf"
  }
}
```

### 10.4 GET /costs

トークンコストの集計結果を返す。

**Query Parameters**

| パラメータ | 型 | 必須 | デフォルト | 説明 |
|-----------|-----|------|-----------|------|
| `period` | string | No | `monthly` | `daily` / `monthly` |
| `year` | integer | No | 現在年 | 対象年 |
| `month` | integer | No | 現在月 | 対象月（period=daily 時に必須） |

**Response 200**

```json
{
  "period": "monthly",
  "year": 2025,
  "month": 1,
  "summary": {
    "total_cost_usd": 1.234,
    "total_queries": 156,
    "total_tokens": 234567,
    "avg_cost_per_query_usd": 0.0079
  },
  "by_operation": {
    "embedding": {
      "total_cost_usd": 0.012,
      "total_tokens": 45678
    },
    "generation": {
      "total_cost_usd": 1.222,
      "prompt_tokens": 156789,
      "completion_tokens": 32100
    }
  },
  "by_model": {
    "gpt-4o-mini": {
      "total_cost_usd": 1.222,
      "total_tokens": 188889
    },
    "text-embedding-3-small": {
      "total_cost_usd": 0.012,
      "total_tokens": 45678
    }
  }
}
```

### 10.5 GET /costs/queries/{query_id}

**Response 200**

```json
{
  "query_id": "550e8400-e29b-41d4-a716-446655440000",
  "records": [
    {
      "operation": "embedding",
      "model": "text-embedding-3-small",
      "total_tokens": 45,
      "cost_usd": 0.0000009
    },
    {
      "operation": "generation",
      "model": "gpt-4o-mini",
      "prompt_tokens": 1500,
      "completion_tokens": 200,
      "total_tokens": 1700,
      "cost_usd": 0.000345
    }
  ],
  "total_cost_usd": 0.0003459
}
```

### 10.6 テンプレート API

#### GET /templates

**Response 200**

```json
{
  "templates": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "default-v1",
      "description": "Phase 1 から引き継ぐ標準テンプレート",
      "is_default": true,
      "version": 1,
      "created_at": "2025-01-15T10:30:00.000Z"
    },
    {
      "id": "660e8400-e29b-41d4-a716-446655440001",
      "name": "summarizer-v1",
      "description": "ドキュメント要約に特化したテンプレート",
      "is_default": false,
      "version": 1,
      "created_at": "2025-02-01T10:30:00.000Z"
    }
  ]
}
```

#### POST /templates

**Request**

```json
{
  "name": "concise-v1",
  "description": "簡潔な回答を生成するテンプレート",
  "system_template": "あなたは簡潔に回答するアシスタントです。\n\n{{ context }}",
  "user_template": "{{ query }}",
  "is_default": false
}
```

**Response 201**

```json
{
  "id": "770e8400-e29b-41d4-a716-446655440002",
  "name": "concise-v1",
  "version": 1,
  "message": "Template created successfully"
}
```

#### PUT /templates/{id}

**Request**

```json
{
  "description": "簡潔な回答を生成するテンプレート（改良版）",
  "system_template": "改良後のテンプレート内容..."
}
```

**Response 200**

```json
{
  "id": "770e8400-e29b-41d4-a716-446655440002",
  "name": "concise-v1",
  "version": 2,
  "message": "Template updated successfully"
}
```

**備考**: 更新時に `version` が自動インクリメントされる。

#### DELETE /templates/{id}

**Response 200**

```json
{
  "message": "Template deleted successfully"
}
```

**Response 400**

```json
{
  "detail": "Cannot delete default template"
}
```

---

## 11. 評価設計（Phase 2 差分）

### 11.1 Phase 2 で追加する指標

| 指標 | 型 | 説明 |
|------|-----|------|
| `rerank_applied` | bool | 再ランキングが適用されたか |
| `rerank_latency_ms` | float | Cross-Encoder 推論レイテンシ |
| `rerank_scores` | list[float] | 再ランキングスコア一覧 |
| `rank_changes` | list[int] | 各チャンクの順位変動 |
| `mrr` | float | Mean Reciprocal Rank |
| `ndcg_at_k` | float | nDCG@k |
| `cost_usd` | float | クエリ単位の総コスト |
| `cost_embedding_usd` | float | Embedding コスト |
| `cost_generation_usd` | float | 生成コスト |
| `filters_applied` | dict | 適用されたフィルタ |
| `template_name` | str | 使用したテンプレート名 |

### 11.2 MLflow 実験構成（Phase 2）

```python
MLFLOW_EXPERIMENT_NAME = "rag-pipeline-v2"
```

| 項目 | 値 |
|------|-----|
| Experiment 名 | `rag-pipeline-v2` |
| Run 命名規則 | `query-{timestamp}` |
| Run タグ | `rerank={true/false}`, `template={name}`, `filters={json}` |

### 11.3 A/B 比較

テンプレートや再ランキングの有無を MLflow で比較評価する。

```python
# MLflow で再ランキング ON/OFF の効果を比較
with mlflow.start_run(run_name=f"query-{timestamp}"):
    mlflow.log_param("rerank_enabled", True)
    mlflow.log_param("template_name", "default-v1")
    mlflow.log_param("top_k_initial", 10)
    mlflow.log_param("top_k_final", 3)
    mlflow.log_metric("mrr", mrr_score)
    mlflow.log_metric("rerank_latency_ms", rerank_latency)
    mlflow.log_metric("cost_usd", total_cost)
```

---

## 12. ディレクトリ構成（Phase 2 差分）

Phase 1 からの追加・変更ファイルを示す。

```
llm-mlops/
├── ...（Phase 1 と同一）
│
├── src/
│   ├── api/
│   │   ├── ...（Phase 1 と同一）
│   │   ├── costs.py                    # GET /costs（Phase 2 新規）
│   │   └── templates.py               # /templates CRUD（Phase 2 新規）
│   │
│   ├── rag/
│   │   ├── ...（Phase 1 と同一）
│   │   ├── reranker.py                 # Cross-Encoder 再ランキング（Phase 2 新規）
│   │   └── template_manager.py         # テンプレート管理（Phase 2 新規）
│   │
│   ├── costs/
│   │   ├── __init__.py
│   │   ├── calculator.py               # コスト計算ロジック（Phase 2 新規）
│   │   └── aggregator.py              # コスト集計（Phase 2 新規）
│   │
│   └── models/
│       └── schemas.py                  # Pydantic スキーマ（Phase 2 拡張）
│
├── db/
│   └── migrations/
│       ├── 001_init.sql
│       ├── 002_add_metadata.sql        # Phase 2 新規
│       ├── 003_token_costs.sql         # Phase 2 新規
│       └── 004_prompt_templates.sql    # Phase 2 新規
│
└── tests/
    ├── ...（Phase 1 と同一）
    ├── test_reranker.py                # Phase 2 新規
    ├── test_filters.py                 # Phase 2 新規
    ├── test_costs.py                   # Phase 2 新規
    ├── test_templates.py               # Phase 2 新規
    └── test_api_costs.py               # Phase 2 新規
```

---

## 13. 環境変数（Phase 2 追加分）

| 変数名 | デフォルト値 | 説明 |
|--------|-------------|------|
| `RERANK_ENABLED` | `true` | Cross-Encoder 再ランキングの有効/無効 |
| `RERANK_MODEL` | `cross-encoder/ms-marco-MiniLM-L-6-v2` | Cross-Encoder モデル名 |
| `TOP_K_INITIAL` | `10` | 再ランキング前の候補取得数 |
| `RERANK_THRESHOLD` | `0.0` | 再ランキングスコアの下限閾値 |

---

## 14. 実装チェックリスト

### Cross-Encoder 再ランキング

- [ ] sentence-transformers インストール・モデルダウンロード
- [ ] Reranker クラス実装（スコアリング・ソート・閾値フィルタ）
- [ ] Query パイプラインへの Reranker 統合
- [ ] `rerank` パラメータによる ON/OFF 切り替え
- [ ] MRR / nDCG 指標の MLflow 記録

### メタデータフィルタ検索

- [ ] documents テーブルへのメタデータカラム追加（マイグレーション）
- [ ] POST /ingest のメタデータ受付対応
- [ ] フィルタ付きベクトル検索クエリ実装
- [ ] POST /query の `filters` パラメータ対応
- [ ] GIN インデックス（tags 配列検索）

### トークンコスト可視化

- [ ] token_costs テーブル作成（マイグレーション）
- [ ] コスト計算ロジック実装（料金テーブル）
- [ ] 全 OpenAI API コールへのコスト記録組み込み
- [ ] `GET /costs` 集計エンドポイント
- [ ] `GET /costs/queries/{query_id}` 詳細エンドポイント

### プロンプトテンプレート管理

- [ ] prompt_templates テーブル作成（マイグレーション）
- [ ] デフォルトテンプレート初期データ投入
- [ ] テンプレート CRUD API（GET / POST / PUT / DELETE）
- [ ] Jinja2 テンプレートレンダリング実装
- [ ] POST /query の `template_id` パラメータ対応
- [ ] テンプレートバージョン自動インクリメント

### テスト

- [ ] Reranker ユニットテスト
- [ ] メタデータフィルタ検索ユニットテスト
- [ ] コスト計算ユニットテスト
- [ ] テンプレート CRUD 統合テスト
- [ ] API 統合テスト（/costs, /templates）

---

## 15. 変更履歴

| バージョン | 日付 | 変更内容 |
|-----------|------|----------|
| 1.0 | 2025-02-12 | 初版作成 |
