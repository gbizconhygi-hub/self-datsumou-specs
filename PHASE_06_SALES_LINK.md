# PHASE_06_SALES_LINK — STORES売上CSV連携仕様 v1.4

## 0. このドキュメントの位置づけ

- 本ドキュメントは、「加盟店サポート自動化プロジェクト v1.4」の**Phase6：データ連携／KPI層** における要素 **SALES_LINK** の仕様を定義する。
- 目的は、STORES管理画面からダウンロードした売上CSV（全店舗分）を**`sales_records` に安全かつ再現性高く取り込み、後続の KPI_ANALYTICS に渡すこと**。
- Phase0 で定義済みの `sales_records` スキーマを前提とし、
「**取り込みフロー／ジョブ管理／エラーハンドリング／重複防止ポリシー**」をここで確定する。

> ⚠️ 注意
> 
> - 本フェーズで定義する新規テーブル・ALTER DDL は、実際の適用は Phase9 PACKAGE_BUILD で行う。
> - 本仕様内で「存在する」と記載しても、v1.4 開発途中のDBにはまだ無いことを前提とする。

---

## 1. 目的・スコープ

### 1-1. 目的（What）

1. STORES管理画面から本部が一括ダウンロードした **全店舗分の売上CSV 3種**
    - 月謝・回数券CSV
    - 都度払いカード売上CSV
    - 予約履歴CSV（現地決済）
    を、**統一フォーマット `sales_records` に変換して蓄積する。**
2. 取込処理を
    - **ジョブ単位で管理**（「何年何月分を、いつ、誰が取込したか」）し、
    - **行数・金額整合性と、エラー／重複を可視化** することで、
    - KPI_ANALYTICS や請求処理の「数字の信用度」を担保する。
3. 将来的な完全自動化（STORES API 経由での自動取込）に向けて、
    - `cron_sales_import.php` の I/O 契約（引数・ジョブ状態遷移）を明示しておく。

### 1-2. 運用前提（When / Who）

- **誰が**：本部HQ担当（ai. ログインユーザ）
- **いつ**：原則として毎月25日前後（任意の運用日だがデフォルト想定）
- **何を**：
    - 「月謝・回数券CSV（全店舗分）」
    - 「都度払いカード売上CSV（全店舗分）」
    - 「予約履歴CSV（現地決済／全店舗分）」
- **どうする**：ai. ポータルの「売上CSV取込」画面からアップロードし、`sales_records` に書き込む取込ジョブを1件作成する。

> ※1回のアップロード＝「ある期間（通常はひと月）について、全店舗分をまとめて取り込むジョブ」という扱い。
> 

### 1-3. スコープ（Scope）

**本フェーズ（SALES_LINK）がカバーするもの**

- ai.（本部ポータル）上の **売上CSV取込画面の仕様**（UIは概要レベル）
- 売上CSV3種 → `sales_records` の **マッピングルール**
- **取込ジョブ管理テーブル**・**エラーログテーブル** の設計
- **重複取込防止**・**エラー時のロールバック／リトライポリシー**
- `cron_sales_import.php` の I/O 契約（CLI引数・挙動の概要）

**本フェーズで明示的に対象外とするもの**

- STORES API を用いた売上データのフル自動取得
    - → Phase9 以降の拡張（APIベース SALES_LINK 拡張）で扱う。
- KPIの定義・ダッシュボード表示
    - → 同じ Phase6 の別要素 **KPI_ANALYTICS** で定義。
- ロイヤリティ計算／請求PDF生成
    - → 既存 `cron_invoices.php` / 請求モジュールに委譲。

---

## 2. 前提・依存関係（Phase0〜5 との関係）

### 2-1. 利用テーブル（既存）

