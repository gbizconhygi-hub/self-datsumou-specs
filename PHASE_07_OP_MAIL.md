# PHASE_07_OP_MAIL.md — OP_MAIL（電話代行メール解析）仕様 v1.4

## 1. コンテキスト・位置づけ

### 1.1 要素名・スコープ

- 要素名: OP_MAIL（電話代行メール解析）
- フェーズ: Phase7
- レイヤ: AI_CORE 配下のサブモジュール
- 本フェーズの目的:
    - 電話代行オペレーターからの「緊急連絡メール」を受信し、
        - 店舗を自動特定（可能な範囲で）
        - AI で「解決済みか / 追加フォローが必要か」「フォロー緊急度」を判定
        - 未解決案件のみ `tickets` としてタスク化し、ChatWork / SMS / SHOP_PORTAL に連携
        - すべてのメールを `operator_mail_logs` にログとして残し、学習・監査の SoR とする

### 1.2 他フェーズとの関係

- `spec_v1.4_base.md`
    - OP_MAIL の思想（電話代行の緊急報告メールを AI で分類し、要対応のみチケット化）の具体化フェーズ。
- `PHASE_04_SHOP_PORTAL.md`
    - OP_MAIL 由来チケットを「電話代行緊急連絡」として店舗側に表示し、
    LINE / メール（noreply + HELP）による顧客対応フローを提供する。
- `PHASE_05_AI_INBOX.md`
    - HQ 向け INBOX として、全チャネル（help./line./shop./OP_MAIL）の `tickets` を統合管理する。
- `PHASE_07_CRON_SYS.md`
    - CRON_SYS とは独立。OP_MAIL は **Push トリガ専用** とし、cron によるメールポーリングは行わない。
- `PHASE_03_LINE_UI.md` / AI_CORE
    - `channel='op_mail'` として AI コアに分類処理を委譲し、
    「フォロー緊急度」「インシデント種別」「加盟店への電話結果」などのラベルを返してもらう。
    - LINE↔STORES 会員の紐づけは、自社 DB（`customer_links` 等）側で保持。

---

## 2. 用語と前提

### 2.1 用語

- 電話代行オペレーター
    
    外注コールセンターを想定。お客様との電話を受け、一次対応と加盟店へのエスカレーションを行う。
    
- 緊急連絡メール
    
    オペレーターが一次対応後に送る業務報告メール。
    
    対象は「無人サロンの運営に影響するインシデント（鍵・機械・設備・クレーム等）」。
    
- 解決済み（resolved）
    
    電話中にオペレーター対応のみで完結しており、加盟店からの追加連絡が不要な案件。
    
    例:
    
    - 暗証番号を誤って入力していた → 正しい番号を案内し、解錠確認済み。
    - 忘れ物 → 数分以内に再入室してお客様自身で回収済み。
- 未解決（unresolved）
    
    オペだけでは完結せず、「加盟店による STORES 予約確認」「補償判断」「折返し連絡」等が必要な案件。
    
    例:
    
    - 謝罪の上お帰りいただいた。
    - 予約が確認できていないように見える。
    - 加盟店に電話したが出ず、そのまま終話せざるを得なかった。

### 2.2 本フェーズの前提

- 真の「超緊急」（電話が切れないレベル）の案件は、
    
    オペレーターが **その場で加盟店に電話してエスカレーション** する運用とする。
    
    → それで対応できた案件については、OP_MAILでの再エスカレーションは不要。
    
- STORES 予約状況の最終判断は **加盟店のみが行える**。
    - OP_MAIL は STORES を直接参照せず、
    - SHOP_PORTAL 上で加盟店が STORES 画面を見ながら「会員番号 / 予約ID」を入力し、
    バックエンドが API で照合する形で本人特定・LINE紐づき確認を行う。
- LINE ID は STORES には存在せず、
    
    LINE↔STORES 会員の紐づけは自社 DB（`customer_links` 等）を SoR とする。
    
    → 「LINEで返信」可否は、`customer_links(channel='line')` によって判定する。
    

