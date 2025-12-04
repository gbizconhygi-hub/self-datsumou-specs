# PHASE_07_OP_MAIL.md — OP_MAIL（電話代行メール解析）仕様 v1.4

## 1. コンテキスト・位置づけ

### 1.1 要素名・スコープ

- 要素名: OP_MAIL（電話代行メール解析）
- フェーズ: Phase7
- レイヤ: AI_CORE 配下のサブモジュール
- 本フェーズの目的:
    - 電話代行オペレーターからの「緊急連絡メール」を受信し、
        - 店舗を自動特定（できる範囲で）
        - AI で「解決済みか / 追加フォローが必要か」「フォロー緊急度」を判定
        - 未解決案件のみ `tickets` としてタスク化し、ChatWork / SMS / SHOP_PORTAL に連携
        - すべてのメールを `operator_mail_logs` にログとして残し、学習・監査の SoR とする

### 1.2 他フェーズとの関係

- spec_v1.4_base で定義された役割の具体化フェーズ。
- PHASE_04_SHOP_PORTAL:
    - OP_MAIL 由来チケットを「電話代行緊急連絡」として店舗側に表示し、
    LINE / メール（noreply + HELP）による顧客対応フローを提供する。
- PHASE_05_AI_INBOX:
    - HQ 向け INBOX として、全チャネル（help./line./shop./OP_MAIL）の tickets を統合管理する。
- PHASE_07_CRON_SYS:
    - CRON_SYS とは独立。OP_MAIL は **Push トリガ専用** とし、cron によるメールポーリングは行わない。
- AI_CORE:
    - `channel='op_mail'` として AI コアに分類処理を委譲し、
    「フォロー緊急度」「インシデント種別」「店舗連絡結果」などのラベルを返してもらう。

---

## 2. 用語と前提

### 2.1 用語

- 電話代行オペレーター:
    - 外注コールセンターを想定。お客様との電話を受け、
    一次対応と加盟店へのエスカレーションを行う。
- 緊急連絡メール:
    - オペレーターが一次対応後に送る業務報告メール。
    - 対象は「無人サロンの運営に影響するインシデント」が前提。
- 解決済み（resolved）:
    - 電話中にオペレーター対応のみで完結しており、加盟店からの追加連絡が不要な案件。
- 未解決（unresolved）:
    - オペだけでは完結せず、「加盟店による STORES 予約確認」「補償判断」「折返し連絡」等が必要な案件。

### 2.2 本フェーズの前提

- 真の「超緊急」（電話が切れないレベル）の案件は、
オペレーターが **その場で加盟店に電話してエスカレーション** する運用とし、
OP_MAIL はその **事後報告とフォロータスク化** を担当する。
- STORES 予約状況の最終判断は **加盟店のみが行える**。
OP_MAIL は STORES を直接見に行かず、
SHOP_PORTAL から加盟店が会員番号 / 予約ID を入力することで会員特定を行う前提とする。
- LINE ID は STORES には存在せず、
LINE↔STORES 会員の紐づけは自社 DB（`customer_links` 等）を SoR とする。

---

## 3. メール受信アーキテクチャと I/O 契約

### 3.1 全体像

1. 電話代行オペレーターは、
緊急連絡用の共通アドレス（例: `emergency@self-datsumou.net`）に報告メールを送る。
2. メールサーバ / プロバイダ側で、
新着メールを HTTP Webhook で `op_mail_ingest` エンドポイントに POST する。
3. `op_mail_ingest` がメール内容を受け取り、
    - 店舗特定
    - AI分類
    - `operator_mail_logs` への記録
    - 必要に応じて `tickets` 起票
    - ChatWork / SMS 通知
    を行う。

### 3.2 エンドポイント仕様

- URL（例）
    - `POST <https://ai.self-datsumou.net/op_mail/ingest`>
    - 実際のパスは SYS_CORE の API パターンに従う（例: `/api/op_mail/ingest.php`）。
- HTTP メソッド
    - `POST` のみ許容。
- 認証
    - リクエストヘッダ:
        - `X-Op-Mail-Token: <固定トークン>`
            - `.env` 等に `OP_MAIL_WEBHOOK_TOKEN` として格納。
        - 送信元 IP を firewall / Web サーバ設定で制限（電話代行メールサーバの IP のみ許可）。
    - オプション:
        - プロバイダが HMAC 署名をサポートする場合、`X-Signature` 等を追加検証。

### 3.3 リクエスト payload（プロバイダ非依存の抽象仕様）

