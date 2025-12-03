# PHASE_07_CRON_SYS — 自動処理／ジョブ管理層 v1.4

## 0. このドキュメントの位置づけ

- 本ドキュメントは、「加盟店サポート自動化プロジェクト v1.4」における**Phase7：自動処理／ジョブ管理層（CRON_SYS）** の仕様を定義する。
- 目的は、これまでの Phase0〜6 で定義された以下の要素を支える**定期処理・バッチ処理（cronスクリプト）を一元的に設計すること** である。
    - SALES_LINK（売上CSV取込）
    - KPI_ANALYTICS（KPI集計）
    - AI-INBOX（請求・チケット）
    - SHOP_PORTAL（オーナーダッシュボード）
    - HELP_UI / LINE_UI（顧客／LINEチャネル）
    - FOUNDATION / DB_CORE（バックアップ・ログ保持）

本フェーズでは、以下を決める。

- ジョブ一覧と役割
- 各ジョブの I/O 契約（CLI引数／参照・更新テーブル）
- ログ・実行履歴・エラー通知・ロック方針
- 日次／月次／高頻度ジョブのスケジュール
- ロイヤリティ特例ルール（`shop_royalty_relief_rules`）評価バッチの組み込み

---

## 1. 対象スコープと非スコープ

### 1-1. スコープ（CRON_SYS が扱うもの）

- cron スクリプト／バッチジョブの一覧と命名規約
- 各ジョブの
    - 実行タイミング（例：毎日／毎月1日／5分間隔など）
    - CLIインタフェース（引数／戻り値）
    - 参照／更新する DB テーブル
    - ログ出力方針（ファイル／DB）
    - 異常時の通知（ChatWork／メール／SMS）方針
- ジョブ実行履歴テーブル（`sys_job_runs`）の設計
- ロイヤリティ特例ルール `shop_royalty_relief_rules` の「条件判定＋アラート」を行う
月次ジョブの I/O 設計（ロジック本体は Phase5差分仕様を参照）

### 1-2. 非スコープ（他フェーズで決まっているもの）

- 各 UI 要素（HELP_UI / LINE_UI / SHOP_PORTAL / AI-INBOX）の画面・文言仕様
- AIコアの意図分類・応答生成ロジック
- SALES_LINK／KPI_ANALYTICS のビジネスロジック（「売上をどう解釈するか」「KPIの定義」など）
- ロイヤリティ計算ロジックそのもの
（本フェーズで扱うのは「特例ルールが成立したことを通知する」まで。
実際のロイヤリティ変更は AI-INBOX から人手で行う）

---

## 2. 関連要素・依存関係

### 2-1. 依存フェーズ

- Phase0 FOUNDATION / DB_CORE
    - `sys_events`／`sys_params`／各種ログテーブル
    - バックアップ・監査ログ・通知ポリシー
- Phase1 SYS_CORE / Phase2 AI_CORE / API_COMM
    - エラー通知・夜間SMS抑制ポリシー
    - 外部サービス連携（ChatWork／SMS／STORES／RemoteLOCK 等）の共通 I/F
- Phase3 HELP_UI / LINE_UI
    - チケット／通知タイミングの期待値
- Phase4 SHOP_PORTAL
    - KPI／請求／契約情報の表示前提
- Phase5 AI-INBOX
    - 請求ワークフロー（`cron_invoices.php` は draft 生成まで）
    - ロイヤリティ特例 UI と `shop_royalty_relief_rules`／`shop_royalty_history`
- Phase6 SALES_LINK / KPI_ANALYTICS
    - `sales_records`／`sales_import_jobs`／`sales_import_errors`
    - `shop_kpis`／`shop_kpi_metrics`
    - KPI・売上の SoR／集計方針

### 2-2. CRON_SYS からの参照先

- テーブル（抜粋）
    - `sales_records`
    - `sales_import_jobs`, `sales_import_errors`
    - `shop_kpis`, `shop_kpi_metrics`
    - `shops`, `shop_royalty_history`, `shop_royalty_relief_rules`
    - `tickets`
    - `sys_events`
    - `audit_logs`（一部の自動処理で重要な変更を記録する場合）
