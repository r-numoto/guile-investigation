# GUILE AWS リソース現状整理

> 調査日: 2026-05-18  
> AWSアカウント: `558631582515` / リージョン: `ap-northeast-1`（東京）/ 使用ロール: `smv_admin`

---

## 目的

GUILE サービスのコスト削減のため、現状の AWS リソースを整理し、削減可能なリソースを特定する。  
**機能の廃止は行わない。admovie は GUILE の主要機能として維持する。**

---

## サービス概要

このAWSアカウントには SonicMoov の2つのサービスが**同一インフラ上に同居**している。

| サービス | 内容 | 稼働状況 |
|---|---|---|
| **GUILE** | インタースティシャル広告配信（静止画・HTML・動画）。admovie は GUILE の動画広告機能 | ✅ 稼働中 |
| **JswfPlayer** | Flash（SWF）→ JavaScript 変換プレイヤー配信 SaaS。**GUILE とは独立したサービス** | ⚠️ **廃止予定** |

### サービス間の依存関係

GUILE は JswfPlayer のインフラを間借りしており、**JswfPlayer 廃止前に移行が必要**。

```
JswfPlayer（2014年〜）
  ├─ jswf CloudFormation スタック → VPC・IGW・ルートテーブル・SG を管理
  │      ↑ GUILE の全 EC2・RDS・ELB がこの VPC 上で動作
  ├─ cdn.jswfplayer.jp（S3）→ admovie の JS・SWF・素材を配信
  │      ↑ guile-frontend-* の本番配信パスが cdn.jswfplayer.jp/admovie/ を参照
  └─ admovie.jswfplayer.jp → admovie のクリック・再生カウント API
```

**根拠（ソースコードより）**
- `guile-frontend-microad/Gruntfile.js`: `BASE_PATH_DEST = '//cdn.jswfplayer.jp/admovie/'`
- `guile-deploy-config/roles/jswf-admovie`: admovie デプロイロール
- `guile-deploy-config/jswf-admovie.prd.json`: Packer AMI 名 `jswf-admovie-{{timestamp}}`

### GUILE アーキテクチャ

```
スマートフォンアプリ
  ↓ 広告リクエスト
is-elb → guile-interstitial → S3(cdn-guile) / CloudFront で素材配信
  ↓ ログ
collector1 → RDS(guile)
  ↓ 動画広告リクエスト
admovie-sonicmoov-elb01/02 → admovie-server01/02 → cdn.jswfplayer.jp/admovie/
```

---

## ⚠️ jswf 廃止前に GUILE 側で必要な移行作業

> コスト削減とは別の話。JswfPlayer 廃止スケジュールに合わせた**必須対応**。

| # | 依存内容 | jswf 廃止で起きること | GUILE 側の対応（廃止前に完了必須） |
|---|---|---|---|
| ① | `jswf` CloudFormation スタック（VPC） | GUILE の全 EC2・RDS・ELB が停止 | GUILE 専用 VPC/スタックへ移行 |
| ② | `cdn.jswfplayer.jp` S3 バケット | admovie 広告が全停止 | admovie コンテンツを `cdn-guile` 等へ移行し配信 URL を変更 |
| ③ | `admovie.jswfplayer.jp` API | admovie トラッキング不能 | admovie サーバーを独自ドメインで運用するよう変更 |

**移行順序**
1. admovie CDN を `cdn-guile`（または新バケット）へ移行、guile-frontend-* の URL を変更
2. admovie API ドメインを独自ドメインに切り替え
3. GUILE の EC2・RDS・ELB を新 VPC へ移行
4. 移行完了後、jswf スタックを削除

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

| VPC | 名前 | CIDR | 用途 |
|---|---|---|---|
| `vpc-4dadb02f` | SMV（本番） | `10.0.0.0/16` | GUILE・admovie 本番環境 |
| `vpc-0bfaf9bb4aec6a3d1` | SMV_staging | `10.1.0.0/16` | GUILE ステージング環境 |
| `vpc-d200f6ba` | デフォルト | `172.31.0.0/16` | 停止中 EC2 のみ。RDS `guile` のサブネットがここに紐づく ⚠️ |

