# PHASE_06_KPI_ANALYTICS — KPI分析／ダッシュボード仕様 v1.4

## 0. このドキュメントの位置づけ

- 本ドキュメントは、「加盟店サポート自動化プロジェクト v1.4」の
    
    **Phase6：データ連携／KPI層** における要素 **KPI_ANALYTICS** の仕様を定義する。
    
- 目的は、以下のデータソースを組み合わせて
    
    **店舗別・オーナー別の「健康状態」を可視化するKPIダッシュボード** を実現すること。
    
    - 売上データ：`sales_records`（Phase6 SALES_LINK により統合済み）
    - 店舗・オーナーマスタ：`shops` / `owners`
    - 問い合わせ・チケット：`tickets`
    - AIチュートリアル／AI応答ログ：`tutorial_logs` / `ai_logs` / `sys_events` 等
- 本フェーズでは、次の4点を仕様として確定する。
    1. **「基本の健康診断セット」となるKPI指標の定義**
    2. **店舗別KPIサマリテーブル `shop_kpis`（コアKPI）の設計**
    3. **拡張KPI用テーブル `shop_kpi_metrics`（後から増やせる枠）の設計**
    4. **本部向けKPIダッシュボードUI（ai.）とパラメータ化方針**

> ⚠️ 注意
> 
> - `shop_kpis` / `shop_kpi_metrics` は Phase0〜5 時点では未定義の新規テーブルであり、本ドキュメントで初めて DDL を規定する。
> - 実際のDBマイグレーションは Phase9 PACKAGE_BUILD で行い、それまでは「存在する前提で話をする」だけとする。

---

## 1. 目的・スコープ

### 1-1. 目的（What / Why）

1. **店舗の健康状態を「ひと目で分かる」形にする**
    - 売上・お試し体験・新規／リピート・問い合わせ・AI自己解決 など
    少数の指標で **「この店は今どんな状態か」** を把握できるようにする。
2. **v1.4 では「基本の健康診断」を優先し、将来拡張可能な構造を残す**
    - まずは、運用・意思決定に直結する **コアKPIのみ** をカッチリ固定する。
    - データが蓄積され、「もっと細かい分析をしたい」となったときに
    DBスキーマを何度も ALTER しなくて済むよう、**汎用メトリクス枠** を用意しておく。
3. **AI導入の「効き具合」を数値で確認できるようにする**
    - 顧客・加盟店の問い合わせのうち、どれだけAIで自己解決できているかを定量化し、
    FAQ・導線・AIパラメータ調整の効果検証に使えるようにする。

### 1-2. スコープ（Scope）

**本フェーズでカバーするもの**

- KPI指標の一覧・定義（「何を」「どう計算するか」のレベル）
- 店舗別KPI集計テーブル `shop_kpis`（コアKPI）の DDL 定義
- 拡張メトリクス用テーブル `shop_kpi_metrics` の DDL 定義
- KPI更新のタイミング・集計粒度と、CRONとのI/O契約（論理仕様）
- ai. ポータルにおけるKPIダッシュボード画面の構成・フィルタ条件
- KPI表示ON/OFFやしきい値の **パラメータ化方針**

**本フェーズで明示的に対象外とするもの**

- 外部BIツールの詳細設定（Looker Studio 等）
- 高度な機械学習モデルやLTV予測
→ v1.4 では単純な合計・平均・比率などの「基本解析」に留める。

---

## 2. データソースとモデル構造

### 2-1. 利用テーブル（Fact / Dimension）

**Fact系**

- `sales_records`
    - Phase6 SALES_LINK により、STORES売上CSV3種から統合された売上データ。
    - 主な利用カラム：
        - `shop_id`, `merchant_public_id`
        - `transaction_datetime`
        - `amount`, `revenue`, `is_trial`, `source_type`
        - `customer_email`, `customer_phone` など
- `tickets`
    - 顧客／加盟店／本部間の全問い合わせチケット。
    - 主な利用カラム（想定）：
        - `shop_id`
        - `direction`（customer_to_shop / owner_to_hq 等）
        - `opened_at`, `first_responded_at`, `closed_at`, `sla_due_at`
        - `status`, `category` など
- `tutorial_logs`（名称は仮。Phase3で定義予定）
    - 顧客／加盟店のAIチュートリアル利用1回分（セッション）を表す。
    - 主な利用カラム（想定）：
        - `shop_id`, `user_role`（customer/owner）
        - `channel`（help/line 等）
        - `started_at`, `ended_at`
        - `resolved_by_ai`（1/0）
        - `escalated_to_ticket`（1/0）
