# GUILE リポジトリ一覧

> 調査日: 2026-05-18

---

## ⚠️ 管理画面リポジトリが未確認

広告の登録・編集・効果確認ができる **管理画面アプリケーションのソースコードは今回 clone した 20 件には含まれていない。**

- EC2 インスタンス `admin`（m3.large）が管理画面サーバーと推測される
- `guile-ops/roles/admin`・`Dockerfile.admin`・`playbook/admin.dev.yml` など、admin サーバーの**構築設定（インフラ側）は存在する**
- `guile_js` の README に「Guile管理画面に登録するテンプレートデータ」という記述があり、管理画面は実在する
- アプリ本体（Rails / PHP 等）のリポジトリが別途存在するはず → **Backlog Wiki や GitHub organization で確認が必要**

---

## 概要

GUILE 関連リポジトリは全 20 件。役割ごとに以下のカテゴリに分類できる。

| カテゴリ | リポジトリ数 | 内容 |
|---|---|---|
| フロントエンド（広告テンプレート） | 7 | 各 ASP・パートナー向けの広告表示テンプレート |
| インフラ・デプロイ | 4 | サーバー構築・プロビジョニング・運用基盤 |
| バックエンド・バッチ | 4 | データ集計・レポート・バッチ処理 |
| プレイヤー・コア | 2 | 動画再生プレイヤー、JS 開発基盤 |
| その他 | 3 | ラボ環境・モック・LP・SSH 鍵 |

---

## フロントエンド（広告テンプレート）

### guile-frontend-core
- **言語/FW**: JavaScript / Grunt
- **用途**: 広告動画プレイヤーの共通コア。再生・計測・SNS 共有などの中核機能を提供する。他の frontend-* リポジトリはこれをベースに各パートナー向けにカスタマイズしている。
- **主要ファイル**: `src/js/`, `Gruntfile.js`

### guile-frontend-amoad
- **言語/FW**: JavaScript / Grunt
- **用途**: Amoad（アドネットワーク）向けの動画広告フロントエンド実装。配信タグ・埋め込みコード周りを担う。
- **主要ファイル**: `public/`, `Gruntfile.js`

### guile-frontend-imobile
- **言語/FW**: JavaScript / Grunt
- **用途**: i-mobile 向けの広告テンプレート実装。guile-frontend-demo をベースにした派生版。
- **主要ファイル**: `src/`, `dest/`

### guile-frontend-microad
- **言語/FW**: JavaScript / Grunt
- **用途**: MicroAd 向けの動画広告テンプレート実装。レクタングル系の配信用コードが中心。
- **主要ファイル**: `src/`, `dest/`

### guile-frontend-nifty
- **言語/FW**: JavaScript / Grunt
- **用途**: nifty 向けの広告テンプレート実装。エキスパンド広告向け構成。
- **主要ファイル**: `src/`, `dest/`

### guile-frontend-united
- **言語/FW**: JavaScript / Grunt
- **用途**: united 向けの広告配信テンプレート。再生中の処理や遷移先制御などの独自要件あり。
- **主要ファイル**: `src/`, `dest/`, `_old/`

### guile-frontend-demo
- **言語/FW**: JavaScript / Grunt
- **用途**: フロントエンド広告テンプレートのデモ・検証用。各種テンプレートの動作確認に使う開発用リポジトリ。`guile_player` をサブモジュールとして参照。
- **主要ファイル**: `src/`, `test/`, `guile_player/`

### guile-frontend-lua
- **言語/FW**: Lua + Perl + JavaScript
- **用途**: サーベイ配信サーバー。Lua スクリプトと Redis / Perl 系モジュールで配信ロジックを構成。他の frontend-* とは性質が異なるサーバーサイド実装。
- **主要ファイル**: `lua-packages/`, `lib-perl/`, `content/`

---

## インフラ・デプロイ

### guile-deploy-config
- **言語/FW**: Ansible / Packer / Shell
- **用途**: サーバー構築・デプロイ用の設定集。AMI 作成や各種環境（本番・ステージング・開発）のプロビジョニングを担う。
- **主要ファイル**: `roles/`, `hosts`, `*.yml`, `*.json`

