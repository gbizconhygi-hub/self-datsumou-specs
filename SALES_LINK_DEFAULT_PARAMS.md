# SALES_LINK_DEFAULT_PARAMS — SALES_LINK パラメータ定義 v1.4

## 0. このドキュメントの位置づけ

- 本ドキュメントは Phase6 要素 **SALES_LINK**（STORES売上CSV取込）で利用する
    
    「挙動を調整するためのパラメータ群」を整理したもの。
    
- 実体は
    - `sys_params` 系テーブル
    - 環境別設定ファイル（.env / config.php 等）
    のいずれかで保持する想定であり、本書は「論理名／意味／デフォルト値」の仕様を示す。
- ここで定義するキーは、
    - `PHASE_06_SALES_LINK.md` の「🔧 パラメータ候補」で挙げた内容をベースとしつつ、
    - v1.4 で運用に必要と判断したものだけを **コアパラメータ** として明示する。

---

## 1. パラメータ一覧（コア）

### 1-1. 基本構成・種別関連

### 1-1-1. `allowed_source_types`

- 型: `string[]`
- デフォルト値: `['monthly_ticket', 'spot_card', 'spot_local']`
- スコープ: システム全体（全店舗共通）
- 説明:
    - SALES_LINK で受け付ける `source_type` の一覧。
    - CSV種別 → `source_type` のマッピングは固定だが、将来新種別が増えたときにここへ追加する。

---

### 1-1-2. `trial_keyword_list`

- 型: `string[]`
- デフォルト値例: `['お試し体験', '体験コース']`
- スコープ: システム全体（全店舗共通）
- 説明:
    - `product_name` に含まれていれば `is_trial = 1` と判定するキーワード一覧。
    - 月謝・回数券／都度カード／現地決済のすべてで共通ルールとして扱う。

---

### 1-2. CSVヘッダ／マッピング関連

### 1-2-1. `csv_header_mappings`

- 型: JSON（構造体）
- デフォルト値（イメージ）:

```json
{
  "monthly_ticket": {
    "merchant_public_id": "merchant-public-id",
    "transaction_type": "取引種別",
    "transaction_datetime": "取引日時",
    "amount": "販売金額",
    "fee": "手数料",
    "revenue": "売上",
    "product_name": "商品名",
    "customer_name": "顧客名",
    "customer_email": "メールアドレス",
    "customer_phone": "電話番号"
  },
  "spot_card": {
    "merchant_public_id": "merchant-public-id",
    "transaction_type": "取引種別",
    "transaction_datetime": "取引日時",
    "amount": "販売金額",
    "fee": "手数料",
    "revenue": "売上",
    "product_name": "タイトル",
    "customer_name": "顧客名",
    "customer_email": "メールアドレス",
    "customer_phone": "電話番号"
  },
  "spot_local": {
    "merchant_public_id": "merchant-public-id",
    "status": "ステータス",
    "transaction_datetime": "希望の予約日時",
    "amount": "金額",
    "product_name": "メニュー",
    "customer_name": "顧客名",
    "customer_email": "メールアドレス",
    "customer_phone": "電話番号",
    "payment_method": "支払方法"
  }
}

```

- スコープ: システム全体
- 説明:
    - STORES CSV のヘッダ名を **論理項目名** に紐付けるマッピング定義。
    - STORES側で列名が変更された場合でも、コード改修なしで追従できるようにする。

---

### 1-3. 取込挙動・重複判定関連

### 1-3-1. `duplicate_detection_keys`

- 型: `string[]`
- デフォルト値:
    
    `['merchant_public_id','source_type','transaction_datetime','amount','product_name','customer_name']`
    
- スコープ: システム全体
- 説明:
    - 「この組み合わせが同じなら、同一取引とみなしてスキップする」という冪等性判定キー。
    - アプリケーションレベルの重複判定に利用し、DBの UNIQUE 制約は張らない。

---

### 1-3-2. `max_rows_per_job`

- 型: `int`
- デフォルト値例: `500000`
- スコープ: システム全体
- 説明:
    - 1ジョブで処理する **データ行数の上限**。
    - 超える場合はエラーにするか、複数ジョブに分割するかはPhase7 CRON_SYS側で決定。

