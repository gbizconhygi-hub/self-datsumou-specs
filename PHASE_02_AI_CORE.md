# AI_CORE - 仕様設計書 (PHASE_02_AI_CORE.md Draft)

## 1. 目的・範囲

### 1.1 フェーズ概要

本フェーズ（Phase 2：AI_CORE）は、以下 4 チャネルすべてから共通で呼び出される

「AIコア（意図分類＋回答生成＋フォールバック＋ログ）」の仕様を定義する。

- エンドユーザー向け
    - `help.self-datsumou.net`（Webチュートリアル）
    - `line.self-datsumou.net`（LINE公式アカウント）
- 加盟店向け
    - `shop.self-datsumou.net`（オーナーポータル・加盟店チュートリアル）
- 本部向け
    - `ai.self-datsumou.net`（本部管理画面）

AI_CORE の主な役割：

- ユーザー発話を受け取り、
    - インテント（意図）の分類
    - FAQ／シナリオ／テンプレ回答の選択・生成
    - フォールバック（テンプレ／有人／チケット化）の判断
    を行い、応答文＋メタ情報を返す。
- Phase1 で定義した SYS_CORE（`SysConfig` / `SysLogger` / `SysEscalation` / `tickets` 等）と連携し、
AI 応答ログとチューニング用メタ情報を一貫して記録・活用できる状態を作る。

### 1.2 本フェーズの範囲

本フェーズで扱うもの：

- AI_CORE の I/O 契約（AI_CORE_REQUEST / AI_CORE_RESPONSE）
- インテントマスタ（`ai_intents_master`）の設計
- Q&A テーブル（`ai_qa_entries`）の設計
- 軽量シナリオ（最大 2〜3 ステップ）の扱い方
- AI 会話ログのデータ構造（`ai_sessions` / `ai_turns`）と `tickets` との紐づけ
- LLM 呼び出しの基本方針（SYS_CORE 経由、モデル指定、閾値の考え方）
- セルフ解決率（`is_self_solved`）の定義と KPI 前提

本フェーズでは扱わない（他フェーズで定義する）もの：

- 各チャネルの UI／画面遷移（ボタン配置、入力フォーム等）
- STORES予約／RemoteLOCK など外部 API の具体的な I/O（Phase3：API_COMM）
- FAQ／シナリオ／テンプレ本文そのものの編集画面仕様（管理 UI）
- 「解決しました／まだです」ボタンや後追い確認の UI 詳細（顧客／加盟店チュートリアルフェーズ）

---

## 2. 前フェーズからの継承ポイント

### 2.1 システム構成（Phase0 / spec_v1.4_base）

- サブドメイン構成
    - エンドユーザー：`help.self-datsumou.net` / `line.self-datsumou.net`
    - 加盟店：`shop.self-datsumou.net`
    - 本部：`ai.self-datsumou.net`
- 共通ライブラリ構造
    - `shared/` 配下に共通機能を集約する。
    - AI_CORE は `shared/ai_core.php`（仮称）として実装される前提。

### 2.2 DB コア（PHASE_00_DB_CORE / db_fields.yaml）

- 既存テーブル（抜粋）
    - `owners`：オーナー情報
    - `shops`：店舗マスタ
    - `shop_secrets`：店舗別シークレット（APIキー等）
    - `options_master`：全体設定マスタ
    - `shop_options`：店舗別設定（`options_master` の上書き）
- AI_CORE は、これらを参照しつつ、以下の新規テーブルを追加する：
    - インテントマスタ：`ai_intents_master`
    - Q&Aマスタ：`ai_qa_entries`
    - 会話セッション：`ai_sessions`
    - ターンログ：`ai_turns`

### 2.3 SYS_CORE 連携（PHASE_01_SYS_CORE）

- SYS_CORE の責務（要約）
    - `SysConfig`：`options_master` / `shop_options` / `.env` から有効な設定値を解決
    - `SysLogger`：`api_call_logs` / `sys_events` / `tickets` など共通ログの窓口
    - `SysEscalation`：ChatWork／メール／SMS／チケット化など有人対応窓口
    - HTTPクライアント：OpenAI／STORES／RemoteLOCK 等の外部API呼び出し共通レイヤー
- AI_CORE からの呼び出し方針（決定事項）
    - LLM 呼び出しは **必ず SYS_CORE の HTTP クライアント経由** とし、AI_CORE は API キー／ベースURL を直接扱わない。

