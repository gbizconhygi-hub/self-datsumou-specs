# ENV_SCHEMA.md — v1.4 環境変数・設定値一覧

このドキュメントは、v1.4 システム全体で利用する `.env` / 設定値の  
「キー名・型・用途・利用フェーズ」を一覧化したものです。

- 目的
  - `.env` の抜け漏れ・ブレを防ぐ
  - 新機能追加時に「どの設定を増やすべきか」を俯瞰しやすくする
  - AI/開発者がコードを書くときの参照元とする
- 注意
  - 値はダミー例です。実運用では各環境ごとに適切な値を設定してください。

---

## 1. グローバルアプリ設定

| KEY              | Type   | Required | Used by                    | Description                               | Example                    |
|------------------|--------|----------|----------------------------|-------------------------------------------|----------------------------|
| APP_ENV          | string | yes      | 全体                       | 環境名（local / staging / production）    | `production`              |
| APP_DEBUG        | bool   | yes      | 全体                       | デバッグモード（trueなら詳細エラー出力） | `false`                   |
| APP_URL          | string | yes      | HELP_UI, SHOP, LINE, AI    | メインアプリURL（リダイレクト基準）      | `https://self-datsumou.net` |
| APP_TIMEZONE     | string | yes      | CRON_SYS, 日時処理         | PHP・DBのデフォルトタイムゾーン           | `Asia/Tokyo`              |
| LOG_CHANNEL      | string | no       | SYS_CORE                   | ログ出力チャンネル                        | `stack`                   |

---

## 2. サブドメイン・フロントURL

| KEY                      | Type   | Required | Used by             | Description                               | Example                              |
|--------------------------|--------|----------|---------------------|-------------------------------------------|--------------------------------------|
| BASE_URL_HELP            | string | yes      | HELP_UI, MAIL       | help. サイトのベースURL                   | `https://help.self-datsumou.net`     |
| BASE_URL_LINE            | string | yes      | LINE_UI             | LINE 用 WebフロントのベースURL           | `https://line.self-datsumou.net`     |
| BASE_URL_SHOP            | string | yes      | SHOP_PORTAL         | オーナー／店舗ポータルのベースURL         | `https://shop.self-datsumou.net`     |
| BASE_URL_AI              | string | yes      | AI_INBOX, OP_MAIL   | HQ向けAI-INBOXなどのベースURL             | `https://ai.self-datsumou.net`       |
| BASE_URL_ASSETS          | string | no       | HELP_UI, LINE_UI    | 静的アセットCDNなどのベースURL            | `https://assets.self-datsumou.net`   |

---

## 3. DB 接続

| KEY           | Type   | Required | Used by         | Description                 | Example           |
|---------------|--------|----------|-----------------|-----------------------------|-------------------|
| DB_HOST       | string | yes      | 全体            | DBホスト名                  | `127.0.0.1`       |
| DB_PORT       | int    | yes      | 全体            | DBポート                    | `3306`            |
| DB_DATABASE   | string | yes      | 全体            | データベース名              | `self_datsumou`   |
| DB_USERNAME   | string | yes      | 全体            | DBユーザー名                | `selfd_user`      |
| DB_PASSWORD   | string | yes      | 全体            | DBパスワード                | `********`        |
| DB_CHARSET    | string | no       | 全体            | 文字コード                  | `utf8mb4`         |
| DB_COLLATION  | string | no       | 全体            | 照合順序                    | `utf8mb4_unicode_ci` |

---

## 4. AI／LLM 関連

| KEY                 | Type   | Required | Used by           | Description                               | Example                                    |
|---------------------|--------|----------|-------------------|-------------------------------------------|--------------------------------------------|
| OPENAI_API_KEY      | string | yes      | AI_CORE           | OpenAI API キー                           | `sk-xxxx`                                  |
| OPENAI_API_BASE_URL | string | no       | AI_CORE           | OpenAI 互換APIのベースURL（任意）         | `https://api.openai.com/v1`                |
| AI_DEFAULT_MODEL    | string | yes      | AI_CORE           | 既定のモデル名                             | `gpt-5.1-thinking`                         |
| AI_LOG_LEVEL        | string | no       | AI_CORE, AI_INBOX | AIログの詳細度                             | `info`                                     |
| AI_MAX_TOKENS       | int    | no       | AI_CORE           | 1リクエストあたりの最大トークン数          | `2048`                                     |

---

## 5. STORES 予約 API

| KEY                         | Type   | Required | Used by          | Description                                        | Example                                            |
|-----------------------------|--------|----------|------------------|----------------------------------------------------|----------------------------------------------------|
| STORES_API_BASE_URL         | string | yes      | API_COMM         | STORES 予約 API ベースURL                          | `https://api.stores.dev/reserve/202101`           |
| STORES_API_KEY              | string | yes      | API_COMM         | STORES API キー                                    | `stores-xxxx`                                     |
| STORES_WEB_DASHBOARD_URL    | string | no       | SHOP_PORTAL      | 管理画面URL（オーナーに案内するときなど）          | `https://admin.stores.jp/...`                     |

---

## 6. RemoteLOCK / キー管理

| KEY                          | Type   | Required | Used by          | Description                                   | Example                        |
|------------------------------|--------|----------|------------------|-----------------------------------------------|--------------------------------|
| REMOTELOCK_API_BASE_URL      | string | yes      | API_COMM         | RemoteLOCK API ベースURL                      | `https://api.remotelock.com`   |
| REMOTELOCK_CLIENT_ID         | string | yes      | API_COMM         | OAuth クライアントID                          | `rl_client_...`                |
| REMOTELOCK_CLIENT_SECRET     | string | yes      | API_COMM         | OAuth クライアントシークレット                | `********`                     |
| REMOTELOCK_DEFAULT_SCOPE     | string | no       | API_COMM         | 取得するスコープ                              | `offline_access profile locks` |