- `ai_logs` / `sys_events`
    - `ai_logs` … AIコアの入出力ログ（intent／信頼度／fallback有無など）
    - `sys_events` … 匿名化されたイベントログ（`event_type`, `shop_id`, `channel` など）。
    - v1.4 では、AI関連KPIは `tutorial_logs` をメインに参照し、足りない場合に `sys_events` で補う。

**Dimension系**

- `shops`
    - 店舗マスタ。`id`, `merchant_public_id`, `name`, `region_name`, `shop_status` 等。
- `owners`
    - オーナーマスタ。`id`, `owner_type`（直営／FC）, `name` 等。
- `owner_contacts`
    - オーナー担当／本部担当者の連絡先情報。
    - KPI集計そのものには直接利用しないが、通知やメンション時に利用される。

---

## 3. KPI定義一覧

> 方針：
> 
> - v1.4 ではまず **「基本健康診断セット（コアKPI）」** を `shop_kpis` に固定カラムとして持つ。
> - それ以外の「あると嬉しい詳細指標」は、汎用メトリクス表 `shop_kpi_metrics` に積む。

### 3-1. コアKPI（`shop_kpis` 直カラム）

コアKPIは、「店舗の健康状態」をざっくり把握するための最低限セット。

### 3-1-1. 売上・体験まわり

| KPI名 | 説明 | 元データ | 概算式 |
| --- | --- | --- | --- |
| `gross_sales_amount` | 総売上額（amountの合計） | `sales_records.amount` | `SUM(amount)` |
| `net_revenue_amount` | 手取りベース売上（revenueの合計） | `sales_records.revenue` | `SUM(revenue)` |
| `trial_sales_amount` | お試し体験売上額 | `sales_records.amount` | `SUM(amount WHERE is_trial=1)` |
| `total_transactions` | 総取引件数 | `sales_records` | `COUNT(*)` |
| `trial_transactions` | お試し体験件数 | `sales_records` | `COUNT(*) WHERE is_trial=1` |

### 3-1-2. 顧客ざっくり行動

| KPI名 | 説明 | 元データ | 概算式（イメージ） |
| --- | --- | --- | --- |
| `new_customer_count` | 期間内に「初めて来た」顧客数 | `sales_records` | 「顧客ごとの初回取引日が集計期間に属する人数」 |
| `repeat_customer_count` | 期間内にリピート来店した顧客数 | `sales_records` | 「過去に来店歴があり、かつ集計期間内にも来店した人数」 |

※ 顧客IDは `COALESCE(customer_email, customer_phone)` 等で擬似ID化する前提。

正確な顧客マスタは将来フェーズで検討。

### 3-1-3. AI・問い合わせまわり（店舗負荷のざっくり把握）

| KPI名 | 説明 | 元データ | 概算式 |
| --- | --- | --- | --- |
| `customer_ai_sessions` | 顧客向けAIチュートリアルセッション数 | `tutorial_logs` | `COUNT(*) WHERE user_role='customer'` |
| `customer_ai_resolved_sessions` | AIのみで解決した顧客セッション数 | `tutorial_logs` | `COUNT(*) WHERE user_role='customer' AND resolved_by_ai=1 AND escalated_to_ticket=0` |
| `customer_ai_escalated_sessions` | 人間対応にエスカレした顧客セッション数 | `tutorial_logs` | `COUNT(*) WHERE user_role='customer' AND escalated_to_ticket=1` |
| `customer_ticket_count` | 顧客→店舗のチケット件数 | `tickets` | `COUNT(*) WHERE direction='customer_to_shop'` |
| `closed_ticket_count` | 期間中にクローズしたチケット件数 | `tickets` | `COUNT(*) WHERE closed_at BETWEEN period` |
| `sla_breached_count` | SLA違反チケット件数 | `tickets` | `COUNT(*) WHERE closed_at > sla_due_at` |
| `avg_first_response_minutes` | 初回回答までの平均時間（分） | `tickets` | `AVG(first_responded_at - opened_at)`（分換算） |

### 3-1-4. 合成スコア（ざっくり店コンディション）

| KPI名 | 説明 | 概算イメージ |
| --- | --- | --- |
| `kpi_score_overall` | 店舗コンディション総合スコア（0〜100） | 売上系・問い合わせ系・AI自己解決率・SLA などを重み付けして算出 |

