# GUILE AWS リソース現状整理

> 調査日: 2026-05-15
> AWSアカウント: `558631582515`
> リージョン: `ap-northeast-1`（東京）
> 使用ロール: `smv_admin`

---

## 目的

GUILE サービスの縮小に向けたコスト削減検討のため、現状の AWS リソースを整理する。

---

## サービス概要

このAWSアカウントには SonicMoov が運営する複数のサービスが同居している。

| サービス名 | 広告フォーマット | 主なリソース | 稼働状況 |
|---|---|---|---|
| **GUILE** | インタースティシャル広告（静止画・HTML、フルスクリーン型）。`is-elb` の "is" は Interstitial の略 | guile-interstitial, is-elb, cdn-guile など | ✅ 稼働中 |
| **admovie** | 動画広告配信。GUILE の動画広告機能として組み込まれている可能性あり（要確認） | admovie-elb01/02, admovie-sonicmoov-elb01/02, ElastiCache(admovie) など | ✅ **稼働中**（週間リクエスト約2,500万件・2026-05-18 実測） |

> ⚠️ admovie が GUILE の一部か独立サービスかは未確認。Backlog Wiki などで確認が必要。

### GUILE のアーキテクチャ概要

```
ユーザー（スマートフォンアプリ等）
    ↓ 広告リクエスト
is-elb（ロードバランサー）
    ↓
guile-interstitial（配信サーバー）
  → 広告クリエイティブを返す
  → S3(cdn-guile) / CloudFront 経由で素材配信

    ↓ インプレッション・クリックログ
collector1（収集サーバー）
  → RDS(guile) または ElastiCache に記録
  → admin で集計レポート確認
```

> ✅ admovie は廃止されていない。2026-05-18 実測で admovie-sonicmoov-elb01/02 に週計約2,500万件のリクエストを確認。

---

## 参考資料

| 資料 | URL |
|---|---|
| Backlog Wiki | https://sonicmoov.backlog.jp/wiki/GUILE/Home |
| 技術関連資料（Google Drive） | https://drive.google.com/drive/folders/1f-a-PyCac1-WeQxheAMIU0omjwTI1t5_ |
| インフラ構成図（Google Drive） | https://drive.google.com/drive/folders/1P_W1a4W8Riqiur6f2t756SUuOb4-ZbAW |

---

## ネットワーク

### VPC

| VPC ID | 名前 | CIDR | 用途 |
|---|---|---|---|
| `vpc-4dadb02f` | SMV | `10.0.0.0/16` | 本番メイン |
| `vpc-0bfaf9bb4aec6a3d1` | SMV_staging | `10.1.0.0/16` | ステージング |
| `vpc-d200f6ba` | （デフォルト） | `172.31.0.0/16` | デフォルト VPC |

#### SMV 本番（vpc-4dadb02f）
全ての現役リソースが集まるメイン VPC。
- EC2（稼働中）: guile-interstitial, admin, collector1, staging, development, NAT-PRX01 等
- ELB: is-elb, admovie-elb01/02, mypage-elb, dev-elb など 8台
- 停止中の古い EC2 も多数残存

#### SMV_staging（vpc-0bfaf9bb4aec6a3d1）
GUILE ステージング環境専用 VPC。リソースは最小限。
- EC2: stg-guile-interstitial（stopped）
- ELB: stg-is-elb のみ

#### デフォルト VPC（vpc-d200f6ba）
AWS がアカウント作成時に自動生成する VPC。現在は停止中 EC2 が残るのみ。
- EC2: DEMO, content, ruizo本番, operation.temporary（全て stopped）
- RDS `guile`（MySQL）のサブネットグループがここに紐づいている

> ⚠️ 本番 DB（guile）がデフォルト VPC に配置されているのはセキュリティ上好ましくない設計。本来は SMV 本番 VPC の Private サブネットに置くべき。

### サブネット（SMV 本番 VPC）

Public / Protected / Private の違いはルートテーブルの設定による：
- **Public**: `0.0.0.0/0 → Internet Gateway`（インターネットに直接出られる）
- **Protected**: `0.0.0.0/0 → NAT-PRX01`（NAT 経由でインターネットに出られる）
- **Private**: インターネットへの経路なし（VPC 内のみ）

