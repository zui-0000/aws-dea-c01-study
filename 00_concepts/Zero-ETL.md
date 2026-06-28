# Zero-ETL

## 概要

ETLパイプラインを**構築・管理せずに**データをRedshiftに連携できる仕組み。
ソースDBの変更が**ほぼリアルタイム**でRedshiftに自動反映される。

従来は Aurora → Glue（ETLパイプライン） → Redshift という手順が必要だったが、
Zero-ETLではその中間のパイプラインが不要になる。

```
【従来のETL】
Aurora → AWS Glue（ETLパイプライン構築・管理） → Redshift
             ↑ 設計・運用コストがかかる

【Zero-ETL】
Aurora ───────────────────────────────→ Redshift
         設定するだけで自動的にリアルタイム連携
```

---

## 対応しているソース

| ソース | ターゲット | 備考 |
|--------|-----------|------|
| Amazon Aurora MySQL | Amazon Redshift | GA済み |
| Amazon Aurora PostgreSQL | Amazon Redshift | GA済み |
| Amazon RDS for MySQL | Amazon Redshift | GA済み |
| Amazon RDS for PostgreSQL | Amazon Redshift | 2025年7月23日にGA |
| Amazon DynamoDB | Amazon Redshift | GA済み |

---

## ターゲットの拡大

Zero-ETLはRedshift以外にもターゲットが拡大している。

| ターゲット | ソース例 | 備考 |
|-----------|---------|------|
| **Amazon OpenSearch Service** | Amazon DynamoDB（GA）、Amazon S3、CloudWatch Logs など | DynamoDB → OpenSearch のZero-ETL統合がGA |
| **Amazon SageMaker Lakehouse** | AWS Glue経由の各種SaaS / DBソース | Glueを介してデータを統合 |
| **Amazon Redshift** | 自己管理型DB（MySQL / PostgreSQL / SQL Server / Oracle） | マネージドDBに加え、自己管理型DBもソースに対応 |

---

## メリット

| メリット | 説明 |
|---------|------|
| **パイプライン管理が不要** | Glue / DMS のジョブを書かなくていい |
| **リアルタイムに近い反映** | バッチETLのような遅延がない |
| **運用コスト削減** | パイプラインの監視・障害対応が不要 |
| **スキーマ変更への追従** | ソースのスキーマ変更を自動で検知・反映 |

---

## 注意点・制限

- 完全な**リアルタイムではない**（数秒〜数分の遅延はある）
- 全てのデータ型・操作に対応しているわけではない
- 複雑な変換処理（集計・結合など）は依然としてGlueが必要
- Zero-ETLはあくまで「レプリケーション」であり、変換はできない

---

## 実務との対応

実際にやっていた Aurora → Glue → CSV → S3 → BigQuery のパイプラインは、
連携先がRedshiftであればZero-ETLで大幅に簡略化できる。

```
【実務でやっていた構成】
Aurora MySQL
    ↓ AWS Glue（ETL処理・CSV変換）
    S3
    ↓ BigQuery（別チームが取り込み）

【Zero-ETLで置き換えた場合】
Aurora MySQL
    ↓ Zero-ETL（自動レプリケーション）
    Amazon Redshift（そのまま分析可能）
```

ただし変換処理（CSV化など）が必要な場合や、BigQueryなど外部への連携が必要な場合は
従来のGlueパイプラインが引き続き有効。

---

## ETL / ELT / Zero-ETL の位置づけ

```
ETL      : 変換してからロード（Glue / EMR）
ELT      : ロードしてから変換（S3 → Redshift / Athena）
Zero-ETL : 変換せずにそのままレプリケーション（Aurora → Redshift 自動連携）
```

Zero-ETLは「変換が不要なレプリケーション」に特化した考え方。

---

## 試験のポイント

- **Aurora → Redshift をパイプラインなしで連携** → Zero-ETL
- **Zero-ETLでできること** → リアルタイムに近いレプリケーション
- **Zero-ETLでできないこと** → 複雑な変換処理（それはGlueの仕事）
- **運用コスト削減・パイプライン不要** がキーワード
