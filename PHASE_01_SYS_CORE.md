# PHASE_01_SYS_CORE — SYS_CORE 仕様書（v1.0 Final）

本ドキュメントは「加盟店サポート自動化プロジェクト v1.4」における  
Phase 1：SYS_CORE（システム共通基盤）の最終仕様書である。

---

# 1. 目的・範囲

## 1.1 目的

本フェーズは、HELP / LINE / SHOP / AI 各レイヤーの共通基盤として機能する  
SYS_CORE のコア構造・ルール・I/O を定義することを目的とする。

SYS_CORE が担う領域は次のとおり。

- 共通クライアント層  
  （STORES / ChatWork / SMS / LINE / OpenAI / RemoteLock 等）
- 共通エラーハンドラ
- パラメータ管理（DB・スコープ別上書き）
- ログ基盤（生ログ／匿名化ログ）
- エスカレーション基盤（通知ルール・条件判定）
- API 通信共通層（curl ベース HttpClient）

## 1.2 前提思想の継承

Phase 0（FOUNDATION）および base 仕様で定義された思想を継承する。

- 加盟店から本部への質問を可能な限り 0 に近づける
- エンドユーザーから加盟店への質問を可能な限り 0 に近づける
- その核となるのは AI チュートリアルである
- DB は UTC 保存・アプリ側で JST 表示
- 日時フォーマットは原則「YYYY-MM-DD HH:mm:ss」
- 固定値は極力パラメータ化し、本部 UI から変更可能にする

## 1.3 範囲

本フェーズでは「仕組みの土台」を対象とし、  
個別のサービスオプション（例：電話代行）の具体挙動は後続フェーズで定義する。

## 1.4 未決事項

なし。

## 1.5 確認項目

なし。

---

# 2. 前フェーズからの継承ポイント

## 2.1 SoR（正式情報源）ポリシー

### 決定事項

1. db_fields.yaml に定義されている既存 DB フィールドを、  
   システム全体の正式情報源（Source of Record; SoR）とする。
2. 既存 DB フィールドと意味が重複する sys_params キーは作成しない（二重管理禁止）。
3. 情報参照の優先順位は次のとおりとする。

    DB（既存フィールド）
     → sys_params（上書き用）
     → .env（秘密情報・固定値）
     → コード内デフォルト値（最終フォールバック）

4. 設計・実装時に、既存 DB と意味が被るパラメータ案が出た場合、  
   ChatGPT は必ず「確認項目」として提示し、二重管理回避の検討を行う。

## 2.2 オプション管理に関する前提

### 決定事項

- shop テーブルには、本部が提供する各種サービスオプション  
  （例：電話代行サービス、将来追加されるオプション等）のフラグが格納される前提とする。
- SYS_CORE は、これらのオプション値を  
  エスカレーション条件などを組み立てる際の「入力情報」として参照できるようにする。
- ただし、  
  「電話代行あり店舗は夜間こう動く」「特定プランはこう動く」といった  
  個別オプションごとの具体的ロジックは後続フェーズ  
  （AI チュートリアル／INBOX 等）の仕様で定義する。  
  本フェーズでは「参照できる仕組み」を用意するところまでを範囲とする。

## 2.3 セキュリティ・マスキング思想

### 決定事項

- 顧客情報は AI に渡す前にマスキングする（例：氏名→イニシャル、電話番号→下 4 桁）。
- 機密情報は .env 管理とし、リポジトリには含めない。
- 通信は HTTPS 前提とし、外部 API 認証は主に Bearer Token / API Key を用いる。

## 2.4 未決事項

- 既存 DB フィールドのうち SYS_CORE が直接参照する候補一覧  
  （別ドキュメントとして整理）。

## 2.5 確認項目

なし。

---

# 3. 新規決定事項（SYS_CORE コンポーネント方針）

## 3.1 ディレクトリ構造（PHP）

### 決定事項

SYS_CORE の主な配置は次のとおりとする。

    /shared/
      sys_core/
        SysConfig.php           … パラメータ取得（DB＋.env＋キャッシュ）
        SysLogger.php           … ログ出力（生ログ・匿名ログ）
        SysErrorHandler.php     … 共通エラーハンドラ
        SysEscalation.php       … エスカレーション共通処理
        HttpClient.php          … curl ベース HTTP クライアント
        Clients/
          StoresClient.php
          ChatworkClient.php
          SmsClient.php         … Twilio ラッパ
          LineClient.php
          OpenAiClient.php
          RemoteLockClient.php

- HELP / LINE / SHOP / AI など各レイヤーから外部 API を利用する際は、  
  必ず上記 Clients 経由とする（直接 API を叩かない）。

## 3.2 共通クライアント層の責務

