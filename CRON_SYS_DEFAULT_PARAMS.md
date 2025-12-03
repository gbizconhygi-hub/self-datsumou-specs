# CRON_SYS_DEFAULT_PARAMS — 自動処理層パラメータ定義 v1.4

## 0. このドキュメントの位置づけ

- 本ドキュメントは Phase7 要素 **CRON_SYS**（自動処理／ジョブ管理層）で利用する  
  「各 cron ジョブの ON/OFF・挙動調整」に関するパラメータ群を整理したもの。
- 実体は
  - `sys_params` 系テーブル
  - または環境別設定ファイル（`.env` / `config.php` 等）
  とし、本書は論理キー名と意味・デフォルト値を示す。
- ここで定義するキーは、
  - `PHASE_07_CRON_SYS.md` の各ジョブごとの「🔧 パラメータ候補」で挙げた内容をベースとした **コアセット** とする。
- スケジュール（何時何分にジョブを走らせるか）は v1.4 時点では OS cron で管理し、  
  本書のパラメータは
  - ジョブの有効／無効
  - 対象件数・挙動のしきい値
  - エラー時の通知有無
  などを制御する役割とする。

---

## 1. パラメータ定義

### 1-1. SALES_IMPORT 関連（`cron_sales_import.php`）

#### 1-1-1. `cron.sales_import.auto_enable`

- 型: `bool`
- デフォルト値: `false`
- スコープ: システム全体
- 説明:
  - `cron_sales_import.php --auto=1` による「未処理ジョブの自動取込」を定期実行するかどうか。
  - `false` の場合、基本は HQ が AI-INBOX 側から「取込ジョブ作成 → 手動実行」する運用を想定。
  - 運用が安定し、CSV アップロードタイミングが固まってきた段階で `true` に切り替える。

---

#### 1-1-2. `cron.sales_import.max_jobs_per_run`

- 型: `int`
- デフォルト値例: `1`
- スコープ: システム全体
- 説明:
  - `--auto` 実行時に、1回の起動で処理する `sales_import_jobs` の上限件数。
  - サーバ負荷や CSV 行数に応じて調整する。
  - 例：
    - 1 → 「1回の cron 実行で1ジョブのみ処理（安全寄り）」。

---

#### 1-1-3. `cron.sales_import.notify_on_warning`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - 取込結果が `completed_with_warning` となった場合に、ChatWork へ通知するかどうか。
  - `true` の場合：
    - 行数／金額差分が `reconcile_warn_percent` を超えているジョブについて、確認を促す。
  - `false` の場合：
    - エラー (`failed`) のみを通知対象とする。

---

#### 1-1-4. `cron.sales_import.error_severity_threshold`

- 型: `int`
- デフォルト値例: `100`
- スコープ: システム全体
- 説明:
  - `sales_import_errors` に記録されるエラー行数がこの値を超えた場合に、
    - 実行結果を「致命的エラー」として扱うしきい値。
  - 例：
    - 100行以上エラー → ChatWork＋SMS レベルのアラート対象にする、など。

---

### 1-2. KPI 集計関連（`cron_kpi_aggregate.php`）

#### 1-2-1. `cron.kpi.daily_enable`

- 型: `bool`
- デフォルト値: `false`
- スコープ: システム全体
- 説明:
  - KPI 日次集計バッチを有効にするかどうか。
  - `KPI_ANALYTICS_DEFAULT_PARAMS` における `kpi_allow_daily` と整合させる前提で、
    - 初期は `false`（月次中心の運用）
    - パフォーマンスや運用ニーズを見て `true` に切り替える。

---

#### 1-2-2. `cron.kpi.monthly_enable`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - KPI 月次集計バッチを有効にするかどうか。
  - v1.4 では月次集計は必須のため、基本は `true` を前提とする。

---

#### 1-2-3. `cron.kpi.daily_target_offset_days`

- 型: `int`
- デフォルト値: `1`
- スコープ: システム全体
- 説明:
  - 日次集計で「何日前の日付を対象日とするか」を指定するオフセット。
  - 例：
    - 1 → 「前日分」を対象として集計。

---

#### 1-2-4. `cron.kpi.monthly_target_offset_months`

- 型: `int`
- デフォルト値: `1`
- スコープ: システム全体
- 説明:
  - 月次集計で「何ヶ月前の月を対象とするか」を指定するオフセット。
  - 例：
    - 1 → 「前月」を対象として集計。

---

#### 1-2-5. `cron.kpi.notify_on_error`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - KPI 集計バッチが失敗した場合に、ChatWork／メールなどで通知するかどうか。

