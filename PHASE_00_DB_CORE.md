# PHASE_00_DB_CORE.md

## Phase0 基本DBコア仕様（オーナー／店舗／契約・料金／備品／リンク／売上）

### このドキュメントの位置づけ

- 本ドキュメントは「加盟店サポート自動化プロジェクト v1.4」における **Phase0 の基本DB仕様** を定義する。
- 範囲は、次のコンテキストの「土台テーブル」のみ：
    - 加盟店オーナー／本部（owners, owner_contacts）
    - 店舗マスタと関連情報（shops, shop_secrets, shop_links, shop_internal_info, shop_options）
    - 無人システム・オプション・ロイヤリティ・請求用の基本構造（fee_* 系, shop_fee_* 系, shop_royalty_history）
    - 備品発注と請求連動（supply_items, shop_supplies）
    - STORES予約との連携による売上データ（sales_records）
- 以降のフェーズ（Phase1 SYS_CORE / Phase2 AI_CORE 等）は、このDBコアを **前提として追加テーブルを生やす側** とし、コアの意味を壊さない。

---

## 0. 共通ルール

### 0-1. 命名ルール

- 本システム側の「店舗」は **shop / shops** で統一する。
- 外部サービス「STORES予約」に関するものは **STORES** と明示し、
STORESの店舗識別子は `merchant_public_id` で表現する。
- 本部（HQ）も含め、オーナーは `owners` テーブルで一元管理する。

### 0-2. 契約の考え方

- 契約単位の実体は「**店舗単位**」。
- **既存店舗**
    - 原則として、契約条件は継続。
    - 値引き・特例等は `shop_fee_package_assignments` / `shop_fee_addons` / `shop_royalty_history` のレコードで表現する。
- **今後の新規店舗**
    - 契約時点で有効な
        - パッケージ（`fee_packages` + `package_components`）
        - アドオン（`fee_items`）
        - ロイヤリティ条件（shops.royalty_*）
        を組み合わせて契約条件とする。
- Phase0では「専用の store_contract テーブル」は持たず、**「店舗 × 期間 × パッケージ／アドオン／ロイヤリティ履歴」＝契約内容** と定義する。

---

## 1. オーナー／本部コンテキスト

### 1-1. owners（オーナーマスタ）

**目的**

- 加盟店オーナーおよび本部（HQ）を一元管理するマスタ。
- 契約情報・請求先・通知先（ChatWork / メール）を保持する。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー（AUTO_INCREMENT）。内部参照用。 |
| owner_code | 加盟店コード。不変の業務識別子。UI上で編集不可。 |
| owner_type | オーナー種別。`hq`=本部直営、`franchisee`=加盟店。 |
| owner_status | 状態。`active` / `suspended` / `closed`。 |
| contract_holder_name | 契約書上の契約者名義。 |
| contract_type | 契約区分。`corporate`=法人、`individual`=個人。 |
| contract_zip / contract_addr_* / contract_phone | 契約住所とその電話。 |
| corporate_number | 法人番号。法人のみ。 |
| representative_* | 代表者名・カナ・生年月日・メール。 |
| mailing_* | 請求書や書類郵送に使う郵送先。契約住所と異なる場合に利用。 |
| chatwork_room_id | オーナー用ChatWorkルームID。複数店舗が1ROOMを共有する。 |
| preferred_contact | 通知手段の希望。`chatwork` / `email` / `both`。 |
| billing_email | 請求書送付先のメール。 |
| post0_contract | Post0契約フラグ（将来の契約種別識別用）。 |
| notes | オーナーに関する社内メモ。 |
| is_profile_complete | オーナー情報の入力完了フラグ。 |
| created_at / updated_at | 登録・更新日時。 |

**運用上の補足**

- `chatwork_room_id` は、請求・MTG・緊急連絡などオーナー単位の通知経路として利用するため、**原則必須** とする（未設定の場合は本部側の共通ROOMにフォールバックする運用を Phase1 以降で定義）。

---

### 1-2. owner_contacts（オーナー担当者）

**目的**

