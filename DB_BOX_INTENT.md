# DB_BOX_INTENT.md  
Phase0 基本DB設計「箱 → 用途」マスタ

## 1. このドキュメントの目的

本ドキュメントは、Phase0 で設計した **基本DBテーブル群の「箱」としての役割・用途** を  
後続フェーズ（SYS_CORE / API_COMM / AI_CORE / HELP_UI / LINE_UI / SHOP_PORTAL / AI-INBOX / 請求まわり）から一貫して参照するためのマスタです。

- 「どのテーブルに、どんな種類の情報を入れるのか」
- 「同じ“箱”を、今後どんな質問・機能で再利用していくのか」

を、**テーブル単位（BOX）で整理**します。

> NOTE:  
> - ここでいう「BOX」は、テーブル自体、または `link_type` / `info_type` / `option_code` などの“種類ラベル”を持つテーブルのことを指します。  
> - 実際のカラム一覧や DDL は `PHASE_00_DB_CORE.md` / `db_fields.yaml` を正とします。本ファイルは「意図の説明」「用途マップ」です。

---

## 2. オーナー・通知まわり BOX

### 2.1 owners — オーナーマスタ（契約・請求・通知ベース）

**BOX名**

- `owners`

**粒度**

- 1レコード = 1オーナー（直営HQ / FCオーナー）

**主な用途**

- 契約者情報・請求宛先の管理
- オーナー単位の ChatWork ルームID / 通知手段の管理
- 直営／FC の区別（owner_type）

**代表的フィールド**

- `owner_code` … 加盟店コード（外部／業務で使う主識別子）
- `owner_type` … HQ or franchisee
- `chatwork_room_id` … オーナー単位の通知ルーム
- `preferred_contact` … email/chatwork/both
- `billing_email` … 請求書送付先

**今後の利用例**

- SYS_CORE:
  - 通知先の決定（オーナー向けお知らせ、請求関連通知）
- AI-INBOX / SHOP_PORTAL:
  - 「このオーナー（複数店舗）の状況」をまとめて見る際のキー
- HELP_UI / LINE_UI:
  - エスカレーション後、「どのオーナーのChatWorkルームに飛ばすか」の解決（shop → owner → chatwork_room_id）

---

### 2.2 owner_contacts — オーナー担当者（メンション先）

**BOX名**

- `owner_contacts`

**粒度**

- 1レコード = 1担当者（max 5名程度）

**主な用途**

- ChatWorkでメンションすべき具体的な担当者（manager / accounting / operations）
- 通知の宛先グルーピング（経理系通知 / 運営系通知）

**代表的フィールド**

- `owner_id` … owners.id
- `role` … manager / accounting / operations / other
- `chatwork_id` … ChatWorkユーザーID
- `is_active` … 在任フラグ

**今後の利用例**

- SYS_CORE / SysEscalation:
  - `role=operations` を優先して加盟店向けエスカレーションを飛ばす
- AI-INBOX:
  - 「このチケットは operations 担当にメンション」のようなルールに利用

---

## 3. 店舗マスタ・売上 BOX

### 3.1 shops — 店舗マスタ

**BOX名**

- `shops`

**粒度**

- 1レコード = 1店舗（直営・FC 区別なし）

**主な用途**

- 店舗の表示名・住所・電話・営業時間などの基本情報
- STORES予約IDとの紐づけ（merchant_public_id）
- RemoteLOCK 子アカウントメール
- ロイヤリティ現在値（type / flat / percent）

**代表的フィールド**

- `code` … 不変の店舗コード（URL・メール・外部連携キー）
- `merchant_public_id` … STORES予約ID
- `shop_email` … 店舗代表メール
- `business_hours` … 営業時間
- `remotelock_child_email` … オーナー用RemoteLOCKログイン
- `royalty_type / royalty_flat / royalty_percent`

**今後の利用例**

- HELP_UI / LINE_UI:
  - `shop_code` パラメータ → `shops` に解決して「どの店舗のHELPか」を特定
  - アクセス案内やGoogleMapの参照（`shop_links` と組み合わせ）
- API_COMM:
  - STORES API で merchant_public_id を使った予約・顧客取得
- 請求：
  - 売上集計時の店舗キー
  - ロイヤリティ計算

---

### 3.2 sales_records — STORES 売上統合 BOX

**BOX名**

- `sales_records`

**粒度**

- 1レコード = 1取引（カード決済／現地決済 含む）

**主な用途**

- STORESの複数CSV（カード売上2種＋現地決済）を統合した売上ログ
- 「店舗別売上」「お試し体験数」「決済構成比」などのKPI計算

**代表的フィールド**

- `merchant_public_id` / `shop_id`
- `source_type` … monthly_ticket / spot_card / spot_local
- `transaction_datetime`
- `amount` / `fee` / `revenue`
- `product_name`
- `customer_name` / `customer_email` / `customer_phone`
- `is_trial`
- `payment_method`

**今後の利用例**

