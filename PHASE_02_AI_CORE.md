# AI_CORE - 仕様設計書 (PHASE_02_AI_CORE.md Draft)

## 1. 目的・範囲

### 1.1 目的

本フェーズ（Phase 2：AI_CORE）は、以下 4 チャネルのすべてから共通で呼び出される

「AIコア（意図分類＋回答生成＋フォールバック＋ログ）」の仕様を定義する。

- エンドユーザー：`help.`（Web）／`line.`（LINE）
- 加盟店：`shop.`（オーナーポータル）
- 本部：`ai.`（本部管理画面）

AI_CORE の主な役割：

- ユーザー発話を受け取り、
    - インテント（意図）の分類
    - FAQ／シナリオ／テンプレ回答の選択・生成
    - フォールバック（テンプレ／有人／チケット化）の判断
    を行い、応答文＋メタ情報を返す。
- Phase1 で定義した SYS_CORE（`SysConfig` / `SysLogger` / `SysEscalation` / `tickets` 等）と連携し、
AI 応答ログとチューニング用メタ情報を一貫して記録・活用できる状態を作る。

### 1.2 範囲

本フェーズで扱うもの：

- AI_CORE の I/O 契約（AI_CORE_REQUEST / AI_CORE_RESPONSE）
- AI_CORE 内部の処理フロー（インテント分類〜フォールバック判断）
- AI 会話ログのデータ構造（`ai_sessions` / `ai_turns`）と `tickets` との紐づけ
- LLM 呼び出しの基本方針（SYS_CORE 経由、モデル指定、閾値の考え方）

本フェーズで扱わない（別フェーズで定義する）もの：

- 各チャネルの UI／文言（画面ラフ・ボタン配置など）
- STORES予約／RemoteLOCK との具体的な API I/O（Phase3：API_COMM）
- FAQ／シナリオ／テンプレ本文そのものの管理画面仕様
- 「セルフ解決しました」など、ユーザーからの明示フィードバック UI 詳細（チュートリアル層フェーズ）

---

## 2. 前フェーズからの継承ポイント

### 2.1 システム構成（Phase0 / spec_v1.4_base）

- サブドメイン構成
    - エンドユーザー：`help.self-datsumou.net` / `line.self-datsumou.net`
    - 加盟店：`shop.self-datsumou.net`
    - 本部：`ai.self-datsumou.net`
- 共通ライブラリ構造
    - `shared/` 配下に共通機能を集約する方針
    - AI_CORE は `shared/ai_core.php`（仮称）として実装される前提

### 2.2 DB コア（PHASE_00_DB_CORE / db_fields.yaml）

- 主な既存テーブル（抜粋）
    - `owners`：オーナー情報
    - `shops`：店舗マスタ
    - `shop_secrets`：店舗別シークレット（APIキー等）
    - `options_master`：全体設定マスタ
    - `shop_options`：店舗別設定（`options_master` の上書き）
- AI_CORE は、これらのテーブルを参照しつつ、
AI会話ログ用に新規テーブル（`ai_sessions` / `ai_turns`）を追加する。

### 2.3 SYS_CORE 連携（PHASE_01_SYS_CORE）

- SYS_CORE の責務（要点）
    - `SysConfig`：`options_master` / `shop_options` / `.env` から有効な設定値を解決
    - `SysLogger`：`api_call_logs` / `sys_events` / `tickets` など共通ログの窓口
    - `SysEscalation`：ChatWork／メール／SMS／チケット化など有人対応窓口
    - HTTPクライアント：OpenAI／STORES／RemoteLOCK 等の外部API呼び出し共通レイヤー
- AI_CORE からの呼び出し方針
    - LLM 呼び出しは必ず SYS_CORE の HTTP クライアント経由とし、
    AI_CORE は API キーやベースURLを直接扱わない。

### 2.4 トーン／スタイル方針（共通ルール）

- エンドユーザー向け（help./LINE）
    - 「人間が丁寧に説明している」トーン
    - 顔文字や柔らかめの表現を適度に使用し、機械感を減らす
