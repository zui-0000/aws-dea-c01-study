# <img src="./images/aws-amazon-s3.png" width="36"/>&nbsp;&nbsp; Amazon S3（Simple Storage Service）

## 概要

**AWSのオブジェクトストレージサービス**。
ファイル（オブジェクト）をバケットに保存する。容量制限なし・高耐久性（99.999999999%）。

```
データエンジニアリングにおける S3 の立ち位置:

データソース → S3（データレイク）→ Glue ETL → Redshift（DWH）
                    ↑
          分析の起点・データの集積地
```

---

## 基本概念

```
バケット（Bucket）
→ オブジェクトを入れるコンテナ。名前はグローバルで一意。

オブジェクト（Object）
→ 保存するファイル本体。最大50TB（2025年re:Invent以降）。

キー（Key）
→ オブジェクトの識別子（= ファイルパス）
→ 例: data/2024/01/sales.csv

プレフィックス（Prefix）
→ キーの先頭部分。フォルダのように見えるが実際はフラット構造。
→ 例: data/2024/01/
```

---

## ストレージクラス

用途・アクセス頻度に応じてコストを最適化できる。

| ストレージクラス | 用途 | 取り出しコスト |
|----------------|------|-------------|
| **S3 Express One Zone** | 超高速・低レイテンシ（AI/ML・リアルタイム分析） | なし |
| **S3 Standard** | 頻繁にアクセスするデータ | なし |
| **S3 Standard-IA** | 低頻度アクセス（月1回程度） | あり |
| **S3 One Zone-IA** | 低頻度・単一AZ（再生成可能なデータ向け） | あり |
| **S3 Glacier Instant Retrieval** | アーカイブ・即時取り出し可 | あり |
| **S3 Glacier Flexible Retrieval** | アーカイブ・取り出しに数分〜数時間 | あり |
| **S3 Glacier Deep Archive** | 長期保存・取り出しに12時間以上 | あり（最安） |
| **S3 Intelligent-Tiering** | アクセスパターン不明なデータ（自動で最適クラスに移動） | なし |

```
「とにかく安く長期保存したい」          → Glacier Deep Archive
「アクセス頻度がわからない」             → Intelligent-Tiering
「ログデータを数年保存したい」           → Glacier Flexible Retrieval
「分析用データ（月1回程度アクセス）」    → Standard-IA
「AI/ML・超低レイテンシが必要」          → Express One Zone
```

### S3 Express One Zone vs S3 One Zone-IA

名前に「One Zone」が入っていて紛らわしいが、用途は全く異なる。

| | S3 One Zone-IA | S3 Express One Zone |
|--|--|--|
| 目的 | コスト削減 | **超低レイテンシ** |
| アクセス頻度 | 低頻度 | 高頻度 |
| レイテンシ | Standard と同程度 | **一桁ミリ秒**（Standard の10倍速） |
| バケット種類 | 汎用バケット | **ディレクトリバケット**（専用） |
| ストレージ単価 | Standard-IA より安い | Standard より高い |
| リクエスト単価 | 取り出し課金あり | Standard の 50% 安い |
| 向いているケース | 再生成可能な低頻度データ | AI/ML・リアルタイム分析の中間データ |

```
Express One Zone の特徴:
├── ディレクトリバケット専用（通常の汎用バケットとは別物）
├── 最大200万リクエスト/秒に対応
└── AI/ML 学習データの高速読み込み・EMR/Athena の中間データ置き場に最適
```

```
「安くためておきたい・たまにしか使わない」  → One Zone-IA
「とにかく速くアクセスしたい・頻繁に読む」  → Express One Zone
```

---

### S3 Intelligent-Tiering の詳細

**アクセスパターンを自動で監視して、最適なティアにオブジェクトを自動移動**するストレージクラス。
アクセス頻度が読めないデータに使う。取り出しコストはかからない。

