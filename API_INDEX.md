# API_INDEX.md — v1.4 エンドポイント一覧

このドキュメントは、v1.4 システムで利用する HTTP/CLI エンドポイントを  
「どのレイヤがどのI/Oを持つか」という観点で一覧化したものです。

- 目的
  - ルーティングの全体像を俯瞰する
  - AI/開発者がコードを書く際の参照元とする
  - 仕様書（PHASE_xx_*）への橋渡し

※ URL パスは原則として例示です。実装時の細部はフレームワークに合わせて調整してください。

---

## 1. HELP_UI（help.self-datsumou.net）

### 1.1 公開API（顧客向け）

| Endpoint                     | Method | Auth            | Description                                      | Spec                       |
|-----------------------------|--------|-----------------|--------------------------------------------------|----------------------------|
| `/help/chat/start`          | GET    | cookie/session? | チャット開始画面（初期フォーム表示）            | PHASE_03_HELP_UI.md       |
| `/help/chat/submit`         | POST   | cookie/session? | 質問送信（AIチュートリアル or 人間エスカレ）   | PHASE_03_HELP_UI.md       |
| `/help/chat/history`        | GET    | cookie/session? | 自分のチャット履歴取得                           | PHASE_03_HELP_UI.md       |

### 1.2 システム内部用

| Endpoint                           | Method | Auth     | Description                                 | Spec                 |
|-----------------------------------|--------|----------|---------------------------------------------|----------------------|
| `/help/webhook/email_reply`       | POST   | token    | noreply メールからの返信URL経由 Webhook    | HELP_UI_POLICY.md    |

---

## 2. LINE_UI（line.self-datsumou.net）

### 2.1 Webhook

| Endpoint                 | Method | Auth           | Description                            | Spec                    |
|-------------------------|--------|----------------|----------------------------------------|-------------------------|
| `/line/webhook`         | POST   | LINE署名       | LINE公式アカウント Webhook 受信       | PHASE_03_LINE_UI.md     |

### 2.2 LIFF / Webフロント

| Endpoint                     | Method | Auth            | Description                                     | Spec                    |
|-----------------------------|--------|-----------------|-------------------------------------------------|-------------------------|
| `/line/chat`                | GET    | LINEログイン    | LINE チャット画面（AIチュートリアル）          | PHASE_03_LINE_UI.md     |
| `/line/chat/submit`         | POST   | LINEログイン    | メッセージ送信                                  | PHASE_03_LINE_UI.md     |
| `/line/link/complete`       | GET    | token           | STORES会員↔LINEユーザーIDの紐づけ完了コールバック | PHASE_03_LINE_UI.md  |

---

## 3. SHOP_PORTAL（shop.self-datsumou.net）

### 3.1 認証／セッション

| Endpoint                     | Method | Auth           | Description                            | Spec                      |
|-----------------------------|--------|----------------|----------------------------------------|---------------------------|
| `/shop/login`               | GET    | —              | ログイン画面                           | PHASE_04_SHOP_PORTAL.md   |
| `/shop/login`               | POST   | —              | ログイン処理                           | PHASE_04_SHOP_PORTAL.md   |
| `/shop/logout`              | POST   | session        | ログアウト                             | PHASE_04_SHOP_PORTAL.md   |

### 3.2 ダッシュボード・チケット

| Endpoint                                     | Method | Auth      | Description                                                     | Spec                      |
|---------------------------------------------|--------|-----------|-----------------------------------------------------------------|---------------------------|
| `/shop/dashboard`                           | GET    | session   | 店舗ダッシュボード（売上・タスク・OP_MAIL履歴など）           | PHASE_04_SHOP_PORTAL.md   |
| `/shop/tickets`                             | GET    | session   | 要対応チケット一覧（help/line/op_mail等）                      | PHASE_04_SHOP_PORTAL.md   |
| `/shop/tickets/{id}`                        | GET    | session   | チケット詳細（OP_MAIL本文・AI判定・対応ボタン含む）           | PHASE_04_SHOP_PORTAL.md   |
| `/shop/tickets/{id}/reply`                  | POST   | session   | 顧客への返信（LINE / メール / メモ）                           | PHASE_04_SHOP_PORTAL.md   |
| `/shop/tickets/{id}/close`                  | POST   | session   | チケットクローズ（対応完了など）                               | PHASE_04_SHOP_PORTAL.md   |
| `/shop/tickets/{id}/mark_no_action`         | POST   | session   | 「対応不要で完了」（OP_MAIL向け close_reason 更新）            | PHASE_07_OP_MAIL.md       |

### 3.3 OP_MAIL 専用

| Endpoint                                      | Method | Auth    | Description                                              | Spec                      |
|----------------------------------------------|--------|---------|----------------------------------------------------------|---------------------------|
| `/shop/tickets/{id}/link_member`             | POST   | session | STORES会員番号/予約IDを入力し、会員情報とLINE紐づきを照合 | PHASE_04_SHOP_PORTAL.md + PHASE_07_OP_MAIL.md |