---

## 3. メール受信アーキテクチャと I/O 契約

### 3.1 全体像（Push 型）

1. 電話代行オペレーターは、緊急連絡用共通アドレス（例: `emergency@self-datsumou.net`）を To に指定し、
    - 必ず該当店舗のメールアドレス（`shops.shop_email`）も To に含めて送信する。
2. メールサーバ / プロバイダ側で、新着メールを HTTP Webhook として `op_mail_ingest` エンドポイントに POST する。
3. `op_mail_ingest` がメール内容を受け取り、
    - 店舗特定（Toヘッダ）
    - AI分類（フォロー緊急度・インシデント種別・加盟店への電話結果など）
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
            - `.env` などに `OP_MAIL_WEBHOOK_TOKEN` として格納。
    - ネットワークレベル:
        - 送信元 IP を firewall / Web サーバ設定で制限（電話代行メールサーバの IP のみ許可）。
    - オプション:
        - プロバイダ側が HMAC 署名をサポートする場合、`X-Signature` 等を追加検証。

### 3.3 リクエスト payload（抽象仕様）

プロバイダに依存しない共通形式として、以下の JSON を受け取る前提とする。

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
  "headers_json": "{... 生ヘッダのJSON ...}",
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

- 必須:
    - `provider`, `provider_message_id`, `subject`, `from`, `to[]`, `received_at`
- `body_text` が無い場合は、サーバ側で HTML→プレーンテキスト変換を行う。

### 3.4 レスポンス

- 正常（新規処理）

```json
{
  "status": "ok",
  "op_mail_log_id": 12345,
  "ticket_created": true,
  "ticket_id": 67890
}

```

- 正常（重複）

```json
{
  "status": "duplicate",
  "op_mail_log_id": 12345
}

```

- 認証エラー: HTTP 401
- バリデーションエラー: HTTP 400 + `{ "error": "..." }`
- サーバ内部エラー: HTTP 500

---

## 4. シーケンス（受信〜分類〜タスク化〜通知）

### 4.1 高レベルシーケンス

1. Webhook 受信
    - 認証・JSON バリデーションを実行。
2. 重複チェック
    - `provider_message_id` をキーに `operator_mail_logs` を検索。
    - 既に存在すれば `status=duplicate` で早期リターン（idempotent）。
3. 店舗特定
    - `to[]` のメールアドレスと `shops.shop_email` を突き合わせて `shop_id` を決定。
4. 正規化テキスト生成
    - 件名 + 本文を元に `normalized_text` を生成（HTML→テキスト変換 + 不要署名の削除など）。
5. AI コア呼び出し
    - `channel='op_mail'` として `normalized_text` + メタ情報を AI_CORE に送信。
6. AI 応答に基づく分類
    - `resolution_status`（resolved/unresolved）
    - `follow_urgency`（high/normal/none）
    - `incident_category` 等を取得。
7. `operator_mail_logs` への記録
    - 生メール + AI 判定 + 店舗 ID + ステータスを 1 レコードにまとめる。
8. 解決済み / 未解決 の分岐
    - `resolution_status='resolved'` or `follow_urgency='none'`
        
        → ログのみ（SHOP_PORTAL には「緊急連絡があった」履歴カード）。
        
    - それ以外（未解決）
        
        → `tickets` にタスク起票。
        
9. 通知
    - 未解決案件のみ、ChatWork / SMS で通知（`follow_urgency` に応じた強度）。

---

## 5. 内部処理ロジック詳細

### 5.1 店舗特定ロジック

### 5.1.1 ポリシー

- メール本文中の店舗名表記（漢字・カナ等）は誤記が多いため、**自動店舗特定には使用しない**。
- 自動判定の唯一の根拠は `to[]` に含まれる店舗メールアドレス（`shops.shop_email`）とする。

### 5.1.2 アルゴリズム