---

### 1-4. 検証・突合・エラー関連

### 1-4-1. `reconcile_warn_percent`

- 型: `float`（%）
- デフォルト値例: `1.0`（＝1%）
- スコープ: システム全体
- 説明:
    - CSV合計金額と `sales_records` に挿入された金額合計の差分率が
        
        この値を超えたら、ジョブを `completed_with_warning` 扱いにする閾値。
        

---

### 1-4-2. `error_row_retention_days`

- 型: `int`
- デフォルト値例: `30`
- スコープ: システム全体
- 説明:
    - `sales_import_errors.raw_row_text` を保持しておく日数。
    - 経過後は生テキストをNULLにし、エラーコードや行番号だけ残す。

---

### 1-5. 実行モード・自動化関連

### 1-5-1. `default_import_mode`

- 型: `string`（`'dry_run'` / `'immediate'`）
- デフォルト値: `'dry_run'`
- スコープ: システム全体
- 説明:
    - ai. 側のCSVアップロード画面を開いたときに選択される標準モード。
    - `'dry_run'`: 検証のみで `sales_records` へは書き込まない。
    - `'immediate'`: 検証後、そのまま取込まで実行。

---

### 1-5-2. `auto_import_enabled`

- 型: `bool`
- デフォルト値: `false`
- スコープ: システム全体
- 説明:
    - 将来の STORES API 完全自動取込（Phase9拡張）に備えたフラグ。
    - v1.4 時点では常に `false`（手動CSVアップロードのみ）。

---

### 1-5-3. `default_monthly_import_day`

- 型: `int`（1〜31）
- デフォルト値例: `25`
- スコープ: システム全体
- 説明:
    - 「売上CSVを本部がアップロードすることを想定している日付」の目安。
    - CRON_SYS 側のスケジュール説明やヘルプ表示に利用（強制ではない）。

---

## 2. 実装メモ（sys_params へのマッピング方針例）

- `sys_params` テーブルを利用する場合、キーは `sales_link.<param_key>` のように prefix を付ける想定：

| param_key | sys_params.key | 例value |
| --- | --- | --- |
| allowed_source_types | `sales_link.allowed_source_types` | `["monthly_ticket","spot_card","spot_local"]` |
| trial_keyword_list | `sales_link.trial_keyword_list` | `["お試し体験","体験コース"]` |
| csv_header_mappings | `sales_link.csv_header_mappings` | JSON |
| duplicate_detection_keys | `sales_link.duplicate_detection_keys` | `["merchant_public_id", ...]` |
| max_rows_per_job | `sales_link.max_rows_per_job` | `500000` |
| reconcile_warn_percent | `sales_link.reconcile_warn_percent` | `1.0` |
| error_row_retention_days | `sales_link.error_row_retention_days` | `30` |
| default_import_mode | `sales_link.default_import_mode` | `"dry_run"` |
| auto_import_enabled | `sales_link.auto_import_enabled` | `false` |
| default_monthly_import_day | `sales_link.default_monthly_import_day` | `25` |

---

## 3. 決定事項 / 未決事項 / 論点

### 3-1. 決定事項

- SALES_LINK に関する挙動調整は、上記パラメータ群を基本セットとして扱う。
- `csv_header_mappings` により、CSVヘッダ名の変更にコード修正なしで追従できる前提とする。
- 冪等性判定キー（`duplicate_detection_keys`）はパラメータで定義し、アプリケーション側から参照する。
- エラー行の生テキスト保持期間は `error_row_retention_days` で制御する。

### 3-2. 未決事項

- `max_rows_per_job` を超えた場合の扱い（ジョブをエラーにするか、自動分割するか）。
- `reconcile_warn_percent` の実運用値（1%で妥当かどうか）は、しばらく稼働させてから調整が必要。

### 3-3. 論点

- `trial_keyword_list` を店舗ごとに変えたい要望が出るかどうか。
    
    （現時点では全店舗共通とし、必要になれば per-shop override を別途設計する）