> kpi_score_overall の具体的な計算式は、KPI_ANALYTICS_DEFAULT_PARAMS のスコアリング設定に従う。
> 
> 
> v1.4 では「ざっくり良好／要注意／危険」を判定できるレベルに留める。
> 

---

### 3-2. 拡張KPI（`shop_kpi_metrics` 側で扱うもの）

拡張KPIは、「データが溜まってから広げる」ための指標群。

例として以下のようなものを想定するが、**DBスキーマをALTERせずに増やせる**前提とする。

- お試し→継続メニュー転換率（例：`metric_key='trial_to_subscription_rate'`）
- 2回目来店率（例：`metric_key='second_visit_rate'`）
- 支払方法別構成比（カード／現地／その他）
- AIの intent カテゴリ別成功率（例：`metric_key='ai_resolution_rate_payment_questions'`）
- 時間帯別・曜日別売上分布（JSON形式で分布を持つ）

これらは、`shop_kpi_metrics` に対して

- `metric_key`（論理名）
- `metric_type`（int / float / percent / money / json）
- `metric_value_numeric` / `metric_value_json`

として保存する方針とする。

---

## 4. テーブル設計

### 4-1. テーブル：`shop_kpis`（コアKPI用）

**目的**

- 店舗別・期間別の **基本的な健康診断KPI** を、日次／月次単位で保持するサマリテーブル。
- ダッシュボード初期表示や一覧表示は原則 `shop_kpis` の値だけで完結する。

**DDL（提案）**

```sql
CREATE TABLE shop_kpis (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主キー',
  shop_id BIGINT UNSIGNED NOT NULL COMMENT '対象店舗ID（shops.id）',
  owner_id BIGINT UNSIGNED NULL COMMENT 'オーナーID（owners.id）。NULLなら不明/未紐付け',
  period_granularity ENUM('daily','monthly') NOT NULL DEFAULT 'monthly'
    COMMENT '集計粒度（日次 or 月次）。v1.4ではmonthlyメイン',
  period_start_date DATE NOT NULL COMMENT '集計期間開始日（含む）',
  period_end_date DATE NOT NULL COMMENT '集計期間終了日（含む）',
  period_label VARCHAR(16) NOT NULL COMMENT '期間ラベル（例：2025-03、2025-03-15など）',

  -- 売上関連（基本）
  gross_sales_amount BIGINT NOT NULL DEFAULT 0 COMMENT '総売上額（amountの合計）',
  net_revenue_amount BIGINT NOT NULL DEFAULT 0 COMMENT '手取りベース売上額（revenueの合計）',
  trial_sales_amount BIGINT NOT NULL DEFAULT 0 COMMENT 'お試し体験売上額',
  total_transactions INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '総取引件数',
  trial_transactions INT UNSIGNED NOT NULL DEFAULT 0 COMMENT 'お試し体験件数',

  -- 顧客ざっくり行動
  new_customer_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '初来店顧客数',
  repeat_customer_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT 'リピート顧客数',

  -- AI・問い合わせ（店のしんどさを把握する基本指標）
  customer_ai_sessions INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '顧客向けAIセッション数',
  customer_ai_resolved_sessions INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '顧客AI自己解決セッション数',
  customer_ai_escalated_sessions INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '顧客AIエスカレーションセッション数',

  customer_ticket_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '顧客→店舗チケット件数',
  closed_ticket_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '期間中クローズしたチケット件数',
  sla_breached_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT 'SLA違反件数',
  avg_first_response_minutes INT UNSIGNED NULL COMMENT '初回応答までの平均時間（分）。NULL=データ不足',

  -- 合成スコア（シンプルに1本だけ）
  kpi_score_overall TINYINT UNSIGNED NULL COMMENT '店舗コンディション総合スコア（0〜100）',

  -- メタ情報
  calculated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
    COMMENT 'このレコードのKPIを計算した日時',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
    COMMENT 'レコード作成日時',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
    COMMENT '更新日時',

  PRIMARY KEY (id),
  UNIQUE KEY uniq_shop_period (shop_id, period_granularity, period_start_date, period_end_date),
  KEY idx_owner_period (owner_id, period_granularity, period_start_date),
  KEY idx_period_label (period_label)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='店舗別KPI集計テーブル（基本健康診断用の日次・月次サマリ）';

```