> ⚠️ 本番 DB（RDS `guile`）がデフォルト VPC に配置されているのはセキュリティ上好ましくない。本来は SMV 本番 VPC の Private サブネットに置くべき。

### サブネット（SMV 本番 VPC）

| CIDR | 種別 | AZ | 状況 |
|---|---|---|---|
| `10.0.0.0/24` | Public | 1a | ✅ 使用中（admin, guile-interstitial 等） |
| `10.0.1.0/24` | Public | 1c | ⚠️ NAT-PRX02（stopped）のみ |
| `10.0.32.0/20` | Public | 1a | ✅ 使用中（ELB 複数） |
| `10.0.48.0/20` | Public | 1c | ✅ 使用中（ELB 複数） |
| `10.0.64.0/19` | Protected | 1a | ✅ 使用中（collector1, staging 等） |
| `10.0.96.0/19` | Protected | 1c | ✅ 使用中（ElastiCache 等） |
| `10.0.10.0/24` | Protected | 1a | ❌ 空 |
| `10.0.254.0/24` | Protected | 1a | ❌ 空 |
| `10.0.255.0/24` | Protected | 1c | ❌ 空 |
| `10.0.160.0/19` | Private | 1a | ❌ 空 |
| `10.0.192.0/19` | Private | 1c | ❌ 空 |

### ルートテーブル（SMV 本番 VPC）

| 名前 | デフォルトルート | 説明 |
|---|---|---|
| PUBLIC | `igw-88afbbea`（Internet Gateway） | インターネット直接接続 |
| PROTECTED | `i-bf6ed0b9`（NAT-PRX01 EC2） | EC2 による自前 NAT |
| PRIVATE | なし | VPC 内のみ |

> ⚠️ PROTECTED サブネットは AWS マネージド NAT Gateway ではなく、**t1.micro EC2（NAT-PRX01）で自前 NAT** を構築。この EC2 が停止すると PROTECTED サブネット内の全 EC2 がインターネット通信不能になる（SPOF）。

### セキュリティグループ（主要）

| グループ名 | 用途 |
|---|---|
| `guile-interstitial` | GUILE インタースティシャル |
| `jswf-SGDefaultELB-*` | ELB 用（jswf スタック管理） |
| `jswf-SGDefaultAPP-*` | アプリ用（jswf スタック管理） |
| `jswf-SGDefaultDB-*` | DB 用（jswf スタック管理） |
| `jswf-SGNATPRX-*` | NAT プロキシ用（jswf スタック管理） |
| `staging_app_sg` | ステージング アプリ用 |

---

## EC2 インスタンス

### 稼働中（8台）

| インスタンスID | 名前 | タイプ | EIP | 用途 |
|---|---|---|---|---|
| `i-07b0d607c001f22ec` | guile-interstitial | t2.micro | `52.198.184.69` | GUILE インタースティシャル広告配信（本番）。is-elb に紐づく |
| `i-00502c06` | admin | m3.large | `54.199.222.231` | 管理画面サーバー |
| `i-a8c2d20d` | collector1 | m3.medium | なし | 広告インプレッション・クリックデータ収集 |
| `i-e0a8cc7f` | staging | t2.micro | なし | ステージング環境 |
| `i-1d305eb8` | development | t2.micro | `52.69.217.85` | 開発環境 |
| `i-bf6ed0b9` | NAT-PRX01 | t1.micro | `54.199.164.45` | PROTECTED サブネットの NAT（SPOF）。jswf スタック管理 |
| `i-06223ad4c5b8d7b32` | admovie-server（1a） | c3.large | なし | admovie 動画広告配信（本番 1a）。admovie-elb01/02 および so-elb01/02 の振り先 |
| `i-07de3e194ab6a27e5` | admovie-server（1c） | c3.large | なし | admovie 動画広告配信（本番 1c）。同上 |

### 停止中（17台）

#### 構成図に記載あり（ステージング）

| インスタンスID | 名前 | タイプ | EIP | 用途 |
|---|---|---|---|---|
| `i-044737b9275526a22` | stg-guile-interstitial | t2.micro | `18.177.50.23` | GUILE ステージング。stg-is-elb に対応 |

