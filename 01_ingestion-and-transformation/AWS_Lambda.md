# <img src="./images/aws-lambda.png" width="36"/>&nbsp;&nbsp; AWS Lambda

## 概要

**イベント駆動のサーバーレスコンピューティングサービス**。
コードをアップロードするだけで実行でき、サーバーの管理が不要。
実行した時間・回数だけ課金される。

```
イベント（S3・Kinesis・DynamoDB Streamsなど）
        ↓ トリガー
AWS Lambda（コードを実行）
        ↓ 結果を出力
S3・DynamoDB・Redshift・外部APIなど
```

データエンジニアリングでは**軽量なデータ変換・イベント駆動の処理・パイプラインの起点**として活用される。

---

## 基本的な制限

| 項目 | 制限 |
|-----|------|
| **タイムアウト** | 最大15分 |
| **メモリ** | 最大10GB |
| **一時ストレージ（/tmp）** | 最大10GB |
| **デプロイパッケージ** | 最大250MB（展開後） |

**15分のタイムアウトが最も重要**。長時間かかる処理には使えない。

```
処理が15分以内に終わる → Lambda で十分
処理が15分を超える可能性がある → Glue / EMR を使う
```

---

## イベントソース（トリガー）

Lambda はさまざまなAWSサービスのイベントで起動できる。

| トリガー | 説明 |
|---------|------|
| **Amazon S3** | ファイルのアップロード・削除を検知して起動 |
| **Amazon Kinesis Data Streams** | ストリームデータをポーリングして処理 |
| **Amazon DynamoDB Streams** | テーブルの変更を検知して処理 |
| **Amazon SQS** | キューのメッセージを処理 |
| **Amazon SNS** | 通知をトリガーに処理 |
| **Amazon EventBridge** | スケジュール・イベントルールで起動 |
| **Amazon API Gateway** | HTTPリクエストで起動 |

---

## イベントソースマッピング

Kinesis・DynamoDB Streams・SQS は**ポーリング型**のトリガー。
Lambdaが自動でストリームをポーリングしてレコードをバッチで受け取る。

```
Kinesis Data Streams（データが流れてくる）
        ↓ Lambda が自動でポーリング
AWS Lambda（バッチでレコードを受け取って処理）
```

詳細は `00_concepts/イベントソースマッピング.md` を参照。

---

## データエンジニアリングでの主なユースケース

### ① S3イベント駆動の軽量ETL

```
S3にファイルが届く
    ↓ S3イベント → Lambda 起動
Lambda（CSVをパース・バリデーション・DynamoDBに書き込み）
```

### ② Firehose のデータ変換

Firehose の配信前にデータを変換するために Lambda を呼び出せる。

```
データソース → Kinesis Data Firehose
                    ↓ Lambda を呼び出して変換
                Lambda（JSONをParquetに変換・不要フィールドを削除など）
                    ↓ 変換済みデータ
              Amazon S3 / Redshift に配信
```

Firehose の中で Lambda を使うことで、配信前にリアルタイムで変換できる。

### ③ DynamoDB Streams との連携

```
DynamoDB テーブルが更新される
    ↓ DynamoDB Streams → Lambda 起動
Lambda（変更内容をElasticsearchに同期・S3に書き出しなど）
```

### ④ Step Functions のタスクとして組み込む

```
Step Functions
    ├── Lambda（データバリデーション）
    ├── Glue ETL Job（大規模変換）
    └── Lambda（完了通知・Slack送信）
```

軽量な前処理・後処理を Lambda、重い変換を Glue に任せる分担が多い。

### ⑤ EventBridge でスケジュール起動

```
EventBridge（毎日0時）
    ↓
Lambda（S3の古いファイルを削除・メタデータを更新など）
```

---

## デプロイ形式

Lambdaのデプロイ形式は2種類ある。

| | ZIP ファイル | コンテナイメージ |
|--|--|--|
| サイズ上限 | 250MB（展開後） | 最大10GB |
| 保存先 | S3 / コンソール直接 | Amazon ECR |
| 向いているケース | 軽いコード・小さいライブラリ | 重い依存ライブラリ（pandas・ML系） |

### コンテナイメージのデプロイ

「LambdaはコードだからDockerと関係ない」と思いがちだが、
**Dockerイメージの中にLambda関数のコードを入れる**という仕組み。中身は同じ handler 関数。

```python
# lambda_function.py（中身は普通のLambda関数と同じ）
import pandas as pd

def handler(event, context):
    df = pd.DataFrame(event['data'])
    # 処理...
    return {'statusCode': 200}
```

