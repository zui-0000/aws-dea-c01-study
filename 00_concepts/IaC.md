# IaC（Infrastructure as Code）

## 概要

インフラの構成をコードで定義・管理する手法。
手動でのコンソール操作をなくし、再現性・バージョン管理・自動化を実現する。

```
従来:
マネジメントコンソールで手動操作 → 再現性なし・ミスが起きやすい

IaC:
コードでインフラを定義 → git管理・レビュー・自動デプロイが可能
```

---

## AWS CloudFormation

**AWSが提供するネイティブのIaCサービス**。
JSON または YAML のテンプレートでAWSリソースを定義する。

```yaml
# CloudFormation テンプレートの例
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  MyGlueJob:
    Type: AWS::Glue::Job
    Properties:
      Name: my-etl-job
      Role: !GetAtt GlueRole.Arn
      Command:
        ScriptLocation: s3://my-bucket/scripts/etl.py
```

### 主な概念

```
テンプレート（Template）
→ リソースを定義する YAML / JSON ファイル

スタック（Stack）
→ テンプレートからデプロイされたリソースの集合
→ スタック単位で作成・更新・削除できる

チェンジセット（Change Set）
→ 更新前に「何が変わるか」を事前確認できる機能
→ 本番環境での意図しない変更を防ぐ

ドリフト検出（Drift Detection）
→ テンプレートと実際のリソースの差分を検出
→ 手動操作で変更されていないかチェックできる
```

### StackSets

**複数のAWSアカウント・リージョンに同じスタックを一括デプロイ**できる機能。

```
StackSets
    ↓ 一括デプロイ
├── アカウントA / ap-northeast-1
├── アカウントB / us-east-1
└── アカウントC / eu-west-1
→ マルチアカウント構成の企業での一括管理に有効
```

---

## AWS SAM（Serverless Application Model）

**CloudFormationの拡張**で、サーバーレスリソース（Lambda・API Gateway・DynamoDB）をより簡潔に定義できる。

```yaml
# SAM テンプレートの例
Transform: AWS::Serverless-2016-10-31  # ← SAMのおまじない

Resources:
  MyFunction:
    Type: AWS::Serverless::Function   # ← SAM独自の型（CloudFormationより簡潔）
    Properties:
      Handler: lambda_function.handler
      Runtime: python3.12
      Events:
        S3Event:
          Type: S3
          Properties:
            Bucket: !Ref MyBucket
            Events: s3:ObjectCreated:*
```

CloudFormationで書くと長くなるLambdaの定義が、SAMなら簡潔に書ける。

### サーバーレスバックエンドの構築

SAMの代表的なユースケースは **API Gateway + Lambda でサーバーレスなバックエンドAPIを提供**すること。
EC2やECSでサーバーを常時稼働させる代わりに、リクエストが来たときだけLambdaを起動する。

```
クライアント（ブラウザ・スマホアプリ）
        ↓ HTTPリクエスト
API Gateway（エンドポイントを提供）
        ↓ ルーティング
Lambda（ビジネスロジックを実行）
        ↓
DynamoDB / RDS / S3 など
```

```
【従来のバックエンド（EC2・ECS）】
サーバーが常時稼働 → リクエストがなくても課金され続ける

【SAM + Lambda + API Gateway】
リクエストが来たときだけ Lambda が起動
→ アイドル時の課金ゼロ・スケールアウトは自動
```

SAMではイベントに `Type: Api` を指定するだけでAPI Gatewayが自動で作られる。

```yaml
Transform: AWS::Serverless-2016-10-31

Resources:
  GetUsersFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/users.getAll
      Runtime: nodejs24.x
      Events:
        GetUsers:
          Type: Api          # ← これだけでAPI Gatewayが自動で作られる
          Properties:
            Path: /users
            Method: GET

  CreateUserFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/users.create
      Runtime: nodejs24.x
      Events:
        CreateUser:
          Type: Api
          Properties:
            Path: /users
            Method: POST
```

```
sam deploy するだけで自動で作られるもの：
├── API Gateway（https://xxxxxx.execute-api.ap-northeast-1.amazonaws.com/）
│   ├── GET  /users  → GetUsersFunction を呼び出す
│   └── POST /users  → CreateUserFunction を呼び出す
└── Lambda 関数 × 2
```

### SAM CLI

**ローカルでLambdaを実行・テストできるCLIツール**。

```bash
sam local invoke MyFunction   # ローカルでLambda関数を実行
sam local start-api           # API Gatewayをローカルでエミュレート
sam deploy                    # AWSにデプロイ
```

デプロイ時は内部でCloudFormationに変換されて実行される。