1. `to[]` の各要素からメールアドレス部分のみ抽出する。
2. `shops` テーブルで `shop_email` が一致する店舗を検索。
3. 結果:
    - 一意に 1 件:
        - `shop_id` にその ID をセット。
        - `shop_resolve_status='resolved'`。
    - 0 件:
        - `shop_id = NULL`
        - `shop_resolve_status='pending'`
        - SHOP_PORTAL 側で「店舗を選択してください」UI を出す想定。
    - 複数件:
        - `shop_id = NULL`
        - `shop_resolve_status='ambiguous'`
        - SHOP_PORTAL 側で選択させる。

### 5.2 AI分類仕様

### 5.2.1 AI_CORE_REQUEST（論理）

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

- `follow_urgency`
    - high: 早め（基本その日中・場合によっては即）にフォローすべき。
    - normal: 今日〜数日以内にフォローできれば良い。
    - none: フォロー不要（オペで完結）。
- `resolution_status`
    - resolved: その場で完全に解決しており、追加フォロー不要。
    - unresolved: 加盟店による追加対応が必要。
- `owner_contact_status`
    - `not_called`: そもそも加盟店には電話していない。
    - `reached`: 加盟店に電話がつながり、引き継ぎ済み。
    - `no_answer`: 呼び出しはしたが出なかった。
    - `line_busy`: 話し中等でつながらなかった。
    - `unknown`: メール文面から判別できない。

### 5.2.3 フォロー緊急度の最終決定

サーバ側では、AI 応答を受けて以下のルールで最終 `follow_urgency` を決定する。

1. AI応答を `ai_follow_urgency` として受け取る。
2. `owner_contact_status` を評価:
    - `owner_contact_status in ('no_answer','line_busy')` の場合:
        - 本来リアルタイムで引き継ぐべき案件が未対応のまま終話しているとみなし、
            
            `follow_urgency = "high"` に上書きする（AI 判定より優先）。
            
3. `resolution_status` を評価:
    - `resolution_status='resolved'` の場合は、最終的に `follow_urgency='none'` とみなす。

この「AI判定 vs ルール上書き vs 最終値」は `operator_mail_logs` に両方保持し、

学習時には最終値をラベルとして使用する。

### 5.3 解決済み / 未解決の分岐とチケット化

### 5.3.1 判定

- 解決済み:
    - `resolution_status='resolved'`
    - もしくはルール上 `follow_urgency='none'` となったもの。
- 未解決:
    - `resolution_status='unresolved'` かつ `follow_urgency in ('high','normal')`。

### 5.3.2 振る舞い

- 解決済み:
    - `operator_mail_logs` のみ作成。
    - SHOP_PORTAL には「〇月△日 xx:xx 緊急連絡があり、オペレーター対応で解決済み」といった履歴カードを表示（Phase4 側）。
    - `tickets` 起票は行わない。
- 未解決:
    - `operator_mail_logs` + `tickets` を作成。
    - SHOP_PORTAL の「要対応チケット一覧」に表示される。
    - フォロー緊急度に応じて通知レベル・SLA を変える（後述）。

### 5.4 `tickets` 起票仕様

### 5.4.1 `tickets.source` の扱い

- `tickets.source` の enum に `'operator_mail'` を追加し、
    
    OP_MAIL由来のチケットは `source='operator_mail'` で統一する。
    
- 既存の `'portal_admin'` は本部起点の手動起票用として維持。

### 5.4.2 category / priority 相当

- `category` 例:
    - `operator_mail.follow_high`
    - `operator_mail.follow_normal`
- priority カラムは存在しないため、
    
    緊急度は `category` と AI-INBOX / SHOP_PORTAL 側の UI ロジックで表現する。
    

### 5.4.3 起票内容例

| フィールド | 値の例 |
| --- | --- |
| source | `operator_mail` |
| shop_id | 店舗特定ロジックで決定した ID |
| owner_id | `shops.owner_id` |
| subject/title | `[OP] 鍵が開かずご利用できない可能性あり` など |
| body/description | AI summary + 報告メール本文のテキスト |
| category | `operator_mail.follow_high` / `.follow_normal` |
| status | `open` |
| created_by_type | `system` |
| created_by_id | `NULL` |

