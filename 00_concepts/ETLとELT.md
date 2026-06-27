# ETL と ELT

## ETL（Extract → Transform → Load）

データを**変換してからロード**する従来の方式。

```
データソース
    ↓ Extract（抽出）
中間ステージング領域
    ↓ Transform（変換・整形）
    ↓ Load（ロード）
データウェアハウス（Redshift など）
```

### 特徴

- 変換処理に専用の計算リソースが必要
- ロード前にデータ品質を担保できる
- データウェアハウスへの負荷が低い
- 大量データの変換は時間がかかる

### AWSサービス

- **AWS Glue** → サーバーレスETLの代表
- **Amazon EMR** → 大規模Spark ETL
- **AWS DMS** → DB間のデータ移行ETL

---

## ELT（Extract → Load → Transform）

データを**生のままロードしてから変換**する現代的な方式。

```
データソース
    ↓ Extract（抽出）
    ↓ Load（ロード）
データレイク / DWH（S3 / Redshift など）
    ↓ Transform（変換・整形）
分析用テーブル・ビュー
```

### 特徴

- データレイクやDWHの計算能力で変換するためスケーラブル
- 生データを保持しておけるので再変換が可能
- クラウドDWHの普及で主流になった方式
- 変換前の生データも残るため柔軟性が高い

### AWSサービス

- **Amazon Redshift** → ロード後にSQL変換
- **Amazon Athena** → S3上のデータをクエリ・変換
- **AWS Glue** → ELT的な使い方も可能

---

## ETL vs ELT 比較

| 観点 | ETL | ELT |
|------|-----|-----|
| 変換タイミング | ロード前 | ロード後 |
| 生データの保持 | しない | する |
| 向いているストレージ | データウェアハウス | データレイク / クラウドDWH |
| スケーラビリティ | 低め | 高い |
| 再処理 | 難しい | 容易 |
| 主流の環境 | オンプレ / 従来型DWH | クラウド / AWS（現在の標準） |

---

## データパイプラインの全体像

```
[ソース] → Extract → (Transform) → Load → (Transform) → [分析]
                ETL の場合 ↑              ELT の場合 ↑
```

実際は **ETL と ELT を組み合わせる** ケースも多い。
例: Kinesis で収集 → S3にロード（EL）→ Glue で変換（T）→ Redshift へ

---

## 具体例

### ETLの例：Aurora MySQL → S3 → BigQuery

```
Aurora MySQL（ソース）
    ↓ Extract（AWS Glueでテーブルデータを抽出）
    ↓ Transform（CSV形式に変換）
    ↓ Load（S3バケットに配置）
    Amazon S3
    ↓ Load（BigQueryへ取り込み）
    BigQuery（分析基盤）
```

**なぜETLか？**
GlueがCSVに変換（Transform）してからS3にロード（Load）しているため。

**ポイント：** AWSだけで完結するならELTが主流だが、BigQueryなど**他システムへの連携では
相手が読みやすい形式に変換してから渡すETLが自然な選択**になることも多い。

---

## なぜAWSではELTが標準なのか

| 理由 | 説明 |
|------|------|
| S3が安い | 生データをそのまま大量保存してもコストが低い |
| 計算リソースが潤沢 | Redshift / Athena / EMR でロード後の変換が高速 |
| 再変換が容易 | 生データが残るので、要件変更時に再処理できる |
| スキーマレス | S3はスキーマ不要（Schema-on-Read）なのでそのままロード可能 |

## 試験のポイント

- **ETLの代表サービス** → AWS Glue、Amazon EMR
- **ELTに向いている** → Amazon Redshift（COPY + SQL変換）、Amazon Athena
- **Glue は ETL / ELT 両方で使える**
- **データレイク構成では ELT が主流**（S3に生データを貯めてから変換）
- **AWS Glue DataBrew** → ノーコードでデータ変換・プロファイリング（ETL補助）
