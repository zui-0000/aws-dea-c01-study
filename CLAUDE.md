# CLAUDE.md — AWS DEA-C01 学習リポジトリ

## 目的

AWS Certified Data Engineer - Associate（DEA-C01）試験の学習ノートを管理するリポジトリ。
Markdownで日本語の学習ノートを作成・管理する。

---

## ディレクトリ構成

```
aws-dea-c01-study/
├── README.md
├── 00_concepts/             # AWS横断の概念（ETL/ELT、ストリーミング、データレイクなど）
├── 01_ingestion-and-transformation/   # ドメイン1: データの取り込みと変換（34%）
│   └── images/              # アイコン画像
├── 02_datastore-management/           # ドメイン2: データストアの管理（26%）
│   └── images/
├── 03_operations-and-optimization/    # ドメイン3: データ運用と最適化（22%）
│   └── images/
└── 04_security-and-governance/        # ドメイン4: データセキュリティとガバナンス（18%）
    └── images/
```

---

## ファイル命名規則

| ディレクトリ | 命名方式 | 例 |
|------------|---------|-----|
| `00_concepts/` | 日本語 | `ETLとELT.md`、`データレイク.md` |
| 各ドメインのAWSサービスノート | 英語（アンダースコア区切り） | `Amazon_Kinesis_Data_Streams.md`、`AWS_Batch.md` |
| 各ドメインのインデックス | 日本語 | `ドメイン1_データの取り込みと変換.md` |

---

## ノートのフォーマット

### 表題とアイコン

アイコンがあるサービスはヘッダー横に埋め込む。

```markdown
# <img src="./images/aws-kinesis-data-streams.png" width="36"/>&nbsp;&nbsp; Amazon Kinesis Data Streams
```

- `width="36"` で統一
- 表題との間に `&nbsp;&nbsp;` を入れてスペースを確保
- 画像は各ドメインの `images/` フォルダに配置

### アイコンがないサービス

公式アイコンが存在しない場合（SCT・DynamoDB Streamsなど）はHTMLコメントで明記する。

```markdown
# AWS SCT（Schema Conversion Tool）
<!-- アイコンなし: SCTはローカルインストール型のツールであり、マネージドサービスではないため公式アイコンが存在しない -->
```

### アイコン画像の形式

- **PNG / webp**: VS Code のMarkdownプレビューで表示可能 → 基本これを使う
- **SVG**: GitHubでは表示されるが、VS CodeのMarkdownプレビューでは表示されない（セキュリティ制限）

### ノートの構成

```markdown
# <img .../>&nbsp;&nbsp; サービス名

## 概要
サービスの一言説明と用途

## 仕組み / 特徴
（コードブロック・図表を活用）

## ユースケース

## 比較表（他サービスとの違い）

## 試験のポイント
```

- 末尾には必ず `## 試験のポイント` セクションを設ける
- コードブロックはアーキテクチャの流れや比較を示すために積極的に使う
- Markdownの表（`|---|---|`形式）でサービス比較をまとめる

---

## Gitコミット規則

- コミットメッセージは日本語でOK
- 学習ノートを追加した場合は `Add: <サービス名>` など端的に
- まとめてプッシュする場合は `docs: ノートまとめ` など

---

## その他の注意事項

- **ノートを作成・更新する際は、必ず現在の年月日を取得し、その時点での最新情報に基づいて内容をまとめること**
  - AWSサービスの仕様・ランタイムバージョン・料金体系などは変わることがある
  - Web検索で最新情報を確認してからノートに反映する
- 実務経験（Aurora MySQL → Glue → S3 のパイプライン構築など）と照らし合わせてノートを作成している