### 2.4 トーン／スタイル方針（共通ルール）

- エンドユーザー（help./LINE）
    - 「人間が丁寧に説明している」トーン。
    - 顔文字や柔らかめの表現を適度に使用し、機械感を減らす。
- 加盟店（shop.）
    - ぶっきらぼう NG。
    - 疑問や不安に寄り添うトーン。
- 本部（ai.）
    - 事務的だが冷たすぎない、簡潔で誤解のない説明。

AI_CORE のプロンプト／テンプレは、このトーン方針を前提とする。

---

## 3. 新規決定事項

### 3.1 決定事項（確定）

1. **頭脳は 1 つ、チャネル差はパラメータで吸収**
    - help. / line. / shop. / ai. は、すべて共通の AI_CORE を利用する。
    - チャネル差（顧客／加盟店／本部）は、プロンプト・温度・履歴長さ・スタイルなどのパラメータで吸収する。
2. **LLM 呼び出しは SYS_CORE 経由**
    - AI_CORE は LLM を直接叩かず、SYS_CORE 提供の HTTP クライアントを通じて実行する。
    - モデル名やベースURL、APIキーは `.env` / `shop_secrets` / `options_master` から `SysConfig` が解決する。
3. **インテントマスタは専用テーブル `ai_intents_master` で管理**
    - インテントごとに `intent_key` / `category` / `subcategory` / `label` / `handling_profile` などを持つ。
    - インテントは 本部 UI から追加・削除・変更可能にする想定。
4. **Q&A は 1 テーブル `ai_qa_entries`＋フラグで顧客／加盟店を区別**
    - `target_type`（customer／merchant／hq）で対象を区別する。
    - `channel_scope`（help／line／shop／ai）で利用チャネルを指定する。
    - 共通Q&Aの共通管理（両方に見せたいもの）も可能。
5. **回答生成は「Q&Aテンプレ＋LLM整形」のハイブリッド方式**
    - `ai_qa_entries.answer_template` は「何を言うか」のガイドライン（マニュアル相当）。
    - 実際の返答は、LLM に
        - ユーザー発話
        - インテント情報
        - answer_template／question_examples
        を渡し、「人間が説明しているような自然な文」に整形させる。
6. **シナリオ（ステップ型）は v1.4 では「最大 2〜3 ステップの直列質問」に限定**
    - 複雑な条件分岐エンジンは実装対象外。
    - 必要なインテント（解錠番号／解約説明など）だけが `followup_question_1` / `followup_question_2` を使う想定とする。
7. **インテントごとの扱いは `handling_profile` ＋ パラメータで制御**
    - `ai_intents_master.handling_profile`
        - `normal`：通常の問い合わせ
        - `cautious`：慎重に扱いたい（鍵・契約・お金系など）
        - `escalate_prefer`：基本、人に回した方がよいもの
    - プロファイルごとの閾値（high／low）は `options_master` 側の設定値で制御し、コードに固定値は持たない。
8. **会話ログ構造は `ai_sessions` / `ai_turns` の二層構造**
    - `ai_sessions`：1件の相談（1ユーザー・1トピック）の単位。チャネル／店舗／解決状況／セルフ解決フラグなどを持つ。
    - `ai_turns`：各発話（ユーザー／AI／system）のログ。インテント・回答種別・メタ情報を保持する。
9. **`tickets` への `ai_session_id` 追加**
    - `tickets` テーブルに `ai_session_id` を追加し、「このチケットはどのAI会話から生まれたか」を辿れるようにする。
10. **セルフ解決率の定義と 24 時間ルール**
    - `ai_sessions.is_self_solved = 1`
        - ユーザー（顧客／加盟店）が、**AI／チュートリアルのみ**で解決したと明示した場合のみ 1。
        - 具体的には、チャット内で「解決しました」ボタンに相当する操作が 24 時間以内に行われた場合。
    - それ以外（人間介入や反応なし）は、`is_self_solved = 0`。
    - セルフ解決率の分母は「24時間以内に判定できたセッション」に限定し、
    判定不能なものは「Unknown」として別カウントする。