- パラメータ
    - `SALES_LINK_DEFAULT_PARAMS.md`, `KPI_ANALYTICS_DEFAULT_PARAMS.md`
    - `AI_INBOX_DEFAULT_PARAMS.md`（ロイヤリティ特例チェック関連）
    - 本フェーズで定義する `CRON_SYS_DEFAULT_PARAMS`（ジョブのON/OFF／挙動調整）

---

## 3. 共通設計ポリシー（全ジョブ共通）

### 3-1. 実行形式 / CLI

- 全ジョブは PHP CLI スクリプトとして `cron_*.php` 名で配置する。
    - 例：`cron_sales_import.php`, `cron_kpi_aggregate.php` など
- 実行ユーザは Web アプリと同じ OS ユーザ（例：`www-data`）を想定。
- CLI 引数は `key=value` 形式で受け付ける。

例:

```bash
php cron_sales_import.php --job-id=123
php cron_kpi_aggregate.php --granularity=daily --date=2025-12-02
php cron_royalty_relief_check.php --month=2025-11

```

### 3-2. ロック方針（同時実行の防止）

- 各ジョブは「同名ジョブの多重起動」を禁止する。
- 実装方針（v1.4）：
    - ジョブ開始時に `/tmp/hygi_cron/{job_name}.lock` へのファイルロックを取得する。
    - すでにロックが存在する場合は
        - ログに `status='error'`, `message='lock_exists'` を記録し即時終了する。
        - 重大度は `warning`（通知のしきい値はパラメータで制御）。
- 将来、DBベースロックが必要になった場合は `sys_job_locks` テーブル追加で対応する。

### 3-3. 実行履歴テーブル（`sys_job_runs`）

### 3-3-1. 目的

- 各ジョブの実行結果（成功／警告／エラー）と所要時間を可視化する。
- AI-INBOX／HQダッシュボードから「バッチが正常に回っているか」を確認できるようにする。

### 3-3-2. DDL（新規）

