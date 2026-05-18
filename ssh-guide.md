# EC2 SSH 接続ガイド

## 前提条件

1. **saml2aws でログイン済み**（トークンは約1時間で期限切れ）
   ```bash
   saml2aws login --profile smv-admin
   ```
2. **SSH 設定が適用済み**（`~/.ssh/config` に `Include ~/.ssh/ssh-guile/config` が追加されていること）
3. **接続元 IP が bastion の SG に許可されていること**（現在許可: `52.195.61.178/32`, `210.172.253.183/32`）

## SSH config ファイル

`ssh-guile` リポジトリの `config` に全ホスト定義が記載されている。  
鍵ファイルは2種類:

| 鍵ファイル | 対象ホスト |
|---|---|
| `ec2-user@bastion.pem` | NAT-PRX01（bastion）、guile-interstitial、stg-guile-interstitial |
| `sonicmoov@aws.pem` | admin、development、staging、collector1/2、admovie-server |

## 接続コマンド一覧

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

## 接続確認結果（2026-05-18）

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