| CIDR | 種別 | AZ | リソース状況 |
|---|---|---|---|
| `10.0.0.0/24` | Public | ap-northeast-1a | ✅ 使用中（admin, guile-interstitial 等） |
| `10.0.1.0/24` | Public | ap-northeast-1c | ⚠️ NAT-PRX02（stopped）のみ |
| `10.0.32.0/20` | Public | ap-northeast-1a | ✅ 使用中（ELB 複数）|
| `10.0.48.0/20` | Public | ap-northeast-1c | ✅ 使用中（ELB 複数）|
| `10.0.64.0/19` | Protected | ap-northeast-1a | ✅ 使用中（collector1, staging 等）|
| `10.0.96.0/19` | Protected | ap-northeast-1c | ✅ 使用中（ElastiCache 等）|
| `10.0.10.0/24` | Protected | ap-northeast-1a | ❌ **空** |
| `10.0.254.0/24` | Protected | ap-northeast-1a | ❌ **空** |
| `10.0.255.0/24` | Protected | ap-northeast-1c | ❌ **空** |
| `10.0.160.0/19` | Private | ap-northeast-1a | ❌ **空** |
| `10.0.192.0/19` | Private | ap-northeast-1c | ❌ **空** |

> ⚠️ 空のサブネット5つは全てルートテーブル・ACL に紐づいているが、リソースは何もない。サブネット自体は無料だが、インフラ整理の観点で削除を検討できる。ただし削除はコスト削減に直結しないため優先度は低い。

### ルートテーブル（SMV 本番 VPC）

| 名前 | 0.0.0.0/0 の向き先 | 説明 |
|---|---|---|
| PUBLIC | `igw-88afbbea`（Internet Gateway） | インターネットに直接出られる |
| PROTECTED | `i-bf6ed0b9`（**NAT-PRX01 EC2**） | EC2 で自前 NAT を構築 |
| PRIVATE | なし | インターネット経路なし |

> ⚠️ PROTECTED サブネットは AWS マネージドの NAT Gateway ではなく、**NAT-PRX01（t1.micro）という EC2 で NAT を自前構築**している。NAT-PRX01 が停止すると PROTECTED サブネット内の全 EC2 がインターネットに出られなくなる。t1.micro は非常に古いインスタンスタイプ。

### ルートテーブル（SMV 本番 VPC）

| 名前 | 用途 |
|---|---|
| PUBLIC | インターネット向け |
| PRIVATE | 内部通信 |
| PROTECTED | 制限付き |

### セキュリティグループ（主要）

| グループ名 | VPC | 用途 |
|---|---|---|
| `guile-interstitial` | SMV 本番 | GUILE インタースティシャル |
| `jswf-SGDefaultELB-*` | SMV 本番 | ELB 用 |
| `jswf-SGDefaultAPP-*` | SMV 本番 | アプリ用 |
| `jswf-SGDefaultDB-*` | SMV 本番 | DB 用 |
| `jswf-SGNATPRX-*` | SMV 本番 | NAT プロキシ用 |
| `staging_app_sg` | SMV_staging | ステージング アプリ用 |

---

## コンピューティング

### EC2 インスタンス（25台）

#### 稼働中（8台）- 構成図と一致

| インスタンスID | 名前 | タイプ | Elastic IP | 推測用途 |
|---|---|---|---|---|
| `i-07b0d607c001f22ec` | **guile-interstitial** | t2.micro | `52.198.184.69` | GUILE 広告配信サーバー（本番）。`is-elb` に紐づく中核サービス |
| `i-00502c06` | admin | m3.large | `54.199.222.231` | 管理画面・管理者向け操作 UI。比較的スペックが高い |
| `i-1d305eb8` | development | t2.micro | `52.69.217.85` | 開発者用の検証・開発環境 |
| `i-e0a8cc7f` | staging | t2.micro | なし | ステージング環境のアプリケーションサーバー |
| `i-a8c2d20d` | collector1 | m3.medium | なし | 広告インプレッション・クリックデータの収集処理 |
| `i-bf6ed0b9` | NAT-PRX01 | t1.micro | `54.199.164.45` | PROTECTED サブネット全体の NAT（インターネット出口）。停止すると collector1 等が外部通信不能になる SPOF |
| `i-06223ad4c5b8d7b32` | （名前なし） | c3.large | なし | 名前・EIP なし。処理能力から広告配信や動画処理のワーカー的役割か（admovie 関連の可能性） |
| `i-07de3e194ab6a27e5` | （名前なし） | c3.large | なし | 同上 |