### 決定事項

- StoresClient  
  認証ヘッダ付与、エンドポイント URL 組み立て、レスポンス統一化。
- ChatworkClient  
  ROOM ID / トークン解決、メッセージ送信、必要に応じてスレッドリンク生成。
- SmsClient（Twilio）  
  送信元/先番号バリデーション、テンプレ ID 解決、送信結果ログ記録。
- LineClient  
  Messaging API / LIFF を通じたユーザーへの送信、Webhook 応答生成。
- OpenAiClient  
  モデル名・temperature・top_p 等を SysConfig から取得し、共通フォーマットで返却。
- RemoteLockClient  
  店舗別資格情報（既存 DB / 秘匿テーブル）を読み込み、PIN / ゲスト情報取得を統一化。

全クライアントは、呼び出し側が個別 API 仕様を意識せず利用できるよう、  
共通レスポンス形式（例：status / code / data / error）を持つことを目標とする。

### 未決事項

- 共通レスポンス JSON の最終スキーマ（フィールド名・ネスト構造）。

### 確認項目

- 既存 PoC で用いているレスポンス形式との互換性要件の有無。

## 3.3 HTTP クライアント実装方針

### 決定事項

- 本フェーズでは外部ライブラリ（Guzzle 等）は必須とせず、  
  curl ベースの自前 HttpClient 実装を標準とする。
- 将来的に Composer 運用が前提になった場合、  
  HttpClient 内部のみを Guzzle 実装等に差し替え、  
  クライアント呼び出し側のコード変更を不要とする。

---

## 3.4 🔧 パラメータ候補（SYS_CORE 全体）

- api_timeout_sec：10（秒）
- api_retry_count：2（ネットワークエラー・5xx のみ）
- log_level：INFO（ERROR / WARN / INFO / DEBUG）
- api_log_retention_days：60（日）
- sys_events_retention_years：3（年）

特に指定がなければ、これらは sys_params に保持し、  
本部管理画面（ai.）から変更可能とする前提で仕様を固める。

---

# 4. データ構造 / I/O 定義

## 4.1 パラメータ管理テーブル：sys_params

### 決定事項

    TABLE sys_params (
      id              BIGINT PK AUTO_INCREMENT,
      scope           VARCHAR(32) NOT NULL,   -- 'global' / 'shop' / 'channel' 等
      scope_id        BIGINT NULL,           -- shop_id 等。scope ごとに意味が変わる
      key             VARCHAR(128) NOT NULL, -- 'api_timeout_sec' など
      value           TEXT NOT NULL,
      value_type      VARCHAR(16) NOT NULL,  -- 'int' / 'string' / 'bool' / 'json'
      description     TEXT NULL,
      updated_at_utc  DATETIME NOT NULL,
      created_at_utc  DATETIME NOT NULL,
      UNIQUE(scope, scope_id, key)
    );

- 初期スコープ種別は以下の 3 種を正式採用する。  
  - global：システム全体で共通  
  - shop：店舗単位  
  - channel：help / line / shop / ai などチャネル単位
- SysConfig は以下の優先順位で値を解決する。  
  1. scope 指定値（例：shop スコープ）  
  2. scope='global'  
  3. .env  
  4. コード内ハードデフォルト
- SoR ポリシーに基づき、既存 DB フィールドと意味が重複する key は作成しない。

### 未決事項

- 将来の拡張として plan（プラン別）等の追加スコープを持つかどうか  
  （現時点では想定のみ。実装は後続フェーズ）。

### 確認項目

- channel 単位のチューニング（例：LINE だけタイムアウト長め）のニーズ。

---

## 4.2 生ログテーブル：api_call_logs

### 決定事項

    TABLE api_call_logs (
      id               BIGINT PK AUTO_INCREMENT,
      occurred_at_utc  DATETIME NOT NULL,
      channel          VARCHAR(32) NOT NULL,  -- 'help' / 'line' / 'shop' / 'ai' / 'cron' 等
      component        VARCHAR(64) NOT NULL,  -- 'StoresClient' / 'ChatworkClient' 等
      shop_id          BIGINT NULL,
      owner_id         BIGINT NULL,
      customer_id      BIGINT NULL,
      request_id       VARCHAR(64) NULL,      -- トレース用 UUID
      endpoint         VARCHAR(255) NOT NULL,
      http_method      VARCHAR(8) NOT NULL,
      status_code      INT NULL,
      latency_ms       INT NULL,
      request_body     MEDIUMTEXT NULL,       -- 必要に応じマスキング済 JSON
      response_body    MEDIUMTEXT NULL,       -- 同上
      error_code       VARCHAR(64) NULL,
      error_message    TEXT NULL
    );