```dockerfile
# Dockerfile
FROM public.ecr.aws/lambda/python:3.12  # AWS公式のLambdaベースイメージ

COPY requirements.txt .
RUN pip install -r requirements.txt     # pandas等をインストール

COPY lambda_function.py .               # 関数コードをコピー

CMD ["lambda_function.handler"]         # 実行するhandlerを指定
```

ECR（Elastic Container Registry）= DockerHubのAWS版・プライベートなイメージ置き場。
ZIPはS3に直接置けるが、コンテナイメージはECRを経由する必要がある。

```bash
# ECRにプッシュする流れ
① docker build でイメージをビルド
② ECR にリポジトリを作成
③ docker push で ECR にプッシュ
④ Lambda の設定で「ECRのこのイメージを使う」と指定
```

```
Lambda 関数が呼ばれる
    ↓
ECR からコンテナイメージを取得
    ↓
コンテナを起動（pandas 等がすでに入っている）
    ↓
handler 関数を実行 → 結果を返す
```

```
ZIPの250MBに収まる   → ZIP形式
pandas・MLライブラリなど重い依存がある → コンテナイメージ（ECR）
```

---

## Lambda Layers（レイヤー）

複数のLambda関数で共通して使うライブラリ・コードを共有する仕組み。
**1回デプロイしておけば、複数のLambda関数から参照できる**。

```
【Layerなし】
Lambda関数A ─── pandas を内包（デプロイパッケージが重い）
Lambda関数B ─── pandas を内包（同じものを重複してデプロイ）
Lambda関数C ─── pandas を内包（無駄が多い）

【Layerあり】
Layer「pandas-layer」（1回だけデプロイ）
    ↑           ↑           ↑
Lambda関数A  Lambda関数B  Lambda関数C
→ 複数の関数が同じLayerを共有。軽い・管理が楽
```

**実際の使い方：**

```
① Layer を作成・AWSに登録
   「pandas-layer」として1回デプロイするだけ

② 各Lambda関数の設定でLayerを指定
   関数A の設定 → pandas-layer を使う
   関数B の設定 → pandas-layer を使う
   関数C の設定 → pandas-layer を使う
```

- 1関数につき最大**5つ**のLayerを適用できる
- Layerはバージョン管理できる（`pandas-layer:v1`・`pandas-layer:v2`など）
- AWSが公式に提供しているLayerもある（AWS SDK・Lambda Powertoolsなど）
- データ処理でよく使うpandasはLayerとして管理するのが定番

---

## コールドスタートと Provisioned Concurrency

### コールドスタート

Lambda は普段は実行環境が起動していない。初回・久しぶりの呼び出し時に起動時間（コールドスタート）が発生する。

```
【コールドスタートあり】
リクエスト → 実行環境を起動（数百ms〜数秒）→ 関数実行

【コールドスタートなし（2回目以降・ウォーム状態）】
リクエスト → 関数実行（即座に）
```

### Provisioned Concurrency（プロビジョニングされた同時実行）

あらかじめ実行環境を起動しておくことでコールドスタートを防ぐ。

```
コールドスタートが許容できる → 通常のLambda
コールドスタートを防ぎたい  → Provisioned Concurrency（コスト増）
```

---

## 同時実行数（Concurrency）

| 種類 | 説明 |
|-----|------|
| **Reserved Concurrency** | 関数ごとに最大同時実行数を予約・制限 |
| **Provisioned Concurrency** | あらかじめ実行環境を起動してコールドスタートを防ぐ |

---

## Glue・EMR との比較

| 観点 | AWS Lambda | AWS Glue | Amazon EMR |
|-----|-----------|---------|-----------|
| タイムアウト | **最大15分** | 実質なし | 実質なし |
| データ規模 | 小規模 | 中〜大規模 | 大〜超大規模 |
| サーバー管理 | 不要 | 不要 | 必要 |
| 向いているケース | 軽量変換・イベント駆動 | 標準ETL | 複数フレームワーク・大規模 |

```
軽量処理・イベント駆動・15分以内 → Lambda
標準ETLパイプライン・サーバーレス → Glue
大規模・Hive/Presto・細かい制御  → EMR
```

---

## 試験のポイント

- **タイムアウト最大15分** → 長時間処理には使えない
- **Firehose + Lambda** → 配信前のリアルタイム変換
- **イベントソースマッピング** → Kinesis・DynamoDB Streams・SQSはポーリング型
- **S3イベント → Lambda** → ファイル到着をトリガーに処理
- **Lambda Layers** → 共通ライブラリを共有（pandas・numpyなど）
- **Provisioned Concurrency** → コールドスタートを防ぐ（コスト増）
- **Step Functionsとの組み合わせ** → 軽量処理をLambda・重い処理をGlueに分担