#### 停止中（15台）

構成図に記載あり：

| インスタンスID | 名前 | タイプ | Elastic IP | 推測用途 |
|---|---|---|---|---|
| `i-044737b9275526a22` | stg-guile-interstitial | t2.micro | `18.177.50.23` | GUILE 広告配信サーバー（ステージング）。`stg-is-elb` に対応。構成図 typo: stg-guile-intertitonal |

構成図に記載なし（過去の遺物の可能性が高い）：

| インスタンスID | 名前 | タイプ | Elastic IP | 推測用途 |
|---|---|---|---|---|
| `i-c60c6f49` | collector2 | m3.medium | なし | collector1 の冗長機または旧バージョン |
| `i-92c87690` | DEMO | t1.micro | なし | デモ・PoC 環境 |
| `i-b3df02b6` | ruizo本番 | t1.micro | `54.238.160.220` ⚠️ | 「ruizo」という別サービスの本番環境（現在は廃止済みの可能性） |
| `i-e3a2cce6` | content | t1.micro | `54.250.175.220` ⚠️ | コンテンツ配信サーバー |
| `i-fe9c95f8` | NAT-PRX02 | t1.micro | なし | NAT-PRX01 の冗長機（現在は未使用） |
| `i-b7d95444` | ad.graasb.com | t2.medium | `54.65.13.167` ⚠️ | 旧広告プラットフォーム（graasb）の配信サーバー |
| `i-6b496372` | rssmerge | t2.micro | なし | RSS フィードの収集・マージ処理バッチ |
| `i-6e5d3977` | admovie-aggregate | c3.large | なし | admovie の集計バッチ処理 |
| `i-8ab05214` | transcoder | c3.xlarge | なし | 動画トランスコード処理（c3.xlarge の高スペックが根拠） |
| `i-0fee8824a4154da14` | operation.temporary | c5.xlarge | なし | 一時的な運用作業用（マイグレーション等） |
| `i-043fa0e92417c2a59` | （名前なし） | t2.micro | なし | 用途不明 |

> ⚠️ stopped かつ構成図に載っていない EC2 は削除候補の最有力。stopped インスタンスに Elastic IP がアタッチされているものはコスト削減候補（ruizo本番・content・ad.graasb.com）

---

## データベース

### RDS

| 識別子 | ステータス | エンジン |
|---|---|---|
| `guile` | ✅ available | MySQL |

### ElastiCache（Redis）

| クラスター名 | エンジン | バージョン | ノードタイプ | 状態 |
|---|---|---|---|---|
| `admovie` | Redis | 6.2.6 | cache.m5.large | ✅ available |
| `admovie-003` | Redis | 6.2.6 | cache.m5.large | ✅ available |

> ※ クラスター名が `admovie` のため、GUILE 用ではなく admovie サービス用の可能性あり。S3 に `guile-redis-backup` バケットが存在するため別途確認が必要。

---

## ロードバランサー（ELB）

AWS CLI で実測した実際の振り先インスタンス：

| 名前 | VPC | 振り先インスタンス | AZ | 備考 |
|---|---|---|---|---|
| `is-elb` | SMV 本番 | i-07b0d607c001f22ec (guile-interstitial) | 1a + 1c | **GUILE 本番** |
| `stg-is-elb` | SMV_staging | -（未取得） | - | **GUILE ステージング** |
| `admovie-elb01` | SMV 本番 | i-06223ad4c5b8d7b32, i-07de3e194ab6a27e5 | 1a + 1c | admovie |
| `admovie-elb02` | SMV 本番 | i-06223ad4c5b8d7b32, i-07de3e194ab6a27e5 | 1a + 1c | admovie（elb01 と同一構成） |
| `admovie-sonicmoov-elb01` | SMV 本番 | i-06223ad4c5b8d7b32, i-07de3e194ab6a27e5 | 1a + 1c | admovie |
| `admovie-sonicmoov-elb02` | SMV 本番 | i-06223ad4c5b8d7b32, i-07de3e194ab6a27e5 | 1a + 1c | admovie（so-01 と同一構成） |
| `staging-admovie-sonicmoov-elb` | SMV 本番 | i-e0a8cc7f (staging) | 1a + 1c | admovie ステージング |
| `mypage-elb` | SMV 本番 | i-00502c06 (admin) | 1a + 1c | マイページ |
| `dev-elb` | SMV 本番 | i-1d305eb8 (development) | 1a + 1c | 開発 |