```
SAM テンプレート
    ↓ sam deploy
CloudFormation テンプレートに自動変換
    ↓
AWSにデプロイ
```

### SAM で使える主なランタイム（2026年6月時点）

```
Python    3.9 / 3.10 / 3.11 / 3.12 / 3.13
Node.js   20.x / 22.x / 24.x      ← TypeScript もここで動く（18.x はEOL済み）
Java      8 / 11 / 17 / 21
.NET      6 / 8
Ruby      3.2 / 3.3
```

**Node.js 24 の注意点（2025年11月追加）：**

コールバック形式のハンドラーが非対応になった。async/await に統一する必要がある。

```javascript
// ❌ Node.js 24 では使えない（古いコールバック形式）
exports.handler = (event, context, callback) => {
    callback(null, 'response')
}

// ✅ async/await を使う（TypeScript なら問題なし）
exports.handler = async (event) => {
    return { statusCode: 200, body: 'response' }
}
```

### TypeScript（Node.js）でのデプロイ

TypeScript はそのままでは Lambda で動かない。JavaScriptにコンパイルしてからデプロイする。

```yaml
MyFunction:
  Type: AWS::Serverless::Function
  Metadata:
    BuildMethod: esbuild          # sam build 時に自動コンパイル
    BuildProperties:
      Minify: true
      Target: "es2020"
      EntryPoints:
        - src/handler.ts
  Properties:
    Runtime: nodejs24.x
    Handler: dist/handler.handler
```

```bash
sam build    # TypeScript → JavaScript に自動コンパイル
sam deploy   # AWSにデプロイ
```

---

## AWS CDK（Cloud Development Kit）

**プログラミング言語でインフラを定義**できるフレームワーク。
Python・TypeScript・Java・Go などで書いたコードが CloudFormation に変換される。

```python
# CDK（Python）でS3バケットとLambdaを定義する例
from aws_cdk import Stack, aws_s3 as s3, aws_lambda as lambda_

class MyStack(Stack):
    def __init__(self, scope, id, **kwargs):
        super().__init__(scope, id, **kwargs)

        bucket = s3.Bucket(self, "MyBucket")

        fn = lambda_.Function(
            self, "MyFunction",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="lambda_function.handler",
            code=lambda_.Code.from_asset("./lambda")
        )
```

### Construct（コンストラクト）

CDKの基本単位。3つのレベルがある。

| レベル | 説明 | 例 |
|-------|------|-----|
| **L1（Cfn系）** | CloudFormationリソースをそのままラップ | `CfnBucket` |
| **L2** | よく使う設定がデフォルトで設定済み | `s3.Bucket`（暗号化・バージョニングなど自動設定） |
| **L3（Patterns）** | 複数リソースをまとめたパターン | `LambdaRestApi`（Lambda + API Gatewayをまとめて定義） |

```
CDK コード
    ↓ cdk synth
CloudFormation テンプレートに変換
    ↓ cdk deploy
AWSにデプロイ
```

---

## 4つのIaCツール比較

| | CloudFormation | SAM | CDK | Terraform |
|--|--|--|--|--|
| 提供元 | AWS | AWS | AWS | HashiCorp |
| 記述形式 | YAML / JSON | YAML / JSON | Python・TypeScript等 | HCL |
| 対応クラウド | AWSのみ | AWSのみ | AWSのみ | **マルチクラウド** |
| サーバーレス特化 | ❌ | ✅ | ❌ | ❌ |
| ローカルテスト | ❌ | ✅（SAM CLI） | △ | △ |
| 学習コスト | 中 | 低（CloudFormation知識があれば） | 低（コードが書ければ） | 低〜中 |
| 状態管理 | CloudFormationが管理 | CloudFormationが管理 | CloudFormationが管理 | tfstateファイル |

```
「AWSだけ・サーバーレス中心」          → SAM
「AWSだけ・コードで書きたい」          → CDK
「AWSだけ・YAMLで書きたい」            → CloudFormation
「マルチクラウド・既存チームの標準」    → Terraform
```

---

## 試験のポイント

- **CloudFormation** → AWSネイティブのIaC。JSON/YAML でリソースを定義
- **StackSets** → 複数アカウント・リージョンに一括デプロイ
- **Change Set** → 更新前に変更内容を事前確認
- **Drift Detection** → テンプレートと実際のリソースの差分を検出
- **SAM** → CloudFormationの拡張。Lambda・API Gatewayを簡潔に定義。SAM CLIでローカルテスト可能
- **CDK** → プログラミング言語でインフラ定義。内部でCloudFormationに変換される
- **Terraform** → マルチクラウド対応。AWSの試験ではCloudFormation・SAM・CDKが中心
