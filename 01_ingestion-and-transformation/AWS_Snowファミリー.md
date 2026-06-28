# <img src="./images/aws-snowball-edge.png" width="36"/>&nbsp;&nbsp; AWS Snow ファミリー

## 概要

インターネット経由では転送が困難な**大量データを物理デバイスでAWSに転送**するサービス群。
ネットワーク帯域が不十分な環境や、エクサバイト規模のデータ移行で使われる。

---

## 2026年6月時点のサービス状況

| サービス | 状況 | 備考 |
|---------|------|------|
| **Snowball Edge Storage Optimized** | ✅ アクティブ（既存顧客のみ） | 210TB NVMe |
| **Snowball Edge Compute Optimized** | ✅ アクティブ（既存顧客のみ） | 104 vCPU・28TB NVMe |
| ~~Snowcone~~ | ❌ 2024年11月12日に販売終了 | 既存顧客サポートは2025年11月12日まで |
| ~~Snowmobile~~ | ❌ 2024年4月に提供停止を表明 | ウェブサイトからの削除は2024年3月 |

> **注意:** Snowball Edgeは**新規顧客には提供終了済み**（2025年11月7日以降、既存顧客のみ利用可能）。
> 新規顧客はAWS DataSync・AWS Data Transfer Terminalの利用が推奨されている。
> エッジコンピューティング用途の代替は **AWS Outposts**。

> **旧世代デバイスの販売終了:** Snowball Edge Storage Optimized 80TB、Compute Optimized 52 vCPU、Compute Optimized with GPU の3モデルは **2024年11月12日に販売終了**。

---

## AWS Snowball Edge（現役サービス）

### Storage Optimized（210TB）
```
用途: 大量データの移行
容量: 210TB NVMe
転送速度: 最大1.5GB/秒
向いているケース: データセンター移行・大規模バックアップ
```

### Compute Optimized（104 vCPU）
```
用途: エッジコンピューティング
容量: 28TB NVMe（フルSSD）
vCPU: 104
向いているケース: ネットワークが届かない現場での処理
```

---

## なぜ物理デバイスを使うのか

```
【ネットワーク転送（DataSync）の限界】
100TB のデータを 1Gbps 回線で転送すると…
  100TB ÷ 1Gbps ≈ 約9日かかる

【Snowball Edgeなら】
物理デバイスをFedExで送るだけ → 数日で完了
コストも大幅に安い
```

ネットワーク転送より**物理的に運んだほうが速い・安い**ケースに使う。

---

## DataSync との使い分け

| 観点 | AWS Snowball Edge | AWS DataSync |
|------|-----------------|-------------|
| 転送方法 | 物理デバイスを郵送 | ネットワーク経由 |
| 向いているデータ量 | 数十TB〜PB規模 | 少量〜中量 |
| ネットワーク環境 | 不要（または劣悪でもOK） | 必要 |
| 転送速度 | 郵送リードタイム依存 | 帯域幅依存 |
| コスト | デバイスレンタル料金 | 転送GB単価 |

```
ネットワークが使える・データ量が少ない → DataSync
ネットワーク不十分・大量データ         → Snowball Edge
```

---

## 廃止されたサービス（参考）

### Snowcone（2024年11月12日に販売終了）
- 最小・最軽量のSnowデバイス（8TB HDD / 14TB SSD）
- 辺境地・IoT用途向けだった
- 2024年11月12日に販売終了（新規・既存とも注文不可）。既存顧客のサポートは2025年11月12日まで

### Snowmobile（2024年4月に提供停止を表明）
- 45フィートのトレーラートラック
- 最大100PBの超大規模転送向けだった
- 2024年4月にAWSが提供停止を表明（ウェブサイトからの削除は2024年3月）
- 代替: Snowball Edgeを複数台使う

---

## 試験のポイント

- **大量データをオフラインで転送** → Snowball Edge
- **ネットワーク転送が現実的でない規模** → Snowball Edge
- **Snowmobile** → 2024年4月に提供停止を表明（ウェブサイト削除は2024年3月）
- **Snowcone** → 2024年11月12日に販売終了（既存顧客サポートは2025年11月12日まで）
- **Snowball Edge** → 新規顧客には提供終了済み（2025年11月7日以降、既存顧客のみ）
- **新規顧客の代替** → オンライン転送はDataSync、物理転送はData Transfer Terminal、エッジコンピューティングはAWS Outposts
- **エッジコンピューティング** → Snowball Edge Compute Optimized（新規はAWS Outposts）