### ELB が多い理由の考察

admovie だけで4本 ELB がある。AWS CLI の実測で **admovie-elb01/02 および admovie-sonicmoov-elb01/02 はそれぞれペア内で全く同じインスタンスに振り分けている**ことが確認できた。

1. **ドメイン別に ELB を分けざるを得なかった（Classic ELB の制約）**
   - `admovie-elb01/02`：`admovie.jp` ドメイン向け（推測）
   - `admovie-sonicmoov-elb01/02`：`sonicmoov.com` ドメイン向け（推測）
   - Classic ELB は SSL 証明書を1枚しか設定できないため、ドメインごとに ELB を分ける必要があった

2. **01/02 は同一構成の ELB が2台並んでいる（冗長・実測確認済み）**
   - elb01 と elb02 は振り先インスタンス・AZ 設定が完全に一致している
   - DNS ラウンドロビンで振り分けていた可能性があるが、ELB 1台で同じことが実現できる
   - 「サーバーを2台並べて冗長化する」という発想を ELB にも誤って適用したと思われる

> 💡 現在の ALB（Application Load Balancer）であれば SNI により1台で複数ドメインの SSL 証明書を扱えるため、整理すれば ELB 台数を大幅削減できる。ただし admovie が稼働中かどうか不明なため、削除前に確認が必要。

---

## ストレージ

### S3 バケット（28件中 GUILE 関連）

| バケット名 | 用途 |
|---|---|
| `cdn-guile` | CDN（本番） |
| `dev-cdn-guile` | CDN（開発） |
| `stg-cdn-guile` | CDN（ステージング） |
| `test-cdn-guile` | CDN（テスト） |
| `dev-cdn-amoad-guile` | amoad CDN（開発） |
| `stg-cdn-amoad-guile` | amoad CDN（ステージング） |
| `dev-light-cdn.guile.jp` | ライト CDN（開発） |
| `origin.i.guile.jp` | オリジンサーバー |
| `origin.j.guile.jp` | オリジンサーバー |
| `dev.logs.guile` | ログ（開発） |
| `guile-redis-backup` | Redis バックアップ |
| `guile-ip-list-storage-destination` | IP リスト |

### EBS スナップショット（19件）

| スナップショット ID | サイズ(GB) | 作成日 |
|---|---|---|
| `snap-079243d9503c518a8` | 256 | 2021-03-15 |
| `snap-013563f2b459fc820` | 256 | 2020-11-12 |
| `snap-004956a3af3f39257` | 256 | 2020-08-04 |
| `snap-0a610b30b9d78c17b` | 20 | 2019-10-07 |
| `snap-0495a1cd84bf20e58` | 40 | 2018-04-01 |
| `snap-0167e4d80270755c2` | 256 | 2017-06-08 |
| その他 13件 | 8 | 2013〜2016年 |

---

## CDN

### CloudFront

| ディストリビューション ID | ドメイン | ステータス | オリジン |
|---|---|---|---|
| `EEX9EOGODLUAW` | `d2pszgce8p13si.cloudfront.net` | Deployed | `cdn-guile.s3.ap-northeast-1.amazonaws.com` |

---

## サーバーレス・API

### Lambda

| 関数名 | ランタイム | ログサイズ |
|---|---|---|
| `get-ip-address` | Node.js 12.x ⚠️ | 約4.6GB |

> ⚠️ Node.js 12.x は EOL（サポート終了）のためランタイム更新が必要

### API Gateway

- REST API: 1件

---

## 監視・運用

### CloudWatch Logs