- BI / ダッシュボード：
  - 店舗別売上／お試し体験率／決済構成比
- 将来のAI回答：
  - 「この店舗の先月売上傾向」「お試し体験が多い店舗は？」など HQ向け問い合わせ
- ロイヤリティ計算：
  - percent / mixed の店舗に対する売上連動ロイヤリティ算出

---

## 4. 店舗機密・API鍵 BOX

### 4.1 shop_secrets — 店舗ごとの機密 KV

**BOX名**

- `shop_secrets`

**粒度**

- 1レコード = 1店舗×1キー（暗号化値）

**主な用途**

- 各種ログインID・パスワード・APIキー・トークンの安全保管
- RemoteLOCK 親アカウント・APIキー
- STORES, Google, Instagram, LINEなどのシステム連携秘密情報

**代表的フィールド**

- `k` … キー名  
  例: `REMOTELOCK_PARENT_EMAIL`, `REMOTELOCK_PARENT_PASSWORD`, `REMOTELOCK_API_KEY`, `GOOGLE_PASSWORD`
- `v_enc` … 暗号化値

**今後の利用例**

- SYS_CORE / API_COMM:
  - 外部APIクライアントがここからキー・シークレットを取得
- AI_CORE:
  - 直接参照しない（あくまでAPI層で利用）
- セキュリティ：
  - 親アカウント情報はオーナーには開示せず、本部のみ閲覧・変更可

---

## 5. 店舗オプション・運用フラグ BOX

### 5.1 options_master / shop_options — 店舗オプション（ON/OFF）

**BOX名**

- `options_master` + `shop_options`

**粒度**

- options_master: 1レコード = 1オプション種別（グローバル定義）
- shop_options: 1レコード = 1店舗×1オプション値（0/1）

**主な用途**

- 現地払い対応・USEN/BGM/カメラ導入・ネット回線などの導入状況
- 予約・忘れ物・清掃などの運用ルール（時間・フラグ）を店舗別に切り替える

**代表的フィールド**

- `option_code` 例:
  - `camera_safie`, `camera_usen`, `bgm_usen`, `net_usen`, `phone_emergency` 等
- `editable_by_store`
  - 0: 本部のみ変更可  
  - 1: 加盟店も変更可能

**今後の利用例**

- HELP_UI:
  - 忘れ物対応用パラメータ（例: `FORGOT_ITEM_FREE_REENTRY_MIN_AFTER_END` 等）を option_code として持たせる
- LINE_UI / SHOP_PORTAL:
  - 店舗別に「現地払い可否」などを切り替える
- ダッシュボード：
  - USEN導入状況・カメラ設置状況の一覧

---

## 6. 課金／ロイヤリティ BOX

### 6.1 fee_items / fee_packages / package_components

**BOX名**

- `fee_items` … 最小課金要素  
- `fee_packages` … パッケージ  
- `package_components` … パッケージ内訳

**主な用途**

- 「無人システム一式」「U-パッケージ」などの定額パッケージの構成管理
- HP掲載、電話代行などの単独項目の管理

**今後の利用例**

- 請求モジュール：
  - 請求書に表示する定額行／明細の生成
- HQ向けダッシュボード：
  - 店舗ごとの契約状態、システム構成可視化  

### 6.2 shop_fee_package_assignments / shop_fee_addons

**BOX名**

- `shop_fee_package_assignments` … 店舗×パッケージ割り当て  
- `shop_fee_addons` … 店舗×アドオン課金

**主な用途**

- 店舗ごとの月額パッケージ・アドオン課金の履歴管理
- HP掲載料、電話代行費など「個別行で請求したい」項目を管理

**今後の利用例**

- HQ向け：
  - どの店舗がどの無人システムパッケージを契約しているか
- 将来のAI（本部サポート）：
  - 「この店舗はU-パッケージに含まれているので、別途ネット費は掛かっていません」などの回答

### 6.3 shop_royalty_history — ロイヤリティ履歴

**主な用途**

- ロイヤリティの期間限定減額・特例の履歴管理
- 過去請求の再現

**今後の利用例**

- HQ向けAI：
  - 「この店舗の現在のロイヤリティ条件は？」  
  - 「いつから減額期間だったか？」

---

## 7. 備品・発注 BOX

### 7.1 supply_items / shop_supplies

**BOX名**

- `supply_items` … 備品カタログ  
- `shop_supplies` … 店舗別発注履歴

**主な用途**

- 備品（ジェル／紙類／消耗品）発注の履歴管理
- 備品費用の請求連動

**今後の利用例**

- SHOP_PORTAL：
  - オーナーが「過去の発注履歴」「発注ステータス」を確認
- HQ向けAI：
  - 「この店舗は今月どれくらい備品を発注しているか」などの集計

---

## 8. 店舗リンク・外部URL BOX

### 8.1 shop_links — 外部リンク集

**BOX名**

- `shop_links`

**粒度**

- 1レコード = 1店舗×1リンク種別

**主な用途**

