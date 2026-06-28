# <img src="./images/aws-step-functions.png" width="36"/>&nbsp;&nbsp; AWS Step Functions

## 概要

**複数のAWSサービスをワークフローとして順番に・並列に・条件分岐しながら実行**できるサーバーレスのオーケストレーションサービス。

```
Step Functions（全体を統括するワークフローエンジン）
├── Lambda（軽量処理）
├── Glue ETL Job（データ変換）
├── EMR（大規模処理）
├── AWS Batch（バッチ処理）
├── DynamoDB（データ読み書き）
└── SNS / SQS（通知・キュー）
```

各サービスを「ステート（状態）」として定義し、**ステートマシン**としてワークフローを管理する。

---

## Workflow Studio

**マネジメントコンソール上でワークフローをビジュアルで組める GUI ツール**。
Glue Studio と同じ発想で、ドラッグ＆ドロップでステートを繋げてワークフローを定義できる。

裏側では **Amazon States Language（ASL）** という JSON が自動生成される。

```json
{
  "StartAt": "GlueETLJob",
  "States": {
    "GlueETLJob": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      ...
    }
  }
}
```

### Terraform との組み合わせ

Workflow Studio と Terraform は対立するものではなく補完的に使える。

```
① Workflow Studio でビジュアルにワークフローを設計・確認
② JSON（ASL）をエクスポート
③ Terraform の定義に貼り付けて IaC として管理
```

```hcl
resource "aws_sfn_state_machine" "etl_pipeline" {
  name     = "etl-pipeline"
  role_arn = aws_iam_role.sfn_role.arn

  definition = <<EOF
  {
    "StartAt": "GlueETLJob",
    "States": { ... }
  }
  EOF
}
```

```
ビジュアルで設計・確認したい → Workflow Studio
IaC で管理したい             → Terraform（ASL の JSON を定義に組み込む）
```

---

## ワークフローの種類

| | Standard Workflow | Express Workflow |
|--|--|--|
| 最大実行時間 | **1年** | 5分 |
| 実行保証 | **exactly-once**（1回だけ実行） | at-least-once（最低1回） |
| 実行履歴 | 90日間保存・監査可能 | CloudWatch Logs に出力 |
| 向いているケース | ETLパイプライン・長時間バッチ | 高頻度・短時間の処理 |
| コスト | 状態遷移回数で課金 | 実行回数・時間で課金（安い） |

```
ETLパイプライン・バッチ処理 → Standard Workflow
IoTイベント処理・高頻度な短時間処理 → Express Workflow
```

---

## ステートの種類

ワークフローは複数の「ステート（状態）」で構成される。

| ステート | 役割 |
|---------|------|
| **Task** | AWSサービスを呼び出す（Lambda・Glue・EMRなど） |
| **Choice** | 条件分岐（if/else） |
| **Parallel** | 複数のブランチを並列実行 |
| **Map** | リストの各要素に対して同じ処理を繰り返す |
| **Wait** | 指定時間または特定時刻まで待機 |
| **Pass** | 入力をそのまま次のステートに渡す（変換可能） |
| **Succeed** | 正常終了 |
| **Fail** | 異常終了 |

---

## Pass ステート

**AWSサービスを呼び出さずに、データを加工・固定値を注入して次のステートに渡す**だけのステート。
コストはゼロ。

### 主な使い方

**① 固定値を注入する**

入力データに固定のパラメータを追加したいとき。

```json
{
  "Type": "Pass",
  "Result": {
    "environment": "production",
    "version": "2.0"
  },
  "ResultPath": "$.config",
  "Next": "GlueETLJob"
}
```

```
入力:  { "bucket": "my-bucket" }
出力:  { "bucket": "my-bucket", "config": { "environment": "production", "version": "2.0" } }
                                 ↑ 固定値が追加された
```

**② データの形を変える（整形）**

次のステートが期待する形式に変換したいとき。

```json
{
  "Type": "Pass",
  "Parameters": {
    "JobName.$": "$.glue_job_name",
    "Arguments.$": "$.params"
  },
  "Next": "GlueETLJob"
}
```

**③ デバッグ・開発時の一時的な置き換え**

重い Task ステートを一時的に Pass に差し替えてワークフローの流れだけ確認できる。

```
本番:
Lambda（重い処理）→ Glue ETL → Redshift

開発時:
Pass（ダミーデータを流す）→ Glue ETL → Redshift
→ Lambda の実装を待たずにワークフロー全体の流れをテストできる
```

---

## Task ステートの統合モード

AWSサービスを呼び出す方法が3種類ある。

| モード | 動作 | 使いどころ |
|-------|------|-----------|
| **Request-Response** | 呼び出して即次のステートへ | 非同期でよい処理 |
| **.sync（同期）** | 処理完了を待ってから次へ | Glue・EMR・Batchなど時間がかかる処理 |
| **.waitForTaskToken** | 外部システムからのトークン返却を待つ | 人の承認待ち・外部APIの完了待ち |

```
# Glue Job を同期で呼び出す例
"GlueETLJob": {
  "Type": "Task",
  "Resource": "arn:aws:states:::glue:startJobRun.sync",  ← .sync で完了を待つ
  "Parameters": { "JobName": "my-etl-job" },
  "Next": "NotifySuccess"
}
```