```sql
CREATE TABLE sys_job_runs (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  job_name VARCHAR(64) NOT NULL COMMENT 'cronスクリプト識別子（例：cron_sales_import）',
  started_at DATETIME NOT NULL COMMENT 'ジョブ開始時刻',
  finished_at DATETIME NULL COMMENT 'ジョブ終了時刻',
  status ENUM('success','warning','error') NOT NULL COMMENT 'ジョブの最終ステータス',
  message VARCHAR(255) NULL COMMENT '概要メッセージ（エラー理由など）',
  affected_rows INT NULL COMMENT '影響行数など、ジョブに応じた任意の件数',
  error_count INT NULL COMMENT 'エラー件数（sales_import_errors 件数等）',
  run_context JSON NULL COMMENT 'ジョブ固有の実行コンテキスト（対象年月など）',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_job_runs_job_time (job_name, started_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

- 各ジョブは
    - 開始時に `started_at`／`status='running'` で1行を INSERT
    - 終了時に `finished_at`／`status`／`message`／`affected_rows` 等を UPDATE
- `status='running'` で一定時間（例：2時間）以上経過したものは「ハング検知」として、別途監視対象とする（将来拡張）。

### 3-4. ログ出力方針

- ファイルログ：
    - `/var/log/hygi_cron/{job_name}.log`（または Xserver の指定パス）に標準出力／標準エラーをリダイレクト。
- DBログ：
    - 重要なイベント（条件達成／重大エラー）は `sys_events` にも記録する。
- 開発環境では、必要に応じて詳細ログレベル（DEBUG）を有効化するが、v1.4 では prod のみ想定。

### 3-5. エラー時の通知ポリシー

- 各ジョブは「致命的な失敗」「一定回数以上の連続失敗」を検知した場合、
    - `sys_events` に `severity='error'` 相当で記録し、
    - SYS_CORE の通知ルールに従って ChatWork／メール／SMS へ通知される。
- 夜間（例：0:00〜6:00）は
    - critical 以外のSMSは送信しない（既存の夜間クワイエットタイムポリシーを踏襲）。
- 「1回の失敗」は通常は ChatWork のみ、
    
    「連続3回以上の失敗」は ChatWork＋SMS を推奨（パラメータで調整）。
    

### 3-6. スケジュール設定に関する前提

- 実際の crontab（OS レベル）の設定は、サーバ運用側で行う。
- 本仕様書では「推奨スケジュール」を明示し、
    - 実行日／頻度は OS cron
    - 対象期間（どの年月・日付を処理するか）は CLI引数＋DBパラメータ
        
        で制御する前提とする。
        
- v1.4 時点では、ジョブ実行時刻そのものは OS cron の変更が必要であり、
    
    完全な「時刻パラメータ化」は将来の改善（Phase9 以降）の課題とする。
    

---

## 4. ジョブ一覧（サマリ）

| ID | スクリプト名 | 役割概要 | 推奨頻度 |
| --- | --- | --- | --- |
| J1 | `cron_sales_import.php` | STORES 売上CSV取込（`sales_records` 更新） | 日次 or 手動トリガ（月末前後） |
| J2 | `cron_kpi_aggregate.php` | KPI 日次・月次集計（`shop_kpis`／`shop_kpi_metrics`） | 日次＋月次 |
| J3 | `cron_invoices.php` | 請求ドラフト生成（自動計上） | 毎月1日 |
| J4 | `cron_royalty_relief_check.php` | ロイヤリティ特例ルール条件評価＋アラート | 毎月1日 |
| J5 | `cron_sms_followup.php` | 未返信チケットへの追撃通知（ChatWork／SMS） | 5分間隔 |
| J6 | `cron_meeting_reminders.php` | HQミーティング前日／1時間前リマインド | 10分間隔 |
| J7 | `cron_backup_db.php` | DB＋請求PDFバックアップ | 毎日 |
| J8 | `cron_cleanup_logs.php` | ログ・KPI・エラーデータ削除 | 毎日 or 週次 |

※OP_MAIL（電話代行メール解析）は Push トリガであり cron の対象外。

※将来の STORES API 連携や AI再学習ジョブは Phase9 以降で J9+ として追加する。

---

## 5. 個別ジョブ仕様

### 5-1. `cron_sales_import.php` — STORES 売上CSV取込

### 5-1-1. 概要

- Phase6 SALES_LINK の実装として、STORES からアップロードされた売上CSV3種を取り込み、
    
    `sales_records` に統合する。
    
- `sales_import_jobs`／`sales_import_errors` の状態管理も担う。

### 5-1-2. CLI I/O 契約

- 想定呼び出し

```bash
# 単一ジョブID指定
php cron_sales_import.php --job-id=123

# 自動モード：status in ('uploaded','validating') を順に処理
php cron_sales_import.php --auto=1

```

- 主な引数
    - `job-id`（任意）: 処理対象の `sales_import_jobs.id`
    - `auto`（bool）: 1 の場合、未処理ジョブを古い順に処理
- 戻り値
    - 正常終了：`exit 0`
    - 入力なし（対象ジョブなし）：`exit 0`（ログに「対象なし」を残す）
    - 異常終了：`exit 1`（sys_events＋通知）

### 5-1-3. 処理フロー概要

1. `sys_job_runs` へ `status='running'` で開始ログを記録。
2. 対象 `sales_import_jobs` の取得
    - `job-id` 指定時：その1件のみ
    - `auto` 時：`status in ('uploaded','validating')` を `created_at` 昇順で1件取得
3. ジョブ状態を `validating` → `importing` → `completed`／`completed_with_warning`／`failed` へ遷移
4. CSV行を走査し
    - 正常行：`sales_records` に INSERT／UPDATE
    - 異常行：`sales_import_errors` に登録
5. 行数／金額の整合性チェック
    - 差分率が `reconcile_warn_percent` を超える場合は `completed_with_warning`
6. 結果を `sys_job_runs` に記録し終了。

### 5-1-4. ログ・エラー通知

- 失敗 (`status='failed'`) 場合
    - ChatWork（本部ルーム）に「売上取込ジョブ失敗」のメッセージ送信。
    - エラー詳細は `sales_import_errors`／ログファイルを参照。
- `completed_with_warning` の場合
    - INFO/Warning レベルの `sys_events` を発行し、KPI集計側で把握できるようにする。

### 🔧 パラメータ候補（CRON_SYS_DEFAUT_PARAMS 連携）

- `cron.sales_import.auto_enable`（bool）
    
    自動モードでの取り込みを定期的に実行するか。
    
- `cron.sales_import.max_jobs_per_run`（int）
    - `auto` 実行時に1回で処理するジョブ数上限（デフォルト1）。
- `cron.sales_import.notify_on_warning`（bool）
    
    `completed_with_warning` の場合に ChatWork 通知するか。
    
- `cron.sales_import.error_severity_threshold`（int）
    
    エラー件数がこの値を超えたら `status='error'` 相当として扱う。
    

---

### 5-2. `cron_kpi_aggregate.php` — KPI 日次／月次集計

### 5-2-1. 概要

- Phase6 KPI_ANALYTICS の集計バッチ。
- `sales_records`／`tickets`／`tutorial_logs`／`sys_events` などから、
    
    `shop_kpis`／`shop_kpi_metrics` を日次／月次で集計する。
    

### 5-2-2. CLI I/O 契約

```bash
# 日次集計（単日）
php cron_kpi_aggregate.php --granularity=daily --date=2025-12-02