- SysLogger 経由でのみ書き込みを行い、直接 INSERT は行わない。
- 生ログ保管期間はデフォルト 60 日とし、日次バッチ等で自動削除する。  
  値は api_log_retention_days パラメータで調整可能とする。

### 未決事項

- request_body / response_body のマスキング詳細ルール  
  （どのフィールドを保持・削除・ハッシュ化するか）。

### 確認項目

- Xserver の DB 容量を踏まえた最大保存期間。  
- 「必ず残したいフィールド」／「必ずマスクしたいフィールド」の運用要件。

---

## 4.3 匿名化イベントログ：sys_events

### 決定事項

    TABLE sys_events (
      id               BIGINT PK AUTO_INCREMENT,
      occurred_at_utc  DATETIME NOT NULL,
      event_type       VARCHAR(64) NOT NULL,   -- 'tutorial_session_started' 等
      channel          VARCHAR(32) NOT NULL,   -- 'help' / 'line' / 'shop' / 'ai'
      shop_id          BIGINT NULL,
      payload_json     JSON NOT NULL           -- 集計に必要な最小情報（ID は持たない）
    );

- 顧客 ID・電話番号など、直接個人を特定できる情報は保存しない。
- 解決有無、カテゴリ、所要時間、年齢帯など、集計に必要な最小情報のみを payload_json に持たせる。
- 匿名化イベントの保管期間はデフォルト 3 年とし、  
  sys_events_retention_years パラメータで調整可能とする。

### KPI として想定する event_type 例

- 顧客 AI チュートリアル系  
  - tutorial_session_started  
  - tutorial_session_resolved  
  - tutorial_session_escalated
- 加盟店サポート AI 系  
  - owner_ai_session_started  
  - owner_ai_session_resolved  
  - owner_ai_session_escalated
- チケット／対応系  
  - ticket_created  
  - ticket_closed
- AI 品質／ユーザー評価系  
  - ai_fallback_used  
  - user_satisfaction_submitted

### 未決事項

- v1.4 で「必須」とする event_type の最小セット。

### 確認項目

- 現時点で必ずモニタリングしたい KPI と、その対応 event_type。

---

## 4.4 🔧 パラメータ候補（データ構造関連）

- api_log_retention_days：60
- sys_events_retention_years：3
- sys_events_payload_max_bytes：32768（32KB 目安）

---

# 5. 共通処理フロー

## 5.1 外部 API 呼び出しフロー

### 決定事項

1. 各レイヤー（HELP / LINE / SHOP / AI）は、StoresClient 等のクライアントに  
   ビジネス上の意図（例：予約取得・解錠番号取得など）を渡す。
2. クライアントクラスは SysConfig から API キー・タイムアウト・リトライ回数などを取得し、  
   HttpClient（curl ベース）に HTTP 通信を委譲する。
3. HttpClient はリトライ制御・タイムアウト制御・レスポンス整形を行い、結果をクライアントに返却する。
4. クライアントは共通レスポンス形式に変換し、SysLogger を通じて api_call_logs に記録する。
5. 失敗時には SysErrorHandler に制御を渡し、エラーレベルとエスカレーション要否を判定する。

### 未決事項

- リトライ間隔（固定秒数／エクスポネンシャルバックオフ等）の具体値。

### 確認項目

- STORES / RemoteLock 等、クリティカルな API のみリトライを厚めにする運用要否。

---

## 5.2 エラーハンドリングフロー

### 決定事項

- エラーは次の 3 区分とする。  
  - USER_ERROR  
    - ユーザー入力不備等。UI 内で完結し、通知は不要。  
  - SYSTEM_WARN  
    - 一時的な外部 API エラー等。再試行または軽度通知の対象。  
  - SYSTEM_CRITICAL  
    - 資格情報不備、特定 API の長時間ダウンなど、人間の早期介入が必要なレベル。

- SysErrorHandler::handle(errorContext) に以下を集約する。  
  - ログ出力（api_call_logs、および必要に応じ別途 error_logs）  
  - エスカレーションルールの適用（SysEscalation 呼び出し）

- 「システムダウン級」は SYS_CORE の文脈では  
  「アプリ自体は動いているが、一部の外部 API 等が深刻な状態」を指す。  
  サーバ自体の停止（Xserver 障害等）は本システム外の監視の責務とする。

### 未決事項

- SYSTEM_WARN から SYSTEM_CRITICAL に格上げする閾値  
  （連続失敗回数・継続時間等）。

### 確認項目

- ChatWork・メール・SMS 通知の現場感覚に基づいた「うるさくない」閾値。

---

# 6. 通知 / 外部連携（エスカレーション基盤）

## 6.1 エスカレーションルール：escalation_rules

