# EC2 サービス役割一覧

SSH接続による実調査（2026-05-18）に基づく各EC2インスタンスの動作まとめ。

## サービス全体の流れ

```
[スマホユーザー]
      │
      ├─① インタースティシャル広告表示
      │     ↓
      │   guile-interstitial (is.guile.jp)
      │   ・nginx + PHP
      │   ・itsads.js（広告SDK）を配信
      │   ・/admin/ で管理画面
      │
      ├─② 動画広告（admovie）表示・計測
      │     ↓
      │   admovie-server ×2 (send-guile.sonicmoov.com / admovie.jswfplayer.jp)
      │   ・nginx + Lua スクリプト
      │   ・/mc.php  → インプレッション計測（1px gif）
      │   ・/cc.php  → クリック計測（リダイレクト）
      │   ・/v1/stats → 統計レポート (Lua)
      │   ・td-agent → ログを collector1 へ転送
      │
      └─③ 管理画面・API操作
            ↓
          admin (mypage.guile.jp / api.guile.jp)
          ・nginx + PHP
          ・mypage.guile.jp → 広告主向けマイページ
          ・api.guile.jp    → GUILE API
          ・毎日0時 cron: admovie 設定リロード
          ・backend.jswfplayer.jp → mypage.guile.jp へ 301 リダイレクト（移行済み）

[ログ集約]
  admovie-server → td-agent → collector1 → S3 (jswf-admovie-logs)
                                    ↑
                              Redis でユニークブラウザ数カウント

[開発・ステージング]
  development → dev.guile.jp / branch.guile.jp / test.guile.jp
  staging     → stg.guile.jp（本番と同じ PHP アプリ構成）
```

---

## インスタンス別詳細

### guile-interstitial
| 項目 | 内容 |
|---|---|
| ドメイン | `is.guile.jp` |
| OS | Amazon Linux 2 |
| ミドルウェア | nginx + php-fpm |
| 役割 | インタースティシャル広告の配信・管理 |

**動作内容**
- `itsads.*.min.js.gz` （インタースティシャル広告SDK）を配信
- PHP で広告リクエストを処理
- `/admin/` に管理画面あり
- ELB（is-elb）の振り先

---

### admovie-server（1a / 1c）
| 項目 | 内容 |
|---|---|
| ドメイン | `send-guile.sonicmoov.com` / `admovie.jswfplayer.jp` |
| OS | Amazon Linux AMI 2015.09 |
| ミドルウェア | nginx（Lua組み込み）+ td-agent |
| 役割 | 動画広告の配信・インプレッション/クリック計測 |

**エンドポイント**
| パス | 処理 |
|---|---|
| `/health` | ヘルスチェック（200返却） |
| `/mc.php` | インプレッション計測（1px 透過 gif を返す） |
| `/cc.php` | クリック計測（`rt` パラメータ先へリダイレクト） |
| `/v1/stats` | 統計レポート（Lua スクリプト処理） |

**特徴**
- nginx に **Lua スクリプト**を組み込んで高速計測処理（PHP 不使用）
- `lua_shared_dict guile 128m` で共有メモリを使いカウント集計
- アクセスログを td-agent 経由で collector1 へ転送
- Redis の `fluent-plugin-redis-unique-browser-count` でユニークブラウザ数を計測
- 1a / 1c の2台で冗長構成（admovie-elb01/02、so-elb01/02 の振り先）
- EBS 252GB × 2本だが使用率は約1%（⚠️ 過剰確保の可能性）

---

### admin
| 項目 | 内容 |
|---|---|
| ドメイン | `api.guile.jp` / `mypage.guile.jp` / `stg.guile.jp` / `stg-api.guile.jp` |
| OS | Amazon Linux AMI 2013.09 |
| ミドルウェア | nginx + php-fpm + Redis + Memcached + Fluentd |
| 役割 | 本番・ステージング バックエンド兼管理サーバー |

**提供サービス**
| ドメイン | 内容 |
|---|---|
| `mypage.guile.jp` | 広告主向けマイページ（PHP フロントエンド） |
| `api.guile.jp` | GUILE API（PHP バックエンド） |
| `stg.guile.jp` | ステージング フロントエンド |
| `stg-api.guile.jp` | ステージング API |
| `backend.jswfplayer.jp` | `mypage.guile.jp` へ 301 リダイレクト（jswf 廃止後の移行済み） |

**cron（毎日 0:00 実行）**
- admovie 本番・ステージング・各アドネットワーク向け設定ファイルのリロード
- 対象: prd / stg / i-mobile / akamai / sanrio / sample

**備考**
- コードベース: `/var/www/guile_prod`（PHP、Composer 管理）
- 追加 EBS 99GB `/data` マウント済み（使用率 2%、ほぼ空）

---

### collector1
| 項目 | 内容 |
|---|---|
| Private IP | `10.0.64.201` |
| OS | Amazon Linux AMI 2015.09 |
| ミドルウェア | td-agent（Fluentd）+ Python |
| 役割 | admovie アクセスログの集約・S3 転送 |

**動作内容**
- admovie-server の td-agent から port 24224 でログを受信（Fluentd forward）
- S3 バケット `jswf-admovie-logs` へ 1時間ごとにバッファ書き出し
- nginx は稼働していない（Web 配信なし）
- EBS 126GB（使用率 1%）

---

### staging
| 項目 | 内容 |
|---|---|
| ドメイン | `stg.guile.jp`（nginx 経由） |
| OS | Amazon Linux AMI 2015.09 |
| ミドルウェア | nginx + td-agent + Redis + Python |
| 役割 | ステージング フロントエンドサーバー |

**動作内容**
- 本番（admin）と同じ PHP アプリ構成のステージング環境
- td-agent でステージング admovie のアクセスログを収集・集計
- Redis でインプレッション/ビュー/スタート数をカウント（本番 collector1 の代替）

---

### development
| 項目 | 内容 |
|---|---|
| ドメイン | `dev.guile.jp` / `branch.guile.jp` / `test.guile.jp` |
| OS | Amazon Linux AMI 2013.09（⚠️ 最も古い） |
| ミドルウェア | nginx + php-fpm + Redis + Memcached + Fluentd |
| 役割 | 開発・テスト環境 |

**動作内容**
- 本番と同じ PHP アプリ（`/var/www/dev.guile.jp`）が動作
- `branch.guile.jp`：ブランチごとの動作確認用
- `test.guile.jp`：テスト用環境
- `send-dev-guile.sonicmoov.com`：admovie 開発用配信エンドポイント