- 加盟店向け（shop.）
    - ぶっきらぼう NG
    - 疑問や不安に寄り添うトーン
- 本部向け（ai.）
    - 事務的だが冷たすぎない、簡潔で誤解のない説明

AI_CORE のプロンプト／テンプレは、このトーン方針を前提とする。

---

## 3. 新規決定事項

### 3.1 決定事項

1. 頭脳は 1 つ・チャネル差はパラメータで吸収
    - help. / line. / shop. / ai. は、すべて共通の AI_CORE を利用する。
    - チャネル差（顧客／加盟店／本部）は、プロンプト・温度・履歴長さなどのパラメータで吸収する。
2. LLM 呼び出しは SYS_CORE 経由
    - AI_CORE は LLM を直接叩かず、SYS_CORE 提供の HTTP クライアントを通じて実行する。
    - モデル名やベースURL、APIキーは `.env` / `shop_secrets` / `options_master` から `SysConfig` が解決。
3. STORES／RemoteLOCK との役割分担
    - AI_CORE：
        - インテント分類
        - 「これは予約変更フロー」「これは解錠番号フロー」などのルーティング判断
    - API_COMM：
        - 実際の STORES／RemoteLOCK API 呼び出し
        - 取得データを元にした最終値（予約情報・解錠番号など）の決定
4. モデル指定はすべてパラメータ管理
    - コードにモデル名を固定せず、
        - `ai_core.default_model`
        - `ai_core.model_per_channel.help`
        - `ai_core.model_per_channel.shop` など
        を `options_master` / `shop_options` 経由で指定可能にする。
5. チャネル別“性格”設定
    - 温度・max_tokens・履歴ターン数などは`ai_core.temperature.*` / `ai_core.max_tokens.*` / `ai_core.max_history_turns.*` といったキーで管理。
    - 初期値は「安全寄り」（暴れすぎない／過激な生成を避ける）で設定する。
6. インテント信頼度とフォールバック方針
    - 加盟店→本部エスカレーション
        - 「なるべくAIで踏ん張る」「かなり慎重にエスカレーション」
        - よほど信頼度が低い／危険（クレーム・法的リスク等）でない限り即チケット化しない。
    - 顧客→加盟店エスカレーション
        - 「なるべくAIで踏ん張る」前提。
        - 予約や解錠番号など、AIだけで完結しづらいものだけをエスカレーション候補とする。
    - リリース初期も、「ある程度 AI に任せる」方向でスタートし、ログを見ながら閾値をチューニングする。
7. ログ構造：`ai_sessions` / `ai_turns` の二層構造を採用
    - `ai_sessions`：会話単位のメタ情報（チャネル／店舗／解決状況など）
    - `ai_turns`：個々の発話単位（ユーザー発話／AI応答／インテント／トークン数など）
8. `tickets` への `ai_session_id` 追加
    - `tickets` テーブルに `ai_session_id` カラムを追加し、
    「このチケットはどのAI会話から生まれたか」を辿れるようにする。
9. セッション切断の共通ルール
    - 「最終アクティビティから 15 分以上経過したら、新規セッションとして扱う」
    - このルールは全チャネル共通とする（Phase2 時点）。
10. 生テキストログと PII の扱い
    - 個人情報が含まれていないと意味をなさないログは「短期間のみフルテキスト保持」。
    - マスク後も意味があるログは、マスク済みで長期間保持。
    - 個人情報特定につながる情報（電話番号・メール・予約番号 等）は、一定期間後にマスキングする。
    - ログ量がシステムに負荷をかけない範囲に収まるよう、保持期間・マスキングで調整する。
11. 加盟店からのログ閲覧粒度
    - 自動応答も含め、店舗ごとの会話履歴は `shop.` から閲覧可能とする（PIIマスク適用前提）。
12. KPI としてのセルフ解決率
    - `ai_sessions` に「セルフ解決フラグ」を持ち、
    セルフ解決率を KPI として本部内部で追う。
    - 加盟店には「セルフ解決率を数値として見せる」ことは想定せず、
    本部のプロジェクト評価・改善指標として利用する。

