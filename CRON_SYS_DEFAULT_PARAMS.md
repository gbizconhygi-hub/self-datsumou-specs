# CRON_SYS_DEFAULT_PARAMS — 自動処理層パラメータ定義 v1.4（ドラフト）

## 0. このドキュメントの位置づけ

- 本ドキュメントは Phase7 要素 **CRON_SYS** で利用する
「各cronジョブのON/OFF・挙動調整」に関するパラメータ群を整理したもの。
- 実体は
    - `sys_params` 系テーブル
    - または設定ファイル
    とし、本書は論理キー名と意味・デフォルト値を示す。
- ここで定義するキーは、
    - `PHASE_07_CRON_SYS.md` の「🔧 パラメータ候補」で挙げた内容をベースとした **コアセット** とする。

---

## 1. パラメータ一覧（案）

### 1-1. SALES_IMPORT 関連

- `cron.sales_import.auto_enable`
    - 型: bool
    - デフォルト値: false
    - 説明: `-auto` モードでの売上取込を定期実行するか。
- `cron.sales_import.max_jobs_per_run`
    - 型: int
    - デフォルト値: 1
    - 説明: 1回の実行で処理する `sales_import_jobs` の上限件数。
- `cron.sales_import.notify_on_warning`
    - 型: bool
    - デフォルト値: true
    - 説明: `completed_with_warning` となった場合に ChatWork 通知を行うか。
- `cron.sales_import.error_severity_threshold`
    - 型: int
    - デフォルト値例: 100
    - 説明: エラー行数がこの値を超えた場合に、致命的エラーとして扱うしきい値。

---

### 1-2. KPI 関連

- `cron.kpi.daily_enable`
    - 型: bool
    - デフォルト値: true
    - 説明: KPI 日次集計を有効にするか。
- `cron.kpi.monthly_enable`
    - 型: bool
    - デフォルト値: true
    - 説明: KPI 月次集計を有効にするか。
- `cron.kpi.daily_target_offset_days`
    - 型: int
    - デフォルト値: 1
    - 説明: 日次集計の対象日を「今日から何日前」として扱うか。
- `cron.kpi.monthly_target_offset_months`
    - 型: int
    - デフォルト値: 1
    - 説明: 月次集計の対象月を「今月から何ヶ月前」として扱うか。
- `cron.kpi.notify_on_error`
    - 型: bool
    - デフォルト値: true
    - 説明: KPI集計の失敗時に通知するか。

---

### 1-3. 請求／ロイヤリティ関連

- `cron.invoices.auto_enable`
    - 型: bool
    - デフォルト値: true
    - 説明: 毎月1日に請求ドラフトを自動生成するか。
- `cron.invoices.default_billing_day`
    - 型: int
    - デフォルト値: 1
    - 説明: 標準の請求締め日（現在は毎月1日想定）。
- `cron.invoices.notify_on_error`
    - 型: bool
    - デフォルト値: true
    - 説明: 請求ドラフト生成に失敗した場合に通知するか。
- `cron.royalty_relief_check.auto_enable`
    - 型: bool
    - デフォルト値: true
    - 説明: ロイヤリティ特例ルールのチェックを月次で実行するか。
- `cron.royalty_relief_check.notify_on_error`
    - 型: bool
    - デフォルト値: true
    - 説明: チェック処理自体の失敗時に通知するか。

---

### 1-4. SMS追撃／リマインド関連

- `cron.sms_followup.enable`
    - 型: bool
    - デフォルト値: true
    - 説明: 未返信チケットへの追撃通知ジョブを有効にするか。
- `cron.sms_followup.max_tickets_per_run`
    - 型: int
    - デフォルト値例: 50
    - 説明: 1回の実行で追撃対象とするチケット上限。
- `cron.sms_followup.notify_on_error`
    - 型: bool
    - デフォルト値: true
    - 説明: 送信処理で致命的エラーが発生した場合に通知するか。
- `cron.meeting_reminders.enable`
    - 型: bool
    - デフォルト値: true
    - 説明: HQミーティングリマインドジョブを有効にするか。
- `cron.meeting_reminders.prev_day_hour`
    - 型: int
    - デフォルト値: 9
    - 説明: 前日リマインドを送る時刻（時単位）。
- `cron.meeting_reminders.use_email`
    - 型: bool
    - デフォルト値: false
    - 説明: ChatWorkに加えてメールも送るか。

---

### 1-5. バックアップ／クリーンアップ関連

- `cron.backup_db.enable`
    - 型: bool
    - デフォルト値: true
    - 説明: DB＋請求PDFバックアップジョブを有効にするか。
- `cron.backup_db.keep_generations`
    - 型: int
    - デフォルト値: 7
    - 説明: 保持するバックアップ世代数。
- `cron.backup_db.notify_on_error`
    - 型: bool
    - デフォルト値: true
    - 説明: バックアップ失敗時に通知するか。
- `cron.cleanup_logs.enable`
    - 型: bool
    - デフォルト値: true
    - 説明: ログ・KPI・エラーデータ削除ジョブを有効にするか。
- `cron.cleanup_logs.max_rows_per_table`
    - 型: int
    - デフォルト値例: 10000
    - 説明: 1回の実行で削除する1テーブルあたりの最大行数。
- `cron.cleanup_logs.notify_on_error`
    - 型: bool
    - デフォルト値: true
    - 説明: クリーンアップに失敗した場合に通知するか。