```json
{
  "provider": "gmail | sendgrid | other",
  "provider_message_id": "xxxxxxxx",
  "message_id": "<xxxx@mail.example>",
  "subject": "【緊急】●●店 鍵が開かない件",
  "from": "operator@example.com",
  "to": [
    "emergency@self-datsumou.net",
    "nagoya-eki@self-datsumou.net"
  ],
  "cc": [
    "hq@example.com"
  ],
  "body_text": "プレーンテキスト本文（省略可）",
  "body_html": "<p>HTML本文（省略可）</p>",
  "headers_json": "{... 生のヘッダをJSON化したもの ...}",
  "attachments": [
    {
      "filename": "photo1.jpg",
      "content_type": "image/jpeg",
      "size": 12345,
      "download_url": "https://..."
    }
  ],
  "received_at": "2025-12-04T21:03:00+09:00"
}

```

- 必須項目:
    - `provider`, `provider_message_id`, `subject`, `from`, `to[]`, `received_at`
- `body_text` が無い場合はサーバ側で HTML→テキスト変換を行う。
- 添付ファイルは v1.4 では分類にのみ利用（あれば AI へのヒントとして渡す）。
    
    UI上での表示や保存位置は将来拡張とする。
    

### 3.4 レスポンス payload

- 正常（新規処理）:

```json
{
  "status": "ok",
  "op_mail_log_id": 12345,
  "ticket_created": true,
  "ticket_id": 67890
}

```

- 正常（重複）:

```json
{
  "status": "duplicate",
  "op_mail_log_id": 12345
}

```

- 認証エラー: HTTP 401
- バリデーションエラー: HTTP 400 + エラー理由
- サーバ内部エラー: HTTP 500

---

## 4. シーケンス（メール受信〜AI分類〜チケット化〜通知）

### 4.1 高レベルシーケンス

1. Webhook 受信
    - 認証・入力バリデーション。
2. 重複チェック
    - `provider_message_id` で `operator_mail_logs` を検索。
    - 既に存在すれば `status=duplicate` を返し終了（idempotent）。
3. 店舗特定
    - `to[]` から店舗メールアドレスを抽出し、`shops` テーブルと突き合わせて `shop_id` を決定。
4. 正規化テキスト生成
    - 件名 + 本文（プレーンテキスト化）をまとめて `normalized_text` を作成。
5. AI コア呼び出し
    - `normalized_text` と補助情報を `AI_CORE_REQUEST` として送信。
6. AI応答に基づく分類
    - `resolution_status`（resolved/unresolved）
    - `follow_urgency`（high/normal/none）
    - `incident_category` 等を受け取る。
7. `operator_mail_logs` への保存
    - 生メール + AI判定 + 店舗ID を1レコードにまとめる。
8. 解決済み/未解決の分岐
    - `resolution_status='resolved'` または `follow_urgency='none'`
        
        → ログのみ（SHOP_PORTAL には「緊急連絡があった」という通知カード）。
        
    - それ以外（未解決）
        
        → `tickets` にタスク起票。
        
9. 通知
    - 未解決案件のみ、ChatWork / SMS で通知（`follow_urgency` に応じた強度）。

---

## 5. 内部処理ロジック

### 5.1 店舗特定ロジック

### 5.1.1 前提

- メール本文中の店舗名表記（漢字・カナ等）は誤記が多いため、
    
    自動店舗特定には使用しない。
    
- `to[]` に、必ず「該当店舗の店舗メールアドレス」が含まれている前提で運用する。

### 5.1.2 アルゴリズム

1. `to[]` の各要素からメールアドレス部分だけを抽出（`name <addr>` → `addr`）。
2. `shops.shop_email` と一致するレコードを検索。
3. 振る舞い:
    - 一意に 1 店舗にマッチした場合:
        - `shop_id` にその店舗IDをセット。
    - 0 件:
        - `shop_id = NULL` とし、`operator_mail_logs.shop_resolve_status='pending'` とする。
        - SHOP_PORTAL 側で「店舗を選択してください」UI を出す前提。
    - 複数件:
        - `shop_id = NULL`、`shop_resolve_status='ambiguous'`。
        - SHOP_PORTAL で選択させる。

### 5.2 AI分類仕様

### 5.2.1 AI_CORE_REQUEST（論理）

AIコアへの入力は概ね次のような JSON を想定する：

```json
{
  "channel": "op_mail",
  "shop_id": 123,
  "input": {
    "subject": "件名...",
    "body_text": "正規化済み本文...",
    "meta": {
      "from": "operator@example.com",
      "to": ["..."],
      "received_at": "2025-12-04T21:03:00+09:00"
    }
  }
}

```

### 5.2.2 AI_CORE_RESPONSE（論理）