---

## エラーハンドリング

### Retry（リトライ）

失敗時に自動でリトライする。

```json
"Retry": [
  {
    "ErrorEquals": ["States.TaskFailed"],
    "IntervalSeconds": 30,   // 30秒後にリトライ
    "MaxAttempts": 3,        // 最大3回
    "BackoffRate": 2         // リトライのたびに待機時間を2倍に
  }
]
```

### Catch（エラーキャッチ）

リトライが全部失敗したら別のステートに移行する。

```json
"Catch": [
  {
    "ErrorEquals": ["States.ALL"],
    "Next": "NotifyFailure"   // 失敗通知のステートへ
  }
]
```

---

## Map ステート

リストの各要素に対して同じ処理を繰り返す。並列実行も可能。

```
対象ファイルリスト: [fileA, fileB, fileC, fileD]
        ↓ Map ステート（並列数: 4）
fileA → Glue Job 実行
fileB → Glue Job 実行  ← 同時に並列実行
fileC → Glue Job 実行
fileD → Glue Job 実行
```

### Inline Map vs Distributed Map

本質的な違いは**データの場所**にある。

```
【Inline Map】
前のステートから配列が渡ってくる
["userA", "userB", "userC"]  ← ステートマシンの入力（メモリ上）
        ↓ Map ステートがリストを受け取って繰り返す
userA・userB・userC それぞれに Lambda を実行

【Distributed Map】
「s3://my-bucket/data/」を指定するだけ
        ↓ Step Functions が S3 を直接スキャン
見つけた全ファイル・全行に対して自動で並列実行
→ 何件あるか事前に知らなくていい！
```

| | Inline Map | Distributed Map |
|--|--|--|
| データの場所 | **ステートマシンの入力（配列）** | **S3を直接参照** |
| 最大並列数 | 40 | 10,000 |
| 実行単位 | 同じ実行内のブランチ | 独立した子ステートマシン |
| 件数の事前把握 | 必要（配列を渡す） | 不要（S3を丸ごと指定） |
| 向いているケース | 数十件程度 | 数万〜数百万件 |

```
「このリスト10件それぞれに処理して」  → Inline Map（配列を入力として渡す）
「S3のあのデータを全部並列処理して」  → Distributed Map（S3を直接指定）
「件数が事前にわからない大量データ」  → Distributed Map
```

---

## データエンジニアリングでの典型的なパターン

### ① ETLパイプラインの標準構成

```
EventBridge（スケジュール）
        ↓
Step Functions
    ├── DataBrew Job（データクレンジング）
    │        ↓ 成功
    ├── Glue ETL Job（変換・ロード）
    │        ↓ 成功
    ├── Lambda（完了通知・Slack）
    │
    └── ❌ 失敗時 → Lambda（エラー通知・Slack）
```

### ② 並列ETLパターン（Parallel ステート）

```
Step Functions
    ├── Parallel ステート
    │    ├── ブランチA: 売上データを処理
    │    ├── ブランチB: 在庫データを処理
    │    └── ブランチC: 顧客データを処理
    │         ↓ 全ブランチ完了後
    └── Glue Job（統合・集計）
```

### ③ 大量ファイルの分散処理（Distributed Map）

```
S3に1万件のCSVファイルが存在
        ↓ Distributed Map（並列数: 1000）
各ファイルを Lambda で並列処理
        ↓ 全件完了
結果を S3 に集約
```

### ④ 条件分岐パターン（Choice ステート）

```
Lambda（ファイルサイズをチェック）
        ↓
Choice ステート
    ├── 1GB未満 → Lambda で処理
    └── 1GB以上 → Glue ETL Job で処理
```

---

## AWS Batch との組み合わせ

AWS Batch の Array Jobs の代わりに Step Functions の Map ステートを使うパターンもある。

```
【Step Functions + Lambda の Map（以前の実務パターン）】
Map ステート → Lambda を並列実行（小〜中規模）

【Step Functions + AWS Batch の Array Jobs】
Map ステート → Batch を呼び出す（大規模・Docker コンテナで処理）

【Step Functions + Distributed Map】
Distributed Map → Lambda / ECS を並列実行（超大規模・S3ファイル処理）
```

---

## Glue・EMR・Batch との役割分担

Step Functions 自体はデータを処理しない。**指揮者**として各サービスを呼び出す。

```
Step Functions = 指揮者（誰に何をやらせるかを管理）
    ├── Lambda    = 軽量処理担当
    ├── Glue      = ETL処理担当
    ├── EMR       = 大規模処理担当
    └── AWS Batch = 大規模バッチ処理担当
```

---

## 試験のポイント

- **ワークフローのオーケストレーション** → Step Functions
- **Standard vs Express** → ETLパイプラインは Standard（1年・exactly-once）
- **.sync** → Glue・EMR・Batchなど完了を待ちたいときに使う
- **Map ステート** → リストの各要素を繰り返し処理・並列実行
- **Distributed Map** → S3の大量ファイルを最大1万並列で処理
- **Retry / Catch** → 失敗時の自動リトライ・エラー通知
- **Choice ステート** → 条件によって処理を分岐
- **Parallel ステート** → 複数の独立したブランチを同時実行
- **Step Functions 自体はデータを処理しない** → 各サービスへの指示役