#### 構成図に記載なし（遺物の可能性が高い・削除候補）

| インスタンスID | 名前 | タイプ | EIP | 推測用途 |
|---|---|---|---|---|
| `i-c60c6f49` | collector2 | m3.medium | なし | collector1 の旧冗長機 |
| `i-92c87690` | DEMO | t1.micro | なし | デモ環境 |
| `i-b3df02b6` | ruizo本番 | t1.micro | `54.238.160.220` ⚠️ | 旧サービス |
| `i-e3a2cce6` | content | t1.micro | `54.250.175.220` ⚠️ | 旧コンテンツサーバー |
| `i-fe9c95f8` | NAT-PRX02 | t1.micro | なし | NAT の旧冗長機 |
| `i-b7d95444` | ad.graasb.com | t2.medium | `54.65.13.167` ⚠️ | 旧広告プラットフォーム |
| `i-6b496372` | rssmerge | t2.micro | なし | RSS フィード処理 |
| `i-6e5d3977` | admovie-aggregate | c3.large | なし | admovie 集計バッチ |
| `i-8ab05214` | transcoder | c3.xlarge | なし | 旧動画トランスコード |
| `i-0fee8824a4154da14` | operation.temporary | c5.xlarge | なし | 一時運用作業用 |
| `i-dbd9d344` | （名前なし） | - | なし | 用途不明 |
| `i-ab5a4f34` | （名前なし） | - | なし | 用途不明 |
| `i-d75a4f48` | （名前なし） | - | なし | 用途不明 |
| `i-fea04370` | （名前なし） | - | なし | 用途不明 |
| `i-043fa0e92417c2a59` | （名前なし） | t2.micro | なし | 用途不明 |
| `i-635d397a` | （名前なし） | - | なし | 用途不明 |

> ⚠️ 停止中 EC2 に EIP がアタッチされたままのもの（ruizo本番・content・ad.graasb.com）は EIP 課金が発生している。

---

## EC2 SSH 接続ガイド

### 前提条件

1. **saml2aws でログイン済み**（トークンは約1時間で期限切れ）
   ```bash
   saml2aws login --profile smv-admin
   ```
2. **SSH 設定が適用済み**（`~/.ssh/config` に `Include ~/.ssh/ssh-guile/config` が追加されていること）
3. **接続元 IP が bastion の SG に許可されていること**（現在許可: `52.195.61.178/32`, `210.172.253.183/32`）

### SSH config ファイル

`ssh-guile` リポジトリの `config` に全ホスト定義が記載されている。  
鍵ファイルは2種類:
| 鍵ファイル | 対象ホスト |
|---|---|
| `ec2-user@bastion.pem` | NAT-PRX01（bastion）、guile-interstitial、stg-guile-interstitial |
| `sonicmoov@aws.pem` | admin、development、staging、collector1/2、admovie-server |

### 接続コマンド一覧

| ホスト | エイリアス | 接続方法 | 実コマンド |
|---|---|---|---|
| NAT-PRX01（bastion） | `guile.bastion` | 直接 | `ssh guile.bastion` |
| development | `guile.dev` | 直接 | `ssh guile.dev` |
| admin | `guile.prd-stg.backend` | 直接 | `ssh guile.prd-stg.backend` |
| guile-interstitial | `guile.interstitial` | 直接 | `ssh guile.interstitial` |
| staging | `guile.stg.frontend` | bastion 経由 | `ssh guile.stg.frontend` |
| collector1 | `guile.prd.fluentd01` | bastion 経由 | `ssh guile.prd.fluentd01` |
| admovie-server（1a） | ※未定義 | bastion 経由 | `ssh -o ProxyCommand='ssh -W %h:%p guile.bastion' -i ~/.ssh/ssh-guile/sonicmoov@aws.pem sonicmoov@10.0.78.28` |
| admovie-server（1c） | ※未定義 | bastion 経由 | `ssh -o ProxyCommand='ssh -W %h:%p guile.bastion' -i ~/.ssh/ssh-guile/sonicmoov@aws.pem sonicmoov@10.0.108.23` |

