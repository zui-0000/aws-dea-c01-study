# <img src="./images/aws-aws-glue.webp" width="36"/>&nbsp;&nbsp; AWS Glue

## 概要

**フルサーバーレスのETL（データ統合）サービス**。
S3・RDS・DynamoDBなどのデータを抽出・変換・ロードするパイプラインを構築できる。

```
データソース（S3・RDS・DynamoDB・Redshiftなど）
        ↓ AWS Glue（ETL処理）
データ保存先（S3・Redshift・RDSなど）
        ↓
Amazon Athena / Amazon Redshift（分析）
```

サーバーの管理不要。Apache Spark ベースで大規模データも処理できる。

---

## 主要コンポーネント

AWS Glue は複数のコンポーネントで構成される。

```
AWS Glue
├── Data Catalog    ← メタデータの一元管理
├── Crawler         ← データソースを自動スキャンしてカタログ登録
├── ETL Jobs        ← SparkベースのETL処理（Python/Scala）
├── Glue Studio     ← ビジュアルETLビルダー
├── DataBrew        ← ノーコードのデータ準備ツール
└── Streaming ETL   ← Kinesis・MSKのストリームデータ処理
```

---

## Glue Data Catalog

**AWSのデータ資産を一元管理するメタデータリポジトリ**。

```
Glue Data Catalog
├── Database（論理的なグループ名 ← 実データは入っていない）
│   └── Table（スキーマ定義・パーティション情報・S3パスなど）
```

### Database はただのラベル（名前空間）

Glue Data Catalog の「Database」は**RDSのような実際のデータベースではない**。
データは入っておらず、テーブル定義をグルーピングするための名前空間（ラベル）。

```
【一般的なDB（RDS・Auroraなど）】
Database = データが実際に格納される場所。レコードがある

【Glue Data Catalog の Database】
Database = テーブル定義を整理するフォルダ名。データは入っていない
実データは S3・RDS・DynamoDB など元の場所にある
```

```
Glue Data Catalog
└── Database: "analytics"（← ただのグループ名）
    ├── Table: "orders"
    │     場所: s3://my-bucket/orders/    ← 実データはここ
    │     スキーマ: id INT, amount INT...  ← 定義だけ持っている
    └── Table: "users"
          場所: rds://my-aurora/users     ← 実データはここ
          スキーマ: id INT, name STRING...
```

### テーブル定義の方法

テーブルは Crawler による自動登録だけでなく、手動でも定義できる。

| 方法 | 説明 |
|-----|------|
| **Glue Crawler** | データソースを自動スキャンしてテーブルを作成・更新 |
| **マネジメントコンソール** | GUIで手動定義（カラム名・型・S3パスなどを自分で入力） |
| **Athena の CREATE TABLE** | AthenaでDDLを実行するとData Catalogに自動登録される |
| **Glue ETL Job** | スクリプト内からAPIでテーブルを作成・更新 |
| **CloudFormation / Terraform** | IaCでテーブル定義を管理 |

```
【自動化したい】 → Crawler（スケジュール実行で常に最新状態を維持）
【スキーマを自分で制御したい】 → コンソール or Athena CREATE TABLE
【IaCで管理したい】 → CloudFormation / Terraform
```

- Athena・Redshift Spectrum・EMR など多くのサービスから参照できる
- Apache Hive Metastore 互換

```
S3（生データ）
    ↓ Glue Crawlerが自動スキャン（または手動定義）
Glue Data Catalog（テーブル定義・スキーマ）
    ↓ 参照
├── Amazon Athena（SQLクエリ）
├── Amazon Redshift Spectrum（外部テーブル）
└── Amazon EMR（Spark/Hive）
```

### パーティションキーとパーティションインデックス

S3のデータはパーティションキーで階層化して保存することが多い。
パーティションキーは**複数設定でき**、左から右に階層を作る。

```
s3://my-bucket/sensor-data/
    year=2026/          ← パーティションキー①
        month=06/       ← パーティションキー②
            day=28/     ← パーティションキー③
                sensor_id=abc/  ← パーティションキー④
```

Glueはこの階層を**左から順番**に辿って絞り込む。
そのため先頭から順に指定するクエリは速いが、途中のキーだけで絞るクエリは遅くなる。

```
【先頭から順に絞り込む → 速い】
WHERE year='2026' AND month='06' AND day='28'
→ 階層を順番に辿れるので問題なし

【先頭をスキップして sensor_id だけで絞り込む → 遅い（インデックスなし）】
WHERE sensor_id = 'abc'
→ year/month/dayの全組み合わせを舐めてからsensor_idを探す
→ パーティションが数百万件あると致命的に遅い
```