- `sales_records`
    - 目的：
        
        > STORES予約から取り込んだ売上データの統合テーブル。
        > 
        > 
        > 月謝・都度カード・現地決済を1フォーマットに統一。
        > 
    - **累積テーブル**であり、過去分も含めて継続保存する前提。
    - 主なカラム（確定済み）：
        - `id` bigint unsigned, PK
        - `merchant_public_id` varchar(64) NOT NULL
        - `shop_id` bigint unsigned NOT NULL（= `shops.id`。merchant_public_idから解決）
        - `source_type` varchar(32) NOT NULL
            - `monthly_ticket`（回数券・月謝）
            - `spot_card`（都度カード）
            - `spot_local`（現地決済）
        - `transaction_type` varchar(64) NULL
        - `transaction_datetime` datetime NOT NULL
        - `amount` int unsigned NOT NULL
        - `fee` int unsigned NULL
        - `revenue` int unsigned NOT NULL
        - `product_name` varchar(255) NOT NULL
        - `customer_name` / `customer_email` / `customer_phone`
        - `is_trial` tinyint(1) NOT NULL default 0
        - `payment_method` varchar(32) NULL
        - `status` varchar(32) NULL
        - `created_at` / `updated_at` datetime
- `shops`
    - `merchant_public_id` と `id` を紐付ける店舗マスタ。
- `owners` / `owner_contacts`
    - オーナー・HQ担当のマスタ。
    - 本仕様では、CSVアップロード実行者を `owner_contacts.id` で持つ前提（HQ担当もここに含める）。

### 2-2. cron・外部連携との関係

- `cron_sales_import.php`
    - Phase7 CRON_SYS で実装される予定の cron スクリプト。
- STORES管理画面
    - CSV ダウンロードUIや列名は STORES 側仕様に依存する。
    - 本仕様では「実際に運用する列名」を前提にしたマッピングを定義し、
    変更があった場合は設定だけで追随できるようにする。

---

## 3. データモデル（SALES_LINK のためのDB設計）

### 3-1. 利用テーブル一覧（SALES_LINK 視点）

- 既存テーブル
    - `sales_records` … 売上実データ（Fact）
    - `shops` … 店舗ディメンション
    - `owners` / `owner_contacts` … オーナー／本部担当ディメンション
- 本フェーズで追加するテーブル（提案）
    - `sales_import_jobs` … 売上CSV取込ジョブの管理
    - `sales_import_errors` … 行単位のエラーログ

---

### 3-2. 新規テーブル：`sales_import_jobs`

**目的**

- STORES売上CSVのアップロード・取込を「1ジョブ」として管理する。
- 「何年何月の全店舗分の帳簿を、いつ、誰が取り込んだか」を追跡可能にする。
- 行数・金額サマリ、エラー有無、対象期間を可視化し、**「この月次データはどの程度信頼できるか」を判断しやすくする。**

**DDL（提案）**

```sql
CREATE TABLE sales_import_jobs (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'DB主キー',
  scope ENUM('all_shops','single_shop') NOT NULL DEFAULT 'all_shops'
    COMMENT '取込範囲: 全店舗分 or 特定店舗分（v1.4の運用はall_shops前提）',
  merchant_public_id VARCHAR(64) NULL COMMENT 'single_shop時のみ対象STORES merchant_public_id。all_shops時はNULL',
  shop_id BIGINT UNSIGNED NULL COMMENT 'single_shop時のみ対象店舗ID（shops.id）。all_shops時はNULL',
  import_period_start DATE NULL COMMENT '取込対象期間（開始日）。通常は月初',
  import_period_end DATE NULL COMMENT '取込対象期間（終了日）。通常は月末',
  uploaded_by_owner_contact_id BIGINT UNSIGNED NULL COMMENT 'アップロード実行者（owner_contacts.id。HQ担当を含む）',
  monthly_label CHAR(7) NULL COMMENT '対象年月（YYYY-MM）。UIでの主キー的なラベル',
  monthly_sequence INT UNSIGNED NOT NULL DEFAULT 1 COMMENT '同じ年月で複数回取込した場合の連番（1=1回目）',
  total_files INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '受理したCSVファイル数（通常3）',
  total_rows INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '全CSVの合計行数（ヘッダ除く）',
  imported_rows INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '正常に取り込んだ行数',
  skipped_rows INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '重複・不整合などでスキップした行数',
  error_rows INT UNSIGNED NOT NULL DEFAULT 0 COMMENT 'エラーにより取り込めなかった行数',
  csv_total_amount BIGINT NULL COMMENT '元CSVの金額合計（amountの合計）',
  db_total_amount BIGINT NULL COMMENT '今回ジョブでINSERTしたsales_records.amount合計',
  status ENUM('uploaded','validating','importing','completed','completed_with_warning','failed')
    NOT NULL DEFAULT 'uploaded'
    COMMENT 'ジョブ状態',
  error_summary TEXT NULL COMMENT 'エラー概要（人間が読む用のサマリ）',
  started_at DATETIME NULL COMMENT '取込開始時刻',
  completed_at DATETIME NULL COMMENT '取込完了時刻',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'ジョブ登録日時（アップロード時）',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新日時',
  PRIMARY KEY (id),
  KEY idx_scope (scope),
  KEY idx_merchant_public_id (merchant_public_id),
  KEY idx_shop_id (shop_id),
  KEY idx_status (status),
  KEY idx_monthly (monthly_label, monthly_sequence),
  KEY idx_period (import_period_start, import_period_end)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='STORES売上CSV取込ジョブ管理テーブル';

```