### 接続確認結果（2025-05-18）

| ホスト | 状態 | SSH 接続 | 備考 |
|---|---|---|---|
| NAT-PRX01（bastion） | 稼働中 | ✅ 成功 | |
| development | 稼働中 | ✅ 成功 | |
| admin | 稼働中 | ✅ 成功 | |
| guile-interstitial | 稼働中 | ✅ 成功 | |
| staging | 稼働中 | ✅ 成功 | bastion 経由 |
| collector1 | 稼働中 | ✅ 成功 | bastion 経由 |
| admovie-server × 2 | 稼働中 | 未確認 | SSH config に未定義のため要手動接続 |
| collector2 | **停止中** | ❌ 不可 | 起動すれば接続可 |
| stg-guile-interstitial | **停止中** | ❌ 不可 | 起動すれば接続可 |
| prd.front01〜04 | 存在しない | ❌ 不可 | SSH config の IP が古い（現在 AWS に該当インスタンスなし） |

> ⚠️ **SSH config の注意点**: bastion のSSHサーバーが古いホストキー（ssh-rsa）のみ提供するため、`config` 内の全ホスト定義に以下が必要（設定済み）:
> ```
> HostKeyAlgorithms +ssh-rsa
> PubkeyAcceptedAlgorithms +ssh-rsa
> ```

---

## データベース

### RDS

| 識別子 | エンジン | インスタンスタイプ | ストレージ | Multi-AZ | 状態 |
|---|---|---|---|---|---|
| `guile` | MySQL | db.t3.micro | 10GB（magnetic） | ✅ 有効 | available |

> 推定コスト: 約 **$40/月**（db.t3.micro Multi-AZ $39 + storage $1）

### ElastiCache（Redis）

| レプリケーショングループ | ノードタイプ | ノード数 | 状態 |
|---|---|---|---|
| `admovie` | cache.m5.large | 2（プライマリ + レプリカ） | available |

> 推定コスト: cache.m5.large × 2 = 約 **$225/月**  
> admovie 用。admovie は継続のため現時点では削除不可。

---

## ロードバランサー（ELB）

全 9 台、すべて Classic ELB（約 $19.4/月/台）。

| 名前 | VPC | 振り先インスタンス | 用途 |
|---|---|---|---|
| `is-elb` | SMV 本番 | guile-interstitial（1a+1c） | GUILE 本番 |
| `stg-is-elb` | SMV_staging | stg-guile-interstitial | GUILE ステージング |
| `admovie-elb01` | SMV 本番 | admovie-server 1a+1c | admovie（admovie.jp ドメイン向け推測） |
| `admovie-elb02` | SMV 本番 | admovie-server 1a+1c | admovie（elb01 と**完全に同一構成**） |
| `admovie-sonicmoov-elb01` | SMV 本番 | admovie-server 1a+1c | admovie（sonicmoov.com ドメイン向け推測） |
| `admovie-sonicmoov-elb02` | SMV 本番 | admovie-server 1a+1c | admovie（so-elb01 と**完全に同一構成**） |
| `staging-admovie-sonicmoov-elb` | SMV 本番 | staging | admovie ステージング |
| `mypage-elb` | SMV 本番 | admin | マイページ |
| `dev-elb` | SMV 本番 | development | 開発環境 |

**admovie 系 ELB が4本ある理由**
1. Classic ELB は SSL 証明書を1枚しか設定できないためドメインごとに ELB を分けた
2. elb01/02 の2台構成は ELB の冗長化として誤って設計されたと推測（ALB であれば SNI で1台にまとめられる）

**2026-05-18 実測トラフィック（CloudWatch）**

| ELB | 週間リクエスト数 |
|---|---|
| admovie-sonicmoov-elb01 | 約 12.4M |
| admovie-sonicmoov-elb02 | 約 12.4M |
| admovie-elb01 | 約 8,500 |
| admovie-elb02 | 約 24,500 |

---

## ストレージ

### S3 バケット（全 34 件）