**パーティションインデックス**はこれを解決する。
RDBのインデックスと同じ発想で、先頭以外のキーへの高速アクセスを実現する。

```
【パーティションインデックスあり（sensor_id にインデックス設定）】
先頭キー（year/month/day）をスキップして sensor_id を直接検索 → 高速
```

| | 個数 | 役割 |
|--|--|--|
| **パーティションキー** | 複数OK（制限なし） | S3のフォルダ階層を定義する |
| **パーティションインデックス** | 最大3つ / テーブル | 先頭以外のキーへの高速アクセス |

- Athena からのクエリ速度改善に効く
- センサー・ログ系などパーティション数が膨大になるケースで特に有効

---

## Glue Crawler

**データソースを自動スキャンしてスキーマを推測し、Data Catalogに登録**するコンポーネント。

CrawlerはData Catalogに仕える存在で、**Data Catalogを自動で最新状態に保つことが目的**。
Data Catalog自体はCrawlerなしでも手動定義で使えるが、Crawlerがあると運用が楽になる。

```
【Crawlerなし】手動でテーブル定義を更新し続ける必要がある
【Crawlerあり】スキャンして自動でData Catalogを更新してくれる
```

### 動作の流れ

```
① 指定したデータソースに接続（S3パス・RDS・DynamoDBなど）
        ↓
② データをサンプリングして中身を確認
        ↓
③ Classifierがファイル形式を判定（CSV・JSON・Parquet・ORC…）
        ↓
④ カラム名・データ型・パーティション構造を推測
        ↓
⑤ Glue Data Catalog にテーブルとして登録・更新
```

### スキーマ変化への対応

```
【新しいパーティションが増えた場合】
/year=2026/month=06/day=27/ → 登録済み
/year=2026/month=06/day=28/ → Crawlerが自動検知して追加 ✅

【カラムが増えた場合】
先月: id, name, amount
今月: id, name, amount, discount ← 新カラム
→ Crawlerが差分を検知してData Catalogを更新
```

### Classifier（クラシファイア）

Crawlerがファイル形式を判定するために使う仕組み。

```
標準Classifier（デフォルト）: CSV・JSON・Parquet・ORC・Avro を自動判定
カスタムClassifier: 独自フォーマットのとき正規表現・Grokパターンで自分で定義
```

### セットアップの流れ

最初からCrawlerを使うのが一般的な運用。テーブルを手動で作る必要はない。

```
① Terraform apply
   → Data Catalog の Database を作成（空っぽ）
   → Crawler を作成（設定だけ。まだ動いていない）

② Crawler を実行（初回 or スケジュール）
   → S3・RDS をスキャン
   → Data Catalog のテーブルを自動作成

③ 以降はスケジュール実行で自動更新
```

```
terraform apply 直後
└── Database: "analytics"（空っぽ）

Crawler 初回実行後
└── Database: "analytics"
    ├── Table: "orders"    ← 自動作成
    ├── Table: "users"     ← 自動作成
    └── Table: "logs"      ← 自動作成
```

Terraformで管理する場合：

```hcl
# Data Catalog の Database（空の名前空間）
resource "aws_glue_catalog_database" "example" {
  name = "analytics"
}

# S3用 Crawler
resource "aws_glue_crawler" "s3_crawler" {
  name          = "s3-crawler"
  database_name = aws_glue_catalog_database.example.name
  role          = aws_iam_role.glue_role.arn

  s3_target {
    path = "s3://my-bucket/data/"
  }

  schedule = "cron(0 0 * * ? *)"  # 毎日深夜に実行
}

# RDS（Aurora MySQL）用 Crawler
resource "aws_glue_crawler" "rds_crawler" {
  name          = "rds-crawler"
  database_name = aws_glue_catalog_database.example.name
  role          = aws_iam_role.glue_role.arn

  jdbc_target {
    connection_name = aws_glue_connection.aurora.name
    path            = "database-a/%"  # 全テーブルが対象
  }
}
```

### Crawlerを使うか手動定義にするかの判断

| ケース | 方法 |
|--|--|
| スキーマが頻繁に変わる・パーティションが自動で増える | Crawler（自動更新） |
| スキーマが固定で変わらない | 手動定義（Crawlerのコスト不要） |
| 独自フォーマットのデータ | カスタムClassifierを使ったCrawler |