> v1.4 の標準運用では scope = 'all_shops'、merchant_public_id / shop_id は NULL を想定。
> 
> 
> `single_shop` は将来のトラブルシュート用の拡張余地として残しておく。
> 

---

### 3-3. 新規テーブル：`sales_import_errors`

**目的**

- 1ジョブにおける行単位のエラーを記録し、
    
    「どの行が、なぜ取り込めなかったか」を後から確認できるようにする。
    

**DDL（提案）**

```sql
CREATE TABLE sales_import_errors (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT 'DB主キー',
  job_id BIGINT UNSIGNED NOT NULL COMMENT '関連ジョブID（sales_import_jobs.id）',
  source_type VARCHAR(32) NOT NULL COMMENT 'CSV種別: monthly_ticket / spot_card / spot_local など',
  csv_filename VARCHAR(255) NULL COMMENT '元CSVファイル名',
  row_number INT UNSIGNED NOT NULL COMMENT 'CSV行番号（1始まり。ヘッダ行を除いた実データ行）',
  error_code VARCHAR(64) NOT NULL COMMENT 'エラー種別コード（必須列欠落, 型不正, 日付パース失敗など）',
  error_message TEXT NOT NULL COMMENT 'エラーメッセージ（人間が読む説明文）',
  raw_row_text TEXT NULL COMMENT '元CSV行（プライバシー配慮のため保存期間は短め）',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '記録日時',
  PRIMARY KEY (id),
  KEY idx_job_id (job_id),
  KEY idx_error_code (error_code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='STORES売上CSV取込エラーログ';

```

---

## 4. 入出力仕様（I/O契約）

### 4-1. 人が操作するUI（ai. 本部ポータル）

**画面名（案）**

- 「売上CSV取込（SALES_LINK）」
    
    URL例：`/sales/import`（ai.self-datsumou.net 配下）
    

**主な入力項目**

- 対象年月（必須）
    - `YYYY-MM` 形式（例：`2025-03`）。
    - 選択された年月から `import_period_start` / `import_period_end` を自動算出（当月1日〜末日）。
- アップロードファイル（3つ想定）
    - 「月謝・回数券CSV（全店舗分）」
    - 「都度払いカード売上CSV（全店舗分）」
    - 「予約履歴CSV（現地決済／全店舗分）」
    - この3ファイルを**それぞれ単一CSVとして指定**する形を基本とし、
        
        代替案として「3ファイルをZIPでまとめて1つ指定」も許容（実装判断）。
        
- オプション
    - 「ドライラン（検証だけでDB書き込みなし）」チェックボックス
    - 「既存データの処理方針」
        - `skip_if_exists`（推奨／デフォルト）
        - `overwrite_same_key`（将来用。初期は無効）

**出力／フィードバック**

- ジョブ作成後のステータス表示：
    - ジョブID
    - 対象年月
    - ステータス（`uploaded` → `validating` → `importing` → `completed` 等）
- 完了後（またはドライラン完了）：
    - 行数サマリ：`total_rows / imported_rows / skipped_rows / error_rows`
    - 金額サマリ：`csv_total_amount / db_total_amount / 差分・差分率`
    - エラー件数と、`sales_import_errors` 一覧へのリンク
    - 「completed_with_warning」の場合は、警告理由を明記（例：金額差分○％超、エラー行○件など）。