# 月次集計（単月）
php cron_kpi_aggregate.php --granularity=monthly --month=2025-11

# 拡張KPI込み再計算
php cron_kpi_aggregate.php --granularity=monthly --month=2025-11 --extended=1

```

- 引数
    - `granularity`（必須）: `daily` / `monthly`
    - `date`（`daily` 時必須）: `YYYY-MM-DD`
    - `month`（`monthly` 時必須）: `YYYY-MM`
    - `extended`（任意）: 1 の場合、`shop_kpi_metrics` も再計算

### 5-2-3. 処理フロー概要

1. `sys_job_runs` に開始記録。
2. 対象期間の既存 `shop_kpis`／`shop_kpi_metrics` を一度クリア（または上書き）。
3. KPI_ANALYTICS 仕様に従って、各ショップについて
    - 売上・件数・trial比率・AI自己解決率・チケット件数・SLA 等を集計。
4. `kpi_data_retention_months` を超える古いデータは削除またはアーカイブ（詳細は 5-8 `cron_cleanup_logs.php` で扱う）。
5. 集計件数・対象店舗数を `sys_job_runs.affected_rows` に記録して終了。

### 🔧 パラメータ候補

- `cron.kpi.daily_enable`（bool）
    
    日次集計を有効にするか。
    
- `cron.kpi.monthly_enable`（bool）
    
    月次集計を有効にするか。
    
- `cron.kpi.daily_target_offset_days`（int）
    
    例：1 → 「昨日」を対象日とする。
    
- `cron.kpi.monthly_target_offset_months`（int）
    
    例：1 → 「前月」を対象月とする。
    
- `cron.kpi.notify_on_error`（bool）
    
    集計失敗時に通知するか。
    

---

### 5-3. `cron_invoices.php` — 請求ドラフト生成

### 5-3-1. 概要

- Phase5 AI-INBOX 仕様に基づき、毎月1日に
    - 対象オーナー／店舗の請求ドラフト (`invoices`／`invoice_items`) を生成し、
    - 売上ベースのロイヤリティ・システム利用料・オプション等の自動計上行を積む。
- PDF生成・送信は AI-INBOX の UI 操作で行う（本ジョブは行わない）。

### 5-3-2. CLI I/O 契約

```bash
php cron_invoices.php --billing-month=2025-11