```
【スケジュール実行のユースケース】
毎日深夜にCrawlerを実行
    ↓
前日分の新しいパーティションを自動検知・カタログ登録
    ↓
翌朝Athenaでそのまま新しい日付のデータをクエリできる
```

---

## Glue ETL Jobs

**Apache Spark ベースの ETL スクリプト**を実行するコンポーネント。
Python（PySpark）または Scala で記述する。

```
ソース → 変換（フィルタ・結合・集計・型変換など）→ ターゲット
```

### ジョブタイプ

| タイプ | 実行環境 | 向いているケース |
|--|--|--|
| **Spark（PySpark / Scala）** | 複数ワーカーで分散処理 | GB〜TB規模の大量データ処理 |
| **Python Shell** | 1台で普通のPythonを実行 | 小規模データ・API呼び出し・軽量処理 |
| **Glue Streaming ETL** | Sparkベースのストリーム処理 | Kinesis・MSKのリアルタイム処理 |

```
データが大きい・複雑な変換 → Spark
データが小さい・シンプルな処理 → Python Shell（安くて速い）
ストリームデータをリアルタイム処理 → Glue Streaming ETL
```

### 実行クラス（Execution Class）

Glue ETL Job には2種類の実行クラスがある。EC2のスポットインスタンスと同じ発想。

| | Standard | Flex |
|--|--|--|
| リソース | 専用リソースを確保 | AWSの余剰リソースを使用 |
| 起動速度 | 速い・安定 | 遅くなる可能性あり |
| 中断リスク | なし | あり |
| コスト | 高め | 約34%安い |

```
【Standard が向いているケース】
毎朝9時までに必ず終わらせたい
リアルタイムダッシュボードのデータ更新
SLA（完了時刻の保証）がある

【Flex が向いているケース】
月次バッチで多少遅れてもOK
開発・テスト環境での動作確認
急ぎじゃない大量データの変換 → コスト削減に有効
```

### DynamicFrame（Gluе独自のデータ構造）

SparkのDataFrameをベースにしたGlue独自のデータ構造。
スキーマが曖昧・不安定なデータ（JSONのネスト構造など）に強い。

```python
# DynamicFrameの読み込み例
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="my_db",
    table_name="my_table"
)
```

### Job Bookmark（ジョブブックマーク）

**すでに処理したデータを記憶して、差分のみ処理**する機能。
増分ETL（新規データだけ処理）を実現できる。

```
【Job Bookmarkなし】
実行のたびに全データを再処理 → 重複・コスト増大

【Job Bookmarkあり】
前回処理済みの位置を記憶 → 新しいデータだけ処理
```

### Pushdown Predicate（プッシュダウン述語）

**データソース側でフィルタを適用**して、読み込むデータ量を最小化する最適化機能。

```python
# 特定のパーティションだけ読み込む例
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="my_db",
    table_name="my_table",
    push_down_predicate="year='2026' and month='06'"
)
```

### JDBCソースの並列読み込みパーティション

MySQLなどのJDBCソース（RDS・Aurora含む）からデータを読む際に設定するパーティション。
**Data Catalogのパーティションとは別物**で、Spark の並列読み取りを制御するもの。

```
【パーティション設定なし】
ワーカー1人が全件を順番に読む → 大量データだと遅い

【パーティションキー = id、分割数 = 4 を指定】
ワーカー1: WHERE id BETWEEN 1      AND 250000
ワーカー2: WHERE id BETWEEN 250001 AND 500000
ワーカー3: WHERE id BETWEEN 500001 AND 750000
ワーカー4: WHERE id BETWEEN 750001 AND 1000000
→ 4並列で読む → 速い
```

数値型・日付型のカラムをパーティションキーに指定するのが一般的。

```python
# Aurora MySQL から並列読み込みする例
datasource = glueContext.create_dynamic_frame.from_options(
    connection_type="mysql",
    connection_options={
        "url": "jdbc:mysql://...",
        "dbtable": "orders",
        "partitionColumn": "id",   # 分割の基準カラム
        "lowerBound": "1",
        "upperBound": "1000000",
        "numPartitions": "4"       # 並列数
    }
)
```

```
【2種類のパーティションの違い】
① S3パーティション（Data Catalog）
   → S3のフォルダ階層。Athenaのクエリ最適化に使う
   → パーティションインデックスが効く

② JDBCソースの並列読み込みパーティション
   → ETL Job スクリプト内の設定。MySQLからの読み取りを高速化
   → Data Catalogとは無関係
```