---

### 4-2. CSV → `sales_records` マッピングルール（確定版）

> ここは、以前検討済みの「売上集計の考え方」を
> 
> 
> Phase0 の `sales_records` に合わせて確定仕様として書き下ろしたセクション。
> 

### 共通の変換ルール（全CSV共通）

- `merchant_public_id`
    - CSV列 `merchant-public-id` をそのまま使用。
- `shop_id`
    - `shops.merchant_public_id = :merchant_public_id` で解決。
    - 一致する店舗が見つからない場合、その行はエラー行として `sales_import_errors` に記録し、スキップ。
- `transaction_datetime`
    - CSVの日時列を `datetime` に変換。
    - 月謝・回数券／都度カード：**「取引日時」列**
    - 現地決済：**「希望の予約日時」列**
    - 変換できない場合はエラー行。
- `amount`
    - CSVの「販売金額」列を整数（税込）に変換。カンマや全角数字は正規化。
- `fee`
    - CSVに手数料列がある場合のみセット（カード売上）。現地決済は NULL または 0。
- `revenue`
    - カード売上：`amount - fee`
    - 現地決済：`amount`
- `product_name`
    - CSVの「商品名／タイトル／メニュー」相当列を、共通してここに格納。
- `customer_name` / `customer_email` / `customer_phone`
    - 各CSVの顧客情報列をそのまま格納（存在しない場合は NULL）。
- `is_trial`
    - `product_name` に「お試し体験」等、指定キーワードを含む場合に `1`、それ以外は `0`。
    - キーワード一覧はパラメータ化（後述）。
- `payment_method`
    - 現地決済CSVの支払方法列から設定（例：`cash`, `card`, `qr` 等）。
    - カード売上CSVでは `card` 固定、または NULL（要運用判断）。
- `status`
    - 現地決済CSVのステータス列を格納
    - 取り込み対象は `"確定"` の行のみとし、それ以外はスキップ。

### CSV種別ごとのマッピング

| CSVファイル種別 | source_type | 主な対応列 → `sales_records` |
| --- | --- | --- |
| 月謝・回数券CSV（全店舗分） | `monthly_ticket` | merchant-public-id → `merchant_public_id`取引種別 → `transaction_type`取引日時 → `transaction_datetime`販売金額 → `amount`手数料 → `fee`売上（入金額） → `revenue`商品名 → `product_name`顧客氏名 → `customer_name`メール → `customer_email`電話 → `customer_phone` |
| 都度払いカード売上CSV | `spot_card` | merchant-public-id → `merchant_public_id`取引種別 → `transaction_type`取引日時 → `transaction_datetime`販売金額 → `amount`手数料 → `fee`売上（入金額） → `revenue`タイトル → `product_name`顧客氏名／メール／電話 → 各カラム |
| 予約履歴CSV（現地決済／全店舗） | `spot_local` | merchant-public-id → `merchant_public_id`ステータス → `status`（"確定"のみ取込）希望の予約日時 → `transaction_datetime`金額 → `amount`（feeはNULL）メニュー → `product_name`顧客氏名／メール／電話 → 各カラム支払方法 → `payment_method` |

**`is_trial` の判定（共通）**

- 月謝・回数券CSV：`product_name` に `"お試し体験【"` を含む行 → `is_trial = 1`
- 都度払いカード売上CSV：`product_name` に `"お試し体験"` を含む行 → `is_trial = 1`
- 予約履歴CSV（現地決済）：`product_name` に `"お試し体験"` を含む行 → `is_trial = 1`

※キーワードは `SALES_LINK_DEFAULT_PARAMS.trial_keyword_list` で調整可能にする。

---

### 4-3. 重複取込防止ポリシー（Idempotency）

**基本方針**

- 同じ期間・同じCSVを再度アップロードしても、**売上が二重計上されない**こと。
- `sales_records` 内で「既に登録済み」と判定できる行はスキップし、`skipped_rows` に計上する。

**重複判定キー（提案）**