11. **ログ／PII の保持日数（初期方針）**
    - `ai_turns`（フルテキスト：PII含む可能性）
        - フルテキスト保持：90 日
        - 90 日経過後、電話番号・メール・予約番号などをマスキングした上で残す。
    - `ai_turns`（マスク済みテキスト）
        - マスク後のログ保持：3 年（1095 日）を初期値とする。
    - `ai_sessions`（メタ情報）
        - 保持：5 年（もしくは無期限運用前提。実運用で調整）
12. **チャネルごとのセルフ解決判定の扱い**
    - help.（Web）
        - セッション内で「この回答で解決しましたか？」を聞き、24 時間以内の反応のみ `is_self_solved` 判定に使う。
        - D+1 フォローなどのプッシュは help. 単体では行わない。
    - LINE
        - セッション内の ✅／❌に加え、必要であれば D+1 フォローを送る設計も許容。
        - 判定基準自体は 24 時間ルールで統一。
    - shop.（加盟店）
        - チャット内の「解決しました」操作、または「未解決の相談一覧」からの「解決済み」操作で `resolved_status` を更新。
        - セルフ解決率は、あくまで AI／チュートリアルのみで解決したものに限る。

---

### 3.2 未決事項（Phase2 内で別途検討／後続フェーズで詳細化）

1. `channel_scope` と `question_examples` の内部フォーマット詳細
    - `channel_scope`
        - 文字列カンマ区切り（例：`"help,line"`）とするか、ビットフラグとするか。
    - `question_examples`
        - TEXT に改行区切りで複数例を格納するか、JSON 配列にするか。
2. CSV フォーマットの細部
    - `question_examples` の区切り文字（改行 vs `||`）
    - インポート／エクスポートツールの具体仕様（管理画面 or バッチ）
3. `handling_profile` ごとのデフォルト閾値 (`conf_high` / `conf_low`)
    - 初期案は本仕様内で推奨値を出すが、最終値は運用開始前に本部で決定。
4. ログ保持期間の最終値
    - 90 日／3 年／5 年案をベースに、規約・顧問の意見を踏まえて最終値を決定。

---

## 4. データ構造 / I/O 定義

### 4.1 AI_CORE_REQUEST（入力）構造

```json
{
  "request_id": "uuid-string",              // ログ相関用の一意ID（未指定時はSYS_CORE側で採番）
  "channel": "help | line | shop | ai",    // 呼び出し元チャネル
  "actor_type": "customer | merchant | hq",// ユーザー種別
  "tenant_code": "hygi",                   // 将来のマルチテナント拡張用（v1.4では固定想定）

  "shop_id": 123,                          // 対象店舗ID（不明時はnull）
  "owner_id": 456,                         // 加盟店オーナーID（加盟店チャット時）
  "end_user_id": "string-or-null",         // エンドユーザーID（会員ID／LINE ID等）

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
      "force_intent": null                 // デバッグ用インテント固定（通常運用ではnull）
    }
  },

  "options": {
    "max_tokens": 800,                     // LLM呼び出しに渡すmax_tokens（未指定時はSysConfigの値）
    "temperature": 0.3,
    "top_p": 0.9
  }
}

```

### 4.2 AI_CORE_RESPONSE（出力）構造