実務での Aurora MySQL → Glue → S3 パイプラインでスクリプトに書いていたのはこちら（②）。

### FindMatches ML Transform（名寄せ・重複排除）

**機械学習を使って「似ているレコード」を検出する**Glue の ML Transform。
通常の完全一致チェックでは検知できない表記ゆれ・略称・フォーマット差異に対応できる。

```
【完全一致チェックの限界】
"山田 太郎" と "山田太郎"   → 別人と判定 ❌（スペースの有無）
"John Smith" と "J. Smith" → 別人と判定 ❌（略称）

【FindMatches なら】
"山田 太郎" と "山田太郎"   → 同一人物と判定 ✅
"John Smith" と "J. Smith" → 同一人物と判定 ✅
```

**仕組み：**

```
① ラベリング（教師データを用意）
   「このレコードとこのレコードは同じ」
   「このレコードとこのレコードは別物」
   → 人間が数十〜数百件のサンプルに正解ラベルを付ける
        ↓
② FindMatches が ML モデルを学習
   → 「このデータにおける"似ている"とは何か」を学習
        ↓
③ 全データに対してモデルを適用
   → 似ているレコードのペアを自動検出
```

**2つのユースケース：**

| ユースケース | 説明 |
|--|--|
| **重複排除（Deduplication）** | 1つのデータセット内の重複レコードを検出・削除 |
| **レコードマッチング（Record Matching）** | 2つの異なるデータセット間で対応するレコードを突き合わせ |

```
【重複排除の例】
顧客マスタに同じ人が複数登録されている
→ FindMatchesで名寄せして1件にまとめる

【レコードマッチングの例】
自社の顧客DB × 外部の購買データを突き合わせたい
→ 名前・住所の表記ゆれがあっても同一人物を特定できる
```

参考書でLake Formationの文脈で紹介されることがあるが、FindMatches自体はGlue ETL Jobの機能。Lake FormationはGlueの上に乗るガバナンス層。

```
AWS Lake Formation（アクセス制御・ガバナンス）
        ↓ 管理対象
AWS Glue（ETL・Data Catalog）
        ↓ 機能の一つ
FindMatches ML Transform（名寄せ・重複排除）
```

---

## Glue Studio

**ビジュアルなドラッグ＆ドロップでETLジョブを構築**できるGUI。
コードを書かずにETLパイプラインを設定できる。

```
ノーコードで設定したい → Glue Studio（ビジュアルETL）
コードで細かく制御したい → Glue ETL Jobs（PySpark/Scala）
```

### 各レイヤーの役割

Glue を使う構成では、役割が3層に分かれる。

```
① Python スクリプト（.py ファイル）
   → ETL の実際の処理内容（どのテーブルを読んで・どう変換して・どこに書く）
   → エンジニアが PySpark で記述する

② Amazon S3
   → Python スクリプトの置き場所

③ Terraform
   → Glue Job の「箱」の設定を管理
   → IAMロール・DPU数・スクリプトのS3パスなどを定義
```

```hcl
# Terraform が管理する範囲（Glue Job の設定）
resource "aws_glue_job" "my_etl" {
  name     = "my-etl-job"
  role_arn = aws_iam_role.glue_role.arn

  command {
    script_location = "s3://my-bucket/scripts/my_etl.py"  # スクリプトの場所を指定
  }
}

# スクリプトファイル自体の S3 配置も Terraform で管理できる
resource "aws_s3_object" "glue_script" {
  bucket = "my-bucket"
  key    = "scripts/my_etl.py"
  source = "./scripts/my_etl.py"  # ローカルの .py ファイルをアップロード
}
```

```
Terraform の範囲  → Glue Job の設定 + スクリプトの S3 配置
Terraform の範囲外 → スクリプトの中身（ETL 処理のロジック）← エンジニアが Python で書く
```

Glue Studio を使えばスクリプトが自動生成されるため、
「生成されたコードをS3に置いてTerraformで管理」という構成も可能。

### ノードの種類と対応範囲

| 種類 | 役割 | 例 |
|--|--|--|
| **Source** | データの読み込み元 | S3・Data Catalog・Kinesis・JDBC |
| **Transform** | データの変換 | Filter・Join・集計・カラム削除・型変換 |
| **Target** | データの書き込み先 | S3・Redshift・Data Catalog |

標準ノードでカバーできない複雑なロジックは **Custom Transform ノード** で部分的にコードを書ける。