---

## 7. メール／OP_MAIL 関連

### 7.1 メール送信

| KEY                 | Type   | Required | Used by         | Description                                | Example                       |
|---------------------|--------|----------|-----------------|--------------------------------------------|-------------------------------|
| MAIL_DRIVER         | string | yes      | HELP_UI, SYSTEM | smtp/sendgrid等                             | `smtp`                        |
| MAIL_HOST           | string | yes      | HELP_UI, SYSTEM | SMTPホスト                                  | `smtp.gmail.com`              |
| MAIL_PORT           | int    | yes      | HELP_UI, SYSTEM | ポート                                      | `587`                         |
| MAIL_USERNAME       | string | yes      | HELP_UI, SYSTEM | SMTPユーザー                               | `noreply@example.com`         |
| MAIL_PASSWORD       | string | yes      | HELP_UI, SYSTEM | SMTPパスワード                             | `********`                    |
| MAIL_ENCRYPTION     | string | no       | HELP_UI, SYSTEM | `tls` / `ssl`                               | `tls`                         |
| MAIL_FROM_ADDRESS   | string | yes      | HELP_UI         | デフォルトFromアドレス                     | `noreply@self-datsumou.net`   |
| MAIL_FROM_NAME      | string | yes      | HELP_UI         | デフォルト送信者名                         | `セルフ脱毛サロン ハイジ`     |

### 7.2 OP_MAIL Webhook

| KEY                      | Type   | Required | Used by         | Description                                             | Example                          |
|--------------------------|--------|----------|-----------------|---------------------------------------------------------|----------------------------------|
| OP_MAIL_WEBHOOK_TOKEN    | string | yes      | OP_MAIL, SYS    | `/op_mail/ingest` 認証用トークン                         | `opmail-token-xxxx`             |
| OP_MAIL_PROVIDER         | string | no       | OP_MAIL         | メールプロバイダ識別用ラベル                            | `gmail`                         |
| OP_MAIL_ENABLED          | bool   | no       | OP_MAIL         | 機能のON/OFF                                            | `true`                          |

---

## 8. 通知（ChatWork / SMS）

### 8.1 ChatWork

| KEY                           | Type   | Required | Used by          | Description                                   | Example          |
|-------------------------------|--------|----------|------------------|-----------------------------------------------|------------------|
| CHATWORK_API_TOKEN            | string | yes      | SYS_CORE, CRON   | ChatWork API トークン                         | `cw-xxxx`        |
| CHATWORK_ROOM_ID_HQ_ALERT     | string | yes      | CRON_SYS, OP_MAIL| HQ向けアラート用ルームID                      | `123456789`      |
| CHATWORK_ROOM_ID_OP_MAIL      | string | no       | OP_MAIL          | 電話代行連絡通知専用ルームID                  | `987654321`      |
| CHATWORK_ROOM_ID_KPI_REPORT   | string | no       | KPI_ANALYTICS    | KPIレポート投げ込み用ルームID                | `111111111`      |

### 8.2 SMS

（利用プロバイダに応じて調整）

| KEY                     | Type   | Required | Used by           | Description                            | Example                 |
|-------------------------|--------|----------|-------------------|----------------------------------------|-------------------------|
| SMS_PROVIDER            | string | no       | CRON_SYS, OP_MAIL | `twilio` など                          | `twilio`                |
| SMS_API_KEY             | string | no       | CRON_SYS, OP_MAIL | SMSプロバイダAPIキー                   | `twilio-xxxx`           |
| SMS_API_SECRET          | string | no       | CRON_SYS, OP_MAIL | SMSプロバイダAPIシークレット           | `********`              |
| SMS_FROM_NUMBER         | string | no       | CRON_SYS, OP_MAIL | 送信元電話番号                         | `+8150xxxxxxxx`         |

---

## 9. KPI / SALES_LINK / CRON_SYS

| KEY                         | Type   | Required | Used by           | Description                                      | Example     |
|-----------------------------|--------|----------|-------------------|--------------------------------------------------|-------------|
| KPI_TIME_BUCKET_MINUTES     | int    | no       | KPI_ANALYTICS     | KPI集計の時間バケット                            | `60`        |
| SALES_LINK_IMPORT_DIR       | string | no       | SALES_LINK        | 売上CSVの配置ディレクトリ                         | `/var/...`  |
| CRON_HEALTHCHECK_SLACK_URL  | string | no       | CRON_SYS          | ヘルスチェック通知用のSlack/Webhook URL          | `https://...` |

---

## 10. フィーチャーフラグ／実験用

| KEY                          | Type   | Required | Used by         | Description                                     | Example   |
|------------------------------|--------|----------|-----------------|-------------------------------------------------|-----------|
| FEATURE_OP_MAIL_ENABLED      | bool   | no       | OP_MAIL         | OP_MAIL 機能全体の ON/OFF                       | `true`    |
| FEATURE_AI_TUNING_ENABLED    | bool   | no       | AI_CORE         | AI学習／パラメータチューニング機能のON/OFF      | `false`   |
| FEATURE_KPI_DASHBOARD_V2     | bool   | no       | KPI_ANALYTICS   | 新ダッシュボードの有効化フラグ                 | `false`   |

---

## 11. 補足

- 各 `*_DEFAULT_PARAMS.md` で決めた「論理的なパラメータ」は、  
  必要に応じてここ（ENV_SCHEMA）にもキー名として登場させる。
- 本ファイルは「何が必要かの一覧」であり、  
  環境ごとの具体値（本番 / ステージング）は別途 `.env.production` 等で管理する。
