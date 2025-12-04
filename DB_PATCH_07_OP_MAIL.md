# DB_PATCH_07_OP_MAIL.md — OP_MAIL 用 DB 差分定義 v1.4

本ファイルは、OP_MAIL（電話代行メール解析）を導入するために

既存 DB に対して必要となる差分 DDL をまとめたものです。

- 対象バージョン: v1.4
- 適用順序:
    1. `tickets.source` への enum 追加
    2. `operator_mail_logs` テーブル作成
    3. `op_mail_training_samples` テーブル作成

---

## 1. `tickets.source` への `'operator_mail'` 追加

既存 `tickets.source` enum に `'operator_mail'` を追加します。

```sql
ALTER TABLE tickets
  MODIFY source ENUM(
    'stores_form',
    'customer_line',
    'hp_form',
    'portal_owner',
    'portal_admin',
    'operator_mail'
  ) NOT NULL COMMENT 'チケット発生元チャネル';
```

※ 実際の既存 enum 値が増減している場合は、最新の定義に `'operator_mail'` を追加する形で調整すること。

---

## 2. `operator_mail_logs` テーブル作成

電話代行メールの取込ログ + AI判定 + 最終判定を保持するテーブルです。

```sql
CREATE TABLE operator_mail_logs (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  provider VARCHAR(32) NOT NULL COMMENT 'メールプロバイダ識別子（gmail/sendgrid等）',
  provider_message_id VARCHAR(191) NOT NULL COMMENT 'プロバイダ側のメッセージID',
  message_id VARCHAR(191) NULL COMMENT 'メールのMessage-IDヘッダ',

  shop_id BIGINT UNSIGNED NULL COMMENT '自動推定された店舗ID（shop_emailから）',
  shop_resolve_status ENUM('resolved','pending','ambiguous') NOT NULL DEFAULT 'resolved'
    COMMENT '店舗IDの解決状況（resolved:一意/pending:不明/ambiguous:複数候補）',

  raw_subject TEXT NOT NULL COMMENT '受信メール件名',
  raw_from VARCHAR(191) NOT NULL COMMENT 'Fromアドレス',
  raw_to TEXT NOT NULL COMMENT 'Toアドレス（カンマ区切り等）',
  raw_cc TEXT NULL COMMENT 'Ccアドレス',
  body_text LONGTEXT NULL COMMENT 'プレーンテキスト本文',
  body_html LONGTEXT NULL COMMENT 'HTML本文',
  headers_json LONGTEXT NULL COMMENT '生ヘッダJSON',
  received_at DATETIME NOT NULL COMMENT '受信日時',

  -- AI判定
  ai_follow_urgency ENUM('high','normal','none') NULL COMMENT 'AI判定のフォロー緊急度',
  ai_incident_category VARCHAR(64) NULL COMMENT 'AI判定のインシデント種別',
  ai_follow_type VARCHAR(64) NULL COMMENT 'AI判定のフォロー内容タイプ',
  ai_needs_compensation TINYINT(1) NULL COMMENT '補償前提と判定されたか',
  ai_needs_store_reservation_check TINYINT(1) NULL COMMENT 'STORES予約確認が必要と判定されたか',
  ai_owner_contact_status ENUM('not_called','reached','no_answer','line_busy','unknown') NULL
    COMMENT 'オペ→加盟店への電話結果',
  ai_resolution_status ENUM('resolved','unresolved') NULL COMMENT 'AI判定の解決/未解決',
  ai_summary TEXT NULL COMMENT 'AIによる要約',
  ai_notes_for_owner TEXT NULL COMMENT '加盟店向けメモ（何をすべきか一行）',
  ai_confidence DECIMAL(4,3) NULL COMMENT 'AI判定の信頼度0〜1',

  -- 最終判定（店舗/HQによる上書き）
  final_follow_urgency ENUM('high','normal','none') NULL COMMENT '最終フォロー緊急度',
  final_incident_category VARCHAR(64) NULL COMMENT '最終インシデント種別',
  final_resolution_status ENUM('resolved','unresolved') NULL COMMENT '最終解決/未解決',
  final_decided_by BIGINT UNSIGNED NULL COMMENT '最終判定を入力したユーザーID（HQ/店舗）',
  final_decided_at DATETIME NULL COMMENT '最終判定日時',

  ticket_id BIGINT UNSIGNED NULL COMMENT '紐づくtickets.id（タスク化された場合）',

  status ENUM('received','classified','ticket_created','closed','error')
    NOT NULL DEFAULT 'received'
    COMMENT 'OP_MAIL処理ステータス',

  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,

  PRIMARY KEY (id),
  UNIQUE KEY uq_operator_mail_msg (provider_message_id),
  KEY idx_operator_mail_shop (shop_id),
  KEY idx_operator_mail_status (status),
  KEY idx_operator_mail_final (final_resolution_status, final_follow_urgency)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='電話代行メール取込ログ（AI判定・最終判定含む）';

```

---

## 3. `op_mail_training_samples` テーブル作成

OP_MAIL 用 AI分類の教師データを保持するテーブルです。

```sql
CREATE TABLE op_mail_training_samples (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  operator_mail_log_id BIGINT UNSIGNED NOT NULL COMMENT '元operator_mail_logs.id',
  shop_id BIGINT UNSIGNED NULL COMMENT '関連する店舗ID',

  input_text LONGTEXT NOT NULL COMMENT '学習用に正規化した本文（件名＋本文など）',
  label_follow_urgency ENUM('high','normal','none') NOT NULL COMMENT 'ラベル：フォロー緊急度',
  label_incident_category VARCHAR(64) NULL COMMENT 'ラベル：インシデント種別',
  label_resolution_status ENUM('resolved','unresolved') NOT NULL COMMENT 'ラベル：解決/未解決',

  label_source ENUM('hq','shop','system') NOT NULL DEFAULT 'hq'
    COMMENT 'ラベルの出所（HQ／店舗／システム）',
  is_active TINYINT(1) NOT NULL DEFAULT 1 COMMENT '学習に利用するかどうか',

  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,

  PRIMARY KEY (id),
  KEY idx_op_mail_train_label (label_follow_urgency, label_incident_category),
  KEY idx_op_mail_train_log (operator_mail_log_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='OP_MAIL分類用 学習サンプル';

```

---

## 4. ロールバック指針（参考）

- `tickets.source` の enum から `'operator_mail'` を削除する場合は、
    
    事前に `tickets.source='operator_mail'` のレコードを他の値に退避させること。
    
- `operator_mail_logs` / `op_mail_training_samples` を DROP する前に、
    
    必要に応じてアーカイブテーブルやダンプを取得しておくこと。
    

```sql
DROP TABLE op_mail_training_samples;
DROP TABLE operator_mail_logs;

-- tickets.source から 'operator_mail' を削除したい場合は、
-- 最新の定義に合わせて ALTER TABLE を再発行すること。

```