---

### 4-2. テーブル：`shop_kpi_metrics`（拡張KPI用）

**目的**

- コアKPI以外の指標（後から増える分析ニーズ）を
    
    **DBスキーマをALTERせずに追加できるようにするための汎用テーブル**。
    
- `metric_key` 単位で、「この期間・この店舗・この指標の値」を保存する。

**DDL（提案）**

```sql
CREATE TABLE shop_kpi_metrics (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主キー',
  shop_id BIGINT UNSIGNED NOT NULL COMMENT '対象店舗ID（shops.id）',
  owner_id BIGINT UNSIGNED NULL COMMENT 'オーナーID（owners.id）',
  period_granularity ENUM('daily','monthly') NOT NULL DEFAULT 'monthly'
    COMMENT '集計粒度（日次 or 月次）',
  period_start_date DATE NOT NULL COMMENT '集計期間開始日（含む）',
  period_end_date DATE NOT NULL COMMENT '集計期間終了日（含む）',
  period_label VARCHAR(16) NOT NULL COMMENT '期間ラベル（例：2025-03）',

  metric_key VARCHAR(64) NOT NULL COMMENT 'KPIの論理名（例：trial_to_subscription_rate）',
  metric_type ENUM('int','float','percent','money','json') NOT NULL DEFAULT 'float'
    COMMENT '値の型を示す分類',
  metric_value_numeric DOUBLE NULL COMMENT '数値系KPIの値（percent/money含む）',
  metric_value_json JSON NULL COMMENT '分布や詳細をJSONで持ちたい場合に使用',
  metric_unit VARCHAR(16) NULL COMMENT '単位（件, %, 円 など）',
  metric_source VARCHAR(64) NULL COMMENT '算出ロジック識別子（集計バッチ名やバージョン）',

  calculated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
    COMMENT 'このKPIを計算した日時',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
    COMMENT 'レコード作成日時',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
    COMMENT '更新日時',

  PRIMARY KEY (id),
  KEY idx_shop_period (shop_id, period_granularity, period_start_date, period_end_date),
  KEY idx_owner_period (owner_id, period_granularity, period_start_date),
  KEY idx_metric_key (metric_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  COMMENT='店舗別KPI拡張メトリクス（任意の指標を柔軟に追加するためのテーブル）';

```

**v1.4 で想定する代表的な `metric_key` 例**

- `trial_to_subscription_rate` … お試し→継続メニュー転換率
- `second_visit_rate` … 2回目来店率
- `payment_card_ratio` / `payment_local_ratio` … 支払手段別構成比
- `owner_ai_resolution_rate` … 加盟店側AI自己解決率
- `customer_ticket_category_distribution` … チケットカテゴリ分布（`metric_type='json'`）

※ どの `metric_key` を使うか／どの画面で見せるかは `KPI_ANALYTICS_DEFAULT_PARAMS` やUI側の設定で制御。

---

## 5. 集計ロジックと更新タイミング

### 5-1. コアKPI集計ロジック（shop_kpis）

1. **売上KPI**
    - `sales_records` を `shop_id` + 期間（`transaction_datetime`）で GROUP BY。
    - `gross_sales_amount` = SUM(amount)
    - `net_revenue_amount` = SUM(revenue)
    - `trial_sales_amount` = SUM(amount WHERE is_trial=1)
    - `total_transactions` = COUNT(*)
    - `trial_transactions` = COUNT(*) WHERE is_trial=1
2. **新規／リピート**
    - 顧客ID（擬似） = `COALESCE(customer_email, customer_phone)` 前提。
    - 顧客ごとの **最初の取引日** を求め、
        - その日付が集計期間内 → new
        - それ以前に初回来店済み かつ 集計期間内に来店あり → repeat
3. **AIセッション（顧客）**
    - `tutorial_logs` を `shop_id` + 期間で GROUP BY。
    - `customer_ai_sessions` = COUNT(*) WHERE user_role='customer'
    - `customer_ai_resolved_sessions` = COUNT(*) WHERE user_role='customer' AND resolved_by_ai=1 AND escalated_to_ticket=0
    - `customer_ai_escalated_sessions` = COUNT(*) WHERE user_role='customer' AND escalated_to_ticket=1