```
オブジェクト作成
        ↓ デフォルトでここから始まる
Frequent Access（頻繁アクセス層）       ← S3 Standard 相当のコスト
        ↓ 30日間アクセスなし（自動）
Infrequent Access（低頻度層）           ← S3 Standard-IA 相当のコスト
        ↓ 90日間アクセスなし（自動）
Archive Instant Access（即時アーカイブ） ← 即時取り出し可
        ↓ 90日以上（オプション設定が必要）
Archive Access（アーカイブ）            ← Glacier 相当（数時間）
        ↓ 180日以上（オプション設定が必要）
Deep Archive Access（深アーカイブ）     ← Glacier Deep Archive 相当
```

上3つは自動で移動。下2つはオプション設定が必要。
**どのティアにあってもアクセスすると自動で Frequent Access 層に戻る**。

#### 監視コスト

Intelligent-Tiering はオブジェクト単位で監視料金がかかる。

```
$0.0025 / 1,000オブジェクト / 月

⚠️ 128KB未満の小さいファイルが大量にある場合、
   監視コストが節約額を上回る可能性がある → Standard-IA の方が安くなる
```

```
「アクセスパターンが読めない・自動で最適化してほしい」  → Intelligent-Tiering
「確実に低頻度しかアクセスしない」                    → Standard-IA（監視コスト不要）
「128KB未満の小さいファイルが大量」                   → Intelligent-Tiering は不向き
```

---

## ライフサイクルポリシー

**オブジェクトを自動で別のストレージクラスに移動・削除**するルール。

```
S3 Standard（作成直後）
    ↓ 30日後
S3 Standard-IA
    ↓ 90日後
S3 Glacier Flexible Retrieval
    ↓ 365日後
削除
```

```json
{
  "Rules": [{
    "Transitions": [
      { "Days": 30,  "StorageClass": "STANDARD_IA" },
      { "Days": 90,  "StorageClass": "GLACIER" }
    ],
    "Expiration": { "Days": 365 }
  }]
}
```

データエンジニアリングでは**ログや生データの長期保存コスト削減**によく使う。

### 移行の最低日数制限

移動先のストレージクラスによって、最低何日経過してから移行できるかが決まっている。

| 移行先 | 最低日数 |
|--------|---------|
| Standard-IA / One Zone-IA | **30日以上** |
| Glacier Instant Retrieval | **30日以上** |
| Glacier Flexible Retrieval | **90日以上** |
| Glacier Deep Archive | **180日以上** |

```
❌ NG（無効な設定）:
Standard → Glacier Instant（4日後）→ Glacier Deep Archive（20日後）
→ Deep Archive への合計日数が 90日未満なので無効

✅ OK:
Standard → Glacier Instant（4日後）→ Glacier Deep Archive（94日後）
                                              ↑ 4 + 90 = 94日以上
```

### 早期削除料金

Glacier に移したオブジェクトを最低保存期間より前に削除すると、残り日数分の料金が請求される。

```
Glacier Flexible Retrieval  → 90日未満で削除すると差額分を課金
Glacier Deep Archive        → 180日未満で削除すると差額分を課金
```

### ライフサイクルポリシー vs Intelligent-Tiering

| | ライフサイクルポリシー | Intelligent-Tiering |
|--|--|--|
| 移動のルール | **日数で固定**（30日後など） | **実際のアクセス状況**で自動判断 |
| 最適化の頻度 | 設定した日数ごと（粗い） | 常時監視（細かい） |
| 監視コスト | なし | あり（$0.0025/1,000オブジェクト/月） |
| 向いているケース | アクセスパターンが予測できる | アクセスパターンが読めない |

```
「頻繁に最適なクラスに移したい・パターンが読めない」  → Intelligent-Tiering
「30日後・90日後など決まったタイミングで移せばいい」  → ライフサイクルポリシー
```

---

## S3 イベント通知

オブジェクトのアップロード・削除などをトリガーに他サービスを呼び出せる。

```
S3（ファイルアップロード）
    ↓ イベント通知
├── Lambda（即座に処理）
├── SQS（キューに積んで非同期処理）
└── SNS（通知）
```

