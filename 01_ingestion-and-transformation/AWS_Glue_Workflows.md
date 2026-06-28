# AWS Glue Workflows（Glueワークフロー）

## 概要

**Glueのコンポーネント（Job・Crawler・Trigger）だけを繋げてパイプラインを定義**できるオーケストレーション機能。
Glueコンソール上でビジュアルに管理できる。

```
Glue Workflow
├── Trigger（スケジュール or 完了イベント）
├── Crawler（S3スキャン・スキーマ更新）
└── ETL Job（変換・ロード処理）
```

---

## スケジュール実行

**Glue Workflows は自身がcron設定を持てる**。EventBridgeなしで自律的に動く。

```
Glue Workflow の Trigger に cron を直接設定
→ EventBridge 不要でスケジュール実行できる
```

これは Step Functions との重要な違い。

| | スケジュール機能 | 外部トリガーが必要 |
|--|--|--|
| **Glue Workflows** | ✅ 自身がcronを持つ | 不要 |
| **Step Functions** | ❌ 持たない | EventBridgeが必要 |
| **AWS Batch** | ❌ 持たない | EventBridgeが必要 |
| **Glue ETL Job 単体** | ✅ Triggerで設定可 | 不要 |

```
Step Functionsをスケジュール実行したい場合:
EventBridge（cron設定）
    ↓ 指定時刻になったら呼び出す
Step Functions（ワークフロー実行）
```

---

## できること

```
Trigger（スケジュール）
    ↓
Crawler（S3をスキャン・スキーマ更新）
    ↓ 完了したら
Glue ETL Job A（前処理・クレンジング）
    ↓ 完了したら
Glue ETL Job B（集計・Redshiftへロード）
```

Glueの構成要素だけで完結するシンプルなETLパイプラインを組める。

---

## Step Functions との比較

| | Glue Workflows | AWS Step Functions |
|--|--|--|
| 連携できるサービス | **Glueのみ**（Job・Crawler・Trigger） | **220以上のAWSサービス** |
| Lambda との連携 | ❌ | ✅ |
| EMR・Batchとの連携 | ❌ | ✅ |
| Slack通知（Lambda経由） | ❌ | ✅ |
| エラーハンドリング | 限定的 | Retry・Catch で細かく制御 |
| 条件分岐 | 限定的 | Choice ステートで柔軟に対応 |
| 実行履歴・監視 | Glueコンソール内 | Step Functionsコンソールで詳細に確認 |
| 設定の複雑さ | シンプル | やや複雑 |
| 追加コスト | なし（Glueの料金のみ） | 状態遷移回数で課金 |

---

## Glue Workflows のメリット（限定的）

正直なところ、**ユースケースは非常に限られている**。

```
メリット①: Glueだけで完結するなら設定がシンプル
メリット②: Step Functionsの追加コストがかからない
メリット③: Glueコンソールで完結するため管理が一元化される
```

ただし以下のような要件が1つでもあると Step Functions の方が適切：

```
❌ Slack通知がしたい         → Lambda が必要 → Step Functions
❌ エラー時に別処理をしたい   → Catch が必要 → Step Functions
❌ EMR や Batch も使いたい   → Glue 外のサービス → Step Functions
❌ 複雑な条件分岐がしたい     → Choice ステート → Step Functions
```

---

## 実務での使い分け

```
「Crawler → Glue Job → Glue Job だけで完結する」
「通知もエラーハンドリングも不要」
「とにかくシンプルに使いたい」
→ Glue Workflows

上記以外のほぼ全てのケース
→ Step Functions の方が柔軟で長期的に使いやすい
```

Slack通知・エラーハンドリング・Lambda連携など実務でよくある要件が加わった時点で、Step Functionsに移行することになるケースが多い。

---

## 試験のポイント

- **Glue Job・Crawler だけを繋げるシンプルなETL** → Glue Workflows
- **Lambda・EMR・Batchなど他サービスと組み合わせる** → Step Functions
- **Slack通知・エラーハンドリングが必要** → Step Functions
- **Glue Workflows は Glue 専用**。他サービスとは連携できない