```

- 引数
    - `billing-month`（任意）: 請求対象月 (`YYYY-MM`)。省略時は「前月」を対象とする。
- 振る舞い
    - 指定月の `invoices`／`invoice_items` を再生成する場合は
        
        既存ドラフトを上書きするかどうかは AI-INBOX 仕様に合わせる
        
        （v1.4 では「未発行の draft のみ上書き」）。
        

### 5-3-3. ロジックの範囲（CRON_SYS 観点）

- ロイヤリティ計算・オプション計上の詳細は Phase5/6 に従う。
- 本ジョブは
    - 対象オーナー・店舗の抽出
    - 該当設定 (`shops`, `shop_royalty_history`, 各種 fee テーブル) の読み込み
    - 自動計上分の `invoice_items` 追加
    - 実行結果のログ・エラー通知
        
        を行う。
        

### 🔧 パラメータ候補

- `cron.invoices.auto_enable`（bool）
    
    毎月1日に自動でドラフト生成するか。
    
- `cron.invoices.default_billing_day`（int）
    
    デフォルトの締め日（例：1）。
    
- `cron.invoices.notify_on_error`（bool）
    
    ドラフト生成に失敗した場合に通知するか。
    

---

### 5-4. `cron_royalty_relief_check.php` — ロイヤリティ特例ルール評価＋アラート

### 5-4-1. 概要

- Phase5/6 差分仕様で定義された `shop_royalty_relief_rules` を評価し、
    - 条件を満たした店舗に対して
        - AI-INBOX 用チケット、または
        - `sys_events`
            
            を生成して HQ に「特例戻し条件を満たした」ことを知らせる。
            
- **重要**：v1.4 では
    - ロイヤリティの自動変更は行わない（`alert_only=1` を前提）
    - 実際のロイヤリティ変更は AI-INBOX 上の HQ 操作により行う

### 5-4-2. CLI I/O 契約

```bash
# 通常運用：前月分をチェック
php cron_royalty_relief_check.php --month=2025-11