| ロググループ | 保持期間 | サイズ |
|---|---|---|
| `/aws/lambda/get-ip-address` | **無制限** ⚠️ | **約4.6GB** |
| `/aws/lambda/akamaiFastPurge` | 無制限 | 285B |
| `/aws/lambda/get_ip_address` | 無制限 | 2KB |

> ⚠️ 保持期間が全て無制限。特に `get-ip-address` の 4.6GB は要対応。

### CloudFormation スタック

| スタック名 | ステータス | 作成日 | 用途 |
|---|---|---|---|
| `jswf` | CREATE_COMPLETE | 2014-03-03 | **GUILE 基盤ネットワーク全体を管理**（VPC・IGW・ルートテーブル・SG・NAT-PRX01/02 等 100リソース） |
| `StackSet-TechorusCloudPortalRole-*` | UPDATE_COMPLETE | 2024-09-03 | AWS 管理ツール系（GUILE 無関係） |
| `StackSet-NHNTechorusSupportIAMRole-*` | UPDATE_COMPLETE | 2024-09-03 | AWS 管理ツール系（GUILE 無関係） |

#### `jswf` スタックが管理する主なリソース

| リソース種別 | 論理ID | 内容 |
|---|---|---|
| Internet Gateway | `InternetGateway` | SMV 本番 VPC のインターネット出口 |
| ルートテーブル | `RTBPublic` / `RTBProtected` / `RTBPrivate` | サブネットのルーティング |
| セキュリティグループ | `SGDefaultAPP` / `SGDefaultDB` / `SGAllowAll` | EC2・DB のアクセス制御 |
| EC2 インスタンス | `NATPRX01` / `NATPRX02` | PROTECTED サブネットの NAT |
| EIP | `EIPNAT` | NAT 用 Elastic IP |

> ⚠️ `jswf` スタックが管理しているリソースを手動で削除・変更すると CloudFormation との整合性が崩れるため注意。GUILE インフラを縮小する際はスタックごと削除するか、スタックから切り離してから削除する必要がある。

### その他

| サービス | 内容 |
|---|---|
| ACM | SSL 証明書 6件 |
| CloudWatch | アラーム 60件 |
| Athena | ワークグループ 1件 |
| MemoryDB | Redis 互換 DB |
| SNS | トピック 4件 |
| KMS | 暗号化キー 4件 |
| App Runner | 自動スケーリング設定 1件 |
| X-Ray | サンプリングルール 1件 |

---

## コスト削減候補

| 優先度 | 対象 | 内容 |
|---|---|---|
| 高 | stopped EC2（多数） | 不要なインスタンスの終了を検討 |
| 高 | stopped EC2 の EIP | ruizo本番・content・ad.graasb.com の EIP 解放 |
| 中 | admovie 系 ELB（4本） | elb01/02・so-elb01/02 は同一構成のため各1台に統合可能。ただし admovie が稼働中かどうか要確認（CloudWatch メトリクスでトラフィック確認） |
| 中 | スナップショット | 2013〜2018年の古いスナップショット削除検討 |
| 中 | CloudWatch Logs | `get-ip-address` ログの保持期間設定（現在 4.6GB・無制限） |
| 中 | Lambda ランタイム | `get-ip-address` を Node.js 12.x（EOL）から更新 |

### ELB 削減による見積もりコスト削減額

Classic ELB は稼働しているだけで時間課金（約 $0.027/時間 ≒ **$19.4/月/台**）。

| 削減シナリオ | 月額削減見込み |
|---|---|
| `admovie-elb01/02` → 1台に統合 | 約 $19 |
| `admovie-sonicmoov-elb01/02` → 1台に統合 | 約 $19 |
| **合計** | **約 $38 / 月** |

> ⚠️ 削減前に CloudWatch メトリクスで admovie 系 ELB へのトラフィックが現在もあるか確認すること。トラフィックがなければ admovie サービス自体が廃止済みの可能性があり、ELB ごと削除できる。

---

## コスト確認

`smv_admin` ロールでは Cost Explorer へのアクセスが組織の SCP（サービスコントロールポリシー）により拒否されている。  
**AWS 管理者にアカウント `558631582515` のサービス別コストレポートの共有を依頼すること。**