### 3.2 未決事項（Phase2 内での検討対象）

- インテントマスタの具体的な保存方式
    - 専用テーブル（例：`ai_intents_master`）を設計するか、`options_master` の一カテゴリとして扱うか。
- FAQ／シナリオデータのテーブル構成
    - AI_CORE フェーズで最低限の構造まで定義するか、
    チュートリアル層フェーズで集中的に扱うか。
- 閾値・保持期間の具体値
    - 各種信頼度閾値（high/low）
    - フルテキスト保持日数／マスキングまでの猶予期間

これらは、数値やテーブル構造の具体案を出した上で、このフェーズ内で確定させる。

---

## 4. データ構造 / I/O 定義

### 4.1 AI_CORE_REQUEST（入力）

```json
{
  "request_id": "uuid-string",              // ログ相関用の一意ID（未指定時はSYS_CORE側で採番）
  "channel": "help | line | shop | ai",    // 呼び出し元チャネル
  "actor_type": "customer | merchant | hq",// ユーザー種別
  "tenant_code": "hygi",                   // 将来のマルチテナント拡張用（v1.4では固定想定）
  "shop_id": 123,                          // 対象店舗ID（不明な場合はnull）
  "owner_id": 456,                         // 加盟店オーナーID（加盟店チャット時）
  "end_user_id": "string-or-null",         // 会員ID／LINE IDなど（匿名の場合はnull）
  "locale": "ja-JP",
  "timezone": "Asia/Tokyo",

  "user_message": "string",                // 最新のユーザー発話
  "conversation_history": [
    {
      "role": "user | assistant | system",
      "content": "string",
      "timestamp": "2025-11-17T01:23:45+09:00"
    }
  ],

  "context": {
    "shop_basic": {
      "name": "セルフ脱毛サロン ハイジ名古屋〇〇店",
      "region_name": "名古屋",
      "shop_status": "open"
    },
    "user_profile": {
      "is_member": true,
      "gender": "male | female | unknown | null",
      "age_range": "20s | 30s | ... | null"
    },
    "session": {
      "ai_session_id": 9999,               // 既存セッションID。新規の場合はnull
      "channel_session_key": "LINEのuserIdなど"
    },
    "runtime_flags": {
      "debug_mode": false,
      "force_intent": null                 // デバッグ用のインテント固定（通常運用ではnull）
    }
  },

  "options": {
    "max_tokens": 800,                     // LLM呼び出しに渡すmax_tokens（未指定時はSysConfigの値）
    "temperature": 0.3,
    "top_p": 0.9
  }
}

```

### 4.2 AI_CORE_RESPONSE（出力）

```json
{
  "request_id": "uuid-string",        // 入力と同じID
  "ai_session_id": 9999,              // AIセッションID（新規セッションの場合はここで採番結果を返す）

  "reply_text": "string",             // ユーザーに返却するメイン応答文
  "reply_text_alt": [                 // （任意）代替案／分割用テキスト
    "string"
  ],

  "intent": {
    "key": "faq.booking.change",      // インテントキー（マスタ参照）
    "label": "予約変更・キャンセル",
    "confidence": 0.87                // 0〜1 のスコア
  },

  "classification": {
    "category": "booking",           // 大分類
    "subcategory": "change",         // 小分類
    "urgency": "normal | high | low" // 緊急度（クレーム／鍵トラブル等）
  },

  "answer": {
    "type": "faq | scenario | template | generated | handoff",
    "source_id": 1234,               // FAQやシナリオID（無い場合はnull）
    "reason": "high_confidence_faq"  // 採用理由（ログ／デバッグ用）
  },

  "escalation": {
    "action": "none | create_ticket | transfer_to_human",
    "reason": "low_confidence | policy_block | user_request | system_error",
    "ticket_draft": {
      "title": "[名古屋〇〇店] 予約変更の問い合わせ",
      "body_preview": "ユーザー: ...\nAI案内: ...",
      "priority": "low | normal | high",
      "tags": ["ai-escalation", "booking"]
    }
  },

  "logging": {
    "log_level": "info | warn | error",
    "should_log_prompt": true,
    "should_log_answer": true
  },

  "meta": {
    "model_name": "resolved-by-SysConfig", // 使用モデル名
    "prompt_tokens": 123,
    "completion_tokens": 456,
    "total_tokens": 579,
    "latency_ms": 842
  }
}

```