---

### 1-3. 請求・ロイヤリティ特例関連

#### 1-3-1. `cron.invoices.auto_enable`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - 毎月1日に `cron_invoices.php` による請求ドラフト生成を自動実行するかどうか。
  - v1.4 では請求ドラフト生成は基本自動運用とする前提のため、デフォルトは `true`。

---

#### 1-3-2. `cron.invoices.default_billing_day`

- 型: `int`
- デフォルト値: `1`
- スコープ: システム全体
- 説明:
  - 請求ドラフト生成ジョブ（cron_invoices.php）を標準で実行する日。
  - v1.4 では前月分を翌月1日に集計する前提。

---

#### 1-3-3. `cron.invoices.notify_on_error`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - 請求ドラフト生成処理でエラーが発生した場合に、ChatWork／メール等で通知するかどうか。

---

#### 1-3-4. `cron.royalty_relief_check.auto_enable`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - 月次のロイヤリティ特例ルール評価ジョブ（`cron_royalty_relief_check.php`）を自動実行するかどうか。
  - `false` にした場合、ロイヤリティ特例に関するアラート（チケット／sys_events）は発生しない。

---

#### 1-3-5. `cron.royalty_relief_check.notify_on_error`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - 特例ルール評価処理自体が失敗した場合に通知するかどうか。

---

### 1-4. SMS追撃／ミーティングリマインド関連

#### 1-4-1. `cron.sms_followup.enable`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - `cron_sms_followup.php`（未返信チケット追撃ジョブ）を有効にするかどうか。
  - システム全体の SLA 担保に直結するため、基本は `true` を推奨。

---

#### 1-4-2. `cron.sms_followup.max_tickets_per_run`

- 型: `int`
- デフォルト値例: `50`
- スコープ: システム全体
- 説明:
  - 1回の実行で追撃通知を行うチケットの上限件数。
  - 大量発生時のスパイクを抑えるための安全弁として利用する。

---

#### 1-4-3. `cron.sms_followup.notify_on_error`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - SMS送信処理で致命的エラーが発生した場合に、ChatWork／メール等で通知するか。

---

#### 1-4-4. `cron.meeting_reminders.enable`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - `cron_meeting_reminders.php`（HQミーティング前日／1時間前リマインドジョブ）を有効にするかどうか。

---

#### 1-4-5. `cron.meeting_reminders.prev_day_hour`

- 型: `int`
- デフォルト値: `9`
- スコープ: システム全体
- 説明:
  - 前日リマインドを送る「時刻（時）」を指定する。
  - 例：9 → 前日 9:00 台に翌日分 MTG のリマインドを送信。

---

#### 1-4-6. `cron.meeting_reminders.use_email`

- 型: `bool`
- デフォルト値: `false`
- スコープ: システム全体
- 説明:
  - ChatWork通知に加えて、メールによるリマインドも送信するかどうか。
  - v1.4 初期は ChatWork 中心の運用を想定し、デフォルトは `false`。

---

### 1-5. バックアップ／クリーンアップ関連

#### 1-5-1. `cron.backup_db.enable`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - `cron_backup_db.php`（DB＋請求PDFバックアップジョブ）を有効にするかどうか。
  - 基本的には常に `true` を維持する前提。

---

#### 1-5-2. `cron.backup_db.keep_generations`

- 型: `int`
- デフォルト値: `7`
- スコープ: システム全体
- 説明:
  - アプリ層バックアップとして保持する世代数。
  - 例：7 → 7日分（7世代）保持し、それより古いバックアップは削除。

---

#### 1-5-3. `cron.backup_db.notify_on_error`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - バックアップ取得や古いバックアップ削除に失敗した場合に通知するかどうか。

---

#### 1-5-4. `cron.cleanup_logs.enable`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - `cron_cleanup_logs.php`（ログ・KPI・エラーデータ削除ジョブ）を有効にするかどうか。
  - retention パラメータに基づく自動削除／アーカイブを実行する。

---

#### 1-5-5. `cron.cleanup_logs.max_rows_per_table`

- 型: `int`
- デフォルト値例: `10000`
- スコープ: システム全体
- 説明:
  - 1回の実行で、1テーブルあたり削除／アーカイブ対象とする最大行数。
  - 負荷を抑えつつ徐々に削除するためのバッチサイズ。

---

#### 1-5-6. `cron.cleanup_logs.notify_on_error`

- 型: `bool`
- デフォルト値: `true`
- スコープ: システム全体
- 説明:
  - クリーンアップ処理で致命的エラーが発生した場合に通知するかどうか。