AIからは以下のような分類結果を受け取る：

```json
{
  "follow_urgency": "high | normal | none",
  "incident_category": "door_lock_issue | reservation_mismatch | device_error | forgotten_item | equipment_usability | other",
  "follow_type": "compensation_or_refund | apology_and_explanation | thanks_and_improvement | information_only",
  "needs_compensation": true | false | null,
  "needs_store_reservation_check": true | false,
  "owner_contact_status": "not_called | reached | no_answer | line_busy | unknown",
  "resolution_status": "resolved | unresolved",
  "summary": "1〜2行の要約",
  "notes_for_owner": "加盟店が何を確認・対応すべきか一行で",
  "confidence": 0.0
}

```

- `owner_contact_status`
    - 「オペレーターが加盟店に電話したか / つながったか」の結果をメール文面から推定。
    - `no_answer` / `line_busy` の場合は、後述のとおり `follow_urgency` を強制的に `high` に引き上げる。

### 5.2.3 フォロー緊急度 `follow_urgency` の決定ルール

サーバ側では、AI応答を受けて次のように最終 `follow_urgency` を決定する。

1. まず AI 応答そのものを受け取る（`ai_follow_urgency`）。
2. 次に `owner_contact_status` を評価:
    - `owner_contact_status in ('no_answer', 'line_busy')` の場合:
        - 加盟店に本来リアルタイムで引き継ぐべきだったのに出来ていないとみなし、
            - `follow_urgency = "high"` に上書きする。
3. `resolution_status` との整合:
    - `resolution_status = 'resolved'` の場合は最終的に `follow_urgency = 'none'` とみなす。

### 5.3 解決済み / 未解決の判定とチケット起票

### 5.3.1 判定

- 解決済み（resolved）:
    - `resolution_status = 'resolved'`
        
        （例: 鍵の番号案内後に解錠確認済み、忘れ物を即回収済み 等）
        
- 未解決（unresolved）:
    - `resolution_status = 'unresolved'` で、かつ
    - `follow_urgency in ('high', 'normal')`

### 5.3.2 振る舞い

- 解決済み:
    - `operator_mail_logs` のみ作成。
    - SHOP_PORTAL には「○月△日 xx:xx 緊急連絡があり、オペレーター対応で解決済み」といった履歴カードを表示する（Phase4 側UI）。
    - `tickets` 起票なし。
- 未解決:
    - `operator_mail_logs` + `tickets` を作成。
    - `follow_urgency=high` / `normal` に応じた通知を行う。

### 5.4 tickets 起票仕様

### 5.4.1 `tickets.source` の扱い（他フェーズからの差分）

- AI_INBOX_POLICY / SHOP_PORTAL の記述に合わせ、
    
    `tickets.source` に `'operator_mail'` を新たに追加する。
    
- PHASE_00_DB_CORE の `tickets.source` enum に対する差分:
    - 既存: `'stores_form','customer_line','hp_form','portal_owner','portal_admin'`
    - 追加: `'operator_mail'`
- 互換性:
    - 既存の `'portal_admin'` は従来どおり本部起点の手動起票等に利用。
    - OP_MAIL 由来チケットは `source='operator_mail'` で統一し、
        
        `category` に `operator_mail.follow_high` 等を設定する。
        

### 5.4.2 起票内容

- 共通フィールド例:

| フィールド | 値の例 |
| --- | --- |
| source | `'operator_mail'` |
| shop_id | 店舗特定ロジックで決定した ID |
| owner_id | `shops.owner_id` |
| title / subject | `[OP] 鍵が開かずご利用できない可能性あり` |
| description/body | AI summary + 原文テキストの一部 |
| category | `operator_mail.follow_high` / `.follow_normal` |
| status | `open`（初期） |
| created_by_type | `system` |
| created_by_id | `NULL` |
- `follow_urgency` ごとのカテゴリ例:
    - high: `category='operator_mail.follow_high'`
    - normal: `category='operator_mail.follow_normal'`

### 5.5 「対応不要で完了」操作（SHOP_PORTAL）

### 5.5.1 加盟店 UI

- SHOP_PORTAL の OP_MAIL チケット詳細画面には常に以下の2ボタンを表示する:
    - [顧客に対応する] … LINE / メール + HELP 誘導フローへ
    - [対応不要で完了] … AI 誤判定・その場解決などで、追加対応不要と判断した場合

### 5.5.2 バックエンド動作

[対応不要で完了] 実行時:

1. `tickets` 更新:
    - `status = 'closed'`
    - `close_reason = 'op_mail_resolved_before'`（追加）