### 4.3 ログ用データモデル（DDL案）

### 4.3.1 `ai_sessions`（会話単位）

```sql
CREATE TABLE ai_sessions (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'AIセッションID',
  channel ENUM('help','line','shop','ai') NOT NULL COMMENT 'チャネル種別',
  actor_type ENUM('customer','merchant','hq') NOT NULL COMMENT 'ユーザー種別',
  tenant_code VARCHAR(32) NOT NULL DEFAULT 'hygi' COMMENT 'テナントコード（将来拡張用）',
  shop_id BIGINT NULL COMMENT '対象店舗ID（不明時NULL）',
  owner_id BIGINT NULL COMMENT '加盟店オーナーID',
  end_user_id VARCHAR(191) NULL COMMENT 'エンドユーザーID（会員ID／LINE ID等）',
  channel_session_key VARCHAR(191) NULL COMMENT 'チャネル固有セッションキー（LINE userId等）',

  started_at DATETIME NOT NULL COMMENT 'セッション開始日時',
  last_activity_at DATETIME NOT NULL COMMENT '最終アクティビティ日時',
  status ENUM('active','closed') NOT NULL DEFAULT 'active' COMMENT 'セッション状態',

  resolved_status ENUM('unknown','resolved_by_ai','resolved_by_human','abandoned')
    NOT NULL DEFAULT 'unknown' COMMENT '解決ステータス',
  is_self_solved TINYINT(1) NOT NULL DEFAULT 0 COMMENT 'セルフ解決できたか',

  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,
  PRIMARY KEY (id),
  KEY idx_ai_sessions_channel (channel),
  KEY idx_ai_sessions_shop (shop_id),
  KEY idx_ai_sessions_owner (owner_id),
  KEY idx_ai_sessions_end_user (end_user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='AI会話セッション';

```

### 4.3.2 `ai_turns`（発話単位）

```sql
CREATE TABLE ai_turns (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'AIターンID',
  ai_session_id BIGINT UNSIGNED NOT NULL COMMENT '紐づくAIセッションID',

  direction ENUM('user','assistant','system') NOT NULL COMMENT '発話方向',
  role_label VARCHAR(64) NOT NULL COMMENT '役割ラベル（user/customer/merchant/hq等）',

  user_message TEXT NULL COMMENT 'ユーザー発話（direction=user時）',
  reply_text TEXT NULL COMMENT 'AI応答（direction=assistant時）',

  intent_key VARCHAR(191) NULL COMMENT 'インテントキー',
  intent_label VARCHAR(191) NULL COMMENT 'インテント表示名',
  intent_confidence DECIMAL(4,3) NULL COMMENT 'インテント信頼度（0.000-1.000）',

  answer_type ENUM('faq','scenario','template','generated','handoff') NULL COMMENT '回答種別',
  answer_source_id BIGINT NULL COMMENT '回答ソースID（FAQ/シナリオ等）',

  escalation_action ENUM('none','create_ticket','transfer_to_human') NULL COMMENT 'エスカレーションアクション',
  escalation_reason VARCHAR(191) NULL COMMENT 'エスカレーション理由',

  model_name VARCHAR(191) NULL COMMENT '使用モデル名',
  prompt_tokens INT NULL,
  completion_tokens INT NULL,
  total_tokens INT NULL,
  latency_ms INT NULL COMMENT 'AI応答までのレイテンシ(ms)',

  error_code VARCHAR(64) NULL COMMENT 'エラー時コード',

  created_at DATETIME NOT NULL,
  PRIMARY KEY (id),
  KEY idx_ai_turns_session (ai_session_id),
  KEY idx_ai_turns_intent (intent_key),
  KEY idx_ai_turns_answer (answer_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='AIターンログ';

```

### 4.3.3 `tickets` テーブル変更案