4. **チケット／SLA**
    - `tickets` を `shop_id` + 期間で GROUP BY。
    - `customer_ticket_count` = COUNT(*) WHERE direction='customer_to_shop'
    - `closed_ticket_count` = COUNT(*) WHERE closed_at in期間
    - `sla_breached_count` = COUNT(*) WHERE closed_at > sla_due_at
    - `avg_first_response_minutes` = AVG(TIMESTAMPDIFF(MINUTE, opened_at, first_responded_at))
5. **合成スコア**
    - `kpi_score_overall` の例：
        - 売上水準（gross_sales_amount / 全店中央値）
        - お試し比率（trial_transactions / total_transactions）
        - 顧客AI自己解決率（customer_ai_resolved_sessions / customer_ai_sessions）
        - SLA遵守率（1 - sla_breached_count / closed_ticket_count）
            
            などを 0〜1 のスケールに正規化し、重み付けして 0〜100 点に換算。
            
    - 重み・しきい値は `KPI_ANALYTICS_DEFAULT_PARAMS` で設定。

### 5-2. 拡張KPI集計ロジック（shop_kpi_metrics）

- `cron_kpi_aggregate.php`（Phase7予定）の中で、
    - `-extended` オプションや内部設定に応じて追加集計を実行。
- 各拡張KPIごとに
    - `metric_key`
    - `metric_type`
    - `metric_unit`
    - 算出ロジックのバージョン（`metric_source`）
        
        を決め、`shop_kpi_metrics` に INSERT / UPDATE。
        

---

### 5-3. CRON のI/O契約（論理）

**バッチ名（想定）：** `cron_kpi_aggregate.php`（Phase7で実装）

**呼び出し例**

- 日次集計：
    - `php cron_kpi_aggregate.php --granularity=daily --date=2025-03-15`
- 月次集計：
    - `php cron_kpi_aggregate.php --granularity=monthly --month=2025-03`
- 拡張KPIも含めて再計算：
    - `php cron_kpi_aggregate.php --granularity=monthly --month=2025-03 --extended=1`

**挙動（概要）**

1. 対象期間・粒度を解決（`period_start_date`, `period_end_date`, `period_label`）。
2. 該当期間の `shop_kpis` レコードを削除 or 上書きモードで再計算。
3. `sales_records` / `tutorial_logs` / `tickets` / `sys_events` からコアKPIを集計し、`shop_kpis` にINSERT。
4. `-extended=1` の場合は、拡張KPI定義に従い `shop_kpi_metrics` を追加更新。
5. 集計結果・異常値はログ出力（詳細はPhase7）。

---

## 6. ダッシュボードUI（ai.）

### 6-1. 画面タブ構成（イメージ）

1. **「売上・体験」タブ**
    - データソース：主に `shop_kpis`
    - 表示：
        - 総売上（net / gross）、お試し売上
        - お試し体験件数・体験比率（trial_transactions / total_transactions）
        - 新規顧客数・リピート顧客数
2. **「AI・問い合わせ」タブ**
    - データソース：`shop_kpis`（コア）、`shop_kpi_metrics`（拡張）
    - 表示：
        - 顧客AIセッション数／AI自己解決率／エスカレーション率
        - 顧客チケット件数／SLA遵守率／平均初回応答時間
        - 将来的には intent別成功率なども `shop_kpi_metrics` から表示
3. **「店舗コンディション」タブ**
    - データソース：`shop_kpis`（kpi_score_overall）
    - 表示：
        - 店舗ごとの `kpi_score_overall` を一覧＋色分け（例：80点以上＝緑、60〜79＝黄、60未満＝赤）
        - 「要注意店舗だけ表示」フィルタ
        - ソート：スコア降順／売上降順／問い合わせ多い順 など

### 6-2. 共通フィルタ条件

- 期間
    - 「今月」「先月」「過去3ヶ月」「任意期間」
- 粒度
    - `monthly`（初期）／ `daily`（将来ONにする想定）
- 店舗
    - 単一店舗／複数選択／全店舗
- オーナー
    - 単一オーナー／直営のみ／FCのみ
- スコア状態
    - 「要注意店舗のみ」（`kpi_score_overall < warning_threshold`）
    - 「SLA違反が多い店舗」など

### 6-3. 文言トーン

- KPIの横には、**解釈を補助する一言コメント** を添える：
    - 例：「AI自己解決率 80％ → 多くの問い合わせがAIで完結できています」
    - 例：「SLA遵守率 60％ → 対応が遅れがちです。優先的なフォローを検討してください」