```
標準ノードだけ（コードゼロ）
        ↓ よくあるETL処理はこれでカバー
標準ノード + Custom Transform（部分的にコード）
        ↓ 独自ロジックが必要なとき
全部コードで書く（Glue ETL Jobs）
        ↓ 完全な自由度が必要なとき
```

### DataBrew との違い

| | Glue Studio | Glue DataBrew |
|--|--|--|
| 目的 | ETLパイプライン全体の構築 | データのクレンジング・プロファイリングに特化 |
| 対象ユーザー | データエンジニア | データアナリスト |
| コード生成 | PySpark を自動生成 | なし |
| データ品質確認 | できない | プロファイリング機能あり |

```
「S3からRedshiftにデータを流すパイプラインを組みたい」 → Glue Studio
「このデータの品質を確認してクレンジングしたい」      → DataBrew
```

---

## Glue DataBrew

**データのクレンジング・品質確認に特化したノーコードツール**。
データアナリスト・データサイエンティスト向けで、コードを書かずにGUIで操作できる。

### データプロファイリング

データを読み込むと自動でデータ品質を分析してくれる。

```
カラム: "age"
├── データ型: Integer
├── 欠損値: 3.2%（320件 / 10000件）
├── ユニーク値: 73種類
├── 最小値: 18 / 最大値: 95 / 平均: 42.3
├── 外れ値: 5件（150歳など明らかに異常な値）
└── 分布グラフ: ヒストグラムで可視化
```

### 主な変換機能（250種類以上）

| カテゴリ | 例 |
|--|--|
| **欠損値処理** | 平均値で補完・行を削除・固定値で埋める |
| **重複処理** | 重複行の削除 |
| **フォーマット統一** | 日付形式・大文字小文字の統一 |
| **カラム操作** | 分割・結合・リネーム・削除 |
| **外れ値処理** | 外れ値の除去・置換 |
| **エンコーディング** | カテゴリ変数を数値に変換 |

### 重要な概念

```
Dataset（データセット）
→ 入力データ（S3・Glue Data Catalog・RDS・Redshiftなど）

Project（プロジェクト）
→ GUIで変換を試すインタラクティブな作業スペース

Recipe（レシピ）
→ 変換手順のリスト（「①欠損値を削除 → ②日付を統一 → ③重複を除去」）
→ 保存して別のデータセットにも再利用できる

Job（ジョブ）
→ レシピを本番の全データに対して実行。結果をS3やData Catalogに出力

Profile（プロファイル）
→ データ品質レポート。欠損値・分布・外れ値などの統計情報
```

### Glue Studio・ETL Jobs との違い

| | DataBrew | Glue Studio | Glue ETL Jobs |
|--|--|--|--|
| 主な用途 | データ品質確認・クレンジング | ETLパイプライン構築 | カスタムETL処理 |
| 対象ユーザー | データアナリスト | データエンジニア | データエンジニア |
| コード | 不要 | 不要（一部Custom） | PySpark/Scala必須 |
| データ品質確認 | ✅ プロファイリングあり | ❌ | ❌ |
| パイプライン構築 | ❌ | ✅ | ✅ |

```
「このデータ品質どうなってる？欠損値どのくらい？」 → DataBrew（プロファイル）
「S3→変換→Redshiftのパイプラインを組みたい」     → Glue Studio
「複雑なカスタムETLをコードで書きたい」            → Glue ETL Jobs
```

### Step Functions への組み込みパターン

DataBrew は **Glue ETL の前段階**として使うのが典型的な使い方。
Step Functions で全体を統括するとこういう流れになる。

```
EventBridge（スケジュール実行）
        ↓
Step Functions
        │
        ├─① DataBrew Job（前処理）
        │     生データをスキャン
        │     欠損値補完・外れ値除去・フォーマット統一
        │     クレンジング済みデータを S3 に出力
        │        ↓ 成功
        ├─② Glue ETL Job（変換・ロード）
        │     クレンジング済みデータを変換・加工
        │     Redshift にロード
        │        ↓ 成功
        ├─③ 完了通知（Slack）
        │
        └─ ❌ いずれかが失敗したらエラー通知（Slack）
```

Step Functions から DataBrew Job を呼び出す際は `databrew:startJobRun.sync` を使う。

```json
{
  "StartAt": "DataBrewCleanJob",
  "States": {
    "DataBrewCleanJob": {
      "Type": "Task",
      "Resource": "arn:aws:states:::databrew:startJobRun.sync",
      "Parameters": { "Name": "my-databrew-job" },
      "Next": "GlueETLJob",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "NotifyFailure" }]
    },
    "GlueETLJob": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": { "JobName": "my-glue-etl-job" },
      "Next": "NotifySuccess",
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "NotifyFailure" }]
    }
  }
}
```