2. `operator_mail_logs` 更新:
    - `final_follow_urgency = 'none'`
    - `final_resolution_status = 'resolved'`
    - `final_decided_by = {現在のポータルユーザーID}`
    - `final_decided_at = NOW()`
3. フォローアップ系 CRON（SLA 追撃）の除外条件:
    - `status='closed' AND close_reason='op_mail_resolved_before'` を対象外とする。
4. 学習用途:
    - `ai_follow_urgency ≠ final_follow_urgency` の場合は、
        
        後述の `op_mail_training_samples` に抽出可能な候補としてフラグ付け。
        

---

## 6. SHOP_PORTAL / AI-INBOX との連携（概要）

※詳細な画面仕様・ルーティングは Phase4 / Phase5 側に委ね、本ドキュメントでは必要な I/O 前提のみ定義する。

### 6.1 SHOP_PORTAL 側

- 解決済み（ログのみ）:
    - 「OP_MAIL 緊急連絡履歴」セクションにカード表示。
- 未解決（チケット化）:
    - 通常の「要対応チケット一覧」に `source='operator_mail'` で混在表示。
    - チケット詳細には:
        - メール原文（プレーンテキスト）
        - AI summary / incident_category / follow_urgency
        - 顧客特定パネル（会員番号 / 予約ID 入力 → API で照合）
        - 「LINEで返信」「メールで返信」「対応不要で完了」ボタン
            
            （LINE紐づき有無に応じて有効/無効）
            

### 6.2 AI-INBOX 側（HQ）

- `source='operator_mail'` のチケットは、
    
    「電話代行」フィルタやバッジ表示で優先して確認できるようにする。
    
- close_reason が `op_mail_resolved_before` のものは
    
    「店舗判断で対応不要」として識別できる。
    

---

## 7. ログ・監査・学習データ

### 7.1 `operator_mail_logs` テーブル（提案DDL）

```sql
CREATE TABLE operator_mail_logs (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  provider VARCHAR(32) NOT NULL,
  provider_message_id VARCHAR(191) NOT NULL,
  message_id VARCHAR(191) NULL,
  shop_id BIGINT UNSIGNED NULL,
  shop_resolve_status ENUM('resolved','pending','ambiguous') NOT NULL DEFAULT 'resolved',

  raw_subject TEXT NOT NULL,
  raw_from VARCHAR(191) NOT NULL,
  raw_to TEXT NOT NULL,
  raw_cc TEXT NULL,
  body_text LONGTEXT NULL,
  body_html LONGTEXT NULL,
  headers_json LONGTEXT NULL,
  received_at DATETIME NOT NULL,

  -- AI判定
  ai_follow_urgency ENUM('high','normal','none') NULL,
  ai_incident_category VARCHAR(64) NULL,
  ai_follow_type VARCHAR(64) NULL,
  ai_needs_compensation TINYINT(1) NULL,
  ai_needs_store_reservation_check TINYINT(1) NULL,
  ai_owner_contact_status ENUM('not_called','reached','no_answer','line_busy','unknown') NULL,
  ai_resolution_status ENUM('resolved','unresolved') NULL,
  ai_summary TEXT NULL,
  ai_notes_for_owner TEXT NULL,
  ai_confidence DECIMAL(4,3) NULL,

  -- 最終判定（店舗/HQによる上書き）
  final_follow_urgency ENUM('high','normal','none') NULL,
  final_incident_category VARCHAR(64) NULL,
  final_resolution_status ENUM('resolved','unresolved') NULL,
  final_decided_by BIGINT UNSIGNED NULL,
  final_decided_at DATETIME NULL,

  ticket_id BIGINT UNSIGNED NULL,

  status ENUM('received','classified','ticket_created','closed','error')
    NOT NULL DEFAULT 'received',

  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,

  PRIMARY KEY (id),
  UNIQUE KEY uq_operator_mail_msg (provider_message_id),
  KEY idx_operator_mail_shop (shop_id),
  KEY idx_operator_mail_status (status),
  KEY idx_operator_mail_final (final_resolution_status, final_follow_urgency)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='電話代行メール取込ログ（AI判定＋最終判定含む）';

```

### 7.2 学習サンプルテーブル `op_mail_training_samples`

```sql
CREATE TABLE op_mail_training_samples (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  operator_mail_log_id BIGINT UNSIGNED NOT NULL,
  shop_id BIGINT UNSIGNED NULL,

  input_text LONGTEXT NOT NULL COMMENT '学習用に正規化した本文（件名＋要約＋本文等）',
  label_follow_urgency ENUM('high','normal','none') NOT NULL,
  label_incident_category VARCHAR(64) NULL,
  label_resolution_status ENUM('resolved','unresolved') NOT NULL,

  label_source ENUM('hq','shop','system') NOT NULL DEFAULT 'hq',
  is_active TINYINT(1) NOT NULL DEFAULT 1,

  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,

  PRIMARY KEY (id),
  KEY idx_op_mail_train_label (label_follow_urgency, label_incident_category),
  KEY idx_op_mail_train_log (operator_mail_log_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='OP_MAIL分類用 学習サンプル';

```

