# Enterprise Knowledge RAG Platform

> OpenAI × pgvector × FastAPI × Object Storage × DuckDB による社内ナレッジ統合基盤（S3互換設計）

企業内ドキュメントを安全に検索・分析できるLLM基盤。
社内文書（PDF / Markdown / CSV）をベクトル化し、RAGによる高精度な検索・回答生成と、利用状況の分析・監査ログを提供する。

ローカル完結で開発し、デプロイ時にAWS（S3 / RDS）へ差し替え可能な設計。

## 技術スタック

| レイヤー | 技術 | ローカル環境 | 本番環境 |
| --- | --- | --- | --- |
| LLM | OpenAI API | - | - |
| Embedding | OpenAI | - | - |
| Vector DB | PostgreSQL + pgvector | Docker | AWS RDS |
| RAG | LlamaIndex | - | - |
| API | FastAPI | - | - |
| Storage | S3互換 | MinIO（Docker） | AWS S3 |
| DWH | DuckDB | ローカルファイル | - |
| MLOps | MLflow + GitHub Actions | ローカルサーバー | - |
| 認証 | 簡易RBAC | - | IAM連携 |

## アーキテクチャ

```
query
 ↓
embedding (OpenAI)
 ↓
pgvector search
 ↓
context assemble
 ↓
LLM (OpenAI API)
 ↓
評価ログ保存 (MLflow)
```

### 入力

- PDF / Markdown / CSV をObject Storageにアップロード

### 処理

- チャンク分割（サイズ / オーバーラップ / セクション単位 or 意味単位）
- OpenAI Embeddingによるベクトル化（正規化 / バッチ処理）
- PostgreSQL + pgvectorへの登録（メタデータ付与）

### 検索

- ベクトル類似検索（cosine / inner product / top_k / インデックス設計）
- フィルタ検索（部署 / 日付など）
- コンテキスト合成（件数制御 / 並び順 / トークン制御 / 重複除去）
- RAGによる生成回答

### 評価

- 正答率 / Hallucination率 / 再現性の測定
- クエリログ保存
- MLflowによる実験記録

### 分析・監査

- DuckDBによる利用分析
  - 検索頻度の高い文書ランキング
  - 部署別ナレッジ利用傾向
- 監査ログ保存

## 開発ロードマップ

### Phase 1 - コア（RAGパイプライン）

- Chunking戦略の設計・実装
- Embedding生成・保存
- pgvectorによるベクトル検索
- コンテキスト合成 + RAG生成
- 評価ログ保存（MLflow）

### Phase 2 - 差別化

- Cross-Encoderによる再ランキング
- メタデータフィルタ検索
- トークンコスト可視化
- プロンプトテンプレート管理

### Phase 3 - 企業向け強化

- RBAC（ロールベースアクセス制御）
- S3連携（MinIO → AWS移行）
- CI/CD（GitHub Actions）
- Docker Compose構成