**DataBrew を毎回使うかどうかの判断：**

```
毎回使う → データソースの品質が不安定な場合
           （外部データ・サードパーティなど欠損値や外れ値が混入しやすい）

使わない → データソースが安定している場合
           （自社の管理された RDS など）
           → Glue ETL だけで十分
```

---

## Glue Streaming ETL

**Kinesis Data Streams や Amazon MSK（Kafka）のストリームデータをリアルタイムで処理**。

```
Kinesis Data Streams
Amazon MSK（Kafka）
        ↓ Glue Streaming ETL（Spark Structured Streaming）
Amazon S3 / Amazon Redshift / DynamoDB
```

---

## トリガーの種類

| トリガー | 説明 |
|---------|------|
| **スケジュール** | 毎日0時など定期実行 |
| **オンデマンド** | 手動実行 |
| **イベント** | 他のGlueジョブ完了後に連鎖実行 |

EventBridgeと組み合わせることで、S3にファイルが到着したタイミングで起動するパターンも組める。

```
S3にファイルが到着
    ↓ EventBridge（イベント検知）
    ↓ Glue ETL Job 起動
    ↓ 変換・クレンジング
S3 or Redshift に書き出し
```

---

## 対応データソース

| カテゴリ | 接続先 |
|---------|-------|
| **ストレージ** | Amazon S3・Amazon S3 Glacier |
| **リレーショナルDB** | Amazon RDS（MySQL・PostgreSQL・Oracle など）・Aurora |
| **分析** | Amazon Redshift |
| **NoSQL** | Amazon DynamoDB |
| **ストリーミング** | Amazon Kinesis・Amazon MSK |
| **カスタム** | JDBC接続（オンプレDBなど） |

---

## 他サービスとの比較

| 観点 | AWS Glue | Amazon EMR | AWS Lambda |
|------|---------|-----------|-----------|
| サーバー管理 | 不要（サーバーレス） | 必要（クラスター管理） | 不要 |
| 処理規模 | 大規模OK | 大規模OK（より細かい制御） | 小規模向き |
| 使用言語 | Python（PySpark）・Scala | Spark・Hive・Presto など | Python・Node.js など |
| 向いているケース | 標準的なETL | 複雑な大規模処理・ML | 軽量なデータ変換 |
| コスト | DPU時間課金 | EC2時間課金 | 実行回数・時間課金 |

```
標準的なETLパイプラインを手軽に → AWS Glue
複雑な処理・カスタムクラスター構成が必要 → Amazon EMR
軽量な変換・小規模処理 → AWS Lambda
```

---

## 実務との対応

実際に構築した **Aurora MySQL → Glue → CSV → S3** のパイプラインは、このGlue ETL Jobsの典型的な使い方。

```
【実務のパターン（ETL）】
Aurora MySQL
    ↓ Glue ETL Job（データ抽出・CSV変換）
Amazon S3（CSV格納）
    ↓
BigQuery（別チームが読み込み）

【より標準的なAWSパターン（ELT）】
Aurora MySQL
    ↓ Glue ETL Job（軽い変換だけ）
Amazon S3（Parquet格納）
    ↓
Amazon Athena / Redshift（SQLで分析）
```

---

## 試験のポイント

- **サーバーレスETL** → AWS Glue（サーバー管理不要）
- **Glue Data Catalog** → メタデータの一元管理。Athena・EMR・Redshift Spectrumが参照する
- **Glue Crawler** → S3などをスキャンしてスキーマを自動検知・カタログ登録
- **Job Bookmark** → 差分処理（増分ETL）を実現。重複処理を防ぐ
- **DynamicFrame** → GlueのDataFrame。スキーマが不安定なデータに強い
- **Pushdown Predicate** → ソース側でフィルタ → 読み込みデータ量を削減
- **Glue DataBrew** → データクレンジング特化のノーコードツール
- **Glue Streaming ETL** → Kinesis・MSKのリアルタイム処理
- **Python Shell** → 小規模・軽量処理向け。Sparkなし・安い
- **Flex 実行クラス** → 余剰リソースを使う。約34%安いが起動遅延・中断リスクあり。非緊急バッチ向け
- **FindMatches** → MLで表記ゆれのある重複レコードを検出・名寄せ
- **EMRとの違い** → Glueはサーバーレス・簡単。EMRはより細かい制御が可能