### 7.3 学習データ投入フロー（要件レベル）

- HQ / 本部管理UI から:
    - `operator_mail_logs` の一覧を検索・絞り込み（期間・店舗・AI判定 vs 最終判定の差分など）。
    - 各レコードに対し、人間が `final_*` を編集可能。
    - 「この判定を学習サンプルにする」チェックONで `op_mail_training_samples` にコピー。
- 学習（モデル改善）自体は AI_CORE / バッチジョブ側の責務とし、
    
    本ドキュメントでは「学習サンプルの構造と SoR」を定義するに留める。
    

### 7.4 sys_events / audit_logs との連携

- `sys_events`:
    - `op_mail.ingest_received` / `op_mail.classify_failed` / `op_mail.ticket_created` / `op_mail.notify_failed` などを記録。
- `audit_logs`:
    - `tickets` の status / close_reason 変更（特に「対応不要で完了」）は監査対象とする。

---

## 8. セキュリティ・無害化・プライバシー

### 8.1 Webhook 認証

- `X-Op-Mail-Token` による固定トークン認証。
- Web サーバ / FW による IP 制限。
- 対応可能であれば、プロバイダ署名（HMAC）を検証し、署名不一致時は 401 を返す。

### 8.2 HTMLメール無害化

- `body_html` は DB に保持しても良いが、
    
    UIに表示する際は必ず:
    
    - `script` / `style` / `iframe` 等の危険タグ除去
    - URL クリック時のターゲット制御（別タブ + 警告）
- SHOP_PORTAL / AI-INBOX では、基本的にプレーンテキスト表示（`body_text`）を優先し、
    
    HTMLは折りたたみ表示する。
    

### 8.3 個人情報・保持期間

- 電話番号 / メールアドレス / 氏名 等の PII を含むため:
    - `operator_mail_logs` の保持期間は最低 X ヶ月（要件に応じて設定）とし、
        
        それ以上はローテーション / アーカイブを検討。
        
    - 学習サンプルでは、本名を直接使わない（可能な範囲で伏字や置き換えも検討）。

---

## 9. CRON_SYS との関係

- OP_MAIL 自体は cron ではなく Push トリガで動作する。
- ただし、以下のような二次的な監視ジョブを CRON_SYS に追加する余地はある（任意）:
    - `op_mail_health_check`:
        - `operator_mail_logs.status='error'` や、
            
            `received_at` から一定時間経っても `ticket_id` が NULL のレコードを検知し、
            
            HQ にアラートする。
            
- SLA 追撃（SMS / ChatWork）は、既存の `cron_sms_followup.php` 等のロジックを流用し、
    
    `source='operator_mail'` かつ `category='operator_mail.follow_high'` 等も対象に含める。
    

---

## 10. 将来拡張

- 添付ファイル（現場写真）の UI 連携（SHOP_PORTAL / AI-INBOX 表示）。
- 音声録音 → 文字起こし → OP_MAIL 連携（電話録音からの自動ログ生成）。
- 電話代行システムからのリアルタイム API 連携（「加盟店への呼び出し履歴」等を追加で取り込む）。
- AIコア側でのモデル再学習:
    - `op_mail_training_samples` を用いたファインチューニングや
        
        embedding + 近傍検索による few-shot 提供。
        

---

## 11. Phase7 から他フェーズへの差分まとめ

- PHASE_00_DB_CORE:
    - `tickets.source` enum に `'operator_mail'` を追加する。
    - 新テーブル `operator_mail_logs`, `op_mail_training_samples` を DB_CORE に編入。
- PHASE_04_SHOP_PORTAL:
    - OP_MAIL チケット詳細画面に:
        - メール原文表示
        - 顧客特定パネル（会員番号入力 → API 照合）
        - 「LINEで返信」「メールで返信」「対応不要で完了」ボタン
        - 解決済みログの履歴カード表示
- PHASE_05_AI_INBOX:
    - `source='operator_mail'` チケットに対するフィルタ / バッジ表示。
    - close_reason `'op_mail_resolved_before'` の表示文言。
- AI_CORE:
    - `channel='op_mail'` 用 classification スキーマを本仕様に合わせて実装。

以上。