---

## 6. SHOP_PORTAL / AI-INBOX 連携（挙動の前提）

詳細な画面仕様は各 Phase に委ね、本フェーズでは挙動の「前提条件」のみ定義する。

### 6.1 SHOP_PORTAL 側（加盟店UI）

- 解決済み (`resolution_status='resolved'`):
    - 「OP_MAIL 緊急連絡履歴」セクションに履歴カードとして表示。
    - クリックで `operator_mail_logs` の詳細（報告全文）を閲覧できる。
    - タスク（要対応チケット）には出さない。
- 未解決（タスク化）:
    - 通常の「要対応チケット一覧」に `source='operator_mail'` で表示。
    - チケット詳細には:
        - 店舗名 / 受信日時 / AI summary / incident_category / follow_urgency
        - 電話代行報告メール全文（プレーンテキスト表示）
        - 顧客特定パネル:
            - 「STORES管理画面で該当の会員 / 予約を開いてください」
            - 「会員番号または予約IDを入力してください」
            - [照合] ボタン → サーバが STORES API / 自社 DB で確認
        - 連絡ボタン:
            - LINE紐づきがある場合:
                - [LINEで返信] ボタン有効
            - 紐づきが無い場合:
                - [メールで返信] ボタン（noreply + HELP誘導）
            - 常に:
                - [対応不要で完了] ボタン（AI誤判定・その場解決時）

### 6.2 SHOP_PORTAL からの「対応不要で完了」

- [対応不要で完了] 実行時:
    - `tickets.status='closed'`
    - `tickets.close_reason='op_mail_resolved_before'`（追加）
    - `operator_mail_logs.final_follow_urgency='none'`
    - `operator_mail_logs.final_resolution_status='resolved'`
    - `operator_mail_logs.final_decided_by` / `final_decided_at` を更新
- このチケットは:
    - SLAフォローアップCRONの対象外とする。
    - AI-INBOX では「店舗判断で対応不要」バッジ付きで closed として表示する。

### 6.3 AI-INBOX 側（HQ UI）

- `source='operator_mail'` のチケットは、フィルタやバッジで「電話代行」と明示。
- `close_reason='op_mail_resolved_before'` の件は、
    
    AI誤判定や「その場解決だった案件」として集計・モニタリングが可能。
    

---

## 7. ログ・監査・学習データ

### 7.1 `operator_mail_logs` テーブル（DDL 概要）

詳細DDLは `DB_PATCH_07_OP_MAIL.md` を参照。

本仕様書では保持すべき情報の構造を定義する。

- 生メール情報:
    - provider / provider_message_id / message_id
    - shop_id / shop_resolve_status
    - raw_subject / raw_from / raw_to / raw_cc
    - body_text / body_html / headers_json
    - received_at
- AI判定:
    - ai_follow_urgency / ai_incident_category / ai_follow_type
    - ai_needs_compensation / ai_needs_store_reservation_check
    - ai_owner_contact_status / ai_resolution_status
    - ai_summary / ai_notes_for_owner / ai_confidence
- 最終判定（店舗・HQの上書き結果）:
    - final_follow_urgency / final_incident_category / final_resolution_status
    - final_decided_by / final_decided_at
- 紐づく tickets:
    - ticket_id
- 内部ステータス:
    - status（received/classified/ticket_created/closed/error）

### 7.2 学習サンプル `op_mail_training_samples`

- 用途:
    - OP_MAIL 用 AI分類の教師データ。
- カラム（概要）:
    - operator_mail_log_id（元メールへの参照）
    - shop_id
    - input_text（学習用に正規化した本文）
    - label_follow_urgency / label_incident_category / label_resolution_status
    - label_source（hq/shop/system）
    - is_active

### 7.3 本部管理UIからの勉強データ投入（要件レベル）

- HQ 向け管理画面（AI管理UIなど）で:
    - `operator_mail_logs` の一覧を期間・店舗・AI判定 vs 最終判定の差分などで絞り込み。
    - 個別レコードの `final_*` ラベルを編集できる。
    - 「このレコードを学習サンプルとして登録」チェックONで `op_mail_training_samples` にコピー。