---

## 4. AI_INBOX（ai.self-datsumou.net）

### 4.1 HQ 用 INBOX

| Endpoint                        | Method | Auth        | Description                                   | Spec                      |
|--------------------------------|--------|-------------|-----------------------------------------------|---------------------------|
| `/ai/inbox`                    | GET    | HQ session  | チケット一覧（全チャネル）                   | PHASE_05_AI_INBOX.md      |
| `/ai/inbox/tickets/{id}`       | GET    | HQ session  | チケット詳細（ログ・AI判定・履歴）           | PHASE_05_AI_INBOX.md      |
| `/ai/inbox/tickets/{id}/note`  | POST   | HQ session  | HQメモ／ラベル付け                          | PHASE_05_AI_INBOX.md      |

### 4.2 OP_MAIL 学習用（管理UI想定）

| Endpoint                                      | Method | Auth        | Description                                          | Spec                          |
|----------------------------------------------|--------|-------------|------------------------------------------------------|-------------------------------|
| `/ai/admin/op_mail/logs`                     | GET    | HQ admin    | operator_mail_logs 一覧・フィルタ                    | PHASE_07_OP_MAIL.md           |
| `/ai/admin/op_mail/logs/{id}`                | GET    | HQ admin    | 個別ログ詳細（ai_* / final_* / 原文）               | PHASE_07_OP_MAIL.md           |
| `/ai/admin/op_mail/logs/{id}/label`          | POST   | HQ admin    | final_* ラベル更新＋学習サンプル登録フラグ          | PHASE_07_OP_MAIL.md           |

（管理UI自体の仕様は AI_CORE / AI_INBOX 側で詳細定義）

---

## 5. OP_MAIL（電話代行メール解析）

### 5.1 Webhook 受信

| Endpoint                 | Method | Auth                 | Description                                  | Spec                      |
|-------------------------|--------|----------------------|----------------------------------------------|---------------------------|
| `/op_mail/ingest`       | POST   | `X-Op-Mail-Token`    | 電話代行メール Webhook 受信                  | PHASE_07_OP_MAIL.md       |

---

## 6. API_COMM（外部APIラッパー）

※ URI は例示。実際には `/api/stores/...` などで実装想定。

| Endpoint                                | Method | Auth      | Description                                      | Spec                      |
|-----------------------------------------|--------|-----------|--------------------------------------------------|---------------------------|
| `/api/stores/customers/{id}`            | GET    | internal  | STORES顧客情報参照                               | PHASE_02_API_COMM.md      |
| `/api/stores/reservations/{id}`         | GET    | internal  | STORES予約情報参照                               | PHASE_02_API_COMM.md      |
| `/api/remotelock/locks/{id}`            | GET    | internal  | RemoteLOCK ロック情報参照                        | PHASE_02_API_COMM.md      |

---

## 7. AI_CORE 内部API（必要に応じ）

| Endpoint                     | Method | Auth      | Description                            | Spec                      |
|-----------------------------|--------|-----------|----------------------------------------|---------------------------|
| `/internal/ai_core/invoke`  | POST   | internal  | AI_CORE_REQUEST/RESPONSE のエントリポイント | PHASE_02_AI_CORE.md      |

※ 実装上は PHP 内部関数呼び出しで済む場合もあり、HTTP である必要はない。

---

## 8. CRON_SYS（CLI / バッチ）

CRON_SYS は HTTP ではなく CLI 実行を前提とする。  
代表的なジョブ名と役割のみ記載する。

| Command                             | Description                                          | Spec                      |
|-------------------------------------|------------------------------------------------------|---------------------------|
| `php cron/cron_sales_link.php`      | 売上CSV取込 → DB 反映                               | PHASE_06_SALES_LINK.md    |
| `php cron/cron_kpi_analytics.php`   | KPI集計・ダッシュボード用テーブル更新               | PHASE_06_KPI_ANALYTICS.md |
| `php cron/cron_sms_followup.php`    | SLA違反チケットへのSMS追撃送信                      | PHASE_07_CRON_SYS.md      |
| `php cron/cron_backup_rotate.php`   | バックアップ・ログローテーション                     | PHASE_07_CRON_SYS.md      |
| `php cron/cron_op_mail_health.php`  | （任意）OP_MAIL ログのエラーチェック                 | PHASE_07_OP_MAIL.md       |

---

## 9. 補足

- 実際のルーティング定義（Slim / Laravel / 独自ルータなど）は、  
  本ドキュメントの一覧を元に実装時に最終調整する。
- 新しいフェーズ／機能を追加した場合は、
  - エンドポイントをここに追記
  - 対応する PHASE_xx_* ファイルからリンク
  を行うこと。