- 「悪い数字を責める」のではなく、
    
    「この数字のとき、こういう可能性が高い」形式の冷静なコメントで統一する。
    

---

## 7. 🔧 パラメータ候補 — `KPI_ANALYTICS_DEFAULT_PARAMS`

> 実体は KPI_ANALYTICS_DEFAULT_PARAMS.md または sys_params 相当の設定テーブルとして管理する。
> 

### 7-1. 集計・表示まわり

- `kpi_default_granularity = 'monthly'`
- `kpi_default_period_months = 3`
    
    → 初期表示で過去3ヶ月を表示。
    
- `kpi_allow_daily = 0`
    
    → 日次集計はオフ（将来ONにできる）。
    

### 7-2. AI／問い合わせ関連しきい値

- `kpi_ai_good_resolution_threshold = 0.7`
    
    → 自己解決率70％以上で「良好」。
    
- `kpi_ai_bad_resolution_threshold = 0.4`
    
    → 40％未満で「要改善」。
    
- `kpi_customer_ticket_high_threshold`
    
    → 顧客チケットがこの数を超えると「問い合わせ多め」と表示。
    

### 7-3. SLA／サポート

- `kpi_sla_good_threshold = 0.9`
    
    → SLA遵守率90％以上で「良好」。
    
- `kpi_sla_bad_threshold = 0.7`
    
    → 70％未満で「要注意」。
    
- `kpi_first_response_target_minutes = 60`
    
    → 初回応答時間の目標（分）。
    

### 7-4. 合成スコアの重み

- `kpi_score_sales_weight = 0.5`
- `kpi_score_support_weight = 0.5`
- `kpi_score_overall_good_threshold = 80`
- `kpi_score_overall_warning_threshold = 60`

### 7-5. 拡張KPI関連

- `kpi_metrics_enabled_keys`
    
    → ダッシュボードに表示する `metric_key` のホワイトリスト。
    
- `kpi_metrics_experimental_keys`
    
    → 試験運用中の指標（デフォルト非表示）。
    

### 7-6. データ保持

- `kpi_data_retention_months = 36`
    
    → `shop_kpis` / `shop_kpi_metrics` の保持期間（月数）。
    
    超えたものはアーカイブ or 削除（Phase7で詳細）。
    

---

## 8. セキュリティ・プライバシー

- `shop_kpis` / `shop_kpi_metrics` には、顧客の氏名・メールアドレス・電話番号などの **個票情報は持たない**。
- 顧客IDとして `customer_email` / `customer_phone` を利用するのは
    
    集計処理（バッチ）内でのみとし、サマリテーブル側には残さない。
    
- KPIダッシュボードに表示するのは、あくまで店舗・オーナー単位の集計値に限定する。
- 個票レベルの売上や問い合わせ内容を確認する場合は、
    
    別画面（AI-INBOX・店舗詳細など）で適切な権限チェックの下で参照する。
    

---

## 9. 決定事項／未決事項／確認論点

### 9-1. 決定事項（Phase6 KPI_ANALYTICS）

- 店舗別KPIサマリは、**新規テーブル `shop_kpis`** に日次／月次単位で保持する。
- `shop_kpis` には、**「基本健康診断セット」** のコアKPIのみを固定カラムとして持つ。
- コアKPI以外の指標は、**新規テーブル `shop_kpi_metrics`** に `metric_key` ベースで保存する。
- KPIダッシュボード（ai.）は、
    - コア指標 → `shop_kpis`
    - 拡張指標 → `shop_kpi_metrics`
        
        を参照する二段構成とする。
        
- コアKPIの重み付け・しきい値は `KPI_ANALYTICS_DEFAULT_PARAMS` として外出しし、コード直書きを禁止する。
- 日次／月次の集計バッチは `cron_kpi_aggregate.php`（Phase7）で実装し、本仕様をI/O契約のベースとする。

### 9-2. 未決事項（要仕様判断）

- `tutorial_logs` の最終スキーマと、「resolved_by_ai」「escalated_to_ticket」の持ち方。
    
    → Phase3の仕様と整合を取る必要がある。
    
- 顧客IDの擬似キーにおいて、`customer_email` / `customer_phone` をどう優先順に扱うか。
    
    → 重複や変更（キャリアメールなど）をどこまで許容するか。
    
- `kpi_score_overall` の具体的なスコアリング式。
    
    → どの指標をどの程度の重みで評価するかは、運用を見ながら調整が必要。
    