```
ユースケース:
S3にCSVが届く → Lambda が起動 → Glue ETL Job を起動 → Redshiftにロード
```

---

## S3 バージョニング

**同じキー（パス）のオブジェクトを上書き・削除しても、過去のバージョンを全部保持**できる機能。

```
バージョニング無効時:
sales.csv をアップロード → 上書き → 古いデータは消える

バージョニング有効時:
sales.csv（v1）→ sales.csv（v2）→ sales.csv（v3）
                                         ↑ 最新
                  v1・v2も残っている → 誤削除・誤上書きから復元可能
```

### 削除したときの動作

```
通常の削除操作:
sales.csv を削除
    ↓
「削除マーカー」が最新バージョンとして追加される
    ↓
一見消えたように見えるが、v1・v2は残っている

削除マーカーを消す → 元のオブジェクトが復活！
```

### バージョニングの3つの状態

```
① 無効（Unversioned）    ← デフォルト
② 有効（Enabled）        ← バージョンを全部保持
③ 一時停止（Suspended）  ← 新規バージョンは作らないが過去は残す
```

一度有効にすると無効には戻せない（一時停止はできる）。

### コストの注意点

過去バージョンも全部ストレージとして課金される。

```
sales.csv（v1: 100MB）
sales.csv（v2: 100MB）
sales.csv（v3: 100MB）
→ 合計 300MB 分のストレージ料金がかかる
```

### 古いバージョンの自動削除

手動で1個1個消すのは現実的ではないため、**ライフサイクルルールで自動削除**するのが定番。

```json
{
  "Rules": [{
    "ID": "delete-old-versions",
    "Status": "Enabled",
    "NoncurrentVersionExpiration": {
      "NoncurrentDays": 30
    }
  }]
}
```

```hcl
# Terraform での設定例
resource "aws_s3_bucket_lifecycle_configuration" "example" {
  bucket = aws_s3_bucket.example.id

  rule {
    id     = "delete-old-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 30   # 最新でなくなってから30日後に自動削除
    }
  }
}
```

```
sales.csv（v3）← 最新・削除されない
sales.csv（v2）← v3が来た瞬間から30日カウント開始 → 30日後に自動削除
sales.csv（v1）← すでに30日経過していたら即削除
```

### MFA Delete

バージョンの完全削除・バージョニングの無効化に **MFA認証を必須**にする機能。

```
誤操作・悪意ある削除を防ぐ最後の砦
→ CLIからでもMFAコードがないとバージョンを完全削除できない
```

### ユースケース

```
├── 誤って上書きしてしまったファイルの復元
├── 誤って削除したファイルの復元
├── データレイクの生データを変更から保護
└── コンプライアンス要件（過去データの保持義務）
```

---

## S3 レプリケーション

| | CRR（Cross-Region Replication） | SRR（Same-Region Replication） |
|--|--|--|
| レプリケート先 | 別リージョン | 同じリージョン |
| 用途 | DR（災害対策）・レイテンシ低減 | ログ集約・検証環境へのコピー |

```
東京リージョンの S3
    ↓ CRR
大阪リージョンの S3（災害対策）
```

---

## S3 Select / Glacier Select（非推奨）

> ⚠️ **2024年7月25日以降、新規顧客は利用不可**。既存顧客は継続利用できるが、新規利用は不可。
> 代替として **Amazon Athena** または **S3 Object Lambda** を使う。

S3オブジェクトの中身をサーバーサイドでフィルタリングしてから取得する機能だったが、廃止方向。

```
【代替手段】
S3 Select の代わりに:
├── Amazon Athena（S3上のデータをSQLで分析）← 推奨
└── S3 Object Lambda（Lambda でフィルタ処理）
```

---

## S3 マルチパートアップロード

**大きなファイルを分割してアップロード**する機能。

```
5GB以上のファイル → マルチパートアップロードを使う（必須）
100MB以上       → 推奨

ファイルを分割 → 並列アップロード → S3側で結合
→ 高速・途中失敗時も該当パートだけ再送できる
```