- オーナーごとの運営担当者（責任者／経理／運営）と、そのChatWorkメンション先を管理する。
- 通知時に「どの担当者をメンションするか」を制御する。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| owner_id | 紐づくオーナー（[owners.id](http://owners.id/)）。 |
| role | 担当区分。`manager`/`accounting`/`operations`/`other`。 |
| display_order | 表示順。UI側で最大5件などの制御を行う。 |
| name / kana | 担当者氏名／カナ。 |
| email / phone | 担当者の連絡先。 |
| chatwork_id | ChatWorkアカウントID（メンション用）。 |
| is_active | 在任中かどうか。 |
| created_at / updated_at | 登録・更新日時。 |

**運用上の補足**

- `chatwork_id` は、緊急案件やMTG確定等のエスカレーション時にメンション先として利用するため、**運用上は必須項目** とする（DBレベルでは NULL を許容する場合でも、入力フォーム側で必須扱い）。
- オーナー単位の通知（owners.chatwork_room_id）と併せて、
    - 「ROOMに投げる」＋「特定担当者をメンションする」
    という二段構えの通知ができるようにする。

---

### 1-3. shop_staff_users（オーナー配下スタッフ）

**目的**

- オーナー配下の「スタッフアカウント」（SHOP_PORTAL にログインする担当者）を管理する。
- メールアドレス／ChatWork ID／ロール（`staff_basic` / `staff_multi_shop`）などを保持する。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| owner_id | 紐づくオーナー（[owners.id](http://owners.id/)）。このオーナー配下のスタッフであることを示す。 |
| name / kana | スタッフ氏名／カナ。 |
| email | ログインIDとして利用するメールアドレス。オーナー配下で一意。 |
| chatwork_id | ChatWorkアカウントID（メンション用）。運用上必須。 |
| default_role | デフォルトロール。`staff_basic` / `staff_multi_shop`。 |
| portal_user_id | 認証テーブル（portal_users.id）への参照。Phase1 以降で利用。 |
| is_active | 在任中かどうか。 |
| created_at / updated_at | 登録・更新日時。 |

**運用上の前提**

- スタッフは必ずいずれかのオーナーに属する（独立した本部ユーザーは Phase1 以降の別テーブルで管理）。
- ChatWork を通じてスタッフに直接メンションするケース（MTGリマインド等）を想定し、`chatwork_id` は必須とする。

**DDL例（参考）**

```sql
CREATE TABLE shop_staff_users (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  owner_id BIGINT UNSIGNED NOT NULL,
  name VARCHAR(191) NOT NULL,
  kana VARCHAR(191) NULL,
  email VARCHAR(191) NOT NULL,
  chatwork_id VARCHAR(64) NOT NULL,
  default_role ENUM('staff_basic','staff_multi_shop') NOT NULL DEFAULT 'staff_basic',
  portal_user_id BIGINT UNSIGNED NULL,
  is_active TINYINT(1) NOT NULL DEFAULT 1,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,
  UNIQUE KEY uniq_staff_email_owner (owner_id, email),
  CONSTRAINT fk_staff_owner FOREIGN KEY (owner_id) REFERENCES owners(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

---

### 1-4. shop_staff_assignments（スタッフと店舗の紐づけ）

**目的**

- スタッフがどの店舗を担当しているかを管理する。
- 1 スタッフが複数店舗を担当することを許容する（マルチ店舗担当の運用を想定）。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| staff_user_id | スタッフID（shop_staff_users.id）。 |
| shop_id | 店舗ID（shops.id）。 |
| is_primary | 代表店舗かどうか（UI表示・ソート用）。 |
| created_at / updated_at | 登録・更新日時。 |

**運用上の前提**

- SHOP_PORTAL での「店舗切り替え」や、staff_multi_shop ロールによる横断ビューは、このテーブルを通じて判定する。
- 代表店舗（`is_primary = 1`）は任意だが、一覧表示時のソート優先度などに利用する。

**DDL例（参考）**

```sql
CREATE TABLE shop_staff_assignments (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  staff_user_id BIGINT UNSIGNED NOT NULL,
  shop_id BIGINT UNSIGNED NOT NULL,
  is_primary TINYINT(1) NOT NULL DEFAULT 0,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,
  UNIQUE KEY uniq_staff_shop (staff_user_id, shop_id),
  CONSTRAINT fk_staff_assign_staff
    FOREIGN KEY (staff_user_id) REFERENCES shop_staff_users(id),
  CONSTRAINT fk_staff_assign_shop
    FOREIGN KEY (shop_id) REFERENCES shops(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

---

### 1-5. shop_staff_mask_rules（スタッフごとのマスク項目）

**目的**

- スタッフごとに「どの情報を見せる／隠すか」を制御するマスク設定を管理する。
- オーナー単位のデフォルトマスクと、特定スタッフ向けの上書きを両方持てるようにする。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| owner_id | オーナーID（owners.id）。 |
| staff_user_id | スタッフID（shop_staff_users.id）。NULL の場合は「オーナー配下の全スタッフ共通デフォルト」。 |
| field_key | マスク対象フィールドキー（例：`billing.total_amount`, `sales.gross_amount`, `kpi.detail` 等）。 |
| mask_mode | マスクモード。`mask` / `show`。`mask` の場合 UI で金額等を伏せる。 |
| created_at / updated_at | 登録・更新日時。 |

**運用イメージ**

- `staff_user_id IS NULL` の行 … オーナー配下の全スタッフに対する **デフォルトマスク**。
- 特定スタッフに対してだけ表示・非表示を変えたい場合、
    - 同じ `field_key` で `staff_user_id = 該当スタッフ` の行を作成し、
    - デフォルト（NULL 行）の設定を上書きする。

**DDL例（参考）**

```sql
CREATE TABLE shop_staff_mask_rules (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  owner_id BIGINT UNSIGNED NOT NULL,
  staff_user_id BIGINT UNSIGNED NULL,
  field_key VARCHAR(64) NOT NULL,
  mask_mode ENUM('mask','show') NOT NULL DEFAULT 'mask',
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,
  CONSTRAINT fk_staff_mask_owner FOREIGN KEY (owner_id) REFERENCES owners(id),
  CONSTRAINT fk_staff_mask_staff FOREIGN KEY (staff_user_id) REFERENCES shop_staff_users(id),
  UNIQUE KEY uniq_staff_mask (owner_id, staff_user_id, field_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```

---

## 2. 店舗コンテキスト

### 2-1. shops（店舗マスタ）

**目的**

- 本部直営・加盟店を問わず「顧客から見える店舗」を表現する。
- オーナーとの紐づけ、住所、問い合わせ先、ロイヤリティの現在値を持つ。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| code | 店舗コード。不変の業務識別子。メールIDやURLスラッグ等で利用。 |
| owner_id | オーナーID（owners.id）。本部直営の場合も本部オーナーと紐づく。 |
| merchant_public_id | STORES予約の `merchant_public_id`。売上CSVとのJOINに使用。 |
| name | 店舗名。基本は本部のみ編集。 |
| region_name | 地域名（名古屋駅前、金山などのざっくり分類）。 |
| contract_date | 店舗単位の本部契約日。 |
| open_date | 各店舗のオープン日。 |
| shop_status | 状態。準備中 / 営業中 / 閉店済。 |
| address_zip / address_line1 / address_line2 / address_building | 店舗住所。 |
| phone | 店舗電話番号。 |
| shop_email | 店舗代表窓口メール。 |
| remotelock_child_email | RemoteLOCK子アカウントのログインメール（オーナーに開示するアカウント）。 |
| business_hours | 営業時間の表示文字列（初回は加盟店入力、確定後は本部管理）。 |
| stores_public_url | STORES予約ページURL。 |
| notes | 一般的な店舗メモ（本部と加盟店の両方が理解してよい情報）。 |
| billing_notes | 請求書備考欄に出したいメモ。 |
| royalty_type | ロイヤリティ方式。`flat`/`percent`/`mixed`。 |
| royalty_flat | 定額額（円）。 |
| royalty_percent | ％（例：5.00=5％）。 |
| created_at / updated_at | 登録・更新日時。 |

---

### 2-2. shop_secrets（店舗機密情報）

**目的**

- 店舗ごとのログイン情報・APIキー・親アカウントなど、**オーナーに開示しない秘密情報**を暗号化して保持する。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| shop_id | 店舗ID（shops.id）。 |
| k | キー名（例：`REMOTELOCK_PARENT_EMAIL` / `REMOTELOCK_PARENT_PASSWORD` / `REMOTELOCK_API_KEY` 等）。 |
| v_enc | 暗号化された値。復号はサーバ側ロジックのみ。 |
| updated_at | 更新日時。 |

**RemoteLOCKの運用前提**

- 親アカウント（本部／店舗用）は `shop_secrets` 側にのみ保存し、オーナーには非開示。
- 子アカウントは `shops.remotelock_child_email` に保持し、オーナーにはこのメールアドレスだけを表示する（PWは原則表示せず）。

---

### 2-3. options_master / shop_options（店舗オプション）

**目的**

- 「現地払い対応」「USEN BGM導入」「USEN光」など、**金額には直接関与しないが運用上重要なON/OFF**を管理する。

**options_master**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| option_code | オプションコード（例：`camera_safie` / `bgm_usen` / `net_usen` / `line_miniapp`）。 |
| option_name | オプション名（画面表示用）。 |
| description | 説明文。 |
| default_value | デフォルトON/OFF（1=ON,0=OFF）。 |
| editable_by_store | 加盟店が編集可能かどうか。Phase0では原則0（本部のみ変更）。 |
| created_at / updated_at | 登録・更新日時。 |

**shop_options**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| shop_id | 店舗ID（shops.id）。 |
| option_id | オプションID（options_master.id）。 |
| value | 値。1=ON,0=OFF。 |
| updated_at | 更新日時。 |

---

### 2-4. shop_links（店舗リンク集）

**目的**

- Googleマップ、Googleクチコミ、LP、アンケートURL、各SNSなどの **外部リンクを構造化** して管理する。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| shop_id | 店舗ID。 |
| link_type | 種別（`google_map` / `google_review` / `instagram` / `lp` / `survey` / `ticket` / `product` / `blog` / `official_site` / `line_add` / `other` 等）。 |
| link_url | URL本体。 |
| notes | 用途メモなど。 |
| updated_at | 更新日時。 |

---

### 2-5. shop_internal_info（店舗内部情報）

**目的**

- 鍵、セコム情報、施工業者、緊急連絡先など、**本部運用専用の内部情報**を管理する。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| shop_id | 店舗ID。 |
| info_type | 種別（`key` / `secom` / `vendor` / `emergency_contact` / `contract_info` / `notice` / `other`等）。 |
| value | 内容（自由記述）。 |
| updated_at | 更新日時。 |

---

## 3. 契約・料金コンテキスト（無人システム費・オプション・ロイヤリティ）

### 3-1. fee_items（課金アイテムマスタ）

**目的**

- 無人システム・電話代行・HP掲載料・USEN系サービスなどの **最小課金要素** を定義する。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| item_code | 課金項目コード（例：`hp_listing`, `system_mgmt`, `camera_safie`, `bgm_usen`, `net_usen`, `phone_emergency`, `lock_maintenance`）。 |
| item_name | 課金項目名。請求書表示名のベース。 |
| default_unit_price | 標準単価（円）。店舗別に上書き可能。 |
| notes | 備考。 |
| updated_at | 更新日時。 |

---

### 3-2. fee_packages / package_components（パッケージ定義）

**目的**

- 「無人システム管理費（第1〜3弾）」「U-パッケージ」「無人システム一式」等の **定額パッケージ** と、その内訳を定義する。

**fee_packages**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| package_code | パッケージコード（例：`sys_mgmt_v1`, `sys_mgmt_v2`, `sys_mgmt_v3`, `usen_pkg`, `full_system`）。 |
| package_name | パッケージ名（請求書表示用）。 |
| monthly_price | 月額定額料金（円）。 |
| notes | パッケージの説明・備考。 |
| updated_at | 更新日時。 |

**package_components**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| package_code | パッケージコード（fee_packages.package_code）。 |
| item_code | 内訳アイテムコード（fee_items.item_code）。 |
| qty | 数量（通常は1）。 |
| notes | 備考。 |

---

### 3-3. shop_fee_package_assignments（店舗ごとのパッケージ割当）

**目的**

- 「この店舗は○月から無人システム管理費（第2弾）を月3万円で契約」
    
    といった **店舗×期間×パッケージ** の情報を持つ。
    

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| shop_id | 店舗ID。 |
| package_code | パッケージコード。 |
| monthly_price_override | 店舗別の上書き価格（NULLならfee_packages.monthly_price）。 |
| visible_on_invoice | 請求書に1行として表示するか。 |
| start_date / end_date | 適用期間（終了日NULL=継続中）。 |
| notes | 備考（交渉内容メモ等）。 |
| created_at / updated_at | 登録・更新日時。 |

---

### 3-4. shop_fee_addons（店舗ごとのアドオン課金）

**目的**

- HP掲載料、電話代行費など、「パッケージに含めない単独課金」を店舗×期間で管理する。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| shop_id | 店舗ID。 |
| item_code | 課金項目コード（fee_items.item_code）。 |
| unit_price | 単価（円）。 |
| qty | 数量。 |
| billing_cycle | 請求周期（`monthly`/`one_time`）。 |
| visible_on_invoice | 請求書に個別行として表示するか。 |
| start_date / end_date | 適用期間。 |
| notes | 備考。 |
| created_at / updated_at | 登録・更新日時。 |

---

### 3-5. shop_royalty_history（ロイヤリティ履歴）

**目的**

- ロイヤリティ率・定額の変更履歴を **期間単位** で管理する。
    
    これにより、過去の請求再現や期間限定減額の記録が可能になる。
    

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| shop_id | 店舗ID。 |
| start_date / end_date | 適用期間（終了日NULL=継続中）。 |
| royalty_type | 方式（`flat`/`percent`/`mixed`）。 |
| royalty_flat | 定額額（円）。 |
| royalty_percent | 率（％）。 |
| notes | 備考（例：○月〜○月までキャンペーンで半額 等）。 |
| created_at / updated_at | 登録・更新日時。 |

**運用上の前提**

- shops.royalty_* は「現在値」を表す。
- shop_royalty_history は「履歴全体」を表し、請求計算時には**当該月が属するレコード**を参照する。

---

## 4. 備品コンテキスト

### 4-1. supply_items（備品マスタ）

**目的**

- 本部から販売する備品（ジェルなど）のカタログ情報を保持する。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| name | 備品名。 |
| unit_price | 販売単価（円）。 |
| cost_price | 仕入原価（円、任意）。 |
| stock_unit | 単位（箱, 本, 枚など）。 |
| category | カテゴリ（消耗品, 備品 など）。 |
| notes | 備考。 |
| updated_at | 更新日時。 |

---

### 4-2. shop_supplies（店舗別備品発注）

**目的**

- 店舗からの備品発注履歴を管理し、請求に連動させる。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| shop_id | 店舗ID。 |
| supply_id | 備品ID（supply_items.id）。 |
| quantity | 数量。 |
| unit_price | 単価（円）。 |
| total_price | 合計金額（円）。 |
| status | 状態（`requested`/`approved`/`shipped`/`completed`/`canceled`）。 |
| requested_at / approved_at / shipped_at / completed_at | 各ステータスに対応する日時。 |
| notes | 備考。 |
| created_at / updated_at | 登録・更新日時。 |

---

## 5. 売上コンテキスト（STORES予約との連携）

### 5-1. sales_records（売上統合テーブル）

**目的**

- STORES予約から月1回などでDLされる複数種のCSV
    
    （月謝・回数券、都度カード、現地決済）を **1フォーマットに正規化** して蓄積する。
    
- 店舗別売上・お試し体験数・決済構成比などの分析に使う。

**新規／累積の扱い**

- sales_recordsは **累積テーブル** とし、過去分も含めて継続保存する前提。
- サーバ負荷や容量が問題化した場合は、将来的に
    - アーカイブ用テーブル
    - 月次サマリテーブル
        
        をPhase1以降で追加検討する。
        

**CSV仕様変更への対応**

- STORES側のCSV項目変更は過去にも発生しているため、
    
    **取り込みロジック側での吸収** を前提とし、sales_recordsのスキーマはなるべく安定させる。
    
- どうしても吸収が難しい場合は、Phase1以降で raw テーブル追加を検討する。

**主なカラム**

| カラム名 | 説明 |
| --- | --- |
| id | DB主キー。 |
| merchant_public_id | STORES予約の `merchant_public_id`。shops.merchant_public_id とJOINし店舗を特定する。 |
| shop_id | 店舗ID（shops.id）。 |
| source_type | データ種別：`monthly_ticket`（月謝・回数券）、`spot_card`（都度カード）、`spot_local`（現地決済）。 |
| transaction_type | 取引種別。STORESの「取引種別」列などを格納。 |
| transaction_datetime | 取引日時。カード売上は取引日時、現地決済は予約日時など、集計の基準となる日時。 |
| amount | 販売金額（基本は税込）。 |
| fee | 手数料。カード決済の場合のみ。 |
| revenue | 手取り売上。`amount - fee` または現地決済の場合は `amount`。 |
| product_name | 商品名／メニュー名。複数CSVの「商品名」「タイトル」「メニュー」を統合して格納。 |
| customer_name | 顧客氏名。 |
| customer_email | 顧客メールアドレス。 |
| customer_phone | 顧客電話番号。 |
| is_trial | お試し体験フラグ。商品名／メニューに「お試し体験」を含む場合1、それ以外0。 |
| payment_method | 支払方法。`card` / `local` 等。現地決済CSVの支払方法列もここへ。 |
| status | 行ステータス。現地決済では「確定」など。確定行のみ取込む前提。 |
| created_at | CSV取込日時。 |
| updated_at | 更新日時。 |

---

## 6. まとめ（Phase0コアとしての扱い）

- 本ドキュメントで定義したテーブル群は、**Phase0 基本DBコア**として扱う。
- 以降のフェーズでは：
    - AIログ、通知ログ、チュートリアル、ポータルUI用の補助テーブルなどを **追加で生やす**。
    - ここで定義したカラムの意味を変える場合は、必ずマイグレーション設計とセットで検討する。
- 契約や料金構造の変化（新しいプラン・新しいオプション・セット化ルールの変更）は、
    - `fee_items` / `fee_packages` / `package_components` のマスタ追加・変更
    - `shop_fee_package_assignments` / `shop_fee_addons` / `shop_royalty_history` のレコード追加
        
        で表現し、**既存店舗の契約条件を壊さない** 方針とする。