- Googleマップ／Google口コミ／Instagram／LP／ブログ／アンケート／チケットURL 等  
  店舗別の外部サイトを構造的に管理

**代表的 link_type（Phase0 で想定していたラベルの整理）**

| link_type          | 想定用途例                                 |
|--------------------|--------------------------------------------|
| google_map         | GoogleマップURL                            |
| google_review      | Googleクチコミページ                       |
| instagram          | Instagramアカウント                        |
| lp                 | ランディングページ                         |
| blog               | 店舗ブログ                                 |
| official_site      | 公式サイト（全体 or 店舗別）              |
| survey             | 利用後アンケートURL                        |
| ticket             | チケット・回数券一覧ページ                |
| product            | 商品一覧・メニュー一覧                     |
| line_add           | 公式LINE追加URL                            |
| other              | 上記に当てはまらない各種URL                |

**今後の AI/HELP_UI 利用例**

- `info.access` / `場所はどこ？`  
  → `shop_links.link_type='google_map'`
- `口コミ見せて`  
  → `google_review`
- `アンケートURL教えて`  
  → `survey`
- `忘れ物チケット買いたい`（将来追加）  
  → `link_type='ticket'` + `label='忘れ物チケット'` or `ticket_forget_item` を導入
- `LINE追加したい`  
  → `line_add`

---

## 9. 店舗内部情報（社内用） BOX

### 9.1 shop_internal_info — 本部専用の現場情報

**BOX名**

- `shop_internal_info`

**粒度**

- 1レコード = 1店舗×1情報種別

**主な用途**

- 鍵情報・セコム情報・施工業者・緊急連絡先・現場注意事項など  
  エンドユーザーには絶対公開しない内部情報

**代表的 info_type（Phase0意図の整理）**

| info_type          | 想定用途例                             |
|--------------------|----------------------------------------|
| key                | 鍵の場所／番号／注意事項               |
| secom              | セコム契約情報／担当者メモ             |
| vendor             | 施工業者・各種ベンダー連絡先          |
| emergency_contact  | 緊急連絡先（本社／監査用など）         |
| notice             | 現場注意事項（階段注意／近隣クレーム） |
| other              | その他社内専用情報                     |

**今後の利用例**

- SHOP_PORTAL / AI-INBOX（本部内部向け）：
  - 緊急対応フロー、清掃トラブル対応時に「注意事項」「緊急連絡先」を参照
- HELP_UI / LINE_UI：
  - エンドユーザーには直接出さないが、  
    エスカレーション先や内部対応のコンテキストとして参照される

---

## 10. BOX → 用途 早見表（Phase0 意図のサマリ）

| カテゴリ                     | BOX（テーブル）                   | 粒度                     | 主な用途のまとめ                                                 |
|------------------------------|------------------------------------|--------------------------|------------------------------------------------------------------|
| オーナー／通知               | owners                             | オーナー単位             | 契約者情報・請求メール・ChatWorkルーム                          |
|                              | owner_contacts                     | 担当者単位               | メンション先・担当区分                                          |
| 店舗マスタ                   | shops                              | 店舗単位                 | 店舗表示情報・STORES ID・RemoteLOCK子メール・ロイヤリティ現在値 |
| 店舗機密                     | shop_secrets                       | 店舗×キー                | APIキー・ログイン・RemoteLOCK親アカ等                           |
| 店舗オプション               | options_master + shop_options      | オプション種別/店舗×種別 | システム導入フラグ・各種運用ルール（忘れ物・支払方式 等）      |
| 課金・パッケージ             | fee_items / fee_packages / package_components | アイテム・パッケージ | 月額パッケージ定義・構成管理                                    |
| 課金割り当て／アドオン       | shop_fee_package_assignments / shop_fee_addons | 店舗×パッケージ／店舗×アイテム | 店舗別の月額課金設定                                            |
| ロイヤリティ                 | shop_royalty_history               | 店舗×期間                | ロイヤリティ特例・期間限定減額の履歴                            |
| 備品・発注                   | supply_items / shop_supplies       | 備品マスタ／店舗×発注     | 備品カタログ・発注履歴                                         |
| 外部リンク（公開情報）       | shop_links                         | 店舗×リンク種別          | GoogleMap・アンケート・チケット・SNSなどのURL                   |
| 店舗内部情報（非公開／社内） | shop_internal_info                 | 店舗×情報種別            | 鍵情報／緊急連絡先／現場注意事項                               |
| 売上ログ                     | sales_records                      | 取引単位                 | STORES売上統合・KPI・ロイヤリティ計算                           |

---

このファイルを追加しておくことで、

- 後続フェーズ（特に HELP_UI / LINE_UI / AI-INBOX）で新しいユースケースが出てきたときに  
  「どの BOX に、どのラベルで入れるか」を **このマスタを基準に逆引き** できます。
- あなたが Phase0 の記憶を毎回頭の中から掘り起こさなくても、  
  「BOX→用途」がここを見れば分かる状態になります。