```

- 引数
    - `month`（任意）: 評価対象月 (`YYYY-MM`)。
        
        省略時は「前月」を対象とする。
        
- 戻り値
    - 正常終了：`exit 0`
    - 異常終了：`exit 1`

### 5-4-3. 評価ロジック

1. `sys_job_runs` に開始ログを記録。
2. `ai_inbox.royalty_relief_check_enable` が false の場合、何もせず終了。
3. `is_active=1` の `shop_royalty_relief_rules` を取得。
4. ルールごとに以下を評価。
    - `rule_type='monthly_sales_over'` の場合
        - 対象店舗の評価対象月の売上合計（税抜）を `sales_records` から集計。
            
            どの `source_type` を対象にするかは SALES_LINK 側の `source_type` 定義に従う。
            
        - `sales_sum >= threshold_value` で「条件達成」と判定。
    - `rule_type='months_since_open_over'` の場合
        - `shops.open_date` から評価対象月末までの経過月数を計算。
        - 経過月数 >= `threshold_value` で「条件達成」と判定。
5. 条件達成時のふるまい
    - `alert_only=1`（v1.4 標準）の場合
        - `ai_inbox.royalty_relief_alert_channel` により分岐。
            - `ticket`：
                - `tickets` に
                    - `source='system'`
                    - `category='royalty_relief'`
                    - `shop_id`／`owner_id`
                    - `subject="{店舗名}：ロイヤリティ特例の戻し条件を満たしました"`
                    - `status='open'`
                    - `priority='warning'`
                        
                        を持つチケットを自動起票する。
                        
                - 同一 `shop_id`／`rule_id`／`month` で既存 open チケットがある場合は重複作成を避ける。
            - `sys_event`：
                - `sys_events` に
                    - `event_type='royalty_relief_condition_met'`
                    - `severity='info' or 'warning'`
                    - `payload_json` に `shop_id`／`rule_id`／`rule_type`／`threshold_value`／`month` 等
                        
                        を記録する。
                        
6. 処理件数（条件達成ルール数／作成チケット数）を `sys_job_runs` に記録して終了。

### 🔧 パラメータ候補

- ※AI_INBOX_DEFAULT_PARAMS に既に追加済みの前提：
    - `ai_inbox.royalty_relief_check_enable`（bool）
    - `ai_inbox.royalty_relief_check_lookback_months`（int）
    - `ai_inbox.royalty_relief_alert_channel`（string: `ticket` or `sys_event`）
    - `ai_inbox.royalty_relief_rule_edit_role`（string）
- CRON_SYS 側として追加する候補：
    - `cron.royalty_relief_check.auto_enable`（bool）
        
        毎月1日に自動実行するか。
        
    - `cron.royalty_relief_check.notify_on_error`（bool）
        
        チェック処理自体の失敗時に通知するか。
        

---

### 5-5. `cron_sms_followup.php` — 未返信チケット追撃

### 5-5-1. 概要

- AI-INBOX／HELP_UI／LINE_UI から発生したチケットのうち
    - 一定時間以上「未返信」のもの、
    - SLA 違反リスクがあるもの
        
        に対して、ChatWork／SMS による追撃通知を行う。
        

### 5-5-2. 実行頻度

- OS cron で 5分間隔を想定（`/5 * * * *`）。

### 5-5-3. ロジック概要

1. `sys_job_runs` に開始ログ。
2. チケットテーブルから
    - `status in ('open','waiting')`
    - 最終更新時刻が
        - `sla_warn_minutes` を超過したもの（警告）
        - `sla_danger_minutes` を超過したもの（危険）
            
            を抽出（AI-INBOX 仕様に準拠）。
            
3. 夜間SMS抑制ポリシーに従い
    - 夜間帯は ChatWork のみ
    - 日中は ChatWork＋SMS（重大度に応じて）
4. 通知送信結果を `sys_events` に記録。
5. 対象件数・エラー件数を `sys_job_runs` に保存。

### 🔧 パラメータ候補

- `cron.sms_followup.enable`（bool）
- `cron.sms_followup.max_tickets_per_run`（int）
    
    1回の実行で通知するチケット上限。
    
- `cron.sms_followup.notify_on_error`（bool）

---

### 5-6. `cron_meeting_reminders.php` — HQミーティングリマインド

### 5-6-1. 概要

- `hq_meeting_slots`／`hq_meetings` を参照し、
    - 前日リマインド（例：前日 9:00）
    - 1時間前リマインド
        
        を行う。
        
- v1.4 時点では MTG ツールは **Google Meet を標準** とし、`hq_meetings.meeting_join_url` に格納された **Google Meet 参加URL** を、リマインド通知本文に必ず含める（NULL の場合のみ URL なしで送信）。

### 5-6-2. 実行頻度

- OS cron：10分間隔（`/10 * * * *`）を想定。
- スクリプト内で
    - 「前日リマインド済みフラグ」
    - 「1時間前リマインド済みフラグ」
        
        を見て、必要なタイミングだけ送信する。
        

### 5-6-3. ロジック概要

1. `sys_job_runs` に開始ログを記録。
2. 対象期間を判定し、リマインド対象の `hq_meetings` を抽出する。
    - 前日リマインド：
        - 現在が `cron.meeting_reminders.prev_day_hour`（例：9:00）±数分のタイミングのとき、翌日開催のミーティングで `reminded_prev_day=0` のものを対象とする。
    - 1時間前リマインド：
        - `meeting_start_at` の60分前〜50分前に該当し、`reminded_one_hour=0` のものを対象とする。
3. 通知チャネル：
    - ChatWork DM／グループ
    - メール（必要なら）
        
        通知本文には少なくとも次を含める。
        
    - 日付・時刻
    - meeting_type（例：初回オリエン／運営相談 等）
    - `meeting_join_url`（設定されている場合。Google Meet 参加URLとして、そのままクリック可能なリンクにする）
4. 通知送信に成功したミーティングについて、`hq_meetings` 側のフラグ（例：`reminded_prev_day`, `reminded_one_hour`）を ON にする。
5. 対象件数・送信件数・失敗件数を集計し、`sys_job_runs` に記録して終了する。

### 🔧 パラメータ候補

- `cron.meeting_reminders.enable`（bool）
- `cron.meeting_reminders.prev_day_hour`（int, 0-23）
- `cron.meeting_reminders.use_email`（bool）

---

### 5-7. `cron_backup_db.php` — DB＋請求PDFバックアップ

### 5-7-1. 概要

- DB 全体と請求PDFを毎日バックアップし、世代管理を行う。
- アプリ層バックアップであり、インフラスナップショットと併用する。

### 5-7-2. 実行頻度

- 毎日 3:30（推奨）。

### 5-7-3. ロジック概要

1. `sys_job_runs` に開始ログ。
2. DBダンプ（mysqldump 等）を取得し、日付付きファイル名で保存。
3. 請求PDF保存ディレクトリをアーカイブ（tar／zip 等）。
4. 過去のバックアップのうち、保持世代数（例：7）を超えるものを削除。
5. 失敗時は `sys_events` に記録し、ChatWork／SMS 通知。
6. バックアップサイズ・ファイル数を `sys_job_runs` に記録。

### 🔧 パラメータ候補

- `cron.backup_db.enable`（bool）
- `cron.backup_db.keep_generations`（int, デフォルト7）
- `cron.backup_db.notify_on_error`（bool）

---

### 5-8. `cron_cleanup_logs.php` — ログ・KPI・エラーデータ削除

### 5-8-1. 概要

- 各種 retention パラメータに従い、古いログ・KPI・エラーレコードを削除またはアーカイブする。
- 対象例：
    - `sales_import_errors`（`error_row_retention_days`）
    - `shop_kpis`／`shop_kpi_metrics`（`kpi_data_retention_months`）
    - AIコアログ（`turn_log_retention_days_full` 等）
    - `sys_events`（`sys_events_retention_years`）

### 5-8-2. 実行頻度

- 毎日 4:00 または週1回（運用状況に応じて）。

### 5-8-3. ロジック概要

1. `sys_job_runs` に開始ログ。
2. 各テーブルごとに
    - 対応する retention パラメータを読み込む。
    - 対象期間を算出。
    - 削除／アーカイブ件数を計測しつつ実行。
3. 件数を `sys_job_runs.affected_rows` に集計（テーブル別詳細は `run_context` に JSON で保存）。
4. 大量削除が発生する場合はバッチサイズ制御を行う。

### 🔧 パラメータ候補

- `cron.cleanup_logs.enable`（bool）
- `cron.cleanup_logs.max_rows_per_table`（int）
- `cron.cleanup_logs.notify_on_error`（bool）

---

## 6. スケジュール設計（推奨）

### 6-1. 日次スケジュール（例）

- 02:30〜：必要に応じて `cron_sales_import.php --auto`（日次運用する場合）
- 03:00：`cron_invoices.php --billing-month=前月`（毎月1日のみ）
- 03:05：`cron_royalty_relief_check.php --month=前月`（毎月1日のみ）
- 03:10：`cron_kpi_aggregate.php --granularity=daily --date=前日`
- 03:15：`cron_kpi_aggregate.php --granularity=monthly --month=前月`（毎月1日のみ）
- 03:30：`cron_backup_db.php`
- 04:00：`cron_cleanup_logs.php`

### 6-2. 高頻度ジョブ

- `cron_sms_followup.php`：5分間隔
- `cron_meeting_reminders.php`：10分間隔

### 6-3. 手動再実行の扱い

- いずれのジョブも CLI から手動再実行可能。
- 再実行時の挙動
    - `cron_sales_import.php`：ジョブID指定で再取込。
    - `cron_kpi_aggregate.php`：対象日／月を明示して再集計。
    - `cron_invoices.php`：未発行ドラフトのみ再生成する前提。
- 多重実行防止のため、ロックにより同時実行を抑止する。

---

## 7. DB／マイグレーション方針（Phase7 分）

### 7-1. 新規テーブル

- `sys_job_runs`（本ドキュメント 3-3-2 参照）

### 7-2. 既存テーブルの利用

- `shop_royalty_relief_rules` は Phase5 差分仕様で定義済み。
    
    Phase7 では DDL 追加なし（評価ロジックのみ）とする。
    

### 7-3. マイグレーション適用

- `sys_job_runs` の CREATE は、他フェーズと同様に
    - Phase9 PACKAGE_BUILD で生成するマイグレーションスクリプトに含める。
- v1.4 設計時点では、実 DB に存在しない前提で扱う。

---

## 8. 将来拡張フック（Phase9 以降）

- STORES API 連携による完全自動売上取込
    - `cron_sales_import.php` とは別に `cron_sales_fetch_api.php` を追加する余地を残す。
- AI再学習ジョブ
    - AIコアログ／問い合わせ事例をもとにした夜間バッチ学習（OpenAI API のバッチ扱いなど）用ジョブ。
- ジョブスケジュールの DB パラメータ化
    - 将来的には
        - ジョブ名／cron式／有効フラグを DB に持ち、
        - 1本の「ジョブディスパッチャ」スクリプトから動的に実行する構成も検討可能。
    - v1.4 では OS cron による固定スケジュールを前提とし、本仕様は「推奨スケジュール」を定義するに留める。