| バケット名 | 用途 | サイズ（実測） |
|---|---|---|
| `cdn-guile` | GUILE CDN 本番（CloudFront オリジン） | **128.2 GB** |
| `cdn.jswfplayer.jp` | JswfPlayer CDN（admovie コンテンツも格納） | **15.0 GB** |
| `guile-redis-backup` | Redis バックアップ | 0.17 GB |
| `dev-cdn-guile` / `stg-cdn-guile` / `test-cdn-guile` | CDN（開発・ステージング・テスト） | - |
| `dev-cdn-amoad-guile` / `stg-cdn-amoad-guile` | amoad CDN（開発・ステージング） | - |
| `origin.i.guile.jp` / `origin.j.guile.jp` | オリジンサーバー | ≈ 0 |
| `jswf-admovie-cf-logs` 等 ログ系バケット | アクセスログ | - |
| `aws-athena-query-results-*` / `dev-athena-result*` | Athena 結果 | - |
| その他（`ec2-usagedata`, `send-guile-logs` 等） | 運用用 | - |

> 推定コスト: (128.2 + 15.0) GB × $0.025 ≒ **約 $4/月**

### EBS ボリューム（全 26 本、2026-05-18 実測）

全ボリューム in-use（インスタンスにアタッチ済み）。

**EBS 全体**

| タイプ | 合計 | 推定月額 |
|---|---|---|
| gp2 | 1,508 GB | 約 $181/月 |
| standard (magnetic) | 264 GB | 約 $21/月 |
| **合計** | **1,772 GB** | **約 $202/月** |

**停止中インスタンスにアタッチされているボリューム（削除候補）**

| インスタンス名 | サイズ | タイプ |
|---|---|---|
| transcoder | 40 GB | gp2 |
| operation.temporary | 128 GB | gp2 |
| stg-guile-interstitial | 20 GB | gp2 |
| （名前なし）×4 | 256+128+100+100 GB | gp2 |
| collector2 | 128 GB | gp2 |
| rssmerge | 8 GB | gp2 |
| （名前なし） | 8 GB | gp2 |
| admovie-aggregate | 100 GB | standard |
| NAT-PRX02 | 8 GB | standard |
| content / DEMO / ad.graasb.com / ruizo本番 / （名前なし） | 各 8 GB | standard |

> 停止中合計: gp2 916 GB + standard 148 GB → 削除で約 **$122/月** 削減

### EBS スナップショット（19 件）

| スナップショット | サイズ | 作成日 |
|---|---|---|
| 4 件 | 256 GB 各 | 2017〜2021年 |
| 2 件 | 40 GB, 20 GB | 2018, 2019年 |
| 13 件 | 8 GB 各 | 2013〜2016年 |

> 合計約 1,188 GB → 推定約 **$59/月**（$0.05/GB）

---

## CDN

### CloudFront

| ディストリビューション ID | オリジン | PriceClass | 月間転送量（実測） |
|---|---|---|---|
| `EEX9EOGODLUAW` | `cdn-guile` S3 | PriceClass_All | **約 73.5 GB** |

> 推定コスト: 約 **$6/月**  
> `PriceClass_All` は最高価格クラス。`PriceClass_200` 等に下げることで小幅削減が可能。

---

## サーバーレス・API

### Lambda

| 関数名 | ランタイム | 月間実行回数（実測） | 推定月額 |
|---|---|---|---|
| `get-ip-address` | Node.js 12.x ⚠️ | 約 168 万回 | 約 $0.34 |

> ⚠️ Node.js 12.x は EOL。ランタイム更新が必要（コスト削減ではなくセキュリティ対応）。

### API Gateway

- REST API: 1 件

---

## 監視・運用

### CloudWatch Logs

| ロググループ | 保持期間 | サイズ |
|---|---|---|
| `/aws/lambda/get-ip-address` | **無制限** ⚠️ | **約 4.6 GB** |
| `/aws/lambda/akamaiFastPurge` | 無制限 | 285 B |
| `/aws/lambda/get_ip_address` | 無制限 | 2 KB |

### CloudFormation スタック