```sql
ALTER TABLE tickets
  ADD ai_session_id BIGINT UNSIGNED NULL COMMENT '起点となったAIセッションID（任意）',
  ADD KEY idx_tickets_ai_session (ai_session_id);

```

### 4.4 🔧 パラメータ候補（AI_CORE）

※ 実体は `options_master` / `shop_options` / `sys_params` 等で管理する前提。

- モデル関連
    - `ai_core.default_model`
    - `ai_core.model_per_channel.help`
    - `ai_core.model_per_channel.line`
    - `ai_core.model_per_channel.shop`
    - `ai_core.model_per_channel.ai`
- 生成関連
    - `ai_core.temperature.default`
    - `ai_core.temperature.help.customer`
    - `ai_core.temperature.shop.merchant`
    - `ai_core.max_tokens.reply`
    - `ai_core.max_history_turns`
- フォールバック閾値
    - `ai_core.intent_confidence_threshold.high`
    - `ai_core.intent_confidence_threshold.low`
    - `ai_core.ticket_auto_escalation_threshold`
- ログ／保持期間
    - `ai_core.turn_log_retention_days_full`（フルテキスト保持日数）
    - `ai_core.turn_log_retention_days_masked`（マスク済みで保持する日数）
    - `ai_core.session_log_retention_days`（`ai_sessions` の保持日数）
- PII マスキング
    - `ai_core.log_pii_masking_rules`（電話／メール／予約番号などの正規表現セット）
    - `ai_core.log_pii_mask_after_days`（マスキングを実施するまでの日数）
- スタイル／トーン
    - `ai_core.style.help.customer`
    - `ai_core.style.shop.merchant`
    - `ai_core.style.ai.hq`

店舗ごとに差異があり得るもの（トーン／温度など）は `shop_options` でオーバーライド可能とする。

---

## 5. フロー / 共通処理フロー

### 5.1 1ターンの基本フロー

1. **リクエスト受領**
    - 各チャネル（help./line./shop./ai.）から AI_CORE_REQUEST を受け取る。
    - `request_id` が無い場合は SYS_CORE 側で採番。
2. **セッション解決**
    - `context.session.ai_session_id` が指定されていれば、その `ai_sessions` レコードを更新（`last_activity_at` 更新）。
    - 指定が無ければ、新規 `ai_sessions` レコードを作成し、`ai_session_id` を採番。
3. **インテント／分類フェーズ**
    - SYS_CORE の HTTP クライアント経由で LLM を呼び出し、
        
        ユーザー発話＋必要なコンテキスト（店舗／ユーザー属性など）から:
        
        - インテントキー
        - カテゴリ／サブカテゴリ
        - 緊急度
        - 回答候補文
        - 信頼度ヒント
            
            を取得する。
            
4. **回答タイプ決定・フォールバック判断**
    - 信頼度、インテント種別、緊急度、設定値をもとに
        - `answer.type`（faq/scenario/template/generated/handoff）
        - `escalation.action`（none/create_ticket/transfer_to_human）
            
            を決定。
            
    - STORES／RemoteLOCK フローが必要な場合は、
        
        「どのフローに乗せるべきか」の判断のみ行い、API_COMM 側へ渡す。
        
5. **応答文整形**
    - 選ばれた回答種別＋トーン設定に基づき、`reply_text` を組み立てる。
    - 顧客／加盟店／本部で言い回しが変わる部分はプロンプトテンプレートで制御する。
6. **ログ記録**
    - `ai_turns` にユーザー発話／AI応答／メタ情報を1レコードとして保存。
    - 必要に応じて `api_call_logs`／`sys_events` にも記録（SYS_CORE責務）。
7. **レスポンス返却**
    - AI_CORE_RESPONSE を呼び出し元へ返す。
    - `escalation.action != 'none'` の場合、フロント or バックエンドから `SysEscalation` を呼び出し、
        
        `tickets` 作成や ChatWork 通知を行う。
        

### 5.2 エスカレーションフロー概要

