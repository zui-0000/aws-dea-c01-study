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
| ~~Snowcone~~ | ❌ 2024年11月終了 | 廃止済み |
| ~~Snowmobile~~ | ❌ 2025年に廃止 | 廃止済み |

> **注意:** 2025年11月7日以降、Snowball Edgeは新規顧客への提供が終了し、**既存顧客のみ**利用可能。
> 新規顧客はAWS DataSync・AWS Data Transfer Terminalの利用が推奨されている。

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

### Snowcone（2024年11月廃止）
- 最小・最軽量のSnowデバイス（8TB HDD / 14TB SSD）
- 辺境地・IoT用途向けだったが廃止に

### Snowmobile（2025年廃止）
- 45フィートのトレーラートラック
- 最大100PBの超大規模転送向けだったが廃止に
- 代替: Snowball Edgeを複数台使う

---

## 試験のポイント

- **大量データをオフラインで転送** → Snowball Edge
- **ネットワーク転送が現実的でない規模** → Snowball Edge
- **Snowcone・Snowmobileは廃止済み** → 試験では古い問題に注意
- **新規顧客はDataSyncを推奨** → AWSの公式推奨の方向性
- **エッジコンピューティング** → Snowball Edge Compute Optimized