| スタック名 | 作成日 | 内容 |
|---|---|---|
| `jswf` | 2014-03-03 | VPC・IGW・ルートテーブル・SG・NAT-PRX01/02 等 100 リソースを管理。JswfPlayer 由来のスタック。GUILE 全体がこの VPC 上で動作 |
| `StackSet-TechorusCloudPortalRole-*` | 2024-09-03 | AWS 管理ツール系（GUILE 無関係） |
| `StackSet-NHNTechorusSupportIAMRole-*` | 2024-09-03 | AWS 管理ツール系（GUILE 無関係） |

> ⚠️ `jswf` スタック管理リソースを手動で削除・変更すると CloudFormation との整合性が崩れる。削除はスタックごと、または事前にスタックから切り離してから実施すること。

### その他

| サービス | 内容 |
|---|---|
| ACM | SSL 証明書 6 件 |
| CloudWatch | アラーム 60 件 |
| MemoryDB | Redis 互換 DB |
| Athena | ワークグループ 1 件 |
| SNS | トピック 4 件（`jswfplayer-admovie-alert` など） |
| KMS | 暗号化キー 4 件 |
| App Runner | 自動スケーリング設定 1 件 |
| X-Ray | サンプリングルール 1 件 |

---

## コスト概算（2026-05-18 時点）

> Cost Explorer は SCP でブロック済み。以下は公開料金表に基づく**推定値**。

| サービス | 内訳 | 推定月額 |
|---|---|---|
| EC2（稼働中 8 台） | c3.large×2, m3.large×1, m3.medium×1, t2.micro×3, t1.micro×1 | 約 $519 |
| EBS（全 26 本） | gp2 1,508 GB + magnetic 264 GB | 約 $202 |
| ElastiCache | cache.m5.large × 2 ノード | 約 $225 |
| ELB（Classic × 9 台） | $0.027/時 × 9 | 約 $175 |
| EBS スナップショット | 1,188 GB × $0.05 | 約 $59 |
| RDS | db.t3.micro Multi-AZ + 10 GB | 約 $40 |
| CloudFront | 73.5 GB/月 | 約 $6 |
| S3 | 143 GB（主要バケット） | 約 $4 |
| Lambda | 168 万回/月 | 約 $0.34 |
| **合計** | | **約 $1,230/月** |

---

## コスト削減候補

**前提: admovie は GUILE の主要機能として継続。機能削減は行わない。**

| 優先度 | 対象 | 内容 | 削減見込み |
|---|---|---|---|
| 🔴 最高 | **停止中 EC2 の EBS 削除** | 停止中 17 台のボリューム削除（gp2 916 GB + standard 148 GB） | **約 $122/月** |
| 🟠 高 | **admovie ELB の統合** | elb01/02 → 1 台、so-elb01/02 → 1 台（同一構成の重複を解消） | **約 $38/月** |
| 🟠 高 | **停止中 EC2 の EIP 解放** | ruizo本番・content・ad.graasb.com の EIP 解放 | **約 $11/月** |
| 🟡 中 | **EBS スナップショット削除** | 2013〜2018 年の古いスナップショット（約 800 GB 相当）削除 | **約 $40/月** |
| 🟡 中 | **CloudFront PriceClass 変更** | `PriceClass_All` → `PriceClass_200` | 数 $ |
| 🟡 中 | **CloudWatch Logs 保持期間設定** | `get-ip-address` ログ（4.6 GB）に保持期間を設定 | 小 |
| 🟢 低 | **Lambda ランタイム更新** | Node.js 12.x（EOL）→ 最新版 | なし（セキュリティ対応） |
| 🟢 低 | **空サブネット整理** | 5 本の空サブネット削除 | なし（整理目的） |

### 削減シナリオ試算

| シナリオ | 内容 | 月額削減見込み |
|---|---|---|
| **今すぐできる整理** | 停止中 EBS 削除 + EIP 解放 + 古いスナップショット削除 | 約 **$173/月** |
| **ELB 統合** | admovie ELB 4本 → 2本 | 約 **$38/月** |
| **合計** | 上記すべて | 約 **$211/月**（現在の約 17% 削減） |

---

## コスト確認

`smv_admin` ロールでは Cost Explorer へのアクセスが組織の SCP により拒否されている。  
**AWS 管理者にアカウント `558631582515` のサービス別コストレポートの共有を依頼すること。**