- 条件（例）
    - 信頼度が `ai_core.intent_confidence_threshold.low` 未満
    - クレーム／返金／鍵トラブルなど、規定上人間判断が必要なカテゴリ
    - ユーザーが明示的に「人と話したい」「電話してほしい」と要求
- 流れ
    1. AI_CORE が `escalation.action` と `ticket_draft` を作成
    2. フロント or バックエンドが `SysEscalation` にチケット案を渡す
    3. `tickets` に登録し、ChatWork／メールなどへ通知
    4. `tickets.ai_session_id` にセッションIDを保存し、後から会話をトレースできるようにする

---

## 6. 通知 / 外部連携

### 6.1 LLM 連携

- AI_CORE → SYS_CORE の HTTP クライアント → LLM（OpenAI など）
- APIキー・モデル名・ベースURLなどは `SysConfig` が解決する。

### 6.2 STORES / RemoteLOCK 等

- AI_CORE は「この問い合わせは STORES の予約確認が必要」「解錠番号案内フロー」と判断するのみ。
- 実際の API 呼び出しは API_COMM フェーズで定義されたエンドポイントを利用。
- AI_CORE は API_COMM から返った結果を元に「どう説明するか」「どこまで言うか」をコントロールする。

### 6.3 エスカレーション通知

- `SysEscalation` を通じて、以下への通知が可能：
    - ChatWork（店舗用／本部用ルーム）
    - メール
    - SMS（必要な場合）
- AI_CORE は「チケット下書き」「優先度」「タグ」など、通知内容の雛形を提供する。

---

## 7. セキュリティ・権限

### 7.1 アクセス制御

- AI_CORE は `shared/ai_core.php` 等の内部ライブラリとして実装し、
    
    直接外部から叩けるエンドポイントは持たない。
    
- 認証・認可はチャネルごとのフロント／SYS_COREで完了済みとし、
    
    AI_CORE には「認証済みユーザー情報」が渡される前提。
    

### 7.2 個人情報（PII）取り扱い

- `ai_turns.user_message`／`ai_turns.reply_text` には、
    
    一時的に個人情報（電話番号／メール／予約番号など）が含まれる可能性がある。
    
- 標準方針：
    - フルテキスト状態は短期間のみ保持（パラメータ `ai_core.turn_log_retention_days_full`）
    - その後、PII マスキングルール（正規表現）に従ってマスクを実施
    - マスク済みのテキストは「一般的な問い合わせ傾向の分析」などに利用可能とする。
- マスキング対象例：
    - 電話番号形式の文字列
    - メールアドレス形式
    - 特定フォーマットの予約番号／会員番号

### 7.3 権限と可視範囲

- 本部（ai.）：
    - 全店舗の `ai_sessions` / `ai_turns` を閲覧可能（内部利用）。
- 加盟店（shop.）：
    - 自店舗に紐づく `ai_sessions` / `ai_turns` のみ閲覧可能。
    - マスク後の内容を前提とし、本部向け内部メモなどは非表示。
- エンドユーザー：
    - 過去の会話履歴をどこまで見せるかは UI/UX フェーズで別途設計。

---

## 8. 未決課題 / 次フェーズへの引き継ぎ

### 8.1 Phase2 内で確定が必要な未決

- インテントマスタの実装方式
    - テーブル構造（例：`ai_intents_master`）／管理画面方針。
- FAQ／シナリオのデータ構造
    - テーブル分割（FAQ / シナリオ / テンプレ）と I/O 契約。
- 閾値・保持期間の具体値
    - インテント信頼度の high/low 値
    - フルテキスト保持日数／マスキング日数／セッション保持日数の初期値

### 8.2 チュートリアル層フェーズへの引き継ぎ

- `ai_sessions.is_self_solved` の更新タイミングと UI
    - 顧客／加盟店に「解決しました／まだ解決していない」をどう聞くか。
- セルフ解決率／エスカレーション率のダッシュボード仕様
    - 本部画面（ai.）でどの粒度（店舗別／チャネル別／期間別）で見せるか。
- 「店舗ごとのAIのクセ調整」（温度／トーン）の UI と運用ルール
    - shop_options の編集画面との連携。