### 9-3. 確認すべき論点（運用設計）

- KPIダッシュボードの一部を将来、shop.（加盟店ポータル）に開放するかどうか。
    
    → 当面は ai.（本部専用）に限定し、要望を見て検討する。
    
- KPIの更新頻度（「日次も必須か、月次で十分か」）。
    
    → v1.4 では月次を主とし、日次はオンオフ切替可能なオプション扱いとする。
    
- 「要注意店舗」の定義ロジック：
    - スコアだけで判定するか
    - 「SLA違反が連続◯ヶ月」「問い合わせ増加トレンド」などの条件も組み込むか。

---

## X. 【任意拡張】ロイヤリティ特例ルール関連KPI（Phase5 反映）

> ※本節は、Phase5 で追加されたロイヤリティ特例ルールに関する  
>  「KPIダッシュボードへの表示ニーズ」を想定した任意拡張であり、  
>  v1.4の必須実装ではない。  

### X.1 目的

- ロイヤリティ特例ルール `shop_royalty_relief_rules` の「条件達成状況」を  
  KPIダッシュボード（ai.）からざっくり把握できるようにする。
- **特例ルールの評価ロジック自体は Phase7 のバッチ側に任せ**、  
  KPI側では「結果（達成したかどうか）」だけを拾って表示する。

### X.2 拡張メトリクス案（`shop_kpi_metrics` 向け）

以下のような `metric_key` を、`shop_kpi_metrics` の任意拡張として定義しておく。

#### X.2.1 `royalty_relief_condition_met`

- 用途:
  - 対象期間において、ロイヤリティ特例ルールの条件を **1回でも満たしたかどうか** をフラグで持つ。
- 格納例:
  - `metric_key = 'royalty_relief_condition_met'`
  - `metric_type = 'int'`
  - `metric_value_numeric = 1`（条件達成あり） or `0`（条件達成なし）
  - `metric_unit = ''`
- サマリ画面での使い方:
  - 「特例条件達成済みの店舗のみ表示」フィルタ  
  - 「条件達成店舗数」のカウント

#### X.2.2 `royalty_relief_condition_count`

- 用途:
  - 対象期間内に条件達成が何回起きたか（ルール複数件があれば合算）。
- 格納例:
  - `metric_key = 'royalty_relief_condition_count'`
  - `metric_type = 'int'`
  - `metric_value_numeric = 達成回数`

#### X.2.3 `royalty_relief_active_rules_count`

- 用途:
  - 対象店舗に現在有効な特例ルールがいくつ設定されているか。
- 格納例:
  - `metric_key = 'royalty_relief_active_rules_count'`
  - `metric_type = 'int'`
  - `metric_value_numeric = is_active=1 の件数`

> これらはすべて **バッチ側で集計し `shop_kpi_metrics` にINSERT** する想定であり、  
> KPI_ANALYTICS 側の責務は「どういう metric_key があれば便利か」を定義するところまでとする。

---

### X.3 KPI_ANALYTICS_DEFAULT_PARAMS への追加案

任意だが、以下のパラメータを追加しておくと UI 制御がしやすい。

#### X.3.1 `kpi_metrics_enabled_keys` への初期値候補

- `['royalty_relief_condition_met','royalty_relief_condition_count']`  
  → ロイヤリティ特例関連の指標をダッシュボード上で有効にする場合。

#### X.3.2 `kpi_royalty_relief_highlight_enabled`

- 型: `bool`
- デフォルト値例: `false`
- 説明:
  - `true` の場合、「ロイヤリティ特例条件を満たした店舗」を  
    コンディションタブ上でハイライト表示する。

---

### X.4 決定事項 / 未決事項（任意拡張分）

#### 決定事項（採用する場合）

- ロイヤリティ特例ルールの条件達成情報を KPI ダッシュボードに反映する場合、  
  その情報は `shop_kpi_metrics` に `metric_key` ベースで保存し、  
  `shop_kpis` の固定カラムは増やさない。
- metric_key は最低限  
  - `royalty_relief_condition_met`  
  - `royalty_relief_condition_count`  
  を用意する。

#### 未決事項

- 実際に v1.4 でどこまで表示するか：
  - 「要注意店舗の絞り込みに使うだけ」に留めるのか、  
  - 専用のロイヤリティ分析タブを作るのかは運用要件次第。
- 直営店舗にも同じメトリクスを使うか（＝FC限定にするかどうか）。
