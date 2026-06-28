# <img src="./images/aws-amazon-managed-service-for-apatch-flink.png" width="36"/>&nbsp;&nbsp; Amazon Managed Service for Apache Flink

## 概要

ストリーミングデータを**リアルタイムで分析・変換・集計**するフルマネージドサービス。
以前は「Kinesis Data Analytics」（厳密には Kinesis Data Analytics for Apache Flink）という名前だったが現在は改名。

なお、旧 **Kinesis Data Analytics for SQL** はこれとは別系統のサービスで、現在は新規利用が制限されており、Apache Flink ベースの本サービスへの移行が推奨されている。

Apache Flink というオープンソースのストリーム処理エンジンをAWSがマネージドで提供している。

### 対応 Apache Flink バージョン

2026年時点で **1.20 / 1.19 / 1.18 / 1.15** をサポート（1.20 は 2024年9月に対応。Flink 1.x 系の LTS）。加えて、メジャーバージョンの **2.2** もサポートされている。

```
Kinesis Data Streams / MSK（Kafka）
    ↓
Amazon Managed Service for Apache Flink（リアルタイム分析・集計）
    ↓
S3 / Redshift / DynamoDB / CloudWatch など
```

---

## Kinesis Data Streams との役割の違い

混同しやすいので整理する。

```
Kinesis Data Streams  → データを「流す・保持する」パイプ
Amazon MSF for Flink  → 流れてくるデータを「リアルタイムで分析・集計する」エンジン
```

| サービス | 役割 |
|---------|------|
| Kinesis Data Streams | データを収集・保持する（ベルトコンベア） |
| Amazon Data Firehose | データを配信先に届ける |
| **Amazon MSF for Flink** | **流れているデータをリアルタイムで処理・分析する** |

---

## できること

### リアルタイム集計

```
Kinesis からデータが流れてくる
    ↓
Flink が「直近1分間の売上合計」をリアルタイムで集計
    ↓
CloudWatch にメトリクスとして送信
```

### ウィンドウ処理

時間の窓（ウィンドウ）を使った集計ができる。

| ウィンドウ種類 | 説明 | 例 |
|-------------|------|-----|
| **タンブリングウィンドウ** | 重複なしの固定時間窓 | 毎分0秒〜59秒の集計 |
| **スライディングウィンドウ** | 重複ありの移動時間窓 | 直近5分間を1分ごとに集計 |
| **セッションウィンドウ** | 一定時間データが来なければ終了 | ユーザーのセッション単位で集計 |

### フィルタリング・変換

```
流れてくるデータの中から条件に合うものだけ抽出
例: エラーログだけを抽出してアラート発火
```

---

## Lambda との違い

どちらもストリームデータを処理できるが用途が違う。

| 観点 | Lambda | Amazon MSF for Flink |
|------|--------|---------------------|
| 向いている処理 | シンプルな変換・DB書込 | 複雑な集計・ウィンドウ処理 |
| 状態管理 | ない（ステートレス） | ある（ステートフル） |
| 集計処理 | 苦手（自前実装が必要） | 得意（ネイティブ機能） |
| コスト | 安い | 高い |

**「直近5分間の平均を出したい」** のようなウィンドウ集計はLambdaでは難しく、Flinkが適している。

---

## 典型的なアーキテクチャ

```
IoT センサー / アプリログ
    ↓
Kinesis Data Streams（収集・保持）
    ↓
Amazon MSF for Flink（リアルタイム分析）
    ├── 異常値検知 → SNS（アラート通知）
    ├── 集計結果  → CloudWatch（モニタリング）
    └── 加工データ → S3 / Redshift（保存・分析）
```

---

## FirehoseとFlinkの組み合わせ

FlinkはFirehoseからデータを受け取ることはできない。方向は逆で、**Flinkの出力先としてFirehoseを使う**。

```
✗ Firehose → Flink（FlinkはFirehoseからは受け取れない）

✅ Kinesis Streams → Flink（分析・集計）→ Firehose → S3 / Redshift
```

---

## FirehoseとLambdaの関係

FirehoseはLambdaをコンシューマーとして使っているわけではない。

```
【Firehoseの内部】
データ受信
    ↓（任意）Lambda を呼び出す ← 変換ヘルパーとして
    │ 例: JSON → Parquet 変換
    ↓
S3 / Redshift へ配信
```

| 役割 | 説明 |
|------|------|
| **コンシューマー** | データを受け取って処理する主体（Lambda・Flink がなる） |
| **変換ヘルパー** | 配信前に加工するだけ（FirehoseにおけるLambdaの役割） |

LambdaはFirehoseの「コンシューマー」ではなく「変換ヘルパー」。しかも**設定は任意**で、使わなくてもFirehoseは動く。

---

## 試験のポイント

- **ストリームデータのリアルタイム集計・分析** → Amazon MSF for Flink
- **ウィンドウ処理が必要** → Flink（Lambdaでは難しい）
- **以前の名前は Kinesis Data Analytics** → 改名されているので注意
- **Kinesis Streams と組み合わせて使う**のが定番パターン
- **ステートフルな処理**（過去のデータを参照しながら処理）ができるのが強み