以下のカラム組み合わせを **重複候補キー** とする：

- `merchant_public_id`
- `source_type`
- `transaction_datetime`
- `amount`
- `product_name`
- `customer_name`（NULL は比較対象外）

**重複判定ロジック**

- インポート対象行について、上記キーで `sales_records` を検索。
- 1件以上ヒットした場合：
    - その行は「重複候補」とみなし、INSERT は行わずスキップ。
    - `skipped_rows` を +1。
    - 必要に応じて `sales_import_errors` に `error_code = 'duplicate_candidate'` として記録（警告扱い）。

> ※ 完全に同一条件の別取引があり得る（例：同姓同名で同メニューを同額で同時刻）ことは否定できないが、
> 
> 
> 現実的には極めて稀と判断し、「二重計上リスクを防ぐメリット」を優先する設計。
> 
> 運用で問題が出た場合、キーの見直しを行う。
> 

**DBレベルの制約**

- 上記キーに対する UNIQUE 制約は張らない（誤検知によるINSERT拒否を避ける）。
- 重複判定はアプリケーションレベルで行う。

---

### 4-4. `cron_sales_import.php` の I/O 契約（概要）

**想定呼び出し**

- 手動実行（HQ／開発者がSSHなどから）
    - `php cron_sales_import.php --job-id=123`
    - 指定ジョブIDのみを処理。
- 定期実行（CRON）
    - `php cron_sales_import.php --auto=1`
    - `status IN ('uploaded','validating')` のジョブを、古い順に処理。

**挙動（ジョブ状態遷移）**

1. `uploaded`
    - CSVアップロード直後。まだ検証・取込は未実施。
2. `validating`
    - CSVヘッダ・必須列・日付・数値などの検証中。
3. `importing`
    - `sales_records` への INSERT を実行中。
4. `completed`
    - 全行が正常に取り込まれ、`error_rows = 0` かつ 金額差分が閾値未満。
5. `completed_with_warning`
    - 取込自体は完了したが、`error_rows > 0` または 金額差分が閾値以上。
6. `failed`
    - システムエラー等により途中で中断した場合。

**CLI戻り値**

- `0` … 正常終了（`completed` / `completed_with_warning`）
- `1` … 入力エラー（job-id 不正など）
- `2` … 二重起動など実行条件不備
- `3` … 予期せぬ例外

---

## 5. 検証・突合（Validation & Reconciliation）

### 5-1. 行数チェック

- 3つのCSVそれぞれについて、
    - データ行数（ヘッダ除く）を数え、`total_rows` に合計値をセット。
- 取込完了後に
    - `imported_rows + skipped_rows + error_rows = total_rows`
        
        が成り立つことを保証。成り立たない場合は `completed_with_warning` とする。
        

### 5-2. 金額チェック

- CSV側合計金額：`csv_total_amount`
    - 3つのCSVの金額列（`amount`に対応する列）の合計。
- DB側合計金額：`db_total_amount`
    - 今回ジョブで新規挿入された `sales_records.amount` 合計。
- 差分率
    - `diff = csv_total_amount - db_total_amount`
    - `diff_rate = |diff| / csv_total_amount * 100`
- `diff_rate` が `SALES_LINK_DEFAULT_PARAMS.reconcile_warn_percent` を超える場合、
    - `status = 'completed_with_warning'` とする。
    - `error_summary` に差分の詳細を記録。

---

## 6. エラー処理・リトライ・ロールバック

### 6-1. 行単位エラーの扱い

- 必須列欠落・日付変換不可・金額変換不可などは**その行だけ**を
    
    `sales_import_errors` に記録し、その行を `error_rows` に計上。
    
- 残りの行は可能な限り取り込む（ベストエフォート）。

### 6-2. システムエラー時のロールバック

- INSERT は一定バッチサイズ（例：1,000行）単位でトランザクションを分割。
- 実行途中で致命的エラーが発生した場合：
    - コミット済みバッチはそのまま残る。
    - ジョブは `status = 'failed'` とし、`imported_rows` には実際の件数を保持。
    - 同じCSVを再度アップロードすることでリトライ可能（重複行はスキップ）。

---