---

## 2. 実装メモ（sys_params へのマッピング方針例）

- `sys_params` テーブルを利用する場合、キーは `cron.<something>` のように prefix を付ける想定：

| param_key                        | sys_params.key                          | 例value      |
| -------------------------------- | --------------------------------------- | ------------ |
| sales_import_auto_enable         | `cron.sales_import.auto_enable`         | `false`      |
| sales_import_max_jobs_per_run    | `cron.sales_import.max_jobs_per_run`    | `1`          |
| sales_import_notify_on_warning   | `cron.sales_import.notify_on_warning`   | `true`       |
| sales_import_error_severity_threshold | `cron.sales_import.error_severity_threshold` | `100` |
| kpi_daily_enable                 | `cron.kpi.daily_enable`                 | `false`      |
| kpi_monthly_enable               | `cron.kpi.monthly_enable`               | `true`       |
| kpi_daily_target_offset_days     | `cron.kpi.daily_target_offset_days`     | `1`          |
| kpi_monthly_target_offset_months | `cron.kpi.monthly_target_offset_months` | `1`          |
| kpi_notify_on_error              | `cron.kpi.notify_on_error`              | `true`       |
| invoices_auto_enable             | `cron.invoices.auto_enable`             | `true`       |
| invoices_default_billing_day     | `cron.invoices.default_billing_day`     | `1`          |
| invoices_notify_on_error         | `cron.invoices.notify_on_error`         | `true`       |
| royalty_relief_auto_enable       | `cron.royalty_relief_check.auto_enable` | `true`       |
| royalty_relief_notify_on_error   | `cron.royalty_relief_check.notify_on_error` | `true`   |
| sms_followup_enable              | `cron.sms_followup.enable`              | `true`       |
| sms_followup_max_tickets_per_run | `cron.sms_followup.max_tickets_per_run` | `50`         |
| sms_followup_notify_on_error     | `cron.sms_followup.notify_on_error`     | `true`       |
| meeting_reminders_enable         | `cron.meeting_reminders.enable`         | `true`       |
| meeting_reminders_prev_day_hour  | `cron.meeting_reminders.prev_day_hour`  | `9`          |
| meeting_reminders_use_email      | `cron.meeting_reminders.use_email`      | `false`      |
| backup_db_enable                 | `cron.backup_db.enable`                 | `true`       |
| backup_db_keep_generations       | `cron.backup_db.keep_generations`       | `7`          |
| backup_db_notify_on_error        | `cron.backup_db.notify_on_error`        | `true`       |
| cleanup_logs_enable              | `cron.cleanup_logs.enable`              | `true`       |
| cleanup_logs_max_rows_per_table  | `cron.cleanup_logs.max_rows_per_table`  | `10000`      |
| cleanup_logs_notify_on_error     | `cron.cleanup_logs.notify_on_error`     | `true`       |

---

## 3. 決定事項 / 未決事項 / 論点

### 3-1. 決定事項

- cron ジョブの
  - 実行有無（enable / auto_enable）
  - 対象件数の上限
  - エラー時の通知有無
  は、ここで定義した `cron.*` パラメータを通じて制御する。
- ジョブの実行時刻そのものは v1.4 では OS cron で管理し、  
  「いつ・どの期間を処理するか」は
  - OS cron（起動タイミング）
  - cron パラメータ（オフセット）
  の組み合わせで決める。
- ロイヤリティ特例ルールに関しては、
  - `cron.royalty_relief_check.*` 系パラメータにより
    - 「条件チェックの自動実行」
    - 「チェック処理のエラー通知」
  を制御し、
  - 実際のロイヤリティ変更は AI-INBOX 側の UI 操作で行う。

### 3-2. 未決事項

- `cron.sales_import.error_severity_threshold` の最適値
  - 実運用でのエラー分布を見ながら調整が必要。
- `cron.cleanup_logs.max_rows_per_table` の標準値
  - DB サイズやインデックス構造により適切値が変わるため、運用初期に様子見が必要。
- KPI 日次集計の運用有無
  - `cron.kpi.daily_enable` と `kpi_allow_daily`（KPI_ANALYTICS 側）の組み合わせ運用のベストプラクティス。

### 3-3. 論点

- 将来的に「ジョブスケジュールを DB パラメータ化する」場合、
  - `cron.*` パラメータと OS cron の役割分担をどう整理するか。
- 重大なジョブ失敗（例：sales_import／backup_db）に対して、
  - 何回連続で失敗したら SMS まで上げるか、といった「連続失敗しきい値」を  
    追加パラメータとして持つかどうか。