### 決定事項

    TABLE escalation_rules (
      id               BIGINT PK AUTO_INCREMENT,
      rule_code        VARCHAR(64) NOT NULL,
      severity         VARCHAR(16) NOT NULL,   -- 'info' / 'warn' / 'critical'
      channel_chatwork TINYINT(1) NOT NULL DEFAULT 1,
      channel_email    TINYINT(1) NOT NULL DEFAULT 0,
      channel_sms      TINYINT(1) NOT NULL DEFAULT 0,
      chatwork_room_id VARCHAR(64) NULL,
      email_to         VARCHAR(255) NULL,
      sms_to           VARCHAR(32) NULL,
      active           TINYINT(1) NOT NULL DEFAULT 1,
      params_json      JSON NULL,              -- 通知テンプレ、タイトル 等
      created_at_utc   DATETIME NOT NULL,
      updated_at_utc   DATETIME NOT NULL,
      UNIQUE(rule_code)
    );

- SysEscalation は rule_code とコンテキスト（shop_id / actor_type 等）を受け取り、  
  ChatworkClient・メール送信モジュール・SmsClient に振り分ける。

## 6.2 オプションとの関係

### 決定事項

- shop テーブルに格納される各種オプションフラグ（例：電話代行サービス契約の有無 等）は、  
  エスカレーション条件を組み立てるための入力情報として参照できるようにする。
- SYS_CORE は、これらオプション値を利用して  
  「どの rule_code を使うか」「どのチャネルに通知するか」を  
  パラメータベースで切り替え可能にするところまでを担当する。
- 個別オプションごとの標準挙動（電話代行あり店舗の夜間運用 等）は、  
  後続フェーズの機能仕様で escalation_rules と sys_params を組み合わせて定義する。  
  本フェーズでは例示のみに留め、ロジックとして固定しない。

## 6.3 夜間通知ポリシー（ベースライン）

### 決定事項

- 夜間（例：0:00〜6:00）の SMS は原則抑制し、  
  severity='critical' かつルール側で SMS 許可されている場合のみ送信する。
- WARN レベルの通知は原則 ChatWork のみとし、SMS・メールは伴わない。

---

## 6.4 🔧 パラメータ候補（通知関連）

- night_sms_quiet_hours：{"start":"00:00","end":"06:00"}
- notify_critical_via_sms：true
- notify_warn_via_chatwork_only：true

### 未決事項

- 利用開始時点で定義しておく rule_code の初期セット。

### 確認項目

- 現在運用している ChatWork ROOM 構成（本部 ROOM / 店舗別 ROOM 等）と、  
  各ルールの送信先イメージ。

---

# 7. セキュリティ・権限

## 7.1 actor コンテキスト

### 決定事項

SYS_CORE では、処理主体を明示するため以下を共通コンテキストとして扱う。

- actor_type  
  - customer  
  - shop_owner  
  - hq_admin  
  - hq_staff  
  - system_cron
- actor_id  
  - 該当する ID（customer_id / owner_id / staff_id 等）

ログ・通知・イベントには可能な範囲で actor_type / actor_id を含め、  
監査性とトレース性を高める。

顧客情報を AI に渡す際は、Phase 0 / base 仕様で定めたマスキング方針に従う。

## 7.2 未決事項

- hq_admin / hq_staff をさらに細分化したロール  
  （例：経理専用、サポート専用）をどのフェーズで導入するか。

## 7.3 確認項目

- 現時点で SYS_CORE レベルでロール追加が必要か。  
  必要なければ上記 5 区分で開始し、後続フェーズで拡張する。

---

# 8. 未決課題 / 次フェーズへの引き継ぎ

## 8.1 Phase 1 内での未決課題一覧

1. 共通レスポンス JSON スキーマ  
   （status / code / data / error 等の構造）。
2. request_body / response_body のマスキングルール詳細。
3. escalation_rules.rule_code の初期セット。
4. SYSTEM_WARN → SYSTEM_CRITICAL への昇格基準（連続失敗・時間ベース等）。
5. v1.4 で必須とする sys_events.event_type の最小セット。

これらは Phase 2 以降または実装段階でのチューニングで破綻しない粒度であり、  
本フェーズの目的（仕組みの土台の確定）は達成されている。

## 8.2 次フェーズ（AI_CORE）への引き継ぎ

- SysConfig（パラメータ取得）
- OpenAiClient（AI モデル呼び出し）
- SysLogger / sys_events（AI 品質・KPI ログ）
- SoR ポリシー（既存 DB 優先）

これらを前提として、Phase 2 では AI コアの  
入出力契約・内部ロジック・ログ連携を設計する。

---

## ✔ PHASE_01_SYS_CORE は以上で「仕様書として完成」とする。