## 7. 🔧 パラメータ候補 — `SALES_LINK_DEFAULT_PARAMS`

> 実体は Phase6 または Phase7 以降で
> 
> 
> `sys_params` 相当の設定テーブル、もしくは
> 
> `SALES_LINK_DEFAULT_PARAMS.md` に定義する。
> 
- `allowed_source_types`
    
    例：`['monthly_ticket', 'spot_card', 'spot_local']`
    
- `trial_keyword_list`
    
    例：`['お試し体験', '体験コース']`
    
- `csv_header_mappings`
    - CSV列名 → 論理項目（`transaction_datetime`, `amount`, `product_name` 等）へのマッピング定義
    - STORES側の列名変更に追随するために必須。
- `duplicate_detection_keys`
    
    例：`['merchant_public_id','source_type','transaction_datetime','amount','product_name','customer_name']`
    
- `max_rows_per_job`
    
    1ジョブで扱う最大行数（性能・タイムアウト対策）。
    
- `reconcile_warn_percent`
    
    金額差分を「警告」とする割合（例：`1.0`）。
    
- `error_row_retention_days`
    
    `sales_import_errors.raw_row_text` の保持日数（例：30日）。
    
- `default_import_mode`
    
    `'dry_run'` or `'immediate'`（初期は `dry_run` 推奨）。
    
- `auto_import_enabled`
    
    将来の STORES API 取込を見据えたフラグ（v1.4 では常に `0`）。
    
- `default_monthly_import_day`
    
    月次取込の想定日（デフォルト値として 25 を設定するなど。UIのヘルプ表示に利用）。
    

---

## 8. セキュリティ・個人情報取り扱い

- `sales_records` には顧客氏名・メール・電話などが含まれるため、
    - DB接続権限管理
    - バックアップファイルの保護
    - 分析用途における匿名化（KPI_ANALYTICS側で対応）
        
        を前提とする。
        
- `sales_import_errors.raw_row_text` はデバッグ用途の一時的保存とし、
    - `error_row_retention_days` 経過後は NULL に更新する。
- 元CSVファイル（アップロードされた生ファイル）も、
    - `uploaded/` ディレクトリ等に保存し、
    - 一定期間（例：30日）後に自動削除する想定（Phase7 CRON_SYS で実装）。

---

## 9. 決定事項 / 未決事項 / 論点整理

### 9-1. 決定事項（Phase6 SALES_LINK）

- 売上取込は「本部が全店舗分のCSV3種を、原則25日前後に月次でアップロード」する運用を前提とする。
- 3種CSVは `source_type` で区別し、`sales_records` に統合する。
- `sales_records` のスキーマは Phase0 定義どおりで、Phase6ではALTERしない。
- `sales_import_jobs` / `sales_import_errors` を追加し、取込ジョブとエラーを管理する。
- 重複判定は
    
    `merchant_public_id, source_type, transaction_datetime, amount, product_name, customer_name`
    
    をキーとしてアプリケーションレベルで行う。
    
- 金額差分が `reconcile_warn_percent` を超えた場合、ジョブは `completed_with_warning` となる。

### 9-2. 未決事項（要仕様判断）

- HQ担当ユーザのIDをすべて `owner_contacts.id` で表現するか、
    
    別途HQユーザテーブルを設けるか。
    
- 現地決済CSVの支払方法を `payment_method` にどこまで細かく反映するか
    
    （例：`cash`, `qr`, `id` などの粒度）。
    
- エラー率が極端に高いジョブ（例：80%以上）が発生した場合、
    - 常に `completed_with_warning` に留めるか
    - 自動的に `failed` 扱いとするか。

### 9-3. 論点（運用・将来拡張）

- 月次以外（四半期・年次・任意期間）での再取込運用をどう位置付けるか。
    
    → 特殊運用として `monthly_sequence` をインクリメントして対応するか。
    
- STORES側で過去売上の修正・返金が行われた場合、
    - 返金行を別行として取り込む前提でよいか（差分更新は行わないか）。
- STORES API による自動取込に切り替える際、
    - `sales_import_jobs` を継続利用するか、
    - 別のジョブ管理を用意するか（互換性をどう保つか）。