---

## パーティション設計

S3上のデータをAthena・Glueで効率よくクエリするためのフォルダ構造の設計。

```
【パーティションなし（非効率）】
s3://my-bucket/sales/all_data.csv
→ 1月分だけ欲しくても全データをスキャン

【パーティションあり（効率的）】
s3://my-bucket/sales/year=2024/month=01/data.parquet
s3://my-bucket/sales/year=2024/month=02/data.parquet
→ WHERE year=2024 AND month=01 → 該当フォルダだけスキャン
```

Hive形式（`key=value`）のプレフィックスにするとAthena・Glueが自動でパーティションを認識する。

---

## セキュリティ

### バケットポリシー

JSONで**バケット単位のアクセス制御**を定義する。

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::123456789:role/GlueRole" },
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-bucket/*"
}
```

### 暗号化

| 方式 | 説明 |
|------|------|
| **SSE-S3** | AWSが管理するキーで暗号化（デフォルト） |
| **SSE-KMS** | AWS KMSのキーで暗号化（監査ログが残る） |
| **SSE-C** | ユーザーが持ち込んだキーで暗号化 |
| **クライアントサイド暗号化** | S3に送る前にクライアント側で暗号化 |

```
「誰がいつ復号したか監査したい」 → SSE-KMS
「とりあえず暗号化したい」       → SSE-S3（デフォルト）
```

### ブロックパブリックアクセス

デフォルトで有効。誤ってバケットを公開してしまうのを防ぐ。

---

## データフォーマット

S3に保存するファイル形式によってAthena・Glueのパフォーマンスが変わる。

| フォーマット | 特徴 | 向いているケース |
|------------|------|----------------|
| **CSV** | 人間が読める・シンプル | 小さなデータ・外部連携 |
| **JSON** | 半構造化データ | ログ・APIレスポンス |
| **Parquet** | 列指向・圧縮効率が高い | 分析クエリ・大規模データ |
| **ORC** | 列指向・Hive最適化 | EMR + Hive |
| **Avro** | スキーマ付き・ストリーミング向け | Kafka・MSKとの連携 |

```
「分析用途・Athenaでクエリしたい」  → Parquet（列指向で必要な列だけ読む）
「ストリーミングデータの保存」       → Avro または JSON → 後でParquetに変換
```

---

## 試験のポイント

- **ストレージクラス** → アクセス頻度・コスト・取り出し時間で選ぶ
- **Express One Zone** → 超低レイテンシ（一桁ミリ秒）・ディレクトリバケット専用・AI/ML向け
- **One Zone-IA vs Express One Zone** → 名前が似ているが別物。IAはコスト削減、Expressは超高速
- **Intelligent-Tiering** → アクセスパターン不明なデータを自動で最適クラスに。30日でIA、90日でArchive Instant Access へ自動移動
- **Intelligent-Tiering の監視コスト** → $0.0025/1,000オブジェクト/月。128KB未満の小さいファイルが大量なら不向き
- **ライフサイクルポリシー** → 古いデータを自動でGlacierに移動・削除
- **S3 Select** → 2024年7月以降新規利用不可。代替は **Athena** または **S3 Object Lambda**
- **マルチパートアップロード** → 5GB以上は必須・並列で高速アップロード
- **パーティション設計** → `year=xxxx/month=xx` 形式でAthena・Glueのスキャン範囲を絞る
- **Parquet** → 分析用途の列指向フォーマット・圧縮効率が高い
- **SSE-KMS** → 監査ログが必要な暗号化
- **バージョニング** → 誤削除・誤上書きからの復元。一度有効にすると無効には戻せない
- **削除マーカー** → バージョニング有効時の削除は実際には消えない。削除マーカーが追加されるだけ
- **古いバージョンの削除** → ライフサイクルルールの `NoncurrentVersionExpiration` で自動削除
- **MFA Delete** → バージョンの完全削除にMFA認証を必須にする
- **CRR** → 別リージョンへのレプリケーション（災害対策）