### guile-ops
- **言語/FW**: Docker / Ansible / Terraform
- **用途**: 開発・本番環境の運用基盤。Docker Compose、AMI、Terraform などをまとめた Ops リポジトリ。
- **主要ファイル**: `docker-compose.yml`, `Dockerfile.*`, `terraform/`, `roles/`

### guile-tracker-provisioner
- **言語/FW**: Docker / Ansible / Packer
- **用途**: トラッカー系プロジェクト共通のプロビジョニングテンプレート。ローカル開発環境と AMI 作成を管理。
- **主要ファイル**: `docker-compose.yml`, `playbook/`, `roles/`, `terraform/`

### ssh-guile
- **言語/FW**: SSH 設定
- **用途**: AWS 等への接続用 SSH 設定・鍵の置き場。コードというより接続資材の保管リポジトリ。
- **主要ファイル**: `config`, `*.pem`, `*.ppk`
- > ⚠️ SSH 秘密鍵が含まれる可能性があるため取り扱い注意。

---

## バックエンド・バッチ

### guile-aggregate-athena
- **言語/FW**: Java / Maven
- **用途**: Athena・各種広告データを集計するバッチ系ツール。JDBC / Java で日次集計を実行する。
- **主要ファイル**: `pom.xml`, `src/main/java/.../AthenaOperator.java`

### guile-csv-report
- **言語/FW**: PHP / Shell
- **用途**: S3 からデータをダウンロードして CSV に集計し、メール送信するレポート生成スクリプト群。
- **主要ファイル**: `guile-id-linkage-daily-report-create-csv-general.php`, `guile-log-download-from-s3-general.sh`, `config.ini`

### guile_batch
- **言語/FW**: PHP / Composer
- **用途**: PHP の単発バッチ・RDS 登録処理。`rdsRegister.php` が中心。
- **主要ファイル**: `composer.json`, `rdsRegister.php`

### guile-tracker-mock
- **言語/FW**: JavaScript / static mock
- **用途**: デバイストラッキングのモック。Cookie とブラウザキャッシュで ID 維持を確認する用途。
- **主要ファイル**: `src/`, `resource/main.min.js`, `postdata.json`

---

## プレイヤー・コア

### guile_player
- **言語/FW**: JavaScript / Flash 系資産
- **用途**: 動画再生プレイヤー本体の実装。広告フロントエンド（`guile-frontend-*`）から利用されるプレイヤーライブラリ。Flash（SWF）資産と ogv.js（HTML5 動画）を含む。
- **主要ファイル**: `src/`, `flash/`, `dist/`, `ogv.js`

### guile_js
- **言語/FW**: JavaScript / Gulp
- **用途**: 管理画面向けの JS 開発基盤・テンプレート生成ツール。`provider.tpl` 生成ロジックなどの開発用コード。
- **主要ファイル**: `gulpfile.js`, `gulp/`, `demo/`

---

## その他

### guile-lab
- **言語/FW**: WordPress / PHP
- **用途**: リッチメディア・動画広告研究用の WordPress 環境。テーマ・プラグイン込みの実運用寄り構成。
- **主要ファイル**: `wordpress/`, `wordpress/wp-config.php`

### guile_lp
- **言語/FW**: HTML / CSS / Grunt
- **用途**: GUILE のランディングページ（LP）管理用リポジトリ。ソースを `dist` にビルドして公開する。
- **主要ファイル**: `src/`, `dist/`, `Gruntfile.js`

---

## リポジトリ間の依存関係

```
guile_player（動画プレイヤー本体）
    ↑ サブモジュールとして参照
guile-frontend-core（共通コア）
    ↑ ベースとして派生
guile-frontend-amoad
guile-frontend-imobile
guile-frontend-microad    → 各パートナー向けテンプレート
guile-frontend-nifty
guile-frontend-united
guile-frontend-demo（検証用）
```

```
guile-deploy-config / guile-ops / guile-tracker-provisioner
    → AWS 上のサーバーへデプロイ
    → EC2: guile-interstitial, collector1 等（aws-resources.md 参照）
```