- モデル学習そのものは AI_CORE / バッチジョブ側の責務とし、
    
    本仕様では「学習サンプル構造とソース」を定義する。
    

### 7.4 sys_events / audit_logs 連携

- `sys_events`:
    - `op_mail.ingest_received`
    - `op_mail.classify_failed`
    - `op_mail.ticket_created`
    - `op_mail.notify_failed`
        
        などのイベントを出力。
        
- `audit_logs`:
    - `tickets` の status / close_reason 変更（特に「対応不要で完了」）を監査対象とする。

---

## 8. セキュリティ・無害化・プライバシー

### 8.1 Webhook 認証

- `X-Op-Mail-Token` による固定トークン認証。
- FW / Webサーバレベルで IP 制限。
- プロバイダ署名対応時は HMAC 署名も併用。

### 8.2 HTML 無害化

- HTMLを DB に保持する場合でも、UI表示時には:
    - `script` / `style` / `iframe` 等の危険タグ除去。
    - リンクは別タブ＋注意喚起。
- SHOP_PORTAL / AI-INBOX では、プレーンテキストを基本とし、HTMLは折りたたみ表示。

### 8.3 PII と保持期間

- 電話番号 / メールアドレス / 氏名等の PII を含むため:
    - `operator_mail_logs` の保持期間は運用に応じて上限を設け、古いログはアーカイブ / 削除方針を別途定義。
    - `op_mail_training_samples` は、必要に応じて匿名化・伏字を検討（直接本名を学習に使わないなど）。

---

## 9. CRON_SYS との関係

- OP_MAIL 自体は cron ではなく **Push トリガ** で動作する。
- ただし、以下のような二次的ヘルスチェックジョブを CRON_SYS に追加する余地はある（任意）:
    - `op_mail_health_check`:
        - `status='error'` のログや、
        - `received_at` から一定時間経過しても `ticket_created` になっていないレコードを検知し、HQ にアラートする。
- SLA 追撃（SMS / ChatWork）は、既存の `cron_sms_followup.php` 等のロジックを流用し、
    
    `source='operator_mail'` かつ `category like 'operator_mail.follow_%'` も対象に含める。
    

---

## 10. 将来拡張

- 添付ファイル（現場写真）の UI 連携（SHOP_PORTAL / AI-INBOX 表示）。
- 音声録音 → 文字起こし → OP_MAIL 連携（電話録音からの自動ログ生成）。
- 電話代行システムからのリアルタイム API 連携（「加盟店への呼び出し履歴」等を追加で取り込む）。
- AIコア側でのモデル再学習:
    - `op_mail_training_samples` を用いたファインチューニングや
        
        embedding + 近傍検索による few-shot 提供。
        
- STORES予約と OP_MAIL の ID 連携を強化し、
    
    将来的に OP メール側に STORES 会員番号を直接埋め込むフロー（今回の仕様はその前段として設計）。
    

---

## 11. 他フェーズへの差分まとめ

- PHASE_00_DB_CORE:
    - `tickets.source` enum に `'operator_mail'` を追加する。
    - 新テーブル `operator_mail_logs`, `op_mail_training_samples` を DB_CORE に編入する。
- PHASE_04_SHOP_PORTAL:
    - OP_MAIL チケット詳細画面に:
        - 電話代行報告メール全文表示
        - 顧客特定パネル（会員番号 / 予約ID 入力 → API 照合）
        - 「LINEで返信」「メールで返信」「対応不要で完了」ボタン
        - 解決済みログの履歴カード表示
- PHASE_05_AI_INBOX:
    - `source='operator_mail'` チケットに対するフィルタ / バッジ表示。
    - `close_reason='op_mail_resolved_before'` の表示文言（「店舗判断で対応不要」など）。
- AI_CORE:
    - `channel='op_mail'` 用 classification スキーマを本仕様に合わせて実装する。