```json
{
  "request_id": "uuid-string",        // 入力と同じID
  "ai_session_id": 9999,              // AIセッションID（新規セッションなら採番結果）

  "reply_text": "string",             // ユーザーに返却するメイン応答文
  "reply_text_alt": [                 // （任意）代替案／分割用テキスト
    "string"
  ],

  "intent": {
    "key": "faq.booking.change",      // インテントキー（ai_intents_master.intent_key）
    "label": "予約変更・キャンセル",
    "confidence": 0.87                // 0〜1 のスコア
  },

  "classification": {
    "category": "booking",           // 大分類
    "subcategory": "change",         // 中分類
    "urgency": "normal | high | low" // 緊急度（鍵・クレーム等）
  },

  "answer": {
    "type": "faq | scenario_step | clarify | handoff",
    "source_qa_id": 1234,            // 使用したai_qa_entries.id（無い場合はnull）
    "reason": "high_confidence_faq"  // ロジック上の採用理由（ログ／デバッグ用）
  },

  "followup": {
    "needed": false,                 // 追加質問が必要か（シナリオ／確認フロー）
    "question": null,                // 次にユーザーへ投げるフォローアップ質問文
    "step_index": null               // シナリオ内のステップ番号（1,2…）
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

### 4.3 DB テーブル定義（DDL 案）

### 4.3.1 インテントマスタ `ai_intents_master`

```sql
CREATE TABLE ai_intents_master (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'インテントID',
  intent_key VARCHAR(191) NOT NULL COMMENT 'インテントキー（例：booking.change）',
  category VARCHAR(64) NULL COMMENT '大分類（booking / contract / lock / complaint 等）',
  subcategory VARCHAR(64) NULL COMMENT '中分類（change / cancel / code 等）',
  detail_label VARCHAR(191) NULL COMMENT 'さらに細かい分類ラベル（必要に応じて使用、NULL可）',

  label_ja VARCHAR(191) NOT NULL COMMENT '日本語表示名（例：予約変更・キャンセル）',
  description TEXT NULL COMMENT '用途説明',

  handling_profile ENUM('normal','cautious','escalate_prefer')
    NOT NULL DEFAULT 'normal' COMMENT '扱いプロファイル（通常／慎重／エスカレ優先）',

  target_type_scope VARCHAR(64) NOT NULL DEFAULT 'customer,merchant'
    COMMENT '対象ユーザー種別の範囲（customer,merchant,hq のカンマ区切り）',

  priority INT NOT NULL DEFAULT 0 COMMENT '一覧表示やマッチングの優先度',
  is_active TINYINT(1) NOT NULL DEFAULT 1 COMMENT '有効フラグ',

  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,

  PRIMARY KEY (id),
  UNIQUE KEY uq_ai_intents_intent_key (intent_key),
  KEY idx_ai_intents_category (category, subcategory)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='AIインテントマスタ';

```

### 4.3.2 Q&A マスタ `ai_qa_entries`

```sql
CREATE TABLE ai_qa_entries (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'Q&AエントリID',
  intent_id BIGINT UNSIGNED NOT NULL COMMENT '紐づくインテントID（ai_intents_master.id）',

  target_type ENUM('customer','merchant','hq')
    NOT NULL COMMENT '対象ユーザー種別',
  channel_scope VARCHAR(64) NOT NULL DEFAULT 'help,line,shop,ai'
    COMMENT '利用チャネル（help,line,shop,ai のカンマ区切り）',

  question_title VARCHAR(191) NOT NULL COMMENT '管理用タイトル（例：予約時間の変更）',
  question_examples TEXT NULL COMMENT '代表的な質問例（複数の場合は区切り文字 or 改行）',

  answer_template TEXT NOT NULL COMMENT '回答テンプレート（マニュアル的な骨子）',
  answer_type ENUM('faq','scenario_entry') NOT NULL DEFAULT 'faq'
    COMMENT 'faq：1発回答／scenario_entry：ステップ型フローの入口',

  followup_question_1 TEXT NULL COMMENT 'ステップ型で最初に聞く質問（例：お名前を教えてください）',
  followup_question_2 TEXT NULL COMMENT '必要に応じた2つ目の質問（例：ご予約日時を教えてください）',

  priority INT NOT NULL DEFAULT 0 COMMENT '同一インテント内での優先度',
  is_active TINYINT(1) NOT NULL DEFAULT 1 COMMENT '有効フラグ',

  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,

  PRIMARY KEY (id),
  KEY idx_ai_qa_intent (intent_id),
  KEY idx_ai_qa_target (target_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='AI Q&Aエントリ';

```

### 4.3.3 会話セッション `ai_sessions`

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

  current_intent_key VARCHAR(191) NULL COMMENT '現在の主要インテントキー（最後に確定したもの）',
  state_json JSON NULL COMMENT 'シナリオ進行状況などの状態保持（必要に応じて使用）',

  started_at DATETIME NOT NULL COMMENT 'セッション開始日時',
  last_activity_at DATETIME NOT NULL COMMENT '最終アクティビティ日時',
  status ENUM('active','closed') NOT NULL DEFAULT 'active' COMMENT 'セッション状態',

  resolved_status ENUM('unknown','resolved_by_ai','resolved_by_human','abandoned')
    NOT NULL DEFAULT 'unknown' COMMENT '解決ステータス',
  is_self_solved TINYINT(1) NOT NULL DEFAULT 0 COMMENT 'セルフ解決できたか（24時間以内に明示があった場合のみ1）',

  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,

  PRIMARY KEY (id),
  KEY idx_ai_sessions_channel (channel),
  KEY idx_ai_sessions_shop (shop_id),
  KEY idx_ai_sessions_owner (owner_id),
  KEY idx_ai_sessions_end_user (end_user_id),
  KEY idx_ai_sessions_intent (current_intent_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='AI会話セッション';

```

### 4.3.4 ターンログ `ai_turns`

```sql
CREATE TABLE ai_turns (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'AIターンID',
  ai_session_id BIGINT UNSIGNED NOT NULL COMMENT '紐づくAIセッションID',

  direction ENUM('user','assistant','system') NOT NULL COMMENT '発話方向',
  role_label VARCHAR(64) NOT NULL COMMENT '役割ラベル（user/customer/merchant/hq 等）',

  user_message TEXT NULL COMMENT 'ユーザー発話（direction=user時）',
  reply_text TEXT NULL COMMENT 'AI応答（direction=assistant時）',

  intent_key VARCHAR(191) NULL COMMENT 'インテントキー',
  intent_label VARCHAR(191) NULL COMMENT 'インテント表示名',
  intent_confidence DECIMAL(4,3) NULL COMMENT 'インテント信頼度（0.000-1.000）',

  answer_type ENUM('faq','scenario_step','clarify','handoff') NULL COMMENT '回答種別',
  answer_source_qa_id BIGINT NULL COMMENT '回答ソースのQ&A ID（ai_qa_entries.id）',

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

### 4.3.5 `tickets` テーブルへの拡張

```sql
ALTER TABLE tickets
  ADD ai_session_id BIGINT UNSIGNED NULL COMMENT '起点となったAIセッションID（任意）',
  ADD KEY idx_tickets_ai_session (ai_session_id);

```

---

### 4.4 🔧 パラメータ候補（AI_CORE 全体）

> 実体は options_master / shop_options / sys_params 等で管理する前提。
> 
> 
> `shop_options` による店舗別オーバーライドを許容する。
> 
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
- プロファイルごとの閾値
    - `ai_core.profile.normal.conf_high`
    - `ai_core.profile.normal.conf_low`
    - `ai_core.profile.cautious.conf_high`
    - `ai_core.profile.cautious.conf_low`
    - `ai_core.profile.escalate_prefer.conf_high`
    - `ai_core.profile.escalate_prefer.conf_low`
- ログ／保持期間
    - `ai_core.turn_log_retention_days_full`（フルテキスト保持日数：初期値 90 日）
    - `ai_core.turn_log_retention_days_masked`（マスク済み保持日数：初期値 1095 日（3 年））
    - `ai_core.session_log_retention_days`（`ai_sessions` の保持日数：初期値 5 年）
- PII マスキング関連
    - `ai_core.log_pii_masking_rules`（電話／メール／予約番号等の正規表現セット）
    - `ai_core.log_pii_mask_after_days`（マスキング実施までの日数：初期値 90 日）
- セルフ解決判定
    - `ai_core.self_solved_decision_window_hours`（セルフ解決判定の有効時間：初期値 24 時間）
- スタイル／トーン
    - `ai_core.style.help.customer`（顧客向けスタイルプロファイルID）
    - `ai_core.style.shop.merchant`（加盟店向けスタイルプロファイルID）
    - `ai_core.style.ai.hq`（本部向けスタイルプロファイルID）

---

## 5. フロー / 共通処理フロー

### 5.1 1ターンの基本フロー

1. **リクエスト受領**
    - 各チャネル（help./line./shop./ai.）から AI_CORE_REQUEST を受け取る。
    - `request_id` が未指定の場合、SYS_CORE 側で採番。
2. **セッション解決**
    - `context.session.ai_session_id` が指定されている場合はそのレコードを `ai_sessions` から取得し、`last_activity_at` を更新。
    - 未指定の場合は新規 `ai_sessions` レコードを作成し、`started_at`／`last_activity_at` を現在時刻でセット。
3. **インテント推定**
    - SYS_CORE の HTTP クライアント経由で LLM を呼び出し、
        
        ユーザー発話＋必要なコンテキスト（店舗情報／ユーザー属性／履歴の一部）を渡して：
        
        - インテントキー (`intent.key`)
        - カテゴリ／サブカテゴリ (`classification.category` / `subcategory`)
        - 緊急度 (`classification.urgency`)
        - 信頼度 (`intent.confidence`)
            
            を取得。
            
    - `ai_intents_master` から `intent_key` に対応するレコードを参照し、`handling_profile` や label を補完。
4. **回答候補抽出**
    - `ai_qa_entries` を以下の条件で検索：
        - `intent_id` = 対応する `ai_intents_master.id`
        - `target_type` = `actor_type`（customer／merchant／hq）
        - `channel_scope` に当該チャネル（help／line／shop／ai）が含まれる
        - `is_active = 1`
    - priority の高いものから順に使用候補とする。
5. **回答タイプ決定とシナリオ判定**
    - `handling_profile` と信頼度 (`confidence`) を見て、次を決定：
        - `answer.type`（faq／scenario_step／clarify／handoff）
        - シナリオ用の `followup.question` が必要かどうか
    - FAQ型：
        - `answer_type = 'faq'` の Q&A があり、信頼度が高い：
            - → テンプレ＋LLM整形で即回答。
    - シナリオ型（ステップ型）：
        - Q&A で `answer_type = 'scenario_entry'` かつ `followup_question_1` が存在：
            - セッション状態を見て現在ステップを決定し、`followup.question` を返す。
    - Clarify型：
        - 信頼度が閾値以下の場合（特に handling_profile = cautious 以上）：
            - → 質問の候補（Q&A タイトルなど）を提示し、「どのパターンか」確認するための質問に落とす。
6. **エスカレーション判断**
    - `handling_profile`／`confidence`／`classification.urgency` に基づき、
        - `escalation.action`（none／create_ticket／transfer_to_human）を決定。
    - `create_ticket` の場合：
        - `tickets` に登録するための `ticket_draft`（title／body_preview／priority／tags）を構成する。
7. **応答テキスト生成（LLM整形）**
    - LLM に対し：
        - ユーザー発話
        - インテント情報
        - Q&Aテンプレ（answer_template）
        - 代表質問例（question_examples）
        - シナリオステップ情報（必要な場合）
            
            を渡し、
            
        - 「人間が丁寧に説明している」トーン
        - チャネル／対象（顧客／加盟店／本部）に応じた言い回し
            
            で回答文を整形させる。
            
8. **ログ記録**
    - `ai_turns` に 1 レコード挿入：
        - セッションID
        - user_message／reply_text
        - intent_key／intent_confidence
        - answer_type／answer_source_qa_id
        - escalation_action／reason
        - model_name／トークン数／レイテンシ
    - 必要に応じて `api_call_logs`／`sys_events` にも記録（SYS_CORE責務）。
9. **レスポンス返却**
    - AI_CORE_RESPONSE をチャネル側に返却。
    - `escalation.action != 'none'` の場合、
        
        フロント or バックエンドが `SysEscalation` を呼び出し、`tickets` 作成や ChatWork 通知を行う。
        

### 5.2 セルフ解決率と 24 時間ルール

- チャネル側で「解決しましたか？」ボタン等を提供し、
    
    押された場合に `ai_sessions` を更新：
    
    - `resolved_status = 'resolved_by_ai'`
    - `is_self_solved = 1`
- 24 時間以内に明示の解決／未解決が得られなかったセッション：
    - `resolved_status` は別ルール（例：`resolved_by_ai` or `abandoned`）で閉じてもよいが、
        
        `is_self_solved` は 0 のまま（KPI には含めない）。
        
- セルフ解決率の分母は
    
    「24 時間以内に `is_self_solved` 判定がついたセッション」のみとし、
    
    判定不能セッションは「Unknown」として別カウントする。
    

---

## 6. 通知 / 外部連携

### 6.1 LLM 連携

- AI_CORE → SYS_CORE の HTTP クライアント → LLM（OpenAI 等）
- モデル名・ベースURL・APIキーはすべて `SysConfig` から取得する。
- プロファイル／チャネルごとのモデル分けは
    
    `ai_core.model_per_channel.*` 等のパラメータで制御。
    

### 6.2 STORES／RemoteLOCK 等（API_COMM フェーズとの連携前提）

- AI_CORE は「どのフローに乗せるか」までを判断する：
    - 例：「これは解錠番号案内フロー」「これは予約変更フロー」など。
- 実際の STORES／RemoteLOCK API 呼び出しは、API_COMM フェーズで定義されるエンドポイントを通じて行う。
- AI_CORE は API_COMM から返却された結果をもとに、
    - どこまでエンドユーザーに見せるか（例：解錠番号の一部だけ）
    - どんなトーンで案内するか
        
        を決定し、返信文に反映する。
        

### 6.3 エスカレーション通知

- `escalation.action = 'create_ticket'` の場合：
    - チャネル／バックエンド側から `SysEscalation` を呼び出し、
        - `tickets` テーブルへ登録
        - ChatWork／メール／SMS への通知
            
            を行う。
            
- `tickets.ai_session_id` に AI セッションIDを紐付けることで、
    
    後から「どの会話からこのチケットが生まれたか」を辿れる。
    

---

## 7. セキュリティ・権限

### 7.1 アクセス制御

- AI_CORE は `shared/ai_core.php` 等の内部ライブラリとして実装し、
    
    外部から直接叩けるエンドポイントは持たない。
    
- 認証・認可はチャネルごとのフロント／SYS_CORE側で完了済みとし、
    
    AI_CORE には「認証済みユーザー情報」が渡る前提。
    

### 7.2 個人情報（PII）の扱い

- `ai_turns.user_message`／`ai_turns.reply_text` には、電話番号／メール／予約番号などの PII が含まれる可能性がある。
- 標準方針：
    - フルテキスト状態での保持は 90 日まで。
    - 90 日経過後、`ai_core.log_pii_masking_rules` に基づいてマスキングを実施。
    - マスク済みテキストは最大 3 年まで保持し、問い合わせ傾向分析や AI 改善に利用可能とする。
- マスキング対象例：
    - 電話番号形式（例：`0X0-XXXX-XXXX` など）
    - メールアドレス形式
    - 特定フォーマットの予約番号／会員番号

### 7.3 権限と可視範囲

- 本部（ai.）
    - 全店舗の `ai_sessions` / `ai_turns` を閲覧可能（内部分析・改善用）。
- 加盟店（shop.）
    - 自店舗に紐づく `ai_sessions` / `ai_turns` のみ閲覧可能。
    - PII マスク後の内容を前提とし、本部向け内部メモやコメントは非表示。
- エンドユーザー（help./LINE）
    - 過去の会話履歴をどこまで見せるかは、UI フェーズ（顧客チュートリアル）で設計。
    - 本仕様では「必要に応じて履歴を参照できる」程度の前提にとどめる。

---

## 8. 未決課題 / 次フェーズへの引き継ぎ

### 8.1 Phase2 内で後続チャットで詰めるべき未決事項

- **AI2-01：`channel_scope`・`question_examples` の内部フォーマット**
    - `channel_scope` を文字列カンマ区切りで固定するか、ビットフラグ型にするか。
    - `question_examples` を TEXT＋改行区切り vs JSON配列のどちらにするか。
- **AI2-02：CSVフォーマットの細部仕様**
    - インテントマスタ用 CSV の確定カラム列
    - Q&A 用 CSV の確定カラム列
    - `question_examples` の区切り文字（改行／`||`／JSON など）
    - 「大雑把な問い合わせデータ → 投入用 CSV への整形」を AI で支援する運用フローの定義。
- **AI2-03：handling_profile ごとの初期閾値**
    - `normal` / `cautious` / `escalate_prefer` 各プロファイルについて、
        - `conf_high` / `conf_low` の推奨値を最終確定する。
- **AI2-04：ログ保持期間の最終値**
    - 規約・顧問（法務・税務）の意見を踏まえ、
        - フルテキスト保持日数
        - マスク済み保持日数
        - セッションメタ保持年数
            
            を最終確定する。
            

### 8.2 次フェーズ（顧客／加盟店チュートリアル層）への引き継ぎ事項

- `ai_sessions.is_self_solved` の更新トリガー
    - 顧客／加盟店チュートリアルの UI で「解決しました／まだです」をどこで出すか。
    - ボタンが押されなかった場合の `resolved_status` 更新ルール（24時間後の扱い）を具体化する。
- シナリオ（ステップ型）の利用対象インテント
    - v1.4 でシナリオを適用するインテント（解錠番号／解約／一部トラブル）をピックアップし、
        
        `followup_question_1`／`followup_question_2` を設計する。
        
- 加盟店向けログ画面の仕様
    - `ai_sessions`／`ai_turns` を shop. 側でどのように見せるか（一覧／詳細／フィルタ）。
- セルフ解決率／エスカレーション率のダッシュボード仕様
    - 本部（ai.）画面で、KPI をどの粒度（店舗別／チャネル別／期間別）で表示するか。
